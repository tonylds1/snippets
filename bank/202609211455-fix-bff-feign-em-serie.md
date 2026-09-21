# Fix: as chamadas do BFF estão rodando em série

```
Os adapters chamam o Feign (bloqueante) direto dentro de suspend fun. Com o dispatcher do
Spring MVC, os async da fase 2 executam um por vez.

Em CADA adapter de adapters/out/http:
1. envolva o corpo do metodo em withContext(Dispatchers.IO)
2. apos a chamada, adicione:
   log.info("<servico> ok ms={} thread={}", ms, Thread.currentThread().name)

Nao altere use case, dominio nem controller. Mostre so as linhas que mudam.
```

## Como conferir em hml

Olhe as linhas `ok` de um mesmo request:

- **threads diferentes** (`DefaultDispatcher-worker-N`) e tempo total ≈ o **maior** `ms` → paralelo ✅
- **mesma thread** (`http-nio-...`) e tempo total ≈ a **soma** dos `ms` → ainda em série ❌
