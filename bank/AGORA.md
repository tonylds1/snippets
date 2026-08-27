# AGORA — o prompt do momento

> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## 🔴 Bloqueio CONFIRMADO em ④

A camada interna só consulta o GSI por **igualdade na partição**. Sem condição na chave de
ordenação, sem token de paginação, e é **biblioteca externa** — não dá para alterar.
Nenhum precedente no repositório de injetar o cliente do SDK direto.

## Passo atual: o último dado que falta

**A pergunta:** quando o resultado é grande, o método devolve **tudo** ou só a **primeira
página**? Se for só a primeira, ele perde itens em silêncio — e isso derruba qualquer saída
que continue usando a camada interna.

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat.
- NAO gere codigo. NAO proponha solucao. So o levantamento.
- Se nao conseguir determinar, escreva "nao consegui determinar" e diga o que faltou.
- Limite de 40 linhas vale so para a parte descritiva.

Contexto: o acesso ao DynamoDB passa por uma biblioteca externa cujo metodo de consulta a
indice secundario aceita apenas igualdade na chave de particao.

Responda:

1. Qual o tipo de retorno exato desse metodo? (Mono, Flux, List, PagePublisher,
   SdkPublisher, outro) Cite a assinatura.
2. Esse tipo de retorno emite TODAS as paginas do resultado, ou apenas a primeira?
   Abra o codigo da dependencia (fontes anexadas, decompilado ou javadoc) e mostre onde
   isso fica claro. Se nao conseguir abrir, diga isso.
3. Se ele emite tudo: a paginacao acontece por dentro automaticamente, ou depende de quem
   consome fazer alguma coisa?
4. Existe algum limite de itens embutido na chamada (Limit, maxResults, pageSize) que o
   metodo aplique sem eu pedir?
5. Qual o nome do artefato e a versao da dependencia? Existe versao mais nova que exponha
   condicao na chave de ordenacao? Se nao souber, diga que nao sabe.
```

**Como ler a resposta:**

| Achado | Consequência |
|---|---|
| emite todas as páginas | a camada interna continua utilizável — a saída **(d)** vira a mais barata |
| só a primeira página | ⚠️ **qualquer uso da camada perde itens em silêncio** — vira argumento forte para o cliente do SDK |
| não conseguiu determinar | testar em homologação com volume acima de uma página, antes de confiar |

---

## As saídas, para a conversa com o TL

| | Saída | Precisa de quê | Custo |
|---|---|---|---|
| **d** | **partição do índice passa a embutir a data** | de ninguém | mais consultas por execução |
| a | injetar o cliente do SDK direto | aprovação; sem precedente na casa | quebra o padrão |
| c | consultar por partição e filtrar em memória | resposta do item 2 acima | lê todos os pendentes |
| b | pedir ao time dono que exponha a condição | outro time, outro backlog | não cabe no prazo |

### Por que (d) primeiro

Se a partição do índice for **data + shard**, a consulta vira igualdade pura — exatamente o
que a camada interna oferece. **Nenhum padrão quebrado, nenhuma permissão pedida.**

⏰ **E o momento é agora:** o índice ainda não existe. Nome de chave é irreversível depois de
criado; hoje trocar é de graça.

⚠️ **O que (d) exige em troca:** consultar uma **janela** de datas, não só hoje. Se o job não
rodar num dia, aquele dia precisa ser recuperado — algo como os últimos 7 dias × 10 shards
por execução, em vez de 10 consultas.

⏸️ **Não mandar a spec do índice para a infra até isto ser decidido.** As chaves podem mudar.

---

## Fato novo, para não perder

O serviço é **reativo** (cliente assíncrono + Reactor). Muda a forma do producer (⑥) também.

Mapa das peças: [INVENTARIO.md](INVENTARIO.md)
