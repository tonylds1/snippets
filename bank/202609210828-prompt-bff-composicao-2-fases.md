# Prompt — BFF de composição em 2 fases (1 obrigatória + N em paralelo degradáveis)

> Gera um endpoint GET que agrega APIs externas: **fase 1** busca a elegibilidade (não pode
> falhar); **fase 2** usa o retorno dela para disparar as demais **em paralelo**, e cada uma
> que falhar vira objeto **vazio** na resposta.
> As APIs externas ainda não existem: o prompt gera a estrutura inteira + adapters **fake**,
> deixando o projeto no ponto de só plugar o client real.
>
> Nomes reais nunca entram aqui: tudo entre `<>` é preenchido na hora de colar.
>
> Pré-requisito: o preâmbulo de [contenção](202608261829-copilot-contencao.md) colado antes.

## Preencher antes de colar

Substitua **todas** as ocorrências de cada um (o mesmo placeholder aparece várias vezes).

```
<ARQUIVO_REF>       = caminho completo do arquivo que tem o endpoint de referencia
<ROTA_REF>          = rota do endpoint de referencia (ex.: /proposals)
<CONTROLLER>        = caminho do controller onde entra o metodo novo
<ROTA>              = rota do endpoint novo
<Nome>              = nome base do agregado/use case/response, em PascalCase
<PARAM>             = nome do path param de entrada (String)

<PRODUTO_A>         = sigla do primeiro produto  (vira valor do enum Produto)
<PRODUTO_B>         = sigla do segundo produto   (vira valor do enum Produto)

<SVC_ELEG>          = nome do service de elegibilidade no application.yml
<SVC_LIMITE_A>      = nome do service de limite do produto A
<SVC_LIMITE_B>      = nome do service de limite do produto B
<SVC_FATURA>        = nome do service de fatura (A e B no mesmo service, param produto)

<CAMPOS_RESPONSE>   = campos da resposta final: nome: Tipo, um por linha
<CAMPOS_ELEG>       = campos que a elegibilidade devolve e que a fase 2 usa como parametro
```

### Se a fatura A e a fatura B forem services diferentes

Troque a linha `FaturaPort` da fase 2 por duas portas, e `<SVC_FATURA>` por dois services:

```
- Fatura<PRODUTO_A>Port -> service <SVC_FATURA_A>
- Fatura<PRODUTO_B>Port -> service <SVC_FATURA_B>
```

## O prompt

```
Kotlin + Spring Boot + OpenFeign, hexagonal. BFF de composicao.
Siga os padroes existentes em <ARQUIVO_REF> (endpoint <ROTA_REF>). Nao invente estrutura nova.

TAREFA: criar GET <ROTA>, que agrega APIs externas em 2 fases e devolve um objeto unico.
As APIs ainda NAO existem: gere a estrutura completa + adapters FAKE com dados fixos.

Entrada: <PARAM>: String (path param, sem RequestObject)
Saida: <Nome>Response {
<CAMPOS_RESPONSE>
}

FASE 1 (sequencial, OBRIGATORIA):
- ElegibilidadePort -> service <SVC_ELEG>
- devolve: <CAMPOS_ELEG>
- se falhar, o endpoint falha (excecao sobe, traduzida no adapter)
- o retorno dela fornece os parametros das chamadas da fase 2

FASE 2 (paralela, todas DEGRADAVEIS, usando dados da fase 1):
- Limite<PRODUTO_A>Port -> service <SVC_LIMITE_A>
- Limite<PRODUTO_B>Port -> service <SVC_LIMITE_B>
- FaturaPort            -> service <SVC_FATURA>, param produto: Produto(<PRODUTO_A>|<PRODUTO_B>)  [2 chamadas]

CRIAR:
1. application/port/out/*.kt       interfaces, suspend fun, so tipos de dominio
2. application/domain/<Nome>.kt
   agregado COM comportamento: monta o resultado a partir da elegibilidade + partes obtidas
   cada tipo de dominio expoe uma instancia VAZIA (companion object VAZIO)
3. application/<Nome>UseCase.kt
   fase 1 sequencial; fase 2 em coroutineScope com async; delega a decisao ao dominio
4. adapters/out/http/<X>Client.kt  @FeignClient + Request/ResponseObject em http/entity + Mapper -> dominio
5. adapters/out/http/fake/<X>FakeAdapter.kt
   implementa a porta com dados fixos realistas, @Profile("local")
6. <CONTROLLER>
   novo metodo + <Nome>Response em entity + mapper dominio -> Response

PARALELISMO E FALHA:
- fase 2 dispara junta: coroutineScope + async { catching { } }
- TODOS os async sao criados ANTES do primeiro await
- catching captura Exception e RELANCA CancellationException (nao usar runCatching)
- falha em uma NAO cancela as outras
- cada falha vira o objeto VAZIO do respectivo tipo + log de warn (nunca null na resposta)
- fase 1 NAO usa catching

REGRAS:
- domain/ e port/ nao importam Spring, Feign, Jackson nem java.io.Serializable
- regra de negocio no dominio; use case so orquestra (nenhum if de negocio no use case)
- excecao do Feign e traduzida no adapter
- application.yml: adicionar cloud.openfeign.client.config.<service>.url para cada service

Nao gere testes nesta etapa.
```

