# AGORA — o prompt do momento

> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## O que se descobriu em 27/08 — e o que isso apaga

A camada interna **sempre soube** fazer o que era preciso. O serviço de query dela tem um
método para GSI que recebe `indexName` e um objeto de opções com **condição de ordenação**:

```
metodoDeQueryPorGsi(tabela, schema, indexName, valorDaPkDoIndice, opcoes): Flux<T>

opcoes:  sortCondition = Between(de, ate)   <- o recorte de data
         pageSize                            <- paginação é assunto conhecido da API
         scanForward, consistentRead, retries
```

`Between(piso, agora)` **é** o "menor ou igual a agora": data em ISO-8601 é string ordenável.

### O que isso apaga

| Já não é problema | Por quê |
|---|---|
| injetar o cliente do SDK | desnecessário — a camada interna faz a consulta |
| o precedente em outro projeto | resolvia uma permissão que ninguém precisava dar |
| mudar a chave do índice p/ embutir a data | o desenho original volta a valer inteiro |
| pedir ao time dono que exponha a condição | já está exposta |

⚠️ **A lição, porque foi cara:** um dia inteiro de investigação para um bloqueio que não
existia. A resposta estava na **assinatura de um método da própria biblioteca** — não numa
conversa, não em outro repositório. **Ler a API antes de desenhar em volta dela.**

### Continua em aberto (para depois da demonstração, não agora)

- 🔴 **O `Flux` percorre todas as páginas ou só a primeira?** Se só a primeira, o time inteiro
  tem um perdedor de itens silencioso. Há um jeito de descobrir em homologação — item F2.3.
- ⚠️ **Índice `KEYS_ONLY` + `schema` da entidade.** A query devolve só as chaves; os demais
  campos vêm nulos. Se a entidade tiver campo não-anulável, isso estoura na desserialização.
  Verificar antes do deploy — item F1.1.

---

## O plano de hoje, em três fases

> ⚠️ **A aplicação ainda não sobe contra o LocalStack.** Ficou para depois, por causa do prazo.
> Isso separa as duas fases de vez:

| | O que prova | Onde roda |
|---|---|---|
| **Fase 0** | o **desenho** — índice esparso, shards, mensagem magra | LocalStack, em bash, na sua máquina |
| **Fase 1** | o **código** — testes unitários com a camada interna mockada | build local, sem AWS nenhuma |
| **Fase 2** | os dois juntos | homologação, com os mesmos comandos da Fase 0 |

🏠 **A Fase 0 precisa de Docker + LocalStack.** Em máquina corporativa isso costuma estar
bloqueado. **Rode em casa, salve a saída do terminal** — ela é a demonstração do desenho, e
vale igual num terminal gravado. Não gaste o dia tentando liberar container no trabalho.

**Faça a Fase 0 antes de abrir o Copilot.** São ~15 minutos e ela sozinha demonstra ④⑤⑥ ponta
a ponta. **Se o resto não fechar, você ainda tem o que mostrar.**

## Preencher antes de colar

```bash
export TABELA=
export PK_TABELA=          # a chave da tabela; único campo da mensagem
export INDICE=
export PK_INDEX=           # atributo do shard, "0".."9" — só existe nos pendentes
export SK_INDEX=           # atributo de data, String ISO-8601
export FILA=
```

Guarde esse bloco preenchido num arquivo `nomes.env` **fora do repositório** e
`source nomes.env` a cada terminal novo.

---

# FASE 0 — a demonstração em bash (15 min)

## 0.1 Subir o LocalStack

```bash
docker run -d --name localstack -p 4566:4566 -e SERVICES=dynamodb,sqs localstack/localstack:3

export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
export AWS_ENDPOINT_URL=http://localhost:4566

aws sts get-caller-identity      # conta 000000000000 = LocalStack
```

> ⚠️ `AWS_ENDPOINT_URL` redireciona **todo** comando `aws` do terminal.
> Antes de falar com a AWS real: `unset AWS_ENDPOINT_URL`.

