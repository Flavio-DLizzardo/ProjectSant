 # Arquitetura On-Premises – Aplicação de Pedidos Online

## Visão Geral

Esta documentação descreve uma arquitetura on‑premises para uma aplicação fictícia de **Pedidos Online** (API de pedidos, produtos e clientes), utilizada por colaboradores da empresa para **cadastro de clientes**, **pesquisa de informações** e **apoio ao processo de venda de produtos**, acessível tanto de redes internas quanto pela Internet, sempre com autenticação.

 - Nginx como **API Gateway / Reverse Proxy**
 - Redis como **cache de leitura**
 - Kafka como **sistema de mensageria**
 - PostgreSQL como **banco de dados relacional**

 O foco é garantir **alta disponibilidade** e **segurança** em todas as camadas.

 ---

 ## Diagrama de Arquitetura (Mermaid)

 ```mermaid
 flowchart LR
     subgraph Internet
         C[Clientes Web / Mobile]
     end

    subgraph DMZ[DMZ - Perímetro]
         N[Nginx\n(API Gateway)]
     end

    subgraph APP[Camada de Aplicação]
         A1[API ConsultClient - Instância 1]
         A2[API ConsultClient - Instância 2]
     end

     subgraph DATA[Camada de Dados]
         subgraph Cache
             R1[Redis 1\n(nó primário)]
             R2[Redis 2\n(réplica)]
         end

         subgraph Messaging
             K1[Kafka Broker 1]
             K2[Kafka Broker 2]
             K3[Kafka Broker 3]
             ZK[Zookeeper (ou KRaft)]
         end

         subgraph DB[PostgreSQL Cluster]
             P1[PostgreSQL Primary]
             P2[PostgreSQL Replica 1]
             P3[PostgreSQL Replica 2]
         end
     end

     subgraph MGMT[Gestão e Monitoramento]
         MON[Monitoring\n(Prometheus/Grafana)]
         BKP[Servidor de Backup\n(Storage)]
         ANS[Ansible Control Node]
     end

     C -->|HTTPS| N
     N -->|Balanceamento L7| A1
     N -->|Balanceamento L7| A2

     A1 -->|Leitura rápida| R1
     A2 -->|Leitura rápida| R1
     R1 --> R2

     A1 -->|Eventos de negócio| K1
     A2 -->|Eventos de negócio| K2
     K1 <-->|Replicação| K2
     K2 <-->|Replicação| K3
     ZK --- K1
     ZK --- K2
     ZK --- K3

     A1 -->|Leitura/Escrita| P1
     A2 -->|Leitura/Escrita| P1
     P1 -->|Streaming/Replicação| P2
     P1 -->|Streaming/Replicação| P3

     P1 -->|Backup lógico/físico| BKP
     P2 --> BKP

     MON --- N
     MON --- A1
     MON --- A2
     MON --- R1
     MON --- K1
     MON --- P1

     ANS -. SSH/Gerência .- N
     ANS -. SSH/Gerência .- A1
     ANS -. SSH/Gerência .- A2
     ANS -. SSH/Gerência .- R1
     ANS -. SSH/Gerência .- K1
     ANS -. SSH/Gerência .- P1
 ```

 ---

 ## Componentes Principais

 ### Nginx – API Gateway / Reverse Proxy

 - **Função**: ponto único de entrada para as requisições HTTP/HTTPS.
 - **Responsabilidades**:
   - Terminação TLS (HTTPS).
   - Balanceamento de carga entre `A1` e `A2`.
   - Rate limiting básico e proteção contra alguns tipos de ataque (ex.: brute force).
   - Reescrita de URLs e roteamento por path (ex.: `/api/v1/...`).

