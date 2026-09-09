---
name: revisar-mr
description: Roteiro para revisar um merge request com foco em defeito, risco e clareza
activate:
  intent: revisar
---

# Revisar merge request

Siga a ordem. Nao pule etapas.

1. Leia a descricao e o titulo. Anote o objetivo declarado em uma frase.
2. Liste os arquivos alterados. Para cada um, leia o diff inteiro antes de
   opinar.
3. Procure, nesta ordem:
   - Defeito de logica: condicao invertida, limite errado, nulo nao tratado.
   - Efeito colateral: escrita em banco, chamada externa, alteracao de estado
     compartilhado sem transacao.
   - Compatibilidade: assinatura publica alterada, migracao sem rollback,
     contrato de API quebrado.
   - Teste: mudanca de comportamento sem teste, teste que nao falha sem a
     correcao.
   - Clareza: nome enganoso, funcao longa demais, comentario dentro de
     funcao.
4. Para cada achado, informe arquivo, linha, o problema em uma frase e a
   correcao sugerida. Sem elogios, sem preambulo.
5. Termine com um veredito: aprovar, aprovar com ajustes, ou bloquear, e o
   motivo em uma linha.
