# DynamoDB — queries de referência

Comandos testados no LocalStack. Nomes de tabela e campos são **fictícios** —
trocar pelos reais na hora de usar.

Tabela de exemplo: `migracao-propostas`, partition key `idCartao`.

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
aws dynamodb describe-table --table-name migracao-propostas \
  --query 'Table.{Itens:ItemCount,Bytes:TableSizeBytes,Cobranca:BillingModeSummary.BillingMode,Chave:KeySchema[].AttributeName,Indices:GlobalSecondaryIndexes[].{Nome:IndexName,Chaves:KeySchema[].AttributeName,Projecao:Projection.ProjectionType}}'
```

**Um item inteiro, para ver o formato:**

```bash
aws dynamodb scan --table-name migracao-propostas --limit 1
```

**Só alguns campos, saída curta:**

```bash
aws dynamodb scan --table-name migracao-propostas --limit 20 \
  --projection-expression "idCartao,dados_confirmacao"
```

**Todos os nomes de atributo distintos numa amostra** — em tabela sem schema,
é assim que se descobre quais campos existem:

```bash
aws dynamodb scan --table-name migracao-propostas --limit 50 \
  --query 'Items[].keys(@)[]' --output text | tr '\t' '\n' | sort -u
```

**Buscar pela chave:**

```bash
aws dynamodb get-item --table-name migracao-propostas \
  --key '{"idCartao":{"S":"abc-123"}}'
```

**Achar itens com um Map preenchido** (filtro no cliente, leitura limitada):

```bash
aws dynamodb scan --table-name migracao-propostas --limit 100 \
  --query 'Items[?length(dados_confirmacao.M) > `0`].idCartao.S'
```

Se o campo for String com JSON dentro, trocar por `length(dados_confirmacao.S) > \`2\``
(vazio é `{}`, dois caracteres).

---

## 3. Criar tabela com índice esparso

O índice só conterá itens que tiverem **os dois** atributos de chave dele.
Item sem eles simplesmente não existe no índice — sem filtro, sem custo.

```bash
aws dynamodb create-table \
  --table-name migracao-propostas \
  --attribute-definitions \
      AttributeName=idCartao,AttributeType=S \
      AttributeName=transbordoPendente,AttributeType=S \
      AttributeName=proximaExecucaoEm,AttributeType=S \
  --key-schema AttributeName=idCartao,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes '[{
    "IndexName": "gsi-transbordo-pendente",
    "KeySchema": [
      {"AttributeName":"transbordoPendente","KeyType":"HASH"},
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
aws dynamodb put-item --table-name migracao-propostas \
  --item '{"idCartao":{"S":"cartao-sem-optin"}}'

# pendente, vencido hoje — entra e é a vez dele
aws dynamodb put-item --table-name migracao-propostas --item '{
  "idCartao":{"S":"cartao-pendente"},
  "dataOptin":{"S":"2026-08-01"},
  "transbordoPendente":{"S":"PEND#0"},
  "proximaExecucaoEm":{"S":"2026-08-20T06:00:00Z"}}'

# pendente, vence amanhã — está no índice, não é a vez dele
aws dynamodb put-item --table-name migracao-propostas --item '{
  "idCartao":{"S":"cartao-futuro"},
  "dataOptin":{"S":"2026-08-10"},
  "transbordoPendente":{"S":"PEND#0"},
  "proximaExecucaoEm":{"S":"2026-08-21T06:00:00Z"}}'

# finalizado — saiu do índice, manteve o dado de negócio
aws dynamodb put-item --table-name migracao-propostas --item '{
  "idCartao":{"S":"cartao-finalizado"},
  "dataOptin":{"S":"2026-07-01"},
  "transbordoFinalizadoEm":{"S":"2026-08-15T06:00:00Z"}}'
```

---

## 5. Consultar o índice

**Tudo que está na fila** (prova da esparsidade — devolve 2, não 4):

```bash
aws dynamodb scan --table-name migracao-propostas \
  --index-name gsi-transbordo-pendente --query 'Items[].idCartao.S'
```

**O que vence hoje** — é esta que o produtor roda:

```bash
aws dynamodb query --table-name migracao-propostas \
  --index-name gsi-transbordo-pendente \
  --key-condition-expression 'transbordoPendente = :s AND proximaExecucaoEm <= :agora' \
  --expression-attribute-values '{":s":{"S":"PEND#0"},":agora":{"S":"2026-08-20T23:59:59Z"}}' \
  --query 'Items[].idCartao.S'
```

Sem `--filter-expression`, então `--limit` vale de verdade.

**Contagem confiável** (o `ItemCount` do `describe-table` atualiza a cada ~6h):

```bash
aws dynamodb scan --table-name migracao-propostas --select COUNT --query 'Count'
```

---

## 6. As três escritas

**Entrada na fila** — no momento em que o cliente opta:

```bash
aws dynamodb update-item --table-name migracao-propostas \
  --key '{"idCartao":{"S":"abc-123"}}' \
  --update-expression 'SET dataOptin=:d, transbordoPendente=:s, proximaExecucaoEm=:p' \
  --condition-expression 'attribute_exists(idCartao) AND attribute_not_exists(transbordoPendente) AND attribute_not_exists(transbordoFinalizadoEm)' \
  --expression-attribute-values '{":d":{"S":"2026-08-24"},":s":{"S":"PEND#0"},":p":{"S":"2026-08-25T06:00:00Z"}}'
```

Cada metade da condição tapa um buraco:
`attribute_exists(idCartao)` porque `update-item` é **upsert** — sem ela, uma chave
errada cria um item pela metade; `attribute_not_exists(transbordoPendente)` impede
reentrada de quem já está na fila; `attribute_not_exists(transbordoFinalizadoEm)`
impede ressuscitar quem já terminou.

**Avanço do cursor** — todo dia que não terminou:

```bash
aws dynamodb update-item --table-name migracao-propostas \
  --key '{"idCartao":{"S":"abc-123"}}' \
  --update-expression 'SET proximaExecucaoEm=:amanha, ultimaExecucaoEm=:agora' \
  --condition-expression 'attribute_exists(transbordoPendente)' \
  --expression-attribute-values '{":amanha":{"S":"2026-08-26T06:00:00Z"},":agora":{"S":"2026-08-25T06:03:00Z"}}'
```

**Saída da fila** — `SET` e `REMOVE` no mesmo comando, atômico:

```bash
aws dynamodb update-item --table-name migracao-propostas \
  --key '{"idCartao":{"S":"abc-123"}}' \
  --update-expression 'SET transbordoFinalizadoEm=:f REMOVE transbordoPendente, proximaExecucaoEm' \
  --condition-expression 'attribute_exists(transbordoPendente)' \
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
