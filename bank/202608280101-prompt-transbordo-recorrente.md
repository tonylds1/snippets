# Prompt — transbordo recorrente, do handler ao fim do ciclo

> Gera o fluxo novo **inteiro**: handler → use case → 2 steps novos → cálculo → escrita final.
> Nomes reais nunca entram aqui: tudo entre `<>` é preenchido na hora de colar.
>
> Pré-requisitos que **não** são gerados por este prompt:
> - o preâmbulo de [contenção](202608261829-copilot-contencao.md), colado em TODO turno;
> - o **bloco da API interna** (AGORA.md F1.0), colado em TODO turno;
> - a `CalculadoraTransbordo` (AGORA.md Fase 4) — classe pura, já escrita e já testada.

## Preencher antes de colar

```
<PK_TABELA>        = chave de particao da tabela; o unico campo que vem na mensagem
<PK_INDEX>         = atributo do shard (PK do GSI)
<SK_INDEX>         = atributo da data do proximo ciclo (SK do GSI)
<XyHandler>        = handler que ouve a fila de transbordo
<XyUseCase>        = use case do fluxo recorrente
<Repository>       = repositorio que le o item no Dynamo
<Ctx>              = objeto de contexto que atravessa os steps
<StepRecorrente>   = interface NOVA dos steps (nao a antiga)
<StepAntigo>       = interface dos steps que ja existem -- so para PROIBIR
<vq>               = a fonte externa do primeiro step
<intervalo>        = de quanto em quanto tempo o ciclo repete
```

## 🎯 As seis mudanças em relação ao rascunho

| # | O rascunho dizia | Por que muda |
|---|---|---|
| 1 | "analise o handler … e crie o use case" | **Ler e escrever no mesmo turno faz ele escrever enquanto deveria ler.** Turno 0 é só leitura, resposta em `arquivo:linha`, zero código |
| 2 | "os steps antigos não podem ser utilizados" | Pedido não se cumpre sozinho: com `List<StepAntigo>` injetada, **um step novo que implemente a interface antiga entra nos dois fluxos em silêncio**. Vira regra estrutural: interface nova, pacote novo |
| 3 | "pegar como referência o passo 11 e 12" | Referência de **forma**, nunca de **regra** — o TL disse que as regras de `/transbordo` estão erradas. A regra vem da `CalculadoraTransbordo` |
| 4 | "avaliar se escreve que finalizou ou que roda em 24h" | São **três** desfechos, não dois, e o terminal é `REMOVE` dos dois atributos (sai do índice), não flag |
| 5 | "[podem ser paralelos?]" | Podem: `Mono.zip`, com timeout explícito nos dois |
| 6 | "teste unitário é importante" | Sem a lista de casos ele escreve 30 testes rasos. A lista fecha o escopo em 8 |

---

## Bloco de referência — cole UMA vez, vale para todos os turnos

💰 O que mais gasta token é ele **procurar**. Você já sabe onde está tudo.

```
ARQUIVOS DE REFERENCIA. Leia SOMENTE estes. Se precisar de outro, PARE e peca pelo nome.

FORMA (como o projeto faz):
- handler de fila ja existente ......... <arquivo:linhas>
- passo 11 de /confirmacao ............. <arquivo:linhas>
- passo 12 de /confirmacao ............. <arquivo:linhas>
- chamada a API de limite .............. <arquivo:linhas>
- consulta ao <vq> ..................... <arquivo:linhas>
- consulta a /proposals ................ <arquivo:linhas>
- escrita condicional no Dynamo ........ <arquivo:linhas>
- um teste unitario do projeto ......... <arquivo:linhas>

REGRA (o que o codigo deve fazer):
- CalculadoraTransbordo (classe pura, ja pronta e testada) ... <arquivo>

⚠️ De /transbordo e dos steps antigos NAO se copia regra nenhuma. Eles nunca rodaram e o
tech lead disse que as regras estao erradas. Servem no maximo como forma, e mesmo assim
so os arquivos listados acima.
```

---

## TURNO 0 — leitura, zero código

```
NAO gere codigo. NAO edite arquivo. Responda com arquivo:linha, em no maximo 15 linhas.

1. O handler <XyHandler> ja recebe mensagem da fila? Qual anotacao/interface o aciona,
   e o consumo e por mensagem ou por lote?
2. Que classe ele desserializa do corpo, e o que ele faz quando a desserializacao falha?
3. Quem faz o acknowledgment: o framework (retornar sem excecao confirma) ou existe
   delete explicito no codigo?
4. O que ja existe de <XyUseCase>: assinatura do metodo publico, o que ele busca no
   <Repository> e o que ja e preenchido no contexto <Ctx>.
5. Liste os campos que <Ctx> tem hoje.
6. Qual interface os steps antigos implementam, e como a lista deles chega no use case
   (injecao de List<T>, @Bean explicito, ou outra coisa)?

A resposta do item 6 decide o desenho do proximo turno. Se nao achar, diga "nao achei".
```

