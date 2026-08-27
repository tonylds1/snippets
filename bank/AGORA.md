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
- ✅ **Índice `KEYS_ONLY` + entidade com campo obrigatório** — era risco de `NullPointerException`
  na desserialização. **Resolvido por desenho:** a consulta usa uma classe de três campos, não
  a entidade completa. Ver F1.1.

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
> São **dois artefatos novos** (uma classe de chaves, um producer) e **um método** somado ao
> repositório que já existe. Nada além disso.
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

## F1.1 A classe de chaves — o que faz o `KEYS_ONLY` não estourar

O índice é `KEYS_ONLY`: a consulta devolve **três campos** — a chave da tabela e os dois
atributos do índice. Mais nada.

⚠️ **Não passe o schema da entidade completa.** Ela tem campos obrigatórios (sem `?`), e o
mapper vai tentar preencher um objeto onde 90% dos atributos não vieram. Campo não-anulável
recebendo nulo = `NullPointerException` na desserialização, em homologação, depois do deploy.

✅ **Crie uma classe pequena com exatamente os três campos** e passe o schema **dela** para a
consulta. O parâmetro `schema` do método existe para isso.

O que você ganha:

| | |
|---|---|
| o risco **some**, em vez de ser testado | não há campo obrigatório para vir nulo |
| a assinatura passa a dizer a verdade | *isto devolve chaves, não entidades de negócio* |
| o teste unitário fica trivial | três campos, sem construir entidade completa |

⛔ **O que NÃO fazer:** trocar a projeção para `INCLUDE` (paga armazenamento e derruba o motivo
do índice ser esparso) nem tornar campos da entidade anuláveis (estraga a entidade por causa
de um caso de borda).

### A classe, pronta para copiar

**Arquivo novo, no mesmo pacote da entidade grande.** Não coloque dentro do arquivo dela: são
conceitos diferentes, e um arquivo por classe é a convenção do Kotlin.

```kotlin
package <o mesmo pacote da entidade>

import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbBean
import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbPartitionKey
import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbSecondaryPartitionKey
import software.amazon.awssdk.enhanced.dynamodb.mapper.annotations.DynamoDbSecondarySortKey

@DynamoDbBean
data class <ClasseDeChaves>(

    @get:DynamoDbPartitionKey
    var <PK_TABELA>: String = "",

    @get:DynamoDbSecondaryPartitionKey(indexNames = ["<INDICE>"])
    var <PK_INDEX>: String = "",

    @get:DynamoDbSecondarySortKey(indexNames = ["<INDICE>"])
    var <SK_INDEX>: String = "",
)
```

### Por que cada pedaço está aí

| Pedaço | Por quê |
|---|---|
| `@DynamoDbBean` | marca a classe como mapeável. Sem isso o schema não é construído |
| `var` e `= ""` em **todos** os campos | o mapper precisa de construtor sem argumentos e de setters. O Kotlin só gera o construtor vazio quando **todo** parâmetro tem valor padrão — se faltar um, quebra em tempo de execução |
| `@get:` antes da anotação | 🎯 **a pegadinha clássica.** O mapper lê a anotação **no getter**. Sem o `@get:`, o Kotlin põe no campo, o mapper não enxerga e a chave "não existe" |
| `@DynamoDbPartitionKey` no `<PK_TABELA>` | o GSI `KEYS_ONLY` devolve a chave da **tabela** junto com as do índice — são os três campos, e é só isso que vem |
| `indexNames = ["<INDICE>"]` | é o que amarra esses dois campos ao seu índice, e não a outro |
| `data class` | dá `equals` de graça, e é o que deixa o teste unitário comparar objetos direto |

### O `companion object` com o schema — a peça que liga tudo

A entidade grande tem um `companion object` com um `TABLE_SCHEMA`. 🎯 **Ele é exatamente o
argumento `schema` que a consulta recebe.** A classe de chaves precisa do dela.

E ele te diz em qual dos dois mundos você está. **Abra o da entidade grande e veja qual é:**

**Mundo A — `TableSchema.fromBean(...)`**  → as anotações são o que vale. Copie assim:

```kotlin
companion object {
    val TABLE_SCHEMA: TableSchema<<ClasseDeChaves>> =
        TableSchema.fromBean(<ClasseDeChaves>::class.java)
}
```

**Mundo B — `StaticTableSchema.builder(...)`** → 🔴 **as anotações são ignoradas.** Apague-as
da sua classe e declare os três atributos à mão:

```kotlin
companion object {
    val TABLE_SCHEMA: TableSchema<<ClasseDeChaves>> =
        StaticTableSchema.builder(<ClasseDeChaves>::class.java)
            .newItemSupplier { <ClasseDeChaves>() }
            .addAttribute(String::class.java) { a ->
                a.name("<PK_TABELA>")
                    .getter { it.<PK_TABELA> }
                    .setter { obj, v -> obj.<PK_TABELA> = v }
                    .tags(StaticAttributeTags.primaryPartitionKey())
            }
            .addAttribute(String::class.java) { a ->
                a.name("<PK_INDEX>")
                    .getter { it.<PK_INDEX> }
                    .setter { obj, v -> obj.<PK_INDEX> = v }
                    .tags(StaticAttributeTags.secondaryPartitionKey("<INDICE>"))
            }
            .addAttribute(String::class.java) { a ->
                a.name("<SK_INDEX>")
                    .getter { it.<SK_INDEX> }
                    .setter { obj, v -> obj.<SK_INDEX> = v }
                    .tags(StaticAttributeTags.secondarySortKey("<INDICE>"))
            }
            .build()
}
```

⚠️ **No mundo B, o `.name(...)` é o nome do atributo NO BANCO**, não o nome em Kotlin. Copie
da entidade grande, que já acertou.

**Não misture os dois mundos.** Faça o que a entidade grande faz, e só isso.

### ⚠️ Três coisas para conferir na entidade grande antes de colar

### ⚠️ Três coisas para conferir na entidade grande antes de colar

**Copie a forma que já funciona lá — não invente.** A entidade grande já roda em produção;
o que ela faz está certo para este projeto.

1. **Ela usa `@get:` ou não?** Se ela escreve `@DynamoDbPartitionKey` sem prefixo e funciona,
   escreva igual. Faça o mesmo que ela — nunca o contrário.
2. **Ela tem `@DynamoDbAttribute("outro_nome")` em algum desses três campos?** Isso acontece
   quando o nome no banco é diferente do nome em Kotlin. 🔴 **Se tiver, replique idêntico** —
   sem isso o mapper procura um atributo que não existe e o campo volta vazio, sem erro.
3. **A entidade usa anotações mesmo, ou um `StaticTableSchema` montado à mão?** Se for
   schema montado, espelhe esse estilo em vez das anotações.

### Como saber que acertou

Se a consulta voltar objetos com `<PK_TABELA>` preenchido, acertou. Se voltar objetos com o
campo vazio (`""`) e **sem lançar exceção**, é o item 2 acima: nome de atributo divergente.

## F1.2 Onde o método entra — no repositório que já existe

Toda consulta a essa tabela já passa pelo repositório do projeto (uma interface e uma
implementação). **O método novo entra ali.** Não crie interface separada: seria um padrão novo
para a mesma tabela, e você está em integração recente — não é hora de estrear convenção.

```kotlin
// na interface que já existe
fun buscarVencidos(agora: Instant): Flux<<ClasseDeChaves>>
```

⚠️ **A implementação é arquivo grande.** Peça ao Copilot **só o método**, no chat, e cole você
mesmo. Mandar o agente editar arquivo grande é o caso clássico de ele reescrever tudo, errar
um delimitador e travar vinte minutos sem escrever nada.

## F1.3 A implementação da consulta (④) — pronta, sem gastar Copilot

Com a classe de chaves e as assinaturas já no lugar, **falta só o corpo.** Troque os `<...>`
pelos seus nomes e cole.

### 1. Duas constantes no `companion object` que já existe

Onde já estão a PK da tabela e os outros índices:

```kotlin
private const val PISO = "0000-01-01T00:00:00Z"    // const: String literal
private val SHARDS = (0..9).map { it.toString() }  // val: é List, não aceita const
```

`const val` só aceita valor conhecido em tempo de compilação — String e primitivos.
**O nome do índice você já tem** no object de nomes de GSI do projeto: use de lá, não redeclare.

### 2. O método

```kotlin
override fun <NomeDoMetodo>(agora: Instant): Flux<<ClasseDeChaves>> {
    val ate = agora.truncatedTo(ChronoUnit.SECONDS).toString()
    val total = AtomicInteger()

    return Flux.fromIterable(SHARDS)
        .flatMap { shard ->
            val doShard = AtomicInteger()

            <servico>.<metodoDeQueryPorGsi>(
                tableName = <TABELA>,
                schema = <ClasseDeChaves>.TABLE_SCHEMA,
                indexName = <INDICE>,
                gsiPkValue = shard,
                gsiPkIsNumber = false,
                options = <ClasseDeOpcoes>(
                    sortCondition = <ClasseDeCondicao>.Between(PISO, ate),
                ),
            )
                .doOnNext { doShard.incrementAndGet() }
                .doOnComplete { log.info("shard {} -> {} itens", shard, doShard.get()) }
                .onErrorResume { erro ->
                    log.error("shard {} falhou, seguindo com os demais", shard, erro)
                    Flux.empty()
                }
        }
        .doOnNext { total.incrementAndGet() }
        .doOnComplete { log.info("indice consultado: {} itens vencidos", total.get()) }
}
```

