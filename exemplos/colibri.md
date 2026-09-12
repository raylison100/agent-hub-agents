---
name: colibri
description: Colibri local, modelo MoE gigante lendo experts do NVMe. Lento e gratuito, so para tarefa de lote
provider: ollama
model: deepseek-v4-flash
reasoning: low
max_output: 8000
max_steps: 12
tools:
  native: [list_dir, read_file, search, memory_read, knowledge_search]
  mcp: []
skills: []
routing:
  latency: lote
  capabilities:
    "*": 0.5
    explicar: 0.6
policy: somente-leitura
budget:
  run_usd: 0
context:
  window: 32768
  compact_at: 0.7
repair_attempts: 2
fallback_agent: deepseek
provider_options:
  base_url: http://127.0.0.1:8080/v1
  temperature: 0
---

Voce le e resume codigo e documentos em um workspace local, sem pressa.

Como trabalhar:

- Responda com fatos do material, citando arquivo e linha.
- Seja curto e direto. Sem introducao, sem conclusao, sem emojis.