🔴 **O item 6 é o que protege o fluxo que já roda em produção.** Se a lista antiga é
`List<StepAntigo>` injetada pelo Spring, todo bean que implementar essa interface entra
**nos dois** fluxos — inclusive os seus dois steps novos, e ninguém teria escrito uma
linha ligando as duas coisas.

---

## TURNO 1 — os dois steps novos e o contrato deles

```
[preambulo de contencao] [bloco da API] [bloco de referencia]

Gere APENAS: a interface nova dos steps e as DUAS classes de step. Nada mais.

REGRA ESTRUTURAL, nao negociavel:
- crie a interface <StepRecorrente> em pacote proprio (<pacote>.recorrente.step).
- os steps novos implementam SOMENTE <StepRecorrente>. NUNCA <StepAntigo>.
- nao acrescente, nao renomeie e nao mova nenhum step antigo.
- motivo: a lista de steps antiga e injetada como colecao; um bean que implemente a
  interface antiga passa a rodar tambem no fluxo que ja esta em producao.

Step 1 <BuscarDadosVqStep>: busca os dados do <vq> como no trecho de referencia, e
devolve o contexto com <atributoVq> preenchido.
Step 2 <BuscarPropostasStep>: busca em /proposals como no trecho de referencia, e
devolve o contexto com <atributoProposals> preenchido.

Requisitos obrigatorios:
1. Assinatura: fun execute(ctx: <Ctx>): Mono<<Ctx>>. O step NAO grava no Dynamo, NAO
   chama a API de limite e NAO decide nada: so busca e preenche o contexto.
2. Toda chamada externa tem timeout explicito. Nenhuma sem timeout.
3. O contexto e imutavel: devolva ctx.copy(...), nunca mutacao de campo.
4. Nenhum step le o relogio. A data do ciclo ja vem no contexto, em ctx.<hoje>.
5. Todo log carrega o <PK_TABELA>.

Kotlin idiomatico, sem !!, sem comentario explicativo no codigo.
Se a resposta passar de 40 linhas, pare e diga o que faltou.
```

⏰ **Nenhum step chama `now()`.** O dia é fixado **uma vez**, no início do use case, e viaja
no contexto. Um lote que atravesse a meia-noite faria dois steps enxergarem dias diferentes —
e o `dias <= qtdDias` da calculadora responderia coisas diferentes para o mesmo item.

---

## TURNO 2 — o use case: orquestração e fim do ciclo

```
[preambulo de contencao] [bloco da API] [bloco de referencia]

Complete <XyUseCase>. Mostre SO as linhas que mudam, com 3 de contexto.

Fluxo:
1. Recebe <PK_TABELA>. Busca o item no <Repository>. Se NAO existir: log WARN com o
   <PK_TABELA> e encerre com SUCESSO. Nao e erro -- o item pode ter saido do indice
   entre a publicacao e o consumo.
2. Monta o <Ctx>, fixando ctx.<hoje> UMA vez aqui (fuso America/Sao_Paulo).
3. Executa os dois <StepRecorrente> em PARALELO com Mono.zip, porque as duas buscas
   sao independentes uma da outra.
4. Chama finalizarPipeline(ctx).

finalizarPipeline:
a. Monta DadosTransbordo a partir do ctx e chama CalculadoraTransbordo.decidir(dados).
   NAO reimplemente a regra: a classe ja existe e ja tem teste.
b. Trate a decisao com `when` SEM else, tres ramos:
   - Transbordar(valor) -> envia o valor para a API de limite como no trecho de
       referencia, e DEPOIS grava avancando <SK_INDEX> para o proximo ciclo.
   - NadaATransbordar  -> nao chama a API. Grava avancando <SK_INDEX>.
   - Finalizar         -> grava o atributo de finalizado E REMOVE os dois atributos de
       controle (<PK_INDEX> e <SK_INDEX>) na MESMA escrita.

Requisitos obrigatorios, nao negociaveis:
1. UMA escrita condicional por desfecho, nunca duas. Duas escritas com uma falha entre
   elas deixam o item fora do indice para sempre, em silencio.
2. A escrita e condicional no valor lido: se outra execucao ja avancou o cursor, esta
   falha a condicao e para. E assim que a duplicata do SQS morre -- NAO invente
   mecanismo de idempotencia novo neste codigo.
3. Sair do indice e REMOVE dos atributos, nunca uma flag "pendente = false": flag
   mantem o item no indice para sempre.
4. <SK_INDEX> = ctx.<hoje> + <intervalo>, calculado a partir do contexto.
   NUNCA Instant.now()/LocalDate.now() dentro do metodo.
5. Formate a data EXATAMENTE no mesmo formato que a consulta ja usa: a comparacao no
   indice e de texto, nao de data.
6. Falha na API de limite: NAO grave nada e deixe a excecao subir. Sem escrita, a
   mensagem volta pela fila e o item continua no indice.
7. Nao altere nenhum step antigo, nenhum use case antigo e nenhuma classe de /transbordo.
```

