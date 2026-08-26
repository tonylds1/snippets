# Consumidor que recebe mensagem magra e enriquece o payload — receita em 3 etapas

> Use quando o producer publica **só a identidade** e todo o resto é buscado no consumidor.
> Nomes de tabela, campos e classes são **fictícios**, sempre — preencher antes de colar.

## 📍 Onde você está

Ver [INVENTARIO.md](INVENTARIO.md) para o fluxo completo das 9 peças.

```
⑥ Producer ──▶ ⑦ Fila ──▶ ⑧ CONSUMIDOR  ◀── esta receita ──▶ ⑨ Fim do ciclo
```

**Esta receita entrega a ⑧.** Não toca no producer, não toca nos steps de negócio,
não toca no ⑨.

**Pré-requisito:** ⑥ existindo e publicando. Sem isso não há o que consumir.

## 🎯 A decisão que define este componente

> **Siga o padrão de erro que o projeto já usa. Não invente classificação nova.**

Existem dois tipos de falha, e projetos maduros os tratam diferente:

| Tipo | Exemplo | Tratamento típico |
|---|---|---|
| **Permanente** | campo faltando, corpo malformado | DLQ imediata; reentregar não vai fazer funcionar |
| **Transitória** | serviço fora, timeout, 5xx | relança e não confirma; o SQS reentrega sozinho |

**Mas muitos projetos tratam tudo como transitória** — relançam a exceção e deixam a
redrive policy levar para a DLQ depois de N tentativas. **Isso é aceitável**: uma mensagem
malformada gasta N tentativas inúteis e acaba na DLQ do mesmo jeito. Nada se perde, só
demora mais.

👉 **Por isso a ETAPA 1 pergunta como o projeto faz hoje, e a ETAPA 2 manda seguir.**
Consistência com o projeto vale mais que a classificação teoricamente correta —
principalmente em código que o time inteiro revisa.

## ⚠️ Idempotência: onde ela mora (e onde não mora)

SQS entrega **pelo menos uma vez**. A mesma mensagem vai chegar duas vezes algum dia.

**Muitas vezes o domínio já protege sozinho:** se a operação é dirigida pelo **estado atual**
(mover o que estiver disponível), a segunda execução não encontra nada para fazer. Isso é
**idempotência natural**, e é mais forte que uma flag, porque nasce do dado.

⚠️ **Mas ela cobre o efeito principal, não a escrituração.** Continuam podendo dobrar:
avanço de data do próximo ciclo, publicação de evento, contador. E se as duas execuções
rodarem em paralelo em réplicas diferentes, ambas podem ler o estado antigo.

🎯 **O conserto dos dois casos é o mesmo e mora no FIM DO CICLO (⑨), não aqui:** uma
**escrita condicional** que só aplica se o valor lido ainda for o mesmo. A segunda execução
falha a condição e para.

**Consequência para esta receita: o consumidor não implementa idempotência.** Ele apenas
não pode introduzir efeito colateral próprio que dobre.

## Preencher antes de colar

```
<FILA>       = nome (ou URL) da fila
<TABELA>     = nome da tabela
<PK_TABELA>  = chave de particao da tabela (o unico campo que vem na mensagem)
```

---

## ETAPA 1 — o que existe hoje (sem código)

O consumidor **já existe**. Antes de mudar, descubra a forma dele.

```
Nao gere codigo. Responda citando arquivo e linha:
1. Qual classe consome a fila <FILA> hoje e como ela e acionada (anotacao, interface,
   configuracao)?
2. Que classe ela desserializa a partir do corpo, e quais campos ela valida?
3. O que ela faz quando a validacao falha: envia para a DLQ explicitamente no codigo,
   ou depende de redrive policy na infraestrutura?
3b. O projeto distingue falha que nunca vai funcionar (corpo malformado, campo faltando)
   de falha temporaria (servico fora, timeout, 5xx)? Ou trata as duas do mesmo jeito?
   Mostre o codigo que faz essa decisao. Se nao houver distincao, diga isso.
4. Quem faz o acknowledgment: o framework (retornar sem excecao confirma, lancar
   devolve a mensagem) ou existe deleteMessage explicito?
5. O consumo e por mensagem ou por lote? Se por lote, onde esta o try/catch?
6. Que caso de uso ela chama, e qual a assinatura dele?
7. O projeto ja tem cliente HTTP para chamar outros servicos? Qual, e com que padrao
   de timeout, retry e tratamento de erro?
```

A resposta do item 7 é o que você vai reusar no enriquecimento — não deixe o modelo
introduzir biblioteca nova.

---

## ETAPA 2 — adaptar para a mensagem magra

