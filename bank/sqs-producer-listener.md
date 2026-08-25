# SQS — producer + listener (Kotlin): prompt mínimo e forma esperada

> Campos e nomes são **fictícios** (`OrderEvent`, `orders-queue`). Nenhuma regra de negócio real.
> O que vale aqui é a **forma**: falha por item não derruba o lote.

## Prompt rápido — quando a prioridade é código certo de primeira ⭐

**Dois turnos, nesta ordem.** O listener é onde mora a dificuldade; pedir os dois juntos
dilui a atenção do modelo. O esqueleto ancora a forma e os pontos frágeis viram requisito
explícito. O desenho é seu; o Copilot preenche — diga isso em voz alta em vez de fingir
que ele achou sozinho.

### Turno 1 — LISTENER

```
Kotlin + AWS SDK v2, SQS, sem framework de mensageria. Gere APENAS o Listener.
Complete o esqueleto abaixo, seguindo a forma exatamente como esta.

sealed class ParseResult {
    data class Parsed(val event: OrderEvent) : ParseResult()
    data class Invalid(val reason: String) : ParseResult()
}

fun Message.toOrderEvent(): ParseResult { /* valida; NUNCA lanca excecao */ }

// no laco do lote
for (msg in resp.messages()) {
    try {
        when (val r = msg.toOrderEvent()) {
            is ParseResult.Parsed  -> { handle(r.event); deleteMessage(msg) }
            is ParseResult.Invalid -> { sendToDlq(msg, r.reason); deleteMessage(msg) }
        }
    } catch (e: Exception) {
        log.error("falha na mensagem {}; lote continua", msg.messageId(), e)
    }
}

Laco de long polling com waitTimeSeconds=10 e maxNumberOfMessages=10.

Requisitos obrigatorios, nao negociaveis:
1. O try/catch fica DENTRO do for, envolvendo cada mensagem individualmente. Nunca em
   volta do lote inteiro: uma mensagem ruim nao pode interromper as demais do lote.
2. toOrderEvent nunca lanca excecao. A falha e valor de retorno (Invalid), e o `when`
   nao tem `else` -- o compilador deve exigir os dois ramos.
3. No ramo Invalid: sendMessage para a URL da DLQ PRIMEIRO, deleteMessage da fila
   principal DEPOIS. Nessa ordem. Invertido, uma falha no envio perde a mensagem.
4. No ramo Invalid, logue o corpo inteiro da mensagem, nao apenas o motivo.
5. deleteMessage e o acknowledgment: so acontece apos o processamento dar certo.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

### Turno 2 — PRODUCER (no mesmo chat, depois de conferir o listener)

```
Agora gere APENAS o Producer, no mesmo estilo do listener acima.

Metodo que recebe uma LISTA de objetos de dominio e publica com SendMessageBatch, em
lotes de ate 10 entradas, com message attribute "traceId" (String) em cada entrada e
ids de entrada unicos dentro do lote.

Requisito obrigatorio, nao negociavel:
SendMessageBatch retorna resultado POR ENTRADA e NAO lanca excecao quando entradas sao
rejeitadas. Leia response.failed() e trate as entradas que falharam (log com o id da
entrada + reenvio). Ignorar failed() faz ate 10 itens sumirem em silencio, com a
chamada retornando sucesso.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

### Turno 3 — CONSULTA PAGINADA (só se a Query também for gerada)

O producer do turno 2 **recebe** a lista pronta: a Query no índice está fora do escopo dele.
Se quiser que o Copilot gere também a consulta, é este turno. Nomes genéricos de propósito.

```
Agora gere APENAS o repositorio de consulta, no mesmo estilo.

DynamoDB SDK v2. Metodo que devolve todos os itens vencidos de um GSI esparso:
- o GSI tem partition key `pendingShard` (valores "PEND#0".."PEND#9") e sort key `dueAt`
  (ISO-8601 em String)
- para cada shard, Query com KeyConditionExpression de igualdade no pendingShard e
  condicao <= :now no dueAt
- sem FilterExpression
- Limit por pagina configuravel

Requisitos obrigatorios, nao negociaveis:
1. Pagine com exclusiveStartKey/lastEvaluatedKey e pare SOMENTE quando lastEvaluatedKey
   for null ou vazio. NUNCA pare porque a pagina veio sem itens: pagina vazia com
   lastEvaluatedKey preenchido significa que ha continuacao, e parar ali descarta o
   resto da fila em silencio.
2. Percorra os 10 shards, acumulando o resultado.
3. Nao use FilterExpression: os dois recortes ja estao na chave, entao o Limit vale.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

⚠️ **`Limit` é tamanho de página, não teto de trabalho.** Tratá-lo como "processo N por dia"
faz a fila nunca drenar. O `paginator` do SDK (`queryPaginator`) resolve o laço sozinho — se o
Copilot oferecer, é a opção mais segura, porque o erro do `items.isEmpty()` deixa de existir.

### Correções prontas, se escapar

Não vale reprompt do zero — uma frase resolve. São os dois erros mais comuns:

**Se o `try/catch` vier em volta do lote:**
```
O try/catch precisa estar dentro do for, envolvendo cada mensagem individualmente,
nao em volta do lote inteiro.
```

**Se a paginação parar no lugar errado:**
```
O laco de paginacao deve parar quando lastEvaluatedKey for null, nao quando a pagina
vier sem itens. Pagina vazia com lastEvaluatedKey preenchido significa continuacao.
```

**Se o producer ignorar o retorno do batch:**
```
Falta tratar response.failed(). SendMessageBatch retorna sucessos e rejeicoes na mesma
resposta e nao lanca excecao: sem ler failed(), entradas rejeitadas somem em silencio.
```

---

## Prompt mínimo — quando quem gera precisa desenhar junto

```
Kotlin + AWS SDK v2. Gere dois componentes para SQS, sem framework de mensageria:

1) Producer: classe com um método que recebe uma LISTA de objetos de domínio e publica
   com SendMessageBatch, em lotes de até 10 entradas, ids de entrada únicos no lote e
   message attribute "traceId" (String) em cada uma. Leia response.failed() e trate as
   entradas rejeitadas — a chamada retorna sucesso mesmo com entradas que falharam.
   Sem lógica de negócio.

2) Listener: laço de long polling (waitTimeSeconds=10, maxNumberOfMessages=10) que, para
   cada mensagem do lote:
   - converte a mensagem para o objeto de domínio numa função de borda que NÃO lança:
     ela retorna uma sealed class com dois casos, Parsed(evento) e Invalid(motivo);
   - consome esse retorno com um `when` SEM `else`;
   - Parsed: processa e só então deleteMessage (deleteMessage é o acknowledgment);
   - Invalid: sendMessage para a URL da DLQ e SÓ ENTÃO deleteMessage da fila principal,
     nessa ordem, logando o corpo inteiro da mensagem;
   - envolva o processamento de CADA mensagem em try/catch DENTRO do laço do lote, de
     modo que uma mensagem com problema não interrompa as demais do mesmo lote.

Sem comentários explicativos no código. Kotlin idiomático, sem !!.
```

## Os 5 pontos que a revisão precisa conferir na saída

Com o prompt mínimo eles falham quase sempre; com o rápido, confira mesmo assim:

1. **`try/catch` por mensagem, dentro do `for` do lote.** O padrão errado é um `try` em volta
   do lote inteiro: uma mensagem ruim aborta as restantes, que voltam por visibility timeout
   e derrubam o lote de novo na próxima entrega.
2. **Conversão fora do `try`.** Se a função de borda lançar, a exceção sobe e mata o lote.
   Por isso ela retorna `sealed`, não lança — a falha vira **valor**, e o `when` sem `else`
   faz o compilador exigir o tratamento (Kotlin ≥1.7: `when`-statement não-exaustivo sobre
   sealed é **erro**).
3. **Ordem no ramo inválido:** envia para a DLQ **primeiro**, deleta depois. Invertido, um
   erro no envio perde a mensagem. `deleteMessage` e "mandar para a DLQ" são operações
   separadas — não existe DLQ implícita.
4. **Log com o corpo inteiro** no ramo inválido. Logar só o motivo descarta a única cópia
   do payload que ia sobrar.
5. **`SendMessageBatch` ignorando `failed()`.** É a mesma falha-por-item do lado do produtor:
   a resposta vem com sucessos **e** rejeições por entrada, e a chamada não lança. Quem não lê
   `failed()` perde até 10 itens por lote, em silêncio, com log limpo.

## Forma esperada (esqueleto)

```kotlin
sealed class ParseResult {
    data class Parsed(val event: OrderEvent) : ParseResult()
    data class Invalid(val reason: String) : ParseResult()
}

fun Message.toOrderEvent(): ParseResult { /* valida; nunca lança */ }

// no laço do lote
for (msg in resp.messages()) {
    try {
        when (val r = msg.toOrderEvent()) {
            is ParseResult.Parsed  -> { handle(r.event); deleteMessage(msg) }
            is ParseResult.Invalid -> { sendToDlq(msg, r.reason); deleteMessage(msg) }
        }
    } catch (e: Exception) {
        log.error("falha na mensagem {}; lote continua", msg.messageId(), e)
    }
}
```

## Escolha do destino da mensagem inválida

| Falha | Reprocessar ajuda? | Destino |
|---|---|---|
| **Transitória** (timeout, throttle, dependência fora) | sim | **não deletar** — volta por visibility timeout; a redrive policy manda para a DLQ ao passar do `maxReceiveCount` |
| **Determinística** (não parseia, campo obrigatório ausente) | não, nunca | **envio explícito para a DLQ + delete** — esperar o `maxReceiveCount` gasta N reentregas garantidamente inúteis |

Descartar sem DLQ só se justifica com volume alto de lixo conhecido — e, mesmo aí, logando o
payload. A regra não é *nunca deletar*: é **nunca deletar sem preservar o payload em algum lugar**.

⚠️ **DLQ sem alarme de profundidade > 0 é delete com passos extras.**
