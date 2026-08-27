# Procurar precedente de GSI com condição de ordenação no GitHub da organização

> Use quando a biblioteca interna não expõe o que você precisa e você quer saber **se alguém
> na casa já resolveu** — seja com outra biblioteca, seja saindo do padrão com aprovação.
>
> Precedente vale mais que argumento: muda a conversa de *"posso fazer diferente?"* para
> *"sigo o que o time X já faz?"*.

## Como buscar

Busca de código do GitHub, com escopo na organização:

```
org:<organizacao> "<string>"
```

Filtros que ajudam: `language:kotlin`, `language:java`, `path:*.kt`, `path:build.gradle`.

---

## Prioridade 1 — a condição exata que falta

| String | Por que |
|---|---|
| `sortLessThanOrEqualTo` | **é a condição que você precisa**, e só existe no Enhanced Client |
| `sortGreaterThanOrEqualTo` | mesma família |
| `QueryConditional.sortBetween` | intervalo de datas — variação do mesmo problema |
| `QueryConditional.sortBeginsWith` | prefixo em chave de ordenação |

Qualquer acerto aqui é ouro: mostre o arquivo inteiro e veja **qual cliente ele injeta**.

## Prioridade 2 — quem já consulta índice de verdade

| String | Por que |
|---|---|
| `DynamoDbIndex` | o tipo que representa um GSI no Enhanced Client |
| `DynamoDbAsyncIndex` | versão reativa |
| `.index(` + `QueryConditional` | uso combinado, típico de consulta a GSI |
| `queryPaginator` | paginação explícita no SDK de baixo nível |
| `keyConditionExpression` | SDK de baixo nível; procure junto com `indexName` |

## Prioridade 3 — ⭐ os outros consumidores da MESMA biblioteca

**A mais valiosa.** Busque pelo **nome do artefato** da dependência interna em
`build.gradle`, `build.gradle.kts` ou `pom.xml`:

```
org:<organizacao> "<grupo>:<artefato>"
org:<organizacao> path:build.gradle "<artefato>"
```

Cada acerto é um time que usa a mesma camada. Abra dois ou três e procure:

- algum deles filtra por **data** ao consultar um índice? Como?
- algum deles injeta o cliente do SDK **direto**, ao lado da camada interna?
- algum deles usa uma versão **mais nova** da biblioteca que exponha mais coisas?

⭐ Você não está procurando quem sabe DynamoDB. Está procurando **quem bateu na mesma parede**.

## Prioridade 4 — índice esparso, o padrão inteiro

| String | Onde costuma aparecer |
|---|---|
| `@DynamoDbSecondarySortKey` | entity com GSI |
| `KEYS_ONLY` | Java, Kotlin ou Terraform |
| `projection_type` | Terraform, dentro de `global_secondary_index` |
| `attribute_not_exists` | escrita condicional — quem faz índice esparso costuma usar |

Acerto em `KEYS_ONLY` + `attribute_not_exists` no mesmo repositório é forte indício de que
alguém já montou um índice esparso de trabalho pendente — o mesmo padrão.

---

## O que fazer com o que achar

| Achado | Consequência |
|---|---|
| condição de ordenação usada em algum lugar | **existe caminho** — copie a forma e veja qual cliente injetam |
| outro consumidor injeta o SDK direto | **precedente na casa**; o risco de sair do padrão cai muito |
| versão mais nova da biblioteca expõe mais | atualizar a dependência pode ser a saída mais barata de todas |
| nada em lugar nenhum | o bloqueio é real e geral — **isso também é resposta**, e fortalece a conversa |

⚠️ Nenhum resultado **não** é fracasso da busca: é evidência de que ninguém na organização
faz isso pela camada interna, e é exatamente o que o TL precisa saber.
