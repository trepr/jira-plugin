# @trepr/mcp-jira
Wrapper do MCP server `mcp-atlassian` para o Jira Server do TRE-PR (`jira.tre-pr.jus.br`). Resolve `JIRA_USERNAME` e `JIRA_API_TOKEN` na ordem: variáveis de ambiente (`JIRA_PASSWORD` como alias), credential store nativo (Windows Credential Manager, libsecret, Keychain) e, na ausência de ambos, encerra com instruções. Em seguida inicia `uvx mcp-atlassian`.
## Execução
Com o Nexus configurado no `~/.npmrc` (`@trepr:registry=https://nexus.tre-pr.jus.br/repository/npm-group/`):
```bash
npx -y @trepr/mcp-jira
```
Requer Node.js 22+ e `uvx` (Python 3.10+).
## Publicação no Nexus do TRE-PR
```bash
npm login --registry=https://nexus.tre-pr.jus.br/repository/npm-hosted/
npm publish --registry=https://nexus.tre-pr.jus.br/repository/npm-hosted/
```
## Variáveis de Ambiente
| Variável | Padrão | Descrição |
| :--- | :--- | :--- |
| `JIRA_URL` | `https://jira.tre-pr.jus.br` | URL base do Jira. |
| `JIRA_USERNAME` | *(credential store)* | Usuário para Basic Auth. |
| `JIRA_API_TOKEN` | *(credential store)* | Senha ou API token. |
| `NODE_OPTIONS` | `--use-system-ca` | Carrega a CA interna do sistema operacional. |
