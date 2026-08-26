# Inventário — as 9 peças do fluxo agendado

> Bloco compartilhado pelas receitas desta pasta. Serve para não perder de vista onde
> você está: cada receita entrega **uma ou duas** peças, nunca o fluxo inteiro.

```
① Agendador (1x/dia)
        v  publica mensagem vazia
② Fila de gatilho
        v
③ TriggerHandler ──chama──▶ ④ CONSULTA NO ÍNDICE
                                  v
                            ⑤ ÍNDICE ESPARSO
                                  v devolve as chaves dos vencidos
                            ⑥ PRODUCER (SendMessageBatch)
                                  v
⑦ Fila
        v
⑧ CONSUMIDOR: valida, enriquece, executa
        v
⑨ Fim do ciclo: avança a data do próximo
   OU remove os atributos → o item sai do índice
```

| | Peça | O que faz | Receita |
|---|---|---|---|
| ① | Agendador | dispara 1×/dia, não sabe nada dos dados | infra |
| ② | Fila de gatilho | ~1 mensagem/dia; **é ela que garante que só uma réplica execute** | infra |
| ③ | TriggerHandler | ignora o corpo, chama o producer | — |
| ④ | Consulta no índice | percorre os shards, devolve as chaves vencidas | [sqs-producer-indice](202608261217-sqs-producer-indice.md) |
| ⑤ | Índice esparso | contém só quem está pendente | [dynamodb-indice-esparso-programatico](202608261122-dynamodb-indice-esparso-programatico.md) |
| ⑥ | Producer | uma mensagem por item, só com a identidade | [sqs-producer-indice](202608261217-sqs-producer-indice.md) |
| ⑦ | Fila | transporte | infra |
| ⑧ | Consumidor | valida, enriquece, chama o caso de uso | [sqs-handler-mensagem-magra](202608261425-sqs-handler-mensagem-magra.md) |
| ⑨ | Fim do ciclo | avança a data **ou** remove os atributos | — |

## Duas regras que atravessam o fluxo inteiro

**O producer decide QUEM. O consumidor decide O QUÊ.**
A mensagem carrega só identidade. Enriquecer do lado do consumidor dá retry, dado fresco
e isolamento por item — de graça.

**Falha por item nunca derruba o lote.**
Vale no laço do batch, no consumidor e no producer. E toda chamada em lote da AWS tem um
campo de falha parcial que precisa ser lido:

| Chamada | Campo | Se ignorar |
|---|---|---|
| `SendMessageBatch` | `failed()` | até 10 itens somem, com a chamada retornando sucesso |
| `BatchGetItem` | `UnprocessedKeys` | itens não lidos passam despercebidos |
| `BatchWriteItem` | `UnprocessedItems` | escritas não aplicadas passam despercebidas |

## Ordem de construção sugerida

⑤ → ④ → ⑥ → ⑧ → ③ → ② → ①

O agendador é a **última** peça, não a primeira: até ele existir, o producer é chamado
na mão, o que é justamente o que torna o teste possível.