## 0.2 O script da demonstração

Salve como `demo-fluxo.sh` **fora do repositório** (ele carrega nomes reais).

```bash
#!/usr/bin/env bash
set -euo pipefail
source ./nomes.env

export AWS_ACCESS_KEY_ID=test AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1 AWS_ENDPOINT_URL=http://localhost:4566

CONTA=$(aws sts get-caller-identity --query Account --output text)
if [ "$CONTA" != "000000000000" ]; then
  echo "ABORTADO: conta $CONTA nao e LocalStack. Este script APAGA a tabela." >&2
  exit 1
fi

HOJE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
ONTEM=$(date -u -d '-1 day' +%Y-%m-%dT%H:%M:%SZ)
AMANHA=$(date -u -d '+1 day' +%Y-%m-%dT%H:%M:%SZ)

echo "==> limpando execução anterior"
aws dynamodb delete-table --table-name "$TABELA" >/dev/null 2>&1 || true
aws sqs delete-queue --queue-url "$(aws sqs get-queue-url --queue-name "$FILA" \
  --query QueueUrl --output text 2>/dev/null)" >/dev/null 2>&1 || true
sleep 2

echo "==> ⑤ criando tabela + índice esparso KEYS_ONLY (nasce vazio)"
aws dynamodb create-table --table-name "$TABELA" \
  --attribute-definitions \
      AttributeName="$PK_TABELA",AttributeType=S \
      AttributeName="$PK_INDEX",AttributeType=S \
      AttributeName="$SK_INDEX",AttributeType=S \
  --key-schema AttributeName="$PK_TABELA",KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --global-secondary-indexes "[{
    \"IndexName\": \"$INDICE\",
    \"KeySchema\": [
      {\"AttributeName\":\"$PK_INDEX\",\"KeyType\":\"HASH\"},
      {\"AttributeName\":\"$SK_INDEX\",\"KeyType\":\"RANGE\"}
    ],
    \"Projection\": {\"ProjectionType\":\"KEYS_ONLY\"}
  }]" >/dev/null
aws dynamodb wait table-exists --table-name "$TABELA"

echo "==> ⑦ criando a fila"
QUEUE_URL=$(aws sqs create-queue --queue-name "$FILA" --query QueueUrl --output text)

echo "==> massa de teste: 4 itens, só 1 deve virar mensagem"
put() { aws dynamodb put-item --table-name "$TABELA" --item "$1" >/dev/null; }

# nunca optou — sem os atributos, não entra no índice
put "{\"$PK_TABELA\":{\"S\":\"sem-inscricao\"}}"
# pendente e vencido — é a vez dele
put "{\"$PK_TABELA\":{\"S\":\"pendente-vencido\"},
      \"$PK_INDEX\":{\"S\":\"0\"},\"$SK_INDEX\":{\"S\":\"$ONTEM\"}}"
# pendente, vence amanhã — está no índice, não é a vez dele
put "{\"$PK_TABELA\":{\"S\":\"pendente-futuro\"},
      \"$PK_INDEX\":{\"S\":\"0\"},\"$SK_INDEX\":{\"S\":\"$AMANHA\"}}"
# finalizado — atributos removidos, saiu do índice, manteve o dado de negócio
put "{\"$PK_TABELA\":{\"S\":\"finalizado\"},\"finalizadoEm\":{\"S\":\"$ONTEM\"}}"
sleep 2

echo
echo "--- prova da esparsidade: a tabela tem 4, o índice tem 2 ---"
echo -n "tabela: "; aws dynamodb scan --table-name "$TABELA" --select COUNT --query Count --output text
echo -n "índice: "; aws dynamodb scan --table-name "$TABELA" --index-name "$INDICE" --select COUNT --query Count --output text

echo
echo "==> ④ consultando os 10 shards do índice, sem FilterExpression e sem Scan"
CHAVES='[]'
for SHARD in 0 1 2 3 4 5 6 7 8 9; do
  PAG=$(aws dynamodb query --table-name "$TABELA" --index-name "$INDICE" \
    --key-condition-expression "$PK_INDEX = :s AND $SK_INDEX <= :agora" \
    --expression-attribute-values "{\":s\":{\"S\":\"$SHARD\"},\":agora\":{\"S\":\"$HOJE\"}}" \
    --query "Items[].$PK_TABELA.S" --output json)
  N=$(echo "$PAG" | jq 'length')
  if [ "$N" -gt 0 ]; then echo "    shard $SHARD -> $N item(ns)"; fi
  CHAVES=$(jq -c -n --argjson a "$CHAVES" --argjson b "$PAG" '$a + $b')
done
echo "    total vencido: $(echo "$CHAVES" | jq 'length')"

echo
echo "==> ⑥ publicando na fila — uma mensagem por item, só a identidade"
ENTRIES=$(echo "$CHAVES" | jq -c --arg pk "$PK_TABELA" \
  'to_entries | map({Id: "m\(.key)", MessageBody: ({($pk): .value} | tojson)})')
if [ "$(echo "$CHAVES" | jq 'length')" -eq 0 ]; then
  echo "    nada vencido — nenhuma chamada feita"; exit 0
fi
RESP=$(aws sqs send-message-batch --queue-url "$QUEUE_URL" --entries "$ENTRIES")
echo "    publicadas: $(echo "$RESP" | jq '.Successful | length')"
echo "    falhas:     $(echo "$RESP" | jq '.Failed // [] | length')   <- ler failed() SEMPRE"

echo
echo "--- ⑦ o que chegou na fila ---"
aws sqs receive-message --queue-url "$QUEUE_URL" --max-number-of-messages 10 \
  --query 'Messages[].Body' --output text
```

