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
mcp.json                servidores MCP e classificacao de risco
routing.json            regras de roteamento (vazio ate a fase 4)
```

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
