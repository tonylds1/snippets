# AGORA — o prompt do momento

> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## 🔴 Bloqueio encontrado em ④ — verificar antes de decidir

**O que o assistente relatou:** o projeto acessa o DynamoDB por uma **camada interna da
empresa**, e essa camada não expõe condição na chave de ordenação (`<= :agora`) nem controle
de paginação. O repositório só recebe essa camada, não o cliente do SDK.

**A saída que ele propôs:** injetar o cliente puro e consultar por fora da camada interna,
só para este fluxo.

⛔ **Não aceite essa saída ainda.** Camadas internas de banco costumam carregar credencial,
tracing, métrica, limite de vazão e auditoria. Passar por fora pode perder tudo isso em
silêncio — e é decisão de arquitetura, não de implementação.

⚠️ **E é afirmação do assistente, não fato verificado.** Confirmar é barato e produz a
evidência para levar ao TL.

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat.
- NAO gere codigo nesta resposta. Nenhuma linha.
- NAO proponha solucao. Eu so quero o levantamento.
- Se nao encontrar algo, escreva "nao encontrei" em vez de supor.
- Limite de 40 linhas vale so para a parte descritiva.

Tarefa: levantar exatamente o que a camada interna de acesso ao DynamoDB permite hoje.

Responda citando arquivo e linha:

1. Liste a assinatura COMPLETA de todos os metodos publicos da classe/interface interna de
   consulta ao DynamoDB usada por este projeto. Todos, sem resumir.
2. Algum desses metodos aceita condicao na chave de ordenacao (maior que, menor que,
   between, begins_with)? Se aceita, mostre a assinatura. Se nao aceita, diga "nao aceita".
3. Algum deles aceita consulta em indice secundario (GSI)? Mostre.
4. Algum deles devolve ou aceita token de paginacao / chave de continuacao? Mostre.
5. Procure em TODO o repositorio qualquer consulta a GSI que use condicao de intervalo na
   chave de ordenacao. Se existir, mostre o trecho inteiro: e a prova de que da para fazer.
6. Existe alguma classe do projeto que ja receba o cliente do SDK diretamente no construtor,
   sem passar pela camada interna? Se existir, mostre -- e o precedente.
7. A camada interna e uma dependencia externa (biblioteca) ou codigo deste repositorio?
   Se for biblioteca, qual o nome do artefato e a versao?
```

**O que cada resposta significa:**

| Achado | Consequência |
|---|---|
| **item 2** aceita condição de ordenação | não há bloqueio — o assistente errou |
| **item 5** achou uso existente | é o molde; copiar aquilo |
| **item 6** achou precedente | injetar o cliente puro já é aceito na casa; o risco cai muito |
| nada disso | bloqueio real → **é conversa com o TL**, com evidência na mão |

---

## Se confirmar o bloqueio: o que levar ao TL

Uma pergunta objetiva, não um pedido de permissão aberto:

> *A camada interna não expõe condição na chave de ordenação. Para consultar o índice por
> data preciso de uma destas três: (a) injetar o cliente do SDK direto neste repositório,
> (b) pedir ao time dono da camada que exponha a condição, ou (c) consultar só pela partição
> e filtrar em memória, lendo mais itens do que o necessário. Qual delas o time prefere?*

⚠️ A opção (c) existe e funciona, mas joga fora metade do motivo de a chave de ordenação
existir: passa a ler **todos** os pendentes, não só os vencidos.

---

## Fato novo do dia, para não perder

O serviço é **reativo** (cliente assíncrono + Reactor). Isso muda a forma do producer também
— o envio em lote será assíncrono, não bloqueante. Anotar antes de chegar em ⑥.

Mapa das peças: [INVENTARIO.md](INVENTARIO.md)
