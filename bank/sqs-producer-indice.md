# Producer que lê de um índice esparso e publica na fila — receita em 3 etapas

> Variante enxuta de [sqs-producer-listener.md](sqs-producer-listener.md), para o caso em que
> **o consumidor já existe** e você só precisa da peça que alimenta a fila.
> Nomes de tabela, campos e classes são **fictícios**, sempre — preencher antes de colar.

## 📍 Onde você está

Ver [INVENTARIO.md](INVENTARIO.md) para as 9 peças, a ordem de construção e as duas regras
que atravessam o fluxo inteiro.

**Esta receita entrega ④ e ⑥.** Pré-requisito: ⑤ (o índice) criado e vazio.
Próxima peça depois desta: ⑧, em [sqs-handler-mensagem-magra.md](sqs-handler-mensagem-magra.md).

## O fluxo

```
① Agendador (1x/dia)
        v  publica mensagem vazia
② Fila de gatilho
        v
③ TriggerHandler ──chama──▶ ④ PRODUCER  ◀── esta receita
                                  v consulta
                            ⑤ ÍNDICE ESPARSO
                                  v devolve as chaves dos vencidos
                            ⑥ SendMessageBatch  ◀── esta receita
                                  v
⑦ Fila
        v
⑧ Consumidor: enriquece o payload e executa
        v
⑨ Fim: avança a data do próximo ciclo
   OU remove os atributos → o item sai do índice
```

**Esta receita entrega o ④ e o ⑥.** Uma entrada (o índice) e uma saída (a fila).

**Não precisa existir ainda:** ① ② ③ e ⑨. Para testar, chame o producer na mão —
o agendador é a última peça, não a primeira.

## 🎯 A regra que define o tamanho da mensagem

> **O producer decide QUEM. O consumidor decide O QUÊ.**

A mensagem carrega **só a identidade**. Nenhum campo de negócio, nenhuma chamada a outro
serviço, nenhuma leitura extra. O enriquecimento acontece do lado do consumidor.

**Por quê:** do lado do consumidor você ganha de graça o que teria de escrever à mão no
producer — **retry** (mensagem falha, SQS reentrega, depois vai para a DLQ), **dado fresco**
(consultado na hora de usar, não fotografado 20 min antes) e **isolamento por item**.
Um producer que enriquece vira lote gigante, ponto único de falha e payload obsoleto.

## Preencher antes de colar

```
<TABELA>     = nome da tabela
<PK_TABELA>  = chave de particao da tabela (o unico campo que vai na mensagem)
<INDICE>     = nome do indice esparso
<PK_INDEX>   = atributo do shard (valores "0" a "9"); so existe nos itens pendentes
<SK_INDEX>   = atributo de data, String ISO-8601
<FILA>       = nome (ou URL) da fila
```

Tudo o mais já está decidido dentro dos prompts. Não há escolha a fazer na hora.

---

## ETAPA 1 — a consulta no índice (④)

```
Kotlin + AWS SDK v2, DynamoDB. Gere APENAS o repositorio de consulta, seguindo o estilo
das classes que ja existem no projeto.

Metodo que devolve as chaves de todos os itens vencidos de um GSI esparso:
- tabela <TABELA>, indice <INDICE>
- partition key do indice: <PK_INDEX>, String, com valores "0" a "9" (10 shards)
- sort key do indice: <SK_INDEX>, String em ISO-8601
- para cada um dos 10 shards: Query com igualdade no <PK_INDEX> e condicao <= :agora
  no <SK_INDEX>
- acumule o resultado dos 10 shards numa lista unica de <PK_TABELA>

Requisitos obrigatorios, nao negociaveis:
1. Pagine com exclusiveStartKey/lastEvaluatedKey e pare SOMENTE quando lastEvaluatedKey
   for null ou vazio. NUNCA pare porque a pagina veio sem itens: pagina vazia com
   lastEvaluatedKey preenchido significa que ha continuacao, e parar ali descarta o
   resto da fila em silencio.
2. Nao use FilterExpression. Os dois recortes ja estao na chave.
3. Se o SDK do projeto oferecer queryPaginator, use-o: ele resolve o laco sozinho e o
   erro do item 1 deixa de existir.
4. O indice e KEYS_ONLY: a Query devolve apenas as chaves do indice mais a chave da
   tabela. Nao tente ler campos de negocio a partir do resultado.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

⚠️ **`Limit` é tamanho de página, não teto de trabalho.** Tratá-lo como "processo N por dia"
faz a fila nunca drenar.

---

## ETAPA 2 — o producer (⑥)

```
Agora gere APENAS o Producer, no mesmo estilo.