## Critério de aceite — confira o use case que ele devolver

Se a forma vier diferente disto, ele errou:

```kotlin
suspend fun execute(param: String): Agregado {
    val eleg = elegibilidadePort.buscar(param)            // fase 1: fora do scope, sem catching

    return coroutineScope {                               // fase 2: fan-out
        val limiteA = async { catching { limiteAPort.buscar(eleg.contaA) } }
        val limiteB = async { catching { limiteBPort.buscar(eleg.contaB) } }
        val faturaA = async { catching { faturaPort.buscar(Produto.A, eleg.contaA) } }
        val faturaB = async { catching { faturaPort.buscar(Produto.B, eleg.contaB) } }

        Agregado.montar(                                  // decisao no dominio
            elegibilidade = eleg,
            limiteA = limiteA.await().ouVazio(Limite.VAZIO, "limiteA"),
            limiteB = limiteB.await().ouVazio(Limite.VAZIO, "limiteB"),
            faturaA = faturaA.await().ouVazio(Fatura.VAZIA, "faturaA"),
            faturaB = faturaB.await().ouVazio(Fatura.VAZIA, "faturaB"),
        )
    }
}
```

Três coisas para olhar:

1. `elegibilidadePort` **fora** do `coroutineScope`.
2. Os 4 `async` **antes** de qualquer `await`. Se ele puser `await()` logo depois de cada
   `async`, serializou tudo — funciona, passa despercebido e perde o paralelismo.
3. Nenhum `if` de negócio no use case. A composição mora no `montar()` do agregado.

### O `catching` que ele deve gerar

```kotlin
suspend fun <T> catching(block: suspend () -> T): Result<T> =
    try {
        Result.success(block())
    } catch (e: CancellationException) {
        throw e                       // engolir isto faz a coroutine ignorar o proprio cancelamento
    } catch (e: Exception) {
        Result.failure(e)
    }
```

Se vier `runCatching { }` no lugar: está errado. Ele captura `Throwable`, inclusive o
cancelamento — timeout que não corta, request cancelado que continua rodando.

## Correções prontas

**Serializou os awaits:**
```
Os async da fase 2 estao sendo aguardados um a um. Crie os 4 async primeiro e so depois
faca os await, para que as 4 chamadas rodem em paralelo. Mostre so o use case.
```

**Usou runCatching:**
```
Troque runCatching pela funcao catching que relanca CancellationException. runCatching
engole o cancelamento da coroutine. Mostre so a funcao e as linhas que a usam.
```

**Pôs regra de negócio no use case:**
```
Mova todo if/when de decisao do use case para o metodo montar() do agregado de dominio.
O use case so dispara as chamadas e entrega os resultados. Mostre os dois arquivos.
```

**Domínio importou framework / Serializable:**
```
Remova de domain/ e port/ qualquer import de Spring, Feign, Jackson ou java.io.Serializable.
Se algum tipo precisa ser serializado, crie o DTO correspondente em adapters/.
```
