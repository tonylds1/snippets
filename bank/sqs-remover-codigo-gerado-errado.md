# Remover código gerado que não pertence ao projeto

> Situação: uma geração anterior criou um **poller autônomo** (`while`, `receiveMessage`,
> `startPolling`/`stop`) num projeto onde o polling é do **framework**. O arquivo não tem
> conserto — tem remoção. Adaptá-lo é dar arquitetura a algo que não deveria existir.

## ⚠️ Por que o Copilot não removeu sozinho

Um prompt que manda **conformar-se ao código existente** protege esse arquivo: ele está no
disco, logo parece código do projeto. Entulho de geração anterior precisa ser apontado como
entulho, explicitamente.

---

## Passo 1 — o terminal decide, em 10 segundos

Não pergunte ao modelo o que o compilador responde de graça.

```bash
# 1. onde a classe aparece
grep -rn "SqsListener" --include=*.kt --include=*.kts --include=*.yml \
     --include=*.yaml --include=*.properties src/

# 2. ela é um bean? (anotação faz o Spring instanciar mesmo sem ninguém chamar)
grep -n "@" src/main/kotlin/**/SqsListener.kt
```

**Leitura do resultado:**

| O que apareceu | Significa |
|---|---|
| Só linhas do próprio `SqsListener.kt` | ninguém usa — **pode apagar** |
| `@Component`, `@Service`, `@Bean`, `@PostConstruct`, `ApplicationRunner` | o Spring instancia. Apagar é o certo, mas confirme que não há **dois consumidores** na mesma fila |
| Import ou injeção em outra classe | 🔴 **pare** — alguém ligou o poller. Dois consumidores concorrendo na mesma fila |

## Passo 2 — apagar e deixar o compilador provar

```bash
rm src/main/kotlin/<caminho>/SqsListener.kt
./gradlew compileKotlin
```

**O compilador é o árbitro, não o grep e não o modelo.** Compilou = nada referenciava a
classe, remoção segura. Falhou = a mensagem de erro aponta exatamente quem dependia dela.

## Passo 3 — só então o Copilot, para o que sobrou

O trabalho de verdade não é apagar: é confirmar que as mudanças caíram na classe que **de
fato** recebe da fila, e não no arquivo que você acabou de deletar.

```
Contexto: o arquivo SqsListener foi removido do projeto. Ele era um poller autonomo gerado
por engano; aqui o polling e do framework (o Main so tem SpringApplication.run).

Confirme que as mudancas estao aplicadas na classe que REALMENTE recebe mensagens da fila
(a que o framework aciona). Para cada item, responda com arquivo e linha, ou "NAO
IMPLEMENTADO":

a) a conversao da mensagem para o objeto de dominio acontece numa funcao de borda que NAO
   lanca excecao e retorna uma sealed class (valido / invalido com motivo);
b) existe um `when` SEM `else` consumindo essa sealed class, dentro da classe que recebe
   da fila;
c) o ramo invalido envia para a DLQ e loga o CORPO INTEIRO da mensagem, nao so o motivo;
d) o publisher usa SendMessageBatch (max 10 entradas, ids unicos no lote) e le
   response.failed(), tratando as entradas rejeitadas.

Se algum item estiver NAO IMPLEMENTADO, implemente seguindo o padrao do projeto (nomes,
logs, DI, tratamento de erro), sem criar pacote novo, sem poller, sem while, sem
receiveMessage, sem thread e sem metodos start/stop.

Ao final, liste o que ficou fora e por que.
```

---

## Divisão de trabalho — a regra geral

| Pergunta | Quem responde | Por quê |
|---|---|---|
| "alguém usa esta classe?" | **compilador** | resposta determinística, 10 s, sem alucinação |
| "esta classe está em conformidade com o projeto?" | **você**, lendo | é julgamento |
| "escreva o código conforme o padrão" | **modelo** | é o que ele faz bem |

⚠️ **Perguntar ao modelo o que uma ferramenta determinística responde é o desperdício mais
comum** — e o mais caro, porque a resposta vem convincente mesmo quando errada.
