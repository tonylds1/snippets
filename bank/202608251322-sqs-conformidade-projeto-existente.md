# Encaixar código gerado num projeto que já existe — prompt em 3 etapas

> Use quando o projeto **já tem** consumidor de fila e o código gerado precisa se encaixar,
> em vez de trazer a própria infraestrutura junto.
>
> **O erro que isso evita:** modelo sem instrução de inspeção gera o padrão mais comum da
> internet — um **poller autônomo** com `while`, `receiveMessage`, thread e `start/stop`.
> Num projeto Spring Boot onde o polling é do framework, esse código não encaixa: ninguém
> chama os métodos, e você só descobre ao colar.

## Sinal de que o polling NÃO é seu

`Main`/`Application` só tem `SpringApplication.run(...)` e existe uma classe que recebe da fila.
Ninguém sobe thread → **quem faz o polling é o container**, e o código gerado não pode trazer o seu.

## O prompt

No Copilot Chat, comece com `@workspace` — sem isso ele só enxerga os arquivos abertos.

```
Contexto: projeto Kotlin + Spring Boot que ja consome de uma fila SQS. Existe uma classe
que recebe as mensagens da fila e delega para um use case, que orquestra steps. Existe um
pacote de publisher. O Main so tem SpringApplication.run — nenhuma thread e iniciada
manualmente, entao o polling e do framework, nao da aplicacao.

NAO gere codigo ainda. Trabalhe em 3 etapas e pare ao fim de cada uma.

=== ETAPA 1 — RELATORIO DE CONFORMIDADE (sem codigo) ===
Inspecione o projeto e me responda, citando arquivo e linha de cada resposta:
1. Qual classe recebe mensagens da fila e como ela e acionada (anotacao, interface,
   configuracao)? Qual biblioteca faz isso?
2. Quem faz o acknowledgment: o framework (retornar sem excecao confirma, lancar devolve
   a mensagem) ou existe deleteMessage explicito no codigo?
3. O consumo e por mensagem ou por lote? Se por lote, quem itera?
4. Existe DLQ? Ela e alcancada por redrive policy na infraestrutura ou por envio
   explicito no codigo?
5. Como o projeto publica mensagens hoje (classe, pacote, estilo)?
6. Qual o padrao de log (biblioteca, structured logging, MDC/correlation id)?
7. Qual o padrao de tratamento de erro e quais excecoes de dominio existem?
8. Injecao de dependencia: construtor, campo, configuracao?
9. Framework de teste e estilo de mock usados.
Se nao encontrar algo, diga "nao encontrado" — nao invente e nao assuma o padrao comum.

=== ETAPA 2 — PLANO (sem codigo) ===
Com base na etapa 1, liste as mudancas arquivo por arquivo, dizendo em cada uma se cria,
altera ou remove, e por que. Objetivo das mudancas:
a) Uma mensagem que nao converte para o objeto de dominio NAO pode derrubar as demais.
   A conversao deve acontecer numa funcao de borda que NAO lanca: retorna uma sealed
   class com dois casos (valido / invalido com motivo), consumida por um `when` sem
   `else`, para o compilador exigir os dois ramos.
b) O caso invalido vai para a DLQ preservando o corpo inteiro da mensagem no log —
   nunca descartado silenciosamente, nunca apenas com o motivo.
c) Se o consumo for por lote, o isolamento de falha e por item dentro do laco do lote.
   Se for por mensagem, respeite o mecanismo de ack do framework: NAO introduza
   deleteMessage manual onde o framework ja confirma.
d) Um publisher que envia em lote com SendMessageBatch (maximo 10 entradas por chamada,
   ids de entrada unicos no lote) e que OBRIGATORIAMENTE le response.failed(): a chamada
   retorna sucesso mesmo com entradas rejeitadas, e ignorar failed() faz itens sumirem
   em silencio.

Restricoes obrigatorias:
- NAO crie poller, laco while, receiveMessage, thread, ExecutorService nem metodos
  start/stop. O polling e do framework.
- NAO crie pacote novo: use os pacotes que ja existem, seguindo a convencao encontrada
  na etapa 1.
- NAO altere assinatura publica de classe existente sem apontar quem a chama.
- Siga o estilo do projeto (nomes, logs, erros, DI) em vez do estilo generico.

=== ETAPA 3 — CODIGO ===
So depois que eu aprovar o plano. Kotlin idiomatico, sem !!, sem comentarios explicativos.
Ao final, liste o que voce NAO conseguiu conformar ao projeto e por que.
```

