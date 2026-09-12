# agent-hub-agents

Repositorio de dados: perfis de agente, skills, tabela de precos, servidores
MCP e politicas de aprovacao. Sem codigo. O daemon le um clone local deste
repositorio.

Formato em `../docs/07-agentes.md`, extensoes em `../docs/09-extensoes.md`.

## Estrutura

```
profiles/               um Markdown com frontmatter por agente
  claude.md
  deepseek.md
  openai.md            (usa a Responses API por padrao; `provider_options.api: chat` volta ao chat completions)
  gemini.md
  qwen3.md
roles/                  papel separado do modelo: prompt, ferramentas e politica
  revisor.md            roda em deepseek, gemini, openai ou claude
  arquiteto.md          roda em claude, openai ou gemini
skills/                 uma pasta por skill, formato Agent Skills
  revisar-mr/SKILL.md
policies/
  padrao.json           leitura allow, escrita ask, execucao ask
  somente-leitura.json  escrita e execucao deny
  budgets.json          teto global mensal e teto diario por agente
  secrets.json          padroes de segredo para redacao de saida
pricing.json            precos por milhao de tokens, versionado
mcp.json                servidores MCP (stdio ou HTTP) e classificacao de risco
routing.json            intencoes, regras, classificador, melhorador de prompt e pontuacao custo x capacidade
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

- `pricing.json` traz Anthropic, DeepSeek V4 (tarifa de pico; fora do pico
  a API cobra metade) e os principais modelos OpenAI em tarifa Standard:
  `gpt-6-astra`, `gpt-5.6-sol`, `gpt-5.6-terra`, `gpt-5.6-luna`, `gpt-5.5`,
  `gpt-5.4`, `gpt-5.4-mini`, `gpt-5.4-nano` e `gpt-5.3-codex`. Modelo fora
  da lista bloqueia o run ate ser cadastrado, de proposito.
- No DeepSeek V4, `reasoning: low` desliga o raciocinio; `medium` e `high`
  ligam com esforco `high`; `max` liga com `max`. O adaptador reenvia o
  `reasoning_content` nas mensagens seguintes, exigencia da API quando ha
  ferramentas.
- Troque o modelo de `qwen3.md` pelo que sua maquina roda no Ollama.
  O modelo precisa suportar tool calling.
- `mcp.json` traz um servidor de exemplo apontando para `/tmp`. Substitua
  pelos seus. Segredos entram por nome de variavel de ambiente com `$`,
  nunca pelo valor.

## Variaveis de ambiente esperadas pelo daemon

| Variavel | Perfil |
|----------|--------|
| `ANTHROPIC_API_KEY` | claude |
| `DEEPSEEK_API_KEY` | deepseek |
| `OPENAI_API_KEY` | openai |
| `GEMINI_API_KEY` | gemini |
| nenhuma | qwen3 (Ollama local) |

## Contexto do projeto no workspace

Cada workspace pode ter uma pasta `.agent-hub/` que o harness le e escreve:

```
.agent-hub/
  memory/     um fato por arquivo, com frontmatter e regra de ativacao
  specs/      especificacao de tarefa, escrita antes de executar
  decisions/  decisao tomada, com o motivo
  INSTRUCOES.md   alternativa a .claude/CLAUDE.md e AGENTS.md
```

As instrucoes do projeto entram sempre no inicio da conversa; o harness usa o
primeiro arquivo que achar entre `.claude/CLAUDE.md`, `CLAUDE.md`, `AGENTS.md` e
`.agent-hub/INSTRUCOES.md`. Cada item de memoria entra so quando a regra de
`activate` casa com o pedido (por `keywords` ou `files`); item sem regra entra
sempre e por isso custa em toda chamada. O total injetado respeita um teto de
15% da janela do modelo, e o que ficou de fora aparece na conversa com o motivo.

Os agentes escrevem com `memory_write` e `spec_write`, sempre com run, agente e
data no cabecalho. Voce ve e apaga tudo pela tela Configuracoes, Contexto do
projeto. O agendamento `revisao-memoria` audita a memoria todo mes.

Versionar ou nao: `specs/` e `decisions/` sao documentacao do projeto e valem
commit; `memory/` depende do time. Para deixar tudo fora do git, acrescente
`.agent-hub/` ao `.gitignore` do seu projeto.

## Base de conhecimento e glossario

`.agent-hub/knowledge/` guarda o material do projeto: processo do time,
transcricoes, notas. O daemon indexa em FTS5 dentro do proprio SQLite, so o que
mudou desde a ultima vez, sem embedding e sem custo. Le `.md`, `.txt`, `.csv`,
`.json` e `.yaml`; outros formatos aparecem como ignorados (PDF precisa ser
convertido antes).

Os agentes buscam com `knowledge_search`, que devolve cada trecho com a citacao
`[arquivo:linha]` pronta, e a descricao da ferramenta cobra a citacao na
resposta. `.agent-hub/glossario.md` e carregado sempre, para os agentes usarem
as palavras do time.

## Classe de latencia e o perfil de lote

Cada perfil declara `routing.latency`: `interativo` (voce esperando),
`lote` (tarefa agendada, sem pressa) ou `ambos`, o padrao. Conversa comum e
tratada como interativa e exclui quem so serve para lote; run de agendamento,
gatilho ou workflow entra como lote, e ai o custo pesa mais na pontuacao
(`batch_cost_weight`, 0.85 contra os 0.5 do interativo), o que empurra o
trabalho da madrugada para o modelo mais barato.

`exemplos/colibri.md` e um perfil pronto para o Colibri, motor local que roda
modelos gigantes lendo experts do disco. Ele nao entra sozinho: copie para
`profiles/` quando tiver o gateway no ar em `127.0.0.1:8080`. Antes disso,
confira o espaco em disco, porque os modelos de ponta passam de 370 GB, e que a
velocidade fica abaixo de 1 token/s em maquina parecida com a sua, ou seja, so
faz sentido como camada de lote.
