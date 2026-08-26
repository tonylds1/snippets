# DynamoDB — queries de referência

Comandos testados no LocalStack. Nomes de tabela e campos são **fictícios** —
trocar pelos reais na hora de usar.

Tabela de exemplo: `ciclos-agendados`, partition key `itemId`.

---

## 1. Preparar o shell

Não precisa de `awslocal`. Desde a versão 2.13 o AWS CLI lê `AWS_ENDPOINT_URL`
nativamente — o mesmo comando serve para LocalStack e para a AWS real.

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://localhost:4566   # só para LocalStack
```

Subir o LocalStack:

```bash
docker run -d --name localstack -p 4566:4566 \
  -e SERVICES=dynamodb,sqs localstack/localstack:3
```

Conferir a conexão:

```bash
aws sts get-caller-identity     # conta 000000000000 = LocalStack
aws dynamodb list-tables
```

> ⚠️ `AWS_ENDPOINT_URL` redireciona **todo** comando `aws` do terminal.
> Antes de falar com a AWS real: `unset AWS_ENDPOINT_URL`.

---

## 2. Explorar uma tabela que já existe

**Forma da tabela, índices e números** — cobrança, tamanho, chaves e projeção de cada GSI:

```bash
aws dynamodb describe-table --table-name ciclos-agendados \
  --query 'Table.{Itens:ItemCount,Bytes:TableSizeBytes,Cobranca:BillingModeSummary.BillingMode,Chave:KeySchema[].AttributeName,Indices:GlobalSecondaryIndexes[].{Nome:IndexName,Chaves:KeySchema[].AttributeName,Projecao:Projection.ProjectionType}}'
```

**Um item inteiro, para ver o formato:**

```bash
aws dynamodb scan --table-name ciclos-agendados --limit 1
```

**Só alguns campos, saída curta:**

```bash
aws dynamodb scan --table-name ciclos-agendados --limit 20 \
  --projection-expression "itemId,dados_extras"
```

**Todos os nomes de atributo distintos numa amostra** — em tabela sem schema,
é assim que se descobre quais campos existem:

```bash
aws dynamodb scan --table-name ciclos-agendados --limit 50 \
  --query 'Items[].keys(@)[]' --output text | tr '\t' '\n' | sort -u
```

**Buscar pela chave:**

```bash
aws dynamodb get-item --table-name ciclos-agendados \
  --key '{"itemId":{"S":"abc-123"}}'
```

**Achar itens com um Map preenchido** (filtro no cliente, leitura limitada):

```bash
aws dynamodb scan --table-name ciclos-agendados --limit 100 \
  --query 'Items[?length(dados_extras.M) > `0`].itemId.S'
```

Se o campo for String com JSON dentro, trocar por `length(dados_extras.S) > \`2\``
(vazio é `{}`, dois caracteres).

---

## 3. Criar tabela com índice esparso

O índice só conterá itens que tiverem **os dois** atributos de chave dele.
Item sem eles simplesmente não existe no índice — sem filtro, sem custo.

```bash
aws dynamodb create-table \
  --table-name ciclos-agendados \
  --attribute-definitions \
      AttributeName=itemId,AttributeType=S \
      AttributeName=pendenteShard,AttributeType=S \
      AttributeName=proximaExecucaoEm,AttributeType=S \
  --key-schema AttributeName=itemId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "gsi-pendentes",
    "KeySchema": [
      {"AttributeName":"pendenteShard","KeyType":"HASH"},
      {"AttributeName":"proximaExecucaoEm","KeyType":"RANGE"}
    ],
    "Projection": {"ProjectionType":"KEYS_ONLY"}
  }]'
```

Em `--attribute-definitions` entram **apenas** atributos que são chave de algo.
Declarar um atributo comum dá erro: DynamoDB não tem schema.

---

## 4. Massa de teste

```bash
# nunca optou — não entra no índice
aws dynamodb put-item --table-name ciclos-agendados \
  --item '{"itemId":{"S":"item-sem-inscricao"}}'

# pendente, vencido hoje — entra e é a vez dele
aws dynamodb put-item --table-name ciclos-agendados --item '{
  "itemId":{"S":"item-pendente"},
  "dataInscricao":{"S":"2026-08-01"},
  "pendenteShard":{"S":"0"},
  "proximaExecucaoEm":{"S":"2026-08-20T06:00:00Z"}}'

# pendente, vence amanhã — está no índice, não é a vez dele
aws dynamodb put-item --table-name ciclos-agendados --item '{
  "itemId":{"S":"item-futuro"},
  "dataInscricao":{"S":"2026-08-10"},
  "pendenteShard":{"S":"0"},
  "proximaExecucaoEm":{"S":"2026-08-21T06:00:00Z"}}'

# finalizado — saiu do índice, manteve o dado de negócio
aws dynamodb put-item --table-name ciclos-agendados --item '{
  "itemId":{"S":"item-finalizado"},
  "dataInscricao":{"S":"2026-07-01"},
  "finalizadoEm":{"S":"2026-08-15T06:00:00Z"}}'
```

