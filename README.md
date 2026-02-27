# ProjectSant – ConsultClients (On-Premises)

Aplicação on-premises **ConsultClients** para consulta de clientes e apoio à venda de produtos.  
Arquitetura: Nginx (API Gateway), Redis (cache), Kafka (mensageria), PostgreSQL (persistência + autenticação).

## Conteúdo do repositório

- **`docs/`** – Documentação de arquitetura (`architecture.md`), diagramas e guia de Version Control / Pull Request.
- **`ansible/`** – Playbook de Day 2 para PostgreSQL (backup, restore, status, start/stop).

## Branch e Pull Request (item 4)

- Código e documentação devem ser commitados na branch **`feature/1.0.0`**.
- Após validação, abrir **Pull Request** para a branch **`master`** e seguir até o merge.

Instruções detalhadas: **[docs/VERSION_CONTROL_PULL_REQUEST.md](docs/VERSION_CONTROL_PULL_REQUEST.md)**.

## Repositório

**https://github.com/Flavio-DLizzardo/ProjectSant**