### Servidores de Aplicação / API ConsultClient (A1, A2)

 - **Função**: hospedar a API da aplicação de pedidos (REST/JSON).
 - **Responsabilidades**:
   - Implementar regras de negócio (criação de pedidos, consulta de catálogo, etc.).
   - Orquestrar chamadas a Redis, Kafka e PostgreSQL.
   - Expor métricas de monitoramento (ex.: `/metrics`).

 ### Redis – Cache de Leitura

 - **Função**: reduzir latência e carga no PostgreSQL para leituras frequentes.
 - **Responsabilidades**:
   - Armazenar em cache dados com alta taxa de leitura (ex.: catálogo de produtos e sessões).
   - Operar em modo **master–replica** (`R1` primário, `R2` réplica).
   - Possível uso de **Redis Sentinel** ou similar para failover automático.

 ### Kafka – Mensageria

 - **Função**: desacoplar produtores e consumidores de eventos de negócio.
 - **Responsabilidades**:
   - Receber eventos como: "pedido criado", "pagamento confirmado", "estoque atualizado".
   - Repassar mensagens para serviços downstream (faturamento, notificação, analytics).
   - Tolerância a falhas via cluster de brokers (`K1`, `K2`, `K3`) e replicação de partições.

 ### PostgreSQL – Persistência de Dados

 - **Função**: banco de dados transacional principal da aplicação.
 - **Responsabilidades**:
   - Garantir integridade referencial de pedidos, clientes, itens e transações.
   - Disponibilizar réplicas para leitura intensiva (`P2`, `P3`).
   - Replicação streaming para alta disponibilidade e menor RPO.

 ### Monitoramento e Gestão

 - **Monitoring**:
   - Prometheus/Grafana coletando métricas de Nginx, aplicação, Redis, Kafka e PostgreSQL.
   - Alertas configurados para disponibilidade, uso de recursos e erros de aplicação.
 - **Backups**:
   - Servidor ou storage dedicado para armazenar backups de banco (e eventualmente configs).
 - **Ansible Control Node**:
   - Responsável por aplicar playbooks de Day 2 (backup, restore, start/stop, etc.).

 ---

 ## Estratégias de Alta Disponibilidade

 ### Camada de Entrada (Nginx)

 - Múltiplas instâncias de Nginx podem ser configuradas atrás de um **load balancer L4**.
 - Configurações armazenadas em repositório (Git) e aplicadas via Ansible para consistência.
 - Uso de **health checks** para remover nós indisponíveis do pool.

 ### Camada de Aplicação

 - Pelo menos **dois servidores de aplicação** (`A1`, `A2`) atrás do Nginx.
 - As sessões de usuário devem ser **stateless** ou armazenadas em Redis para permitir failover transparente.

 ### Redis

 - Topologia **master–replica** com:
   - Escritas indo para o nó primário.
   - Leituras podendo ser delegadas à réplica.
 - Uso de **Sentinel** ou mecanismo equivalente para:
   - Detectar falha no primário.
   - Promover automaticamente uma réplica a primário.

 ### Kafka

 - Mínimo de **3 brokers** (`K1`, `K2`, `K3`) com:
   - Replicação de partições (`replication.factor >= 3` quando possível).
   - `min.insync.replicas` configurado para garantir durabilidade antes de reconhecer escrita.
 - Zookeeper (ou KRaft, nas versões mais novas) garantindo coordenação do cluster.

 ### PostgreSQL

 - Cluster com **1 primário** (`P1`) e **2 réplicas** (`P2`, `P3`):
   - Replicação streaming assíncrona ou síncrona (conforme requisitos de RPO/RTO).
 - Mecanismo de failover (ex.: Patroni, pg_auto_failover ou scripts gerenciados) para promover réplicas.
 - Aplicação configurada para:
   - Usar o primário para operações de escrita.
   - Utilizar réplicas para queries de leitura pesadas (relatórios, dashboards).

 ---

 ## Estratégias de Segurança

 ### Rede e Segmentação

 - Separação clara por zonas:
   - **DMZ**: Nginx exposto à Internet.
   - **Camada de Aplicação**: somente acessível a partir da DMZ e rede interna.
   - **Camada de Dados**: somente acessível a partir da camada de aplicação e gestão.
- Firewalls e regras de ACLs limitando portas e origens permitidas.
- Firewalls dedicados para acesso à API **ConsultClient**:
  - **Clientes externos**:
    - Firewall de borda permitindo apenas tráfego HTTPS (porta 443) de origens externas confiáveis para o Nginx na DMZ.
    - Bloqueio direto de tráfego externo para a camada de aplicação e dados (somente Nginx é exposto).
  - **Clientes internos (rede corporativa)**:
    - Regras específicas permitindo acesso HTTPS dos segmentos internos autorizados (ex.: redes de atendimento, backoffice) ao Nginx.
    - Opcionalmente, firewall interno adicional entre rede corporativa e camada de aplicação, restringindo origem e portas para as instâncias da API `ConsultClient`.

 ### Tráfego Criptografado

 - **HTTPS** entre clientes e Nginx, com TLS moderno.
 - Comunicação interna usando TLS sempre que possível (Kafka, PostgreSQL, Redis, SSH).
 - Gestão de certificados via AC interna ou serviços como HashiCorp Vault.

### Autenticação e Autorização

- Autenticação realizada diretamente pela **aplicação/API ConsultClient** contra uma **tabela de usuários no PostgreSQL** (sem uso de Active Directory ou IdP externo).
- Todos os **usuários da API (internos e externos)** são **funcionários da empresa**, cadastrados na tabela de usuários:
  - Campos típicos: `username`, `password_hash`, `role`, `status`.
  - As senhas são armazenadas apenas como **hash seguro** (ex.: bcrypt/Argon2), nunca em texto puro.
- Fluxo de login:
  - O funcionário acessa a URL da aplicação/API (`https://apps.empresa.com/consultclient`) via HTTPS.
  - Nginx encaminha a requisição para o endpoint de autenticação da API (ex.: `POST /auth/login`).
  - A API valida `username` e `password` consultando o PostgreSQL e, em caso de sucesso, emite um **token de sessão/JWT** que será usado nas demais chamadas.
  - As próximas requisições à API ConsultClient são validadas pela própria aplicação, que verifica o token/sessão antes de processar a operação.
- PostgreSQL:
  - Usuários separados para aplicação, manutenção e relatórios.
  - Políticas de privilégios mínimos por schema/tabela.
- Kafka:
  - Autenticação SASL (quando habilitado).
  - ACLs por tópico (produtores/consumidores autorizados).

 ### Proteção de Dados e Backups

 - Backups de PostgreSQL armazenados com:
   - Criptografia em repouso.
   - Rotação de chaves e controle de acesso.
 - Logs de auditoria habilitados para:
   - Acessos administrativos ao banco.
   - Execução de playbooks sensíveis (via Ansible).

 ---

 ## Operações de Day 2 com Ansible

 As operações de **Day 2** cobrem rotinas de:

 - **Backup** e **restore** do PostgreSQL.
 - Verificação de **status** do serviço.
 - **Start/stop/restart** do banco.
 - Testes simples de conectividade.

 O playbook de exemplo deste projeto foca no PostgreSQL, mas a mesma abordagem pode ser adaptada para Redis ou Kafka.

 ---

 ## Como Este Projeto Atende ao Desafio

 - **Diagrama de arquitetura**: fornecido em formato Mermaid dentro deste arquivo.
 - **Documentação de arquitetura**: esta página explica componentes, alta disponibilidade e segurança.
 - **Playbook Ansible de Day 2**: disponível em `ansible/postgresql_day2.yml` com operações rotineiras.
 - **Boas práticas**:
   - Documentação em Markdown clara e estruturada.
   - Comentários explicativos no playbook apenas onde agregam entendimento de operação.

