# Producer que lê de um índice esparso e publica na fila — receita em 3 etapas

> Variante enxuta de [sqs-producer-listener.md](sqs-producer-listener.md), para o caso em que
> **o listener já existe** e você só precisa da peça que alimenta a fila.
> Nomes de tabela, campos e classes são **fictícios**, sempre — preencher antes de colar.

## Onde isto entra no fluxo

```
① Agendador (1x/dia)
        v  publica mensagem vazia
② Fila de gatilho
        v
③ TriggerHandler ──chama──▶ ④ PRODUCER  ◀── esta receita
                                  v consulta
                            ⑤ ÍNDICE ESPARSO
                                  v devolve os ids vencidos
                            ⑥ SendMessageBatch  ◀── esta receita
                                  v
⑦ Fila (já existe)
        v
⑧ Listener (já existe)
        v
⑨ Use case → os steps de negócio
        v
⑩ Fim do laço: avança a data do próximo ciclo
   OU remove os atributos → o item sai do índice
```

**Esta receita entrega o ④ e o ⑥.** Uma entrada (o índice) e uma saída (a fila).
Não encosta nos steps de negócio, não decide nada de domínio.

**Não precisa existir ainda:** ① ② ③ e ⑩. Para testar, chame o producer na mão —
o agendador é a última peça, não a primeira.

**A única amarra externa:** o ⑥ publica na fila que o ⑧ já consome. O corpo da mensagem
tem de casar com o que aquele listener sabe ler. Por isso a ETAPA 0 existe.

## Preencher antes de colar

```
<TABELA>    = nome da tabela
<INDICE>    = nome do indice esparso
<PK_INDEX>  = atributo do shard (valores "0" a "9"); so existe nos itens pendentes
<SK_INDEX>  = atributo de data, String ISO-8601
<FILA>      = nome (ou URL) da fila que o listener ja consome
```

Tudo o mais já está decidido dentro dos prompts. Não há escolha a fazer na hora.

---

## ETAPA 0 — descobrir a forma da mensagem (sem gerar código)

**Por quê:** se o producer publicar num formato que o listener não sabe ler, **toda mensagem
vai para a DLQ** — sem erro em lugar nenhum até chegar lá. Dois minutos aqui evitam isso.

No Copilot Chat, com `@workspace`:

```
Nao gere codigo. Responda citando arquivo e linha:
1. Qual classe consome a fila <FILA> e qual classe ela desserializa a partir do corpo
   da mensagem?
2. Me mostre a definicao completa dessa classe (todos os campos e tipos).
3. O corpo e lido direto como JSON dessa classe, ou existe algum envelope em volta?
4. Existe message attribute obrigatorio (traceId, correlationId, tipo)? Se sim, quais.
```

Guarde a resposta: o nome dessa classe entra na ETAPA 2 como `<CLASSE_MENSAGEM>`.

---

## ETAPA 1 — a consulta no índice (④)

```
Kotlin + AWS SDK v2, DynamoDB. Gere APENAS o repositorio de consulta, seguindo o estilo
das classes que ja existem no projeto.

Metodo que devolve todos os itens vencidos de um GSI esparso:
- tabela <TABELA>, indice <INDICE>
- partition key do indice: <PK_INDEX>, String, com valores "0" a "9" (10 shards)
- sort key do indice: <SK_INDEX>, String em ISO-8601
- para cada um dos 10 shards: Query com igualdade no <PK_INDEX> e condicao <= :agora
  no <SK_INDEX>
- acumule o resultado dos 10 shards numa lista unica

Requisitos obrigatorios, nao negociaveis:
1. Pagine com exclusiveStartKey/lastEvaluatedKey e pare SOMENTE quando lastEvaluatedKey
   for null ou vazio. NUNCA pare porque a pagina veio sem itens: pagina vazia com
   lastEvaluatedKey preenchido significa que ha continuacao, e parar ali descarta o
   resto da fila em silencio.
2. Nao use FilterExpression. Os dois recortes ja estao na chave.
3. Se o SDK do projeto oferecer queryPaginator, use-o: ele resolve o laco sozinho e o
   erro do item 1 deixa de existir.
4. O indice e KEYS_ONLY: a Query devolve apenas as chaves. Nao tente ler campos de
   negocio a partir do resultado do indice.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

⚠️ **`Limit` é tamanho de página, não teto de trabalho.** Tratá-lo como "processo N por dia"
faz a fila nunca drenar.

---

## ETAPA 2 — o producer (⑥)

```
Agora gere APENAS o Producer, no mesmo estilo.

Kotlin + AWS SDK v2, SQS. Metodo que recebe a lista devolvida pela consulta da etapa
anterior e publica na fila <FILA> com SendMessageBatch:
- uma mensagem por item
- corpo no formato da classe <CLASSE_MENSAGEM>
- lotes de ate 10 entradas
- ids de entrada unicos dentro de cada lote

Requisito obrigatorio, nao negociavel:
SendMessageBatch retorna resultado POR ENTRADA e NAO lanca excecao quando entradas sao
rejeitadas. Leia response.failed() e trate as entradas que falharam: log com o id da
entrada e reenvio. Ignorar failed() faz ate 10 itens sumirem em silencio, com a chamada
retornando sucesso.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

---

## Testar em homologação, sem o agendador

1. Grave **2 ou 3 itens de teste** com `<PK_INDEX>` = `"0"` e `<SK_INDEX>` com data no passado.
2. Chame o producer na mão (teste de integração ou endpoint temporário).
3. Confira a fila:

```bash
aws sqs get-queue-attributes --queue-url <URL_DA_FILA> \
  --attribute-names ApproximateNumberOfMessages
```

| Resultado | Leitura |
|---|---|
| chegaram 2 ou 3 mensagens | ✅ ④ ⑤ ⑥ funcionando ponta a ponta |
| chegou 0 | a consulta não achou — confira se o item tem **os dois** atributos e se a data está no passado |
| chegaram e caíram na DLQ | o corpo não casa com o listener — refaça a ETAPA 0 |

## Correções prontas, se escapar

**Se a paginação parar na página vazia:**
```
Pare o laco somente quando lastEvaluatedKey for null ou vazio. Pagina sem itens com
lastEvaluatedKey preenchido significa que ha continuacao.
```

**Se o `failed()` for ignorado:**
```
SendMessageBatch nao lanca excecao quando entradas sao rejeitadas. Leia response.failed()
e trate cada entrada que falhou.
```