```bash
chmod +x demo-fluxo.sh && ./demo-fluxo.sh
```

## 0.3 O que a saída prova

| Linha da saída | O que está provado |
|---|---|
| `tabela: 4` / `índice: 2` | ⑤ o índice é **esparso** — quem não tem os dois atributos não existe nele |
| `shard 0 -> 1 item` | ④ a consulta percorre shards e o recorte de data está **na chave**, sem filtro |
| `publicadas: 1` | ⑥ só o vencido virou mensagem; o de amanhã ficou no índice |
| corpo com um campo só | a regra da mensagem magra: **producer decide QUEM, consumidor decide O QUÊ** |
| `falhas: 0` | o campo existe e é lido — não é sucesso presumido |

⚠️ **Se `índice: 4`** — algum item tem os atributos sem dever ter. Índice esparso errado é
backfill caro em produção. Confira a massa antes de seguir.

**Pare aqui e respire.** ✅ **A demonstração está pronta.** O que vem agora é melhoria.

---

# FASE 1 — o código, provado por teste unitário

> A aplicação não roda local. **O teste unitário é a única coisa que te diz se está certo
> antes de homologação.** Não é burocracia: é o seu único instrumento.
>
> Cole o [preâmbulo de contenção](202608261829-copilot-contencao.md) no começo de cada prompt
> em modo agente. **Um arquivo novo por vez** — nunca peça reescrita de arquivo grande.

## F1.0 💰 O bloco da API — o que mais economiza token

**A maior parte do custo do Copilot é ele procurar.** Varre o projeto, lê arquivo, erra,
tenta de novo. E você **já achou** o que ele procuraria.

Preencha isto **uma vez**, a partir do seu código, e guarde num `api-interna.txt`
**fora do repositório** (mesmo lugar do `nomes.env` — carrega nomes reais):

```
A biblioteca interna de acesso ao DynamoDB JA esta no projeto e ja e usada em outras
classes. Use EXATAMENTE a API abaixo. Nao procure outra, nao inspecione o projeto atras
dela, nao injete cliente do AWS SDK.

  <servico>.<metodoDeQueryPorGsi>(
      tableName, schema, indexName, <valorDaPkDoIndice>, <flagPkNumerica>, options
  ): Flux<T>

  options = <ClasseDeOpcoes>(
      sortCondition = <ClasseDeCondicao>.Between(from, to),
      pageSize = ...
  )
```