🎯 **O `when` sobre `sealed interface` é o ponto do desenho.** Um `BigDecimal` de retorno
seria ambíguo — zero é *"acabou"* ou *"hoje não"*? — e os dois têm efeitos **opostos** no
índice. Com a sealed, quem esquecer um ramo não compila.

---

## TURNO 3 — os testes, com a lista fechada

```
[preambulo de contencao] [bloco de referencia: so o teste de exemplo]

Gere APENAS os testes unitarios, no estilo do teste de exemplo. Mocke o <Repository>,
os dois steps e o cliente da API de limite. Nenhuma AWS de verdade, nenhum Spring context.

Um teste para cada caso, nesta ordem:
1. Item nao encontrado no Dynamo: encerra com sucesso, nao chama step nenhum, nao grava.
2. Caminho feliz Transbordar: chama a API de limite UMA vez com o valor da calculadora,
   e grava avancando a data.
3. NadaATransbordar: NAO chama a API de limite, e mesmo assim grava avancando a data.
4. Finalizar: grava o finalizado e REMOVE os dois atributos de controle. Verifique a
   remocao explicitamente -- e o unico caminho que tira o item do indice.
5. Falha na API de limite: a excecao sobe e NENHUMA escrita acontece.
6. Os dois steps sao executados; o contexto que chega em finalizarPipeline tem os dois
   atributos preenchidos.
7. ctx.<hoje> e o mesmo valor nos dois steps e no calculo (o relogio e lido uma vez so).
8. A data gravada e ctx.<hoje> + <intervalo>, no mesmo formato de texto da consulta.

Nao teste a CalculadoraTransbordo (ja tem teste proprio) nem a biblioteca de SQS.
```

🔴 **O teste 4 é o que vale mais.** É o único caminho que tira o item do índice; se ele
gravar flag em vez de `REMOVE`, o item volta todo dia, para sempre, sem erro e sem log.

---

## TURNO 4 — revisão, antes de abrir o PR

```
Revise o que voce gerou. NAO reescreva do zero. NAO edite arquivo. Responda CONFORME ou
NAO CONFORME com arquivo:linha, e no fim mostre APENAS o diff das correcoes.

1. Nenhuma classe nova implementa a interface dos steps antigos.
2. Nenhum arquivo de /transbordo ou dos steps antigos foi alterado.
3. A regra de calculo esta SO na CalculadoraTransbordo -- nenhum if de negocio duplicado
   no use case ou nos steps.
4. O `when` da decisao nao tem `else`.
5. Existe exatamente UMA escrita por desfecho, e ela e condicional.
6. O ramo Finalizar REMOVE os dois atributos de controle.
7. Nenhum now()/LocalDate.now() fora do ponto unico no inicio do use case.
8. Toda chamada externa tem timeout explicito.
9. Item ausente no Dynamo encerra com sucesso e log WARN, sem ir para a DLQ.
10. Todo log carrega o <PK_TABELA>.
11. Nenhuma biblioteca nova foi introduzida.
```

---

## O que ainda precisa de resposta humana

| Pergunta | Por que trava |
|---|---|
| O fluxo recorrente também publica no SNS de democratização? | Se sim, é uma linha no fim do `when`; se não, é blast radius em consumidor de terceiro |
| `<intervalo>` é 24h fixas ou "próximo ciclo" configurável? | 24h fixas somam atraso a cada execução tardia; data-base fixa não |
| O que distingue transbordo inicial de recorrente? | 👉 Derive do **estado do item** (presença dos atributos de controle), não de campo novo na mensagem: a mensagem magra é o que dá dado fresco |