Kotlin + AWS SDK v2, SQS. Metodo que recebe a lista de <PK_TABELA> devolvida pela consulta
da etapa anterior e publica na fila <FILA> com SendMessageBatch:
- uma mensagem por item
- o corpo da mensagem contem SOMENTE o campo <PK_TABELA>
- lotes de ate 10 entradas
- ids de entrada unicos dentro de cada lote

Requisitos obrigatorios, nao negociaveis:
1. NAO enriqueca o payload. Nenhuma chamada a outro servico, nenhuma leitura adicional
   na tabela, nenhum campo de negocio na mensagem. O enriquecimento e responsabilidade
   do consumidor.
2. SendMessageBatch retorna resultado POR ENTRADA e NAO lanca excecao quando entradas
   sao rejeitadas. Leia response.failed() e trate as entradas que falharam: log com o
   id da entrada e reenvio. Ignorar failed() faz ate 10 itens sumirem em silencio, com
   a chamada retornando sucesso.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

---

## ETAPA 3 — revisão antes de commitar

Use depois que o código estiver gerado, antes de abrir o PR. Ele força uma resposta
item a item em vez de um "está tudo certo" genérico.

```
Revise o Producer e a consulta que voce gerou. NAO reescreva do zero.
NAO edite nenhum arquivo. Responda no chat e mostre o diff proposto; eu aplico a mao.
Editar arquivo grande faz voce regerar o arquivo inteiro, e cada iteracao paga uma
rodada de verificacao de erros.

Para cada item abaixo responda CONFORME ou NAO CONFORME, citando o arquivo e a linha.
No fim, mostre APENAS o diff das correcoes.

1. O corpo da mensagem contem SOMENTE <PK_TABELA>. Nenhum campo de negocio, nenhuma
   chamada a servico externo, nenhuma leitura extra na tabela.
2. A consulta percorre os 10 shards do indice.
3. Nao existe FilterExpression em nenhuma Query.
4. Nao existe Scan em lugar nenhum.
5. A paginacao para SOMENTE quando lastEvaluatedKey e null ou vazio. Nao existe nenhuma
   condicao de parada baseada em pagina vazia ou em contagem de itens.
6. SendMessageBatch envia no maximo 10 entradas por chamada.
7. Os ids de entrada sao unicos dentro de cada lote.
8. response.failed() e lido, e cada entrada que falhou e identificada no log e
   reenviada ou reportada. Logar apenas a quantidade nao basta.
9. Falha em um shard ou em um lote nao interrompe os demais.
10. O metodo publico pode ser invocado sem depender de agendador nenhum, para execucao
    manual em homologacao.
11. Existe log com: quantos itens vieram por shard, quantos foram publicados, quantos
    falharam. Sem isso o teste em homologacao nao e verificavel.
12. Nao ha estado mutavel compartilhado entre execucoes.

Se algo estiver NAO CONFORME, corrija. Nao acrescente nenhuma funcionalidade que nao
esteja nesta lista.
```

---

## Testar em homologação, sem o agendador

1. Grave **2 ou 3 itens de teste** com `<PK_INDEX>` = `"0"` e `<SK_INDEX>` com data no passado.
2. Invoque o producer na mão (teste de integração ou endpoint temporário).
3. Confira o log (item 11 da revisão) e a fila:

```bash
aws sqs get-queue-attributes --queue-url <URL_DA_FILA> \
  --attribute-names ApproximateNumberOfMessages
```

| Resultado | Leitura |
|---|---|
| chegaram 2 ou 3 mensagens | ✅ ④ ⑤ ⑥ funcionando ponta a ponta |
| chegou 0 | a consulta não achou — confira se o item tem **os dois** atributos e se a data está no passado |
| log diz que publicou, fila vazia | leia `failed()` — as entradas foram rejeitadas |

## Correções prontas, se escapar

**Se a paginação parar na página vazia:**
```
Pare o laco somente quando lastEvaluatedKey for null ou vazio. Pagina sem itens com
lastEvaluatedKey preenchido significa que ha continuacao.
```

**Se o `failed()` for ignorado:**
```
SendMessageBatch nao lanca excecao quando entradas sao rejeitadas. Leia response.failed()
e trate cada entrada que falhou, identificando-a no log.
```

**Se o producer enriquecer o payload:**
```
Remova o enriquecimento. O corpo da mensagem deve conter somente <PK_TABELA>. Quem
enriquece e o consumidor.
```