Copie os nomes reais do seu código — assinatura completa, na ordem certa dos parâmetros.

**Cole esse bloco no começo de todo prompt da Fase 1.** Sem ele, cada prompt paga uma
varredura do projeto. Com ele, o modelo escreve direto.

💰 **Segunda economia: use o chat, não o modo agente.** Agente relê e reescreve arquivo a
cada iteração; chat devolve o texto e você cola. Mais barato e mais previsível.

## F1.1 Antes de qualquer prompt: a entidade aguenta `KEYS_ONLY`?

O índice é `KEYS_ONLY`. A query devolve **só** as chaves — os outros campos da entidade vêm
nulos. Abra a classe de entidade e responda:

**Existe campo não-anulável (sem `?`) fora das chaves do índice e da tabela?**

| Resposta | O que fazer |
|---|---|
| não, todos anuláveis | ✅ segue |
| sim | ⚠️ a desserialização estoura. Ou usa uma projeção `INCLUDE` com os campos de chave, ou lê o resultado sem passar pelo schema da entidade |

**Descobrir isso em homologação custa um deploy. Descobrir agora custa 2 minutos.**

## F1.2 O contrato local (escreva à mão, são 4 linhas)

Não peça isto ao Copilot — é o nome que você vai conviver.

```kotlin
interface <NomeDaQuery> {
    fun buscarVencidos(agora: Instant): List<String>
}
```

**Por que existe:** o teste unitário mocka a camada interna e verifica **esta** interface.
E se um dia a biblioteca mudar, muda uma classe só.

## F1.3 Prompt — a consulta (④)

```
[COLE AQUI O BLOCO DA API, do seu api-interna.txt]

Kotlin. Gere APENAS a classe que implementa a interface <NomeDaQuery>, que ja existe no
projeto. Nao altere a interface. Nao inspecione o resto do projeto: tudo que voce precisa
saber da biblioteca esta no bloco acima.

O metodo devolve as chaves de todos os itens vencidos de um GSI esparso:
- indice <INDICE>
- particao do indice <PK_INDEX>: String com valores "0" a "9" (10 shards)
- ordenacao do indice <SK_INDEX>: String ISO-8601
- para cada um dos 10 shards, uma consulta com a condicao de ordenacao Between,
  de um piso fixo ate o instante recebido por parametro
- acumule os 10 resultados numa lista unica de <PK_TABELA>

Requisitos obrigatorios:
1. O piso do Between e uma constante ISO-8601 anterior a qualquer data possivel.
   Nao invente data "de hoje menos N dias": item atrasado ha muito tempo tem de aparecer.
2. Falha num shard NAO interrompe os outros nove. Registre e siga.
3. Nada de FilterExpression e nada de Scan. Os dois recortes ja estao na chave.
4. Log com quantos itens vieram por shard e o total. Sem isso o teste em homologacao
   nao e verificavel.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

## F1.4 Prompt — os testes da consulta

```
[COLE AQUI O BLOCO DA API]

Agora gere APENAS os testes unitarios dessa classe, no mesmo estilo dos testes que ja
existem no projeto. Mocke o servico de query da biblioteca interna descrito no bloco.

Casos obrigatorios, um teste cada:
1. Sao feitas exatamente 10 chamadas, uma por shard, com os valores "0" a "9".
2. Toda chamada usa o indice <INDICE> e condicao de ordenacao Between, com o limite
   superior igual ao instante recebido por parametro. Capture o argumento e verifique.
3. Os resultados dos 10 shards aparecem todos na lista devolvida, sem perder nem duplicar.
4. Quando um shard lanca excecao, os outros nove continuam e a lista traz os itens deles.
5. Nenhum item vencido em nenhum shard devolve lista vazia, sem excecao.

