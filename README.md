# Jira Plugin (TRE-PR)
Plugin no padrão aberto **[Agent Plugins](https://agent-plugins.org/)** para integração com o Jira Server interno do TRE-PR (`jira.tre-pr.jus.br`, versão 7.3.3). Permite que agentes de IA interajam com issues, projetos, boards Scrum/Kanban, sprints e worklogs através do Model Context Protocol (MCP), usando a REST API v2 (`/rest/api/2/`) e a Agile API (`/rest/agile/1.0/`).
O MCP server é o `mcp-atlassian` (Python, executado via `uvx`), iniciado pelo pacote wrapper `@trepr/mcp-jira`, disponibilizado no **Nexus do TRE-PR (`nexus.tre-pr.jus.br`)**. O wrapper resolve as credenciais de Basic Auth (variáveis de ambiente ou credential store nativo do sistema operacional) antes de iniciar o `mcp-atlassian`.
---
## Estrutura do Plugin
```text
jira-plugin/
├── .env.example                    # Modelo para configuração das credenciais por variáveis de ambiente
├── plugin.json                     # Manifesto do plugin (Agent Plugins v1.0.0)
├── mcp.json                        # Configuração do MCP server (@trepr/mcp-jira)
├── packages/
│   └── mcp-jira/                   # Pacote npm wrapper (@trepr/mcp-jira) publicado no Nexus
│       ├── bin/run.mjs             # Resolve credenciais e inicia o mcp-atlassian
│       ├── bin/read-credential.ps1 # Leitura do Windows Credential Manager (CredRead)
│       ├── bin/store-credential.ps1# Gravação no Windows Credential Manager (CredWrite)
│       ├── package.json            # Metadados do pacote npm
│       └── README.md               # Instruções de publicação no Nexus
├── skills/
│   └── jira-operator/
│       ├── SKILL.md                # Ponto focal com convenções e workflows
│       └── references/
│           └── commit-traceability.md  # Regra de rastreabilidade commit-issue
├── icon.svg                        # Ícone do plugin
├── tre-root-v3.crt                 # Certificado raiz da AC interna do TRE-PR
├── README.md                       # Documentação do plugin
└── CHANGELOG.md                    # Histórico de versões
```
> [!IMPORTANT]
> **Não coloque o arquivo `.env` na raiz deste plugin!** Gerenciadores de plugins substituem os arquivos do plugin durante as atualizações. Prefira o credential store do sistema operacional ou variáveis de ambiente.
---
## Pré-requisitos
1. **Node.js** 22+ (necessário para `--use-system-ca`)
2. **npx** disponível no PATH
3. **Python 3.10+** e **uv** (`uvx`) para executar o `mcp-atlassian`
4. **Configuração do Nexus no `~/.npmrc`**:
   ```ini
   @trepr:registry=https://nexus.tre-pr.jus.br/repository/npm-group/
   ```
5. **Certificado da CA interna** do TRE-PR instalado no Certificate Store (geralmente distribuído via GPO)
```bash
node --version    # v22.x.x ou superior
npx --version
uvx --version
npm config get @trepr:registry
```
Se `uvx` não estiver instalado:
```powershell
# Windows (via winget):
winget install astral-sh.uv
# Linux/macOS:
curl -LsSf https://astral.sh/uv/install.sh | sh
```
### SSL e Certificados
```powershell
certutil -store Root "ACRAIZ"                      # Verificar
certutil -addstore Root "tre-root-v3.crt"          # Instalar (arquivo na raiz do plugin)
```
---
## Autenticação

Este plugin utiliza **Basic Auth** (autenticação HTTP básica) com as credenciais do usuário no Jira do TRE-PR. As credenciais são resolvidas automaticamente pelo **credential helper** (`credential-helper.mjs`) na seguinte ordem de prioridade:

1. **Variáveis de ambiente** (`JIRA_USERNAME` + `JIRA_PASSWORD`) — se definidas, são utilizadas diretamente
2. **Credential store nativo do OS** — se as variáveis não estiverem definidas, busca no gerenciador de credenciais do sistema operacional
3. **Erro com instruções** — se nenhuma fonte for encontrada, exibe mensagem com as opções de configuração

### Configuração via Credential Store (recomendado)

Armazene suas credenciais no gerenciador nativo do sistema operacional. Isso evita manter senhas em variáveis de ambiente ou arquivos de configuração.

#### Windows (Credential Manager)

```powershell
# Armazenar credenciais (solicita a senha interativamente):
powershell -ExecutionPolicy Bypass -File packages/mcp-jira/bin/store-credential.ps1
```

O script `store-credential.ps1`:
- Assume o username do usuário logado no Windows (Active Directory)
- Solicita a senha interativamente (input mascarado — não aparece no terminal)
- Armazena no Windows Credential Manager via API nativa (`CredWrite` do `advapi32.dll`)

Se precisar especificar um username diferente:
```powershell
powershell -ExecutionPolicy Bypass -File packages/mcp-jira/bin/store-credential.ps1 -Username outro.usuario
```

Para verificar ou remover:
```powershell
cmdkey /list:jira-trepr      # Verificar
cmdkey /delete:jira-trepr    # Remover
```

O credential helper lê as credenciais via API nativa do Windows (`CredRead` do `advapi32.dll`) — não requer módulos PowerShell adicionais.

#### Linux (libsecret / GNOME Keyring)

```bash
# Pré-requisito:
sudo apt install libsecret-tools   # Debian/Ubuntu

# Armazenar credenciais:
echo -n 'seu.usuario' | secret-tool store --label='Jira TRE-PR' service jira-trepr field username
echo -n 'sua-senha' | secret-tool store --label='Jira TRE-PR' service jira-trepr field password
```

#### macOS (Keychain)

```bash
security add-generic-password -s jira-trepr -a seu.usuario -w sua-senha
```

### Configuração via Variáveis de Ambiente (alternativa)

Se preferir não utilizar o credential store, defina as variáveis de ambiente no seu perfil de shell:

```powershell
# PowerShell (Windows):
$env:JIRA_USERNAME = "seu.usuario"
$env:JIRA_API_TOKEN = "sua-senha"

# Bash/Zsh (Linux/macOS):
export JIRA_USERNAME="seu.usuario"
export JIRA_API_TOKEN="sua-senha"
```

As variáveis de ambiente têm **prioridade sobre o credential store** — se estiverem definidas, o credential helper as utiliza diretamente sem consultar o OS.

> **Nota:** A variável `JIRA_PASSWORD` também é aceita como alias de `JIRA_API_TOKEN` para retrocompatibilidade.

### Sobre o valor da senha

O valor da senha (`JIRA_PASSWORD` ou o campo password no credential store) pode ser:
- A **senha de rede** do usuário (mesma utilizada para login no Jira via browser)
- Um **API token** gerado nas configurações pessoais do Jira (se a instância suportar)

A utilização de API token é preferível por ser mais segura e permitir revogação independente da senha de rede, porém a instância 7.3.3 pode não oferecer essa funcionalidade.

### Segurança

- Credenciais **nunca são armazenadas** em `mcp.json` ou qualquer arquivo versionado
- O credential helper resolve credenciais em tempo de execução e as injeta apenas no processo do MCP server
- O agente opera com as **mesmas permissões** que o usuário autenticado no Jira
- Se nenhuma credencial for encontrada, o helper exibe mensagem de erro e encerra sem iniciar o server

---
## Workflows Comuns

### Buscar issues por JQL

```
Buscar issues abertas atribuídas a mim no projeto SISCONV.
Buscar issues com prioridade Critical no projeto INFRA.
Listar bugs abertos do sprint atual no board SISCONV.
```

Exemplos de queries JQL que o agente pode executar:
- `assignee = currentUser() AND status != Done`
- `project = SISCONV AND issuetype = Bug AND status = "In Progress"`
- `priority = Critical AND resolution = Unresolved`
- `sprint in openSprints() AND project = INFRA`

### Criar issues

```
Criar uma issue do tipo Bug no projeto SISCONV com título "Erro no relatório de consolidação".
Criar uma Task no projeto INFRA para "Atualizar certificado SSL do servidor de homologação".
```

### Transições de status

```
Mover a issue SISCONV-123 para "In Progress".
Finalizar a issue INFRA-45 (transição para Done).
Reabrir a issue SISCONV-200.
```

### Boards e Sprints

```
Listar boards disponíveis no Jira.
Mostrar o sprint atual do board SISCONV.
Listar issues do sprint ativo no board INFRA.
```

### Worklogs (registro de horas)

```
Registrar 2 horas na issue SISCONV-123 com descrição "Implementação do endpoint de autenticação".
Listar worklogs da issue INFRA-45.
Registrar 30 minutos na issue SISCONV-200 com descrição "Code review e ajustes".
```

### Comentários

```
Adicionar comentário na issue SISCONV-123: "Implementação concluída, aguardando review."
Listar comentários da issue INFRA-45.
```

---
## Troubleshooting
### Erro de SSL: "self-signed certificate in certificate chain"
**Causa**: certificado da CA interna ausente no Certificate Store, ou Node.js inferior a 22.
1. Verifique `node --version` (22+).
2. Verifique `certutil -store Root "ACRAIZ"`.
3. Se necessário, instale `tre-root-v3.crt` (veja "SSL e Certificados").
4. Reinicie o cliente (Kiro) e reconecte o MCP server `jira`.
### Erro: "401 Unauthorized"
**Causa**: credenciais inválidas ou ausentes.
1. Verifique o credential store (`cmdkey /list:jira-trepr` no Windows; `secret-tool lookup service jira-trepr field username` no Linux) ou as variáveis `JIRA_USERNAME` e `JIRA_API_TOKEN`.
2. Confirme usuário e senha acessando o Jira pelo browser.
3. Se a senha de rede foi alterada, atualize o credential store.
### Erro: "404 Not Found" ao acessar issue ou projeto
Verifique o formato da issue key (`PROJ-123`), se o projeto existe em `https://jira.tre-pr.jus.br/browse/PROJ` e se o usuário tem permissão de visualização.
### MCP Server não inicia ou aparece "No tools available"
1. Verifique `node --version` (22+) e `uvx --version`.
2. Verifique o acesso ao pacote: `npx -y @trepr/mcp-jira` (deve iniciar ou informar credenciais ausentes).
3. Confirme o `@trepr:registry` no `~/.npmrc`.
4. Consulte os logs do MCP server `jira` no cliente.
---
## Boas Práticas
- Use sempre o formato completo da issue key (`SISCONV-45`) e o project key em maiúsculas.
- Teste JQL complexas no browser antes de solicitar ao agente.
- Crie issues com informações completas (summary, description, issuetype, priority).
- Ao registrar worklogs, inclua uma descrição do trabalho realizado.
---
## Variáveis de Ambiente
### Configuradas no `mcp.json`
| Variável | Valor | Descrição |
|----------|-------|-----------|
| `JIRA_URL` | `https://jira.tre-pr.jus.br` | URL base da instância Jira |
| `JIRA_SSL_VERIFY` | `false` | Desabilita a verificação SSL no pacote Python (fallback para CA interna) |
| `NODE_OPTIONS` | `--use-system-ca` | Confia nos certificados do sistema operacional |
### Resolvidas pelo `@trepr/mcp-jira`
| Variável | Fonte | Descrição |
|----------|-------|-----------|
| `JIRA_USERNAME` | Env var ou credential store | Usuário para Basic Auth |
| `JIRA_API_TOKEN` | Env var ou credential store | Senha ou API token (`JIRA_PASSWORD` é aceito como alias) |
---
## Skill
### `jira-operator`
Orienta o agente no uso das ferramentas do MCP server `jira`: convenções de issue key e project key, JQL, criação, transições, boards, sprints, worklogs, tratamento de erros e segurança. Veja [`skills/jira-operator/SKILL.md`](skills/jira-operator/SKILL.md).
A regra de rastreabilidade commit-issue está em [`skills/jira-operator/references/commit-traceability.md`](skills/jira-operator/references/commit-traceability.md). Diferente do Power original, o plugin não distribui hook nem steering file; para impor a regra, copie o conteúdo para o steering do seu workspace.
## Uso com o GitLab Plugin
O `jira-plugin` (servidor `jira`) e o `gitlab-plugin` (servidor `gitlab`) são complementares e podem coexistir: o GitLab gerencia código-fonte e CI/CD, e o Jira gerencia demandas, sprints e horas.
