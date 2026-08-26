# snippets

Trechos de referência para capturar rápido na máquina de trabalho, onde o ambiente é
travado e não dá para instalar nem experimentar à vontade.

> ## ⚠️ Repositório PÚBLICO — regra antes de commitar
>
> Nomes de tabela, campos, valores e volumes aqui são **fictícios**, sempre.
>
> **Nunca** commitar: regra de negócio real, número de produção, nome de sistema
> interno, nome de empregador, endpoint ou domínio corporativo.
>
> Material real fica fora deste repositório. Na dúvida, não commite — repositório
> público não se desfaz: clones e caches sobrevivem ao `git rm`.

## Conteúdo

| Pasta | Assunto |
|---|---|
| [bank/](bank/) | DynamoDB: exploração de tabela, índice esparso, escritas condicionais · SQS: producer/listener com falha por item · encaixar código gerado em projeto existente · remover código gerado errado |

### Fluxo agendado (índice esparso → fila → consumidor)

Quatro receitas encadeadas. Comece pelo inventário — ele diz qual peça cada uma entrega:

1. [bank/INVENTARIO.md](bank/INVENTARIO.md) — as 9 peças, a ordem de construção, as regras que atravessam tudo
2. [bank/dynamodb-indice-esparso-programatico.md](bank/dynamodb-indice-esparso-programatico.md) — ⑤ o índice
3. [bank/sqs-producer-indice.md](bank/sqs-producer-indice.md) — ④ ⑥ consulta e producer
4. [bank/sqs-handler-mensagem-magra.md](bank/sqs-handler-mensagem-magra.md) — ⑧ o consumidor

## Capturar um arquivo pelo terminal

```bash
curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/dynamodb-queries.md
```