Nao teste a biblioteca interna. Teste o comportamento da SUA classe.
```

## F1.5 Prompt — o producer (⑥)

**ETAPA 2** de [sqs-producer-indice.md](202608261217-sqs-producer-indice.md), sem alteração.

🔴 `SendMessageBatch` **não lança exceção** quando entradas são rejeitadas. Ler `response.failed()`.

## F1.6 Prompt — os testes do producer

```
Agora gere APENAS os testes unitarios do Producer. Mocke o cliente de SQS.

Casos obrigatorios, um teste cada:
1. 25 itens geram 3 chamadas, com 10, 10 e 5 entradas. Nenhuma chamada com mais de 10.
2. Os ids de entrada sao unicos dentro de cada lote.
3. O corpo da mensagem contem SOMENTE o campo <PK_TABELA>. Desserialize e verifique que
   nao ha nenhum outro campo.
4. Quando a resposta traz entradas em failed(), cada entrada falha aparece no log ou no
   retorno, identificada. Este teste e o mais importante da lista: sem ele, ate 10 itens
   somem em silencio com a chamada retornando sucesso.
5. Lista vazia nao faz nenhuma chamada ao SQS.

Nao teste o SQS. Teste o comportamento do SEU codigo.
```

## F1.7 Revisão antes do PR

**ETAPA 3** de [sqs-producer-indice.md](202608261217-sqs-producer-indice.md) — os 12 itens,
CONFORME/NÃO CONFORME com arquivo e linha. Força resposta item a item em vez de "está tudo certo".

```bash
./gradlew test
```

---

# FASE 2 — verificar em homologação

São os mesmos comandos da Fase 0, **sem** o `AWS_ENDPOINT_URL`.

```bash
unset AWS_ENDPOINT_URL      # ⚠️ obrigatório, senão você fala com o container
```

## F2.1 O índice nasceu vazio?

```bash
aws dynamodb describe-table --table-name "$TABELA" \
  --query "Table.GlobalSecondaryIndexes[?IndexName=='$INDICE'].[IndexStatus,ItemCount,Backfilling]"
```

`ACTIVE`, `0`, e `Backfilling` ausente. **`ItemCount` diferente de 0 significa que algum item
já tinha os atributos** — o índice não nasceu vazio.

## F2.2 Marcar 2–3 itens e rodar

Marque itens de teste com os dois atributos e data no passado, invoque o producer na mão, e:

```bash
aws sqs get-queue-attributes --queue-url "$URL_DA_FILA" \
  --attribute-names ApproximateNumberOfMessages
```

| Resultado | Leitura |
|---|---|
| chegaram 2 ou 3 | ✅ ④⑤⑥ ponta a ponta, em homologação |
| 0 | o item tem **os dois** atributos? a data está no passado? |
| log diz publicou, fila vazia | leia `failed()` — as entradas foram rejeitadas |

## F2.3 🔴 O teste que responde a pergunta do `Flux`

**Marque mais itens do que o `pageSize` padrão** — se o padrão for 100, marque 150, todos no
mesmo shard, todos vencidos.

| Voltaram | Conclusão |
|---|---|
| 150 | ✅ o `Flux` pagina sozinho. A dúvida morre |
| 100 (ou o `pageSize`) | 🔴 **devolve só a primeira página.** Todo mundo que usa essa biblioteca perde itens em silêncio — e isso vira assunto do time, não seu |

Este teste custa um script de seed e responde uma pergunta que ninguém do time respondeu.

---

## Depois desta

1. **Contar ao time o que se achou** — não o SDK, não o precedente: que a camada interna
   expõe condição de ordenação e ninguém sabia. É a informação mais útil que saiu do dia.
2. **Fazer a aplicação subir contra o LocalStack.** Ficou de lado pelo prazo, e é o que
   transformaria a Fase 1 inteira em algo verificável antes do deploy.
3. **Auditoria das regras dos 4 steps** — lista step a step para o TL validar, não conserto
   adivinhado.

Mapa das peças: [INVENTARIO.md](INVENTARIO.md) · Comandos de referência:
[dynamodb-queries](202608241731-dynamodb-queries.md)