## Como rodar (não erre aqui)

**Cole o prompt UMA VEZ, inteiro.** As 3 etapas já estão nele. Você **não** cola resultado de
volta — o chat mantém o histórico, o modelo já sabe o que respondeu.

1. **Abra no editor** os arquivos que importam (classe que recebe da fila, use case, publisher).
   Copilot usa arquivos abertos como contexto.
2. **Chat novo.** `@workspace` + o prompt inteiro, numa mensagem só.
3. Ele responde a **etapa 1** e para.
4. **Confira** (ver abaixo).
5. `Etapa 1 conferida. Siga para a etapa 2.`
6. Leia o plano. Erro? Corrija em **uma frase** — não recomece.
7. `Plano aprovado. Siga para a etapa 3.`
8. Confira o código contra os pontos de revisão.

⚠️ **Tudo no mesmo chat.** Chat novo entre etapas = contexto perdido.

### Conferir a etapa 1 — é aqui que se ganha ou perde

Não leia como texto: **teste duas respostas**, abrindo os arquivos citados. Priorize a
**pergunta 2** (quem faz o acknowledgment) — ela muda mais código que todas as outras.

Resposta sem arquivo e linha, ou com "provavelmente"/"normalmente"/"em geral", é chute:

```
As respostas 3 e 6 nao citam arquivo e linha. Releia o projeto e responda
apenas com o que voce encontrou, ou diga "nao encontrado".
```

### Falha mais provável: ele ignora o "pare"

Despeja relatório + plano + código de uma vez. **Não aproveite o código:**

```
Voce pulou as etapas. Descarte o codigo por enquanto e responda APENAS a etapa 1,
citando arquivo e linha em cada resposta.
```

Aceitar o despejo abre mão do que o prompt existe para comprar: o erro aparecer em texto,
onde custa uma frase, em vez de em código.

### Se ele se perder (contexto longo)

Sinal: volta a sugerir `while`, `receiveMessage` ou `start/stop`. Re-ancore em uma linha:

```
Lembrete das restricoes: sem poller, sem while, sem receiveMessage, sem thread,
sem metodos start/stop. O polling e do framework. Sem pacote novo.
```

## Por que 3 etapas

| Etapa | O que ela compra |
|---|---|
| **1 — conformidade** | impede o modelo de gerar o padrão da internet em vez do padrão do projeto |
| **2 — plano** | o erro aparece em texto, onde custa uma frase corrigir, não em código |
| **3 — código** | só depois de você concordar com o desenho |

**Exigir arquivo e linha** em cada resposta da etapa 1 é o detalhe que faz funcionar: sem isso o
modelo descreve o que "provavelmente" existe. Com isso, ou ele leu, ou aparece "não encontrado" —
que também é resposta útil.

## ⚠️ A pergunta 2 é a que mais muda código

Se o framework confirma por retorno (padrão do Spring Cloud AWS: **retornar = ack, lançar =
devolve à fila**), então `deleteMessage` manual não é só desnecessário — **quebra o contrato**.
Você deleta, o framework tenta deletar de novo, e o tratamento de erro dele deixa de devolver
qualquer coisa para a fila.

Nesse modo, "delete é o acknowledgment" vira "**retornar** é o acknowledgment", e o try/catch por
item muda de lugar: o laço do lote é do framework, não seu.