---

## 5. Consultar o índice

**Tudo que está na fila** (prova da esparsidade — devolve 2, não 4):

```bash
aws dynamodb scan --table-name ciclos-agendados \
  --index-name gsi-pendentes --query 'Items[].itemId.S'
```

**O que vence hoje** — é esta que o produtor roda:

```bash
aws dynamodb query --table-name ciclos-agendados \
  --index-name gsi-pendentes \
  --key-condition-expression 'pendenteShard = :s AND proximaExecucaoEm <= :agora' \
  --expression-attribute-values '{":s":{"S":"0"},":agora":{"S":"2026-08-20T23:59:59Z"}}' \
  --query 'Items[].itemId.S'
```

Sem `--filter-expression`, então `--limit` vale de verdade.

**Contagem confiável** (o `ItemCount` do `describe-table` atualiza a cada ~6h):

```bash
aws dynamodb scan --table-name ciclos-agendados --select COUNT --query 'Count'
```

---

## 6. As três escritas

**Entrada na fila** — no momento em que o cliente opta:

```bash
aws dynamodb update-item --table-name ciclos-agendados \
  --key '{"itemId":{"S":"abc-123"}}' \
  --update-expression 'SET dataInscricao=:d, pendenteShard=:s, proximaExecucaoEm=:p' \
  --condition-expression 'attribute_exists(itemId) AND attribute_not_exists(pendenteShard) AND attribute_not_exists(finalizadoEm)' \
  --expression-attribute-values '{":d":{"S":"2026-08-24"},":s":{"S":"0"},":p":{"S":"2026-08-25T06:00:00Z"}}'
```

Cada metade da condição tapa um buraco:
`attribute_exists(itemId)` porque `update-item` é **upsert** — sem ela, uma chave
errada cria um item pela metade; `attribute_not_exists(pendenteShard)` impede
reentrada de quem já está na fila; `attribute_not_exists(finalizadoEm)`
impede ressuscitar quem já terminou.

**Avanço do cursor** — todo dia que não terminou:

```bash
aws dynamodb update-item --table-name ciclos-agendados \
  --key '{"itemId":{"S":"abc-123"}}' \
  --update-expression 'SET proximaExecucaoEm=:amanha, ultimaExecucaoEm=:agora' \
  --condition-expression 'attribute_exists(pendenteShard)' \
  --expression-attribute-values '{":amanha":{"S":"2026-08-26T06:00:00Z"},":agora":{"S":"2026-08-25T06:03:00Z"}}'
```

**Saída da fila** — `SET` e `REMOVE` no mesmo comando, atômico:

```bash
aws dynamodb update-item --table-name ciclos-agendados \
  --key '{"itemId":{"S":"abc-123"}}' \
  --update-expression 'SET finalizadoEm=:f REMOVE pendenteShard, proximaExecucaoEm' \
  --condition-expression 'attribute_exists(pendenteShard)' \
  --expression-attribute-values '{":f":{"S":"2026-08-25T06:03:00Z"}}'
```

Rodar duas vezes: a segunda falha com `ConditionalCheckFailedException`.
No código isso é "já foi feito, segue" — não é erro.

---

## 7. Armadilhas

**`put-item` substitui o item inteiro.** O que você não mandou, some — sem erro e
sem aviso. Para acrescentar atributo a um item existente, é `update-item` + `SET`.

**`Limit` é aplicado antes do `FilterExpression`.** Com filtro, `--limit 100` não
devolve 100: ele lê 100 e devolve os que passaram. Você paga pela leitura do que
descartou. Enquanto explora: **limite sim, filtro não.**

**`update-item` é upsert.** Sem `attribute_exists(<chave>)` na condição, uma chave
inexistente **cria** um item novo em vez de falhar.

**Número trafega como string:** `{"N": "10000"}`. Não é bug — preserva a precisão
decimal de 38 dígitos. No código, desserializar para `BigDecimal`, nunca `Double`.

**Atributo aninhado não pode ser chave de GSI.** Só atributo escalar de topo.

**Chave de GSI é imutável.** Errar custa índice novo e migração.

**Um GSI por vez.** Não dá para criar ou apagar dois índices simultaneamente.

**`<` e `>` são redirecionamento no shell.** Nunca use ângulo como placeholder em
comando para colar — `--table-name <tabela>` vira "arquivo não encontrado".
