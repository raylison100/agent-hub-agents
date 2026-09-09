# agent-hub-agents

Repositorio de dados: perfis de agente, skills, tabela de precos, servidores
MCP e politicas de aprovacao. Sem codigo. O daemon le um clone local deste
repositorio.

Formato em `../docs/07-agentes.md`, extensoes em `../docs/09-extensoes.md`.

## Estrutura

```
profiles/               um Markdown com frontmatter por agente
  claude-arquiteto.md
  deepseek-dev.md
  local-leitor.md
skills/                 uma pasta por skill, formato Agent Skills
  revisar-mr/SKILL.md
policies/
  padrao.json           leitura allow, escrita ask, execucao ask
  somente-leitura.json  escrita e execucao deny
  budgets.json          teto global mensal e teto diario por agente
  secrets.json          padroes de segredo para redacao de saida
pricing.json            precos por milhao de tokens, versionado
mcp.json                servidores MCP (stdio ou HTTP) e classificacao de risco
routing.json            intencoes por palavra chave e regras de roteamento
hooks.json              hooks de ciclo de vida no formato proprio
plugins.json            plugins no layout do Claude Code, por caminho local
overrides.json          provedor, modelo e contexto para agentes vindos de plugin
webhooks.json           webhooks de saida no formato Standard Webhooks
schedules/*.json        agendamentos por cron (opcional)
triggers/*.json         gatilhos externos (opcional)
```

## Plugins do Claude Code

Aponte `plugins.json` para a pasta de um plugin instalado, por exemplo
`~/.claude/plugins/marketplaces/<nome>/plugins/<plugin>`. O daemon le
`skills/`, `commands/`, `agents/`, `.mcp.json` e `hooks/hooks.json`, tudo
com prefixo do nome do plugin. Agentes de plugin nao trazem provedor nem
modelo: `overrides.json` decide, por plugin (`"meu-plugin"`), por agente
(`"meu-plugin/revisor"`) ou pelo padrao (`"default"`).

## Hooks

`hooks.json` no formato proprio:

```json
{
  "hooks": [
    { "event": "tool.before", "command": "node gates/segredo.js", "match": { "tool": "git|write_file" }, "timeout_ms": 10000 }
  ]
}
```

O hook recebe JSON na entrada padrao e responde com
`{ "decision": "allow" | "deny", "reason": "...", "output": "..." }`.
Saida diferente de zero nega. Hooks de plugin no formato do Claude Code
sao traduzidos automaticamente (`PreToolUse`, `PostToolUse`, `Stop`,
`UserPromptSubmit`; codigo de saida 2 nega).

## Antes de usar

- Preencha os precos do DeepSeek em `pricing.json`. O daemon recusa rodar um
  modelo com preco nulo.
- Troque o modelo de `local-leitor.md` pelo que sua maquina roda no Ollama.
  O modelo precisa suportar tool calling.
- `mcp.json` traz um servidor de exemplo apontando para `/tmp`. Substitua
  pelos seus. Segredos entram por nome de variavel de ambiente com `$`,
  nunca pelo valor.

## Variaveis de ambiente esperadas pelo daemon

| Variavel | Perfil |
|----------|--------|
| `ANTHROPIC_API_KEY` | claude-arquiteto |
| `DEEPSEEK_API_KEY` | deepseek-dev |
| nenhuma | local-leitor (Ollama local) |
