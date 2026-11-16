# Changelog

Todas as mudanças notáveis deste plugin serão documentadas neste arquivo.

O formato é baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/) e este projeto adota [Semantic Versioning](https://semver.org/).

## [1.0.1] — 2026-11-16

### Corrigido
- `@trepr/mcp-jira`: adicionado o shebang `#!/usr/bin/env node` em `bin/run.mjs`. Sem ele, o `npx` executava o script com `sh` e o MCP server encerrava imediatamente ("Connection closed").

## [1.0.0] — 2026-11-16

### Adicionado
- Conversão do Kiro Power `jira-trepr` para o padrão aberto **[Agent Plugins](https://agent-plugins.org/)** (v1.0.0), com `plugin.json` e `mcp.json` (`"type": "stdio"`).
- Pacote wrapper `@trepr/mcp-jira` (`packages/mcp-jira`), que resolve credenciais (env vars ou credential store do SO) e inicia o `mcp-atlassian`, substituindo o `credential-helper.mjs` referenciado por caminho absoluto.
- Skill `jira-operator` adaptada para MCP direto, sem comandos `kiro_powers`.

### Removido
- Hook `enforce-commit-jira-link` e steering `commit-jira-traceability`. A regra foi mantida como `skills/jira-operator/references/commit-traceability.md`.
- Instruções de onboarding e cópia de artefatos do `POWER.md`.
