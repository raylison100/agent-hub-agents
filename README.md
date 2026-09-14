# agent-hub-agents

Configuracao do Agent Hub em texto, sem codigo: perfis de agente, papeis,
skills, workflows, agendamentos, tabela de precos, roteamento e politicas de
aprovacao e orcamento. O [daemon](https://github.com/raylison100/agent-hub-daemon)
le uma copia local desta pasta, apontada por `agents_dir` no `config.toml`.

Use este repositorio como ponto de partida e mantenha a sua copia, com os seus
perfis e conectores, fora de qualquer repositorio publico.

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
mcp.example.json        exemplo de servidor MCP; copie para mcp.json, que fica fora do git
routing.json            intencoes, regras, classificador, melhorador de prompt, pontuacao custo x capacidade e cascata
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
- Copie `mcp.example.json` para `mcp.json` e troque pelos seus servidores, ou
  cadastre pela tela Conectores. O `mcp.json` e configuracao da sua maquina:
  lista seus conectores e caminhos locais, e esta no `.gitignore`. Segredos
  entram por nome de variavel (`${VARIAVEL}`), nunca pelo valor, e o valor fica
  no cofre cifrado do daemon.
- Os perfis vem com `mcp: []`. Liste ali os conectores que cada agente pode usar.

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

## Cascata: modelo local primeiro, verificacao por codigo

Com `cascade` no `routing.json`, o pedido de uma intencao listada roda antes no
modelo local. O resultado passa por verificacoes feitas por codigo, sem modelo
nenhum, e so vai para o agente escolhido pelo roteador quando alguma falha:

```json
"cascade": { "agent": "qwen3", "escalate_to": "deepseek", "intents": ["explicar"] }
```

- `parada`: o run precisa terminar normalmente.
- `resposta`: nao pode vir vazia, com raciocinio vazado nem dizendo que nao achou.
- `fundamentacao`: precisa ter lido algo do workspace, e so pode citar arquivo que leu.
- `citacoes`: arquivo citado tem que existir, e a linha tem que caber no arquivo.
- `ferramentas`: nao pode ter falhado em todas as chamadas.

Vale so com agente automatico, sem papel, sem imagem e com o pedido cabendo no
`max_prompt_tokens` do perfil local. A tentativa recusada fica guardada como
mensagem filha do run, fora do historico da conversa.

Vem desligada. Medido em 2026-09-13 com 12 perguntas sobre este codigo, no
qwen3:8b com contexto de 8k: 3 respostas aceitas no local, uma delas errada
(leu o arquivo errado e citou certo), 9 escaladas com resposta certa. Custou
0,019 USD contra 0,035 USD do deepseek sozinho e demorou 216 s contra 177 s.
Vale ligar com um modelo local maior ou para uma intencao em que o local acerte
quase sempre.

## MCP remoto com OAuth

Servidor MCP por HTTP que pede OAuth ganha um bloco `oauth` no `mcp.json`:

```json
"meu-servidor": {
  "url": "https://mcp.exemplo.com/mcp",
  "oauth": { "issuer": "https://auth.exemplo.com", "scopes": ["read"] }
}
```

Com `issuer`, o daemon descobre os endpoints sozinho e, se o provedor aceitar
registro dinamico, cria o cliente na hora; `authorization_url`, `token_url` e
`client_id` tambem podem vir escritos a mao. Voce clica em Autorizar na tela de
Conectores, o navegador abre, e o token volta para o cofre cifrado, com
renovacao automatica pelo refresh_token. Sem autorizacao, o conector recusa a
conexao com a mensagem dizendo o que fazer.

## A2A: outro sistema falando com os seus agentes

O daemon publica o cartao em `/.well-known/agent.json`, com um skill por agente,
e aceita JSON-RPC em `/a2a` com `message/send` e `tasks/get`. A credencial e o
mesmo token do daemon, no cabecalho `Authorization: Bearer`.

## Acesso remoto com senha

Na propria maquina a conexao e automatica e nao pede nada. De fora, o caminho e
senha: defina uma em Configuracoes, Conexao, ou pelo terminal, sem deixar rastro
no historico do shell:

```
echo -n "sua senha" | agent-hub-daemon senha
```

O dispositivo de fora entra com a senha uma vez e recebe uma credencial propria,
que fica guardada nele. Voce ve os dispositivos autorizados com nome e ultimo
acesso na mesma tela, e revoga um sem mexer nos outros. Tres erros seguidos de
senha e a origem fica esperando, com a espera crescendo a cada nova tentativa.

## Parte do Agent Hub

Este repositorio e uma das partes do [Agent Hub](https://github.com/raylison100/agent-hub),
um gerenciador de modelos de IA que roda na sua maquina. A documentacao geral
esta na [wiki](https://github.com/raylison100/agent-hub/wiki).

| Repositorio | Papel |
|---|---|
| [agent-hub](https://github.com/raylison100/agent-hub) | ponto de partida, Makefile, scripts e wiki |
| [agent-hub-core](https://github.com/raylison100/agent-hub-core) | biblioteca TypeScript: adaptadores, laco do agente, custo, roteamento, ferramentas, protocolo |
| [agent-hub-daemon](https://github.com/raylison100/agent-hub-daemon) | servico local: sessoes, runs, aprovacoes, automacao, conectores, API WebSocket |
| [agent-hub-web](https://github.com/raylison100/agent-hub-web) | interface Vue 3 como PWA, a mesma no navegador, no celular e no desktop |
| [agent-hub-agents](https://github.com/raylison100/agent-hub-agents) | perfis, papeis, skills, workflows, precos, roteamento e politicas, em texto |
| [agent-hub-desktop](https://github.com/raylison100/agent-hub-desktop) | app Tauri 2 para Windows e Linux |
| [agent-hub-relay](https://github.com/raylison100/agent-hub-relay) | retransmissor sem estado para acesso remoto |
| [agent-hub-channels](https://github.com/raylison100/agent-hub-channels) | clientes em plataformas de mensagem, hoje Telegram |
| [agent-hub-docs](https://github.com/raylison100/agent-hub-docs) | planejamento, arquitetura, ADRs e a fonte das paginas da wiki |

## Licenca

[PolyForm Noncommercial 1.0.0](LICENSE). Pode ler, estudar, modificar e usar
para fins pessoais, de pesquisa, ensino ou em organizacao sem fins lucrativos.
Uso comercial nao e permitido sem autorizacao do autor.

Required Notice: Copyright (c) 2026 Raylison Nunes (https://github.com/raylison100)
