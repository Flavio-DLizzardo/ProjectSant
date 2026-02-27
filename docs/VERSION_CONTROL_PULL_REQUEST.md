# Item 4 – Version Control and Pull Request

Repositório: **https://github.com/Flavio-DLizzardo/ProjectSant**

---

## Opção A: Projeto já está na pasta local (você ainda não clonou do GitHub)

No terminal, na pasta do projeto (`ProjectSant`):

```bash
cd /Users/flizzardo/ProjectSant

# Inicializar repositório Git (se ainda não foi feito)
git init

# Adicionar o remote do GitHub
git remote add origin https://github.com/Flavio-DLizzardo/ProjectSant.git

# Criar e mudar para a branch feature/1.0.0
git checkout -b feature/1.0.0

# Adicionar todos os arquivos (documentação e código)
git add .

# Ver o que será commitado
git status

# Commit
git commit -m "docs: documentação de arquitetura, Ansible Day 2 e diagramas (feature 1.0.0)"

# Enviar a branch feature/1.0.0 para o GitHub
git push -u origin feature/1.0.0
```

Se o GitHub pedir autenticação, use um **Personal Access Token (PAT)** ou configure **SSH** em vez de senha.

---

## Opção B: Repositório já clonado do GitHub

Se você já clonou `https://github.com/Flavio-DLizzardo/ProjectSant` em outra pasta, copie para lá os arquivos deste projeto (docs, ansible, README.md, .gitignore) e depois:

```bash
cd /caminho/onde/esta/o/ProjectSant   # pasta do clone

git checkout -b feature/1.0.0
git add .
git status
git commit -m "docs: documentação de arquitetura, Ansible Day 2 e diagramas (feature 1.0.0)"
git push -u origin feature/1.0.0
```

---

### 3. Abrir Pull Request para `master`

1. Acesse: **https://github.com/Flavio-DLizzardo/ProjectSant**
2. Se o push foi feito, o GitHub costuma exibir um banner: **"Compare & pull request"** para a branch `feature/1.0.0`. Clique nele.
3. Se não aparecer:
   - Clique em **Branches** → selecione `feature/1.0.0` → **New pull request**.
4. Defina:
   - **Base:** `master`
   - **Compare:** `feature/1.0.0`
5. Preencha título e descrição (ex.: "Documentação e código – arquitetura ConsultClients, Ansible, diagramas").
6. Clique em **Create pull request**.

---

### 4. Concluir até o merge

1. Revise o PR (e faça os ajustes que quiser em `feature/1.0.0` e dê push de novo).
2. Se houver revisão obrigatória, aguarde aprovação.
3. Clique em **Merge pull request**.
4. Confirme o merge (e, se quiser, exclua a branch `feature/1.0.0` após o merge).

Após o merge, o código e a documentação estarão na branch `master`.
