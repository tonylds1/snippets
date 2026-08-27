# AGORA — o prompt do momento

> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## Passo atual: analisar o PRECEDENTE encontrado na organização

**O achado:** outro projeto da organização usa `QueryConditional.sortLessThanOrEqualTo` —
exatamente a condição que a camada interna não expõe.

⚠️ **Leia com cuidado:** o trecho encontrado usa `.scanIndexForward(false).limit(1)`, que é
*"me dá o mais recente, só um"* — busca pontual, não varredura em lote. **O precedente é
sobre o acesso ao cliente, não sobre o caso de uso.** Não diga ao TL que o projeto X faz o
mesmo que você precisa.

**Rodar com o projeto baixado aberto no workspace.**

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat.
- NAO gere codigo. Isto e um levantamento.
- Se nao encontrar algo, escreva "nao encontrei" em vez de supor.
- Limite de 40 linhas vale so para a parte descritiva; trechos de codigo nao contam.

Contexto: este repositorio usa QueryConditional.sortLessThanOrEqualTo numa consulta ao
DynamoDB. Quero saber se posso reproduzir esse padrao em outro microservico da mesma
organizacao.

Responda citando arquivo e linha:

1. Mostre a classe INTEIRA que monta esse QueryEnhancedRequest: construtor, injecoes e o
   metodo completo.
2. Que cliente ela injeta no construtor: DynamoDbEnhancedClient, DynamoDbEnhancedAsyncClient,
   DynamoDbClient, ou alguma camada interna da organizacao? Cite a linha do construtor.
3. Onde esse cliente e criado ou configurado? Mostre a classe de configuracao inteira:
   @Bean, @Configuration, e as propriedades que ela le.
4. A consulta e sobre a tabela principal ou sobre um indice secundario? Se for indice,
   mostre a linha com .index(...).
5. Como o resultado e percorrido: PageIterable, PagePublisher, stream, ou apenas o primeiro
   item? Mostre o trecho.
6. Liste as dependencias de DynamoDB deste projeto com grupo, artefato e versao
   (build.gradle, build.gradle.kts ou pom.xml).
7. Este projeto tambem depende de alguma biblioteca INTERNA da organizacao para acesso ao
   DynamoDB? Se sim: qual artefato e versao, e ela e usada em outros pontos deste mesmo
   repositorio? Este item e o mais importante da lista.
8. Existe README, ADR, decisao registrada ou comentario explicando por que usaram o cliente
   do SDK diretamente?
9. Qual a data do ultimo commit deste repositorio e deste arquivo?
```

---

## Como ler a resposta

| Achado | Consequência |
|---|---|
| **item 7**: usa a camada interna **E** o cliente do SDK | ⭐ **coexistência é aceita na casa.** Seu pedido vira "igual ao projeto X" |
| **item 2**: injeta o cliente do SDK direto | precedente de acesso existe — o risco de sair do padrão cai muito |
| **item 3**: tem classe de configuração | é o molde a copiar; sai do zero |
| **item 6**: mesmo artefato e versão que o seu | nada novo a introduzir, só usar o que já está no classpath |
| **item 8**: existe ADR justificando | argumento pronto, escrito por outra pessoa |
| **item 9**: commit recente | precedente vivo vale mais que repositório parado |

⚠️ Se o item 7 disser que aquele projeto **não** usa a camada interna, o precedente enfraquece:
pode ser um projeto que nasceu fora do padrão, não um que convive com ele.

---

## Depois deste

1. Se o precedente sustentar: montar a proposta para o TL — **uma pergunta**, com o projeto
   de referência citado por nome e arquivo.
2. Ainda em aberto e independente disto: **o método da camada interna pagina ou devolve só a
   primeira página?** Vale mesmo que você passe a usar o SDK.
3. ⏸️ **Não mandar a spec do índice para a infra** até as chaves estarem decididas.

Mapa das peças: [INVENTARIO.md](INVENTARIO.md)
