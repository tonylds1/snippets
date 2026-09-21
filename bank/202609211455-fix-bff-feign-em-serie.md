# Fix: as chamadas do BFF estão rodando em série

> **v2.** A v1 mandava pôr `withContext` no adapter. Quando a porta **não** é `suspend`, o
> Copilot embrulha em `runBlocking` para compilar — e aí continua em série. O lugar certo é o
> `async` do use case: nenhuma assinatura muda, outros chamadores não são afetados.

```
Desfaca a mudanca nos adapters: remova runBlocking e withContext, volte assinatura e corpo
ao original (fun normal, chamada Feign direta). Mantenha o log.info com thread.

No use case, fase 2, passe o dispatcher para cada async:
  async(Dispatchers.IO + MDCContext()) { catching { ... } }

Nao altere dominio, controller nem assinatura de porta. Mostre so as linhas que mudam.
```

Se `MDCContext` não compilar: falta a dependência `org.jetbrains.kotlinx:kotlinx-coroutines-slf4j`.

## Por que não `runBlocking`

`runBlocking` segura a thread que chamou até o bloco terminar. Dentro do `async`, isso é a
thread do request esperando parada: as 4 chamadas saem uma por vez. E sob carga, `runBlocking`
dentro de coroutine pode esgotar as threads e travar o serviço.

## Como conferir em hml

Olhe as linhas `ok` de um mesmo request:

- **threads diferentes** (`DefaultDispatcher-worker-N`) e tempo total ≈ o **maior** `ms` → paralelo ✅
- **mesma thread** (`http-nio-...`) e tempo total ≈ a **soma** dos `ms` → ainda em série ❌
- traceId presente nas 4 linhas → `MDCContext` funcionando ✅