```
Adapte o consumidor para receber uma mensagem magra e enriquecer o payload. Siga o estilo
das classes que ja existem no projeto e reuse o cliente HTTP que ele ja usa.

A mensagem agora contem SOMENTE o campo <PK_TABELA>, String.

Fluxo do metodo:
1. Ler <PK_TABELA> do corpo. Ausente, vazio ou em branco => FALHA PERMANENTE.
2. Carregar o item da tabela <TABELA> por essa chave. Se nao existir: log WARN com a
   chave e encerrar com sucesso. Nao e erro: o item pode ter sido removido entre a
   publicacao e o consumo.
3. Enriquecer chamando os servicos externos.
4. Chamar o caso de uso com o resultado do enriquecimento.

Requisitos obrigatorios, nao negociaveis:

1. TRATAMENTO DE ERRO: siga EXATAMENTE o padrao que a ETAPA 1 revelou no projeto. Nao
   introduza esquema de classificacao novo.
   - Se o projeto trata tudo como falha temporaria (relanca a excecao e deixa a redrive
     policy levar para a DLQ), mantenha assim.
   - Se o projeto distingue os dois casos, siga a mesma distincao, com os mesmos tipos
     de excecao que ele ja usa.
   Diga em uma frase qual padrao voce encontrou e esta seguindo.

2. IDEMPOTENCIA: NAO acrescente mecanismo de idempotencia neste componente. A garantia
   vem do dominio (a operacao e dirigida pelo estado atual) e de uma escrita condicional
   no fim do ciclo, que esta fora deste escopo. Apenas nao introduza efeito colateral
   proprio que dobre se a mesma mensagem chegar duas vezes.

3. Toda chamada externa tem timeout explicito. Nenhuma chamada sem timeout.

4. Se o consumo for por lote, o try/catch fica DENTRO do laco, envolvendo cada mensagem
   individualmente. Uma mensagem ruim nao pode interromper as demais.

5. A validacao nunca lanca excecao: a falha e valor de retorno. Se usar sealed class,
   o `when` nao tem `else`, para o compilador exigir todos os ramos.

6. Todo log carrega o <PK_TABELA>, para dar para rastrear um item especifico.

7. NAO altere o caso de uso nem os steps de negocio. O escopo e so o consumidor.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

---

## ETAPA 3 — revisão antes de commitar

```
Revise o consumidor que voce gerou. NAO reescreva do zero.
NAO edite nenhum arquivo. Responda no chat e mostre o diff proposto; eu aplico a mao.
Editar arquivo grande faz voce regerar o arquivo inteiro, e cada iteracao paga uma
rodada de verificacao de erros.

Para cada item responda CONFORME ou NAO CONFORME, citando arquivo e linha. No fim,
mostre APENAS o diff das correcoes.

1. O tratamento de erro e o MESMO que as outras classes do projeto usam. Cite a classe
   existente que serviu de referencia.
2. Nenhum catch engole a excecao em silencio: ou ela e relancada, ou ha registro do
   destino da mensagem.
3. Nenhum mecanismo novo de idempotencia foi inventado neste componente.
4. Item nao encontrado na tabela encerra com sucesso e log WARN, nao vai para a DLQ.
5. Toda chamada externa tem timeout explicito.
6. Se o consumo e por lote, o try/catch esta DENTRO do laco.
7. A validacao devolve valor, nao lanca excecao.
8. No ramo de falha permanente, o log contem o corpo INTEIRO da mensagem.
9. Todo log carrega o <PK_TABELA>.
10. Existe um ponto identificado que garante idempotencia, OU esta declarado que nao existe.
11. O caso de uso e os steps de negocio nao foram alterados.
12. Nenhuma biblioteca nova foi introduzida.

Se algo estiver NAO CONFORME, corrija. Nao acrescente funcionalidade fora desta lista.
```

---

## Decisões já embutidas — mude só se o time disser outra coisa

| Situação | Decisão embutida |
|---|---|
| campo da identidade ausente | falha — pelo caminho de erro que o projeto já usa |
| item não existe na tabela | log WARN + sucesso, sem DLQ |
| classificação de erro | **a que o projeto já usa** — descoberta na ETAPA 1 |
| mensagem duplicada | o consumidor não trata; a garantia é do ⑨ (escrita condicional) |

## Testar

Publique **uma** mensagem na mão, com o corpo mínimo:

```bash
aws sqs send-message --queue-url <URL_DA_FILA> \
  --message-body '{"<PK_TABELA>":"valor-que-existe-na-tabela"}'
```

| Resultado | Leitura |
|---|---|
| processou e o log mostra a chave | ✅ caminho feliz |
| foi para a DLQ | veja o log: validação ou enriquecimento |
| ficou reprocessando sem parar | erro transitório sendo relançado, e a redrive policy não está configurada |

Depois repita com o corpo `{}` — tem de acabar na DLQ. Se o projeto trata tudo como
transitória, ela chega lá **depois das N tentativas** da redrive policy, e isso está certo.