Imports novos: `java.time.Instant`, `java.time.temporal.ChronoUnit`,
`java.util.concurrent.atomic.AtomicInteger`, `reactor.core.publisher.Flux`.

### Lendo de fora para dentro

`Flux.fromIterable(SHARDS)` emite dez Strings, `"0"` a `"9"`. O `flatMap` chama a lambda uma vez
para cada, **em paralelo**, e cola os dez `Flux` resultantes num só. Quem consome o retorno
recebe os itens de todos os shards misturados, sem saber que houve dez consultas.

| Trecho | Requisito que ele cumpre |
|---|---|
| `onErrorResume` **dentro** da lambda | 🎯 falha num shard não derruba os outros nove. Depois do `flatMap`, o primeiro erro cancelaria tudo |
| `Between(PISO, ate)` | o recorte de data está **na chave** — sem `FilterExpression`, sem `Scan` |
| `AtomicInteger` **dentro** do método | contador por execução. No `companion object` viraria estado compartilhado entre chamadas |
| `log` por shard e total | sem isso o teste em homologação não é verificável |

### ⚠️ A comparação é de texto, não de data

`<SK_INDEX>` é String, então o `Between` compara **caractere a caractere**. E `Instant.toString()`
inclui os nanos quando existem:

```
gravado:   2026-08-27T06:00:00Z
consulta:  2026-08-27T06:00:00.123456Z
```

O `.` vem antes do `Z` na tabela ASCII — o valor mais preciso ordena **antes**, e o item some do
resultado **sem erro nenhum**.

👉 **Formate o `agora` no mesmo formato em que a data é gravada.** `truncatedTo(SECONDS)` serve se
a escrita grava com precisão de segundos. **Confira o código de escrita**; se ele usa um
formatador próprio, use o mesmo aqui.

### 📌 O shard não se configura

Não há nada a criar na AWS, no Terraform ou na biblioteca. O shard é **um atributo comum do item**,
com valor `"0"` a `"9"` que alguém grava. O índice tem esse atributo como chave de partição — dez
valores distintos, dez partições. É por isso que a consulta percorre os dez: o conjunto completo
está espalhado entre eles.

⚠️ **Enquanto ninguém gravar esse atributo, o índice fica vazio e a consulta devolve zero.**
Isso é o esperado — a escrita é um passo posterior, e é ela que coloca o item no índice.

### Ajuste antes de colar

- **A ordem e os nomes dos parâmetros** vêm do seu `api-interna.txt` — confira contra a assinatura real.
- **`gsiPkIsNumber = false`** porque o shard é String. Se gravar o shard como número, é `true`.
- **`log`** — use o logger no estilo do resto do projeto.

## F1.4 Prompt — os testes da consulta

```
[COLE AQUI O BLOCO DA API]

Agora gere APENAS os testes unitarios desse metodo, no mesmo estilo dos testes que ja
existem no projeto. Mocke o servico da biblioteca interna descrito no bloco.

Casos obrigatorios, um teste cada:
1. Sao feitas exatamente 10 chamadas, uma por shard, com os valores "0" a "9".
2. Toda chamada usa o indice <INDICE> e condicao de ordenacao Between, com o limite
   superior igual ao instante recebido por parametro. Capture o argumento e verifique.
3. Os resultados dos 10 shards aparecem todos no Flux devolvido, sem perder nem duplicar.
4. Quando um shard lanca excecao, os outros nove continuam e o resultado traz os itens deles.
5. Nenhum item vencido em nenhum shard devolve Flux vazio, sem excecao.

Nao teste a biblioteca interna. Teste o comportamento do SEU metodo.
```

## F1.5 Prompt — o producer (⑥)

⚠️ Onde a receita disser "recebe a lista de `<PK_TABELA>`", ajuste: ele recebe o **Flux da
classe de chaves** da F1.1 e tira o `<PK_TABELA>` de cada uma. O corpo da mensagem continua
tendo **um campo só**.

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
