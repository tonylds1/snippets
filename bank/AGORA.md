# AGORA — o prompt do momento

> Arquivo vivo, sem data no nome: **sempre contém o único prompt a executar agora.**
>
> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## ⑤ índice — ENCERRADO do seu lado

O TL confirmou: **quem cria o índice é a pipeline**, não a aplicação. Então os itens 1, 4, 5,
8 e 9 da revisão saem do seu escopo — todos eram propriedade da chamada de criação.

⛔ **Não construa o `DynamoDBConfig` nem o `ApplicationRunner` que o assistente sugeriu.**
Seriam uma segunda fonte criando o mesmo índice.

✅ O que ficou pronto e validado: constante com dono único, tipos das chaves corretos,
atributos nuláveis (nasce esparso), nada os grava ainda (nasce vazio).

### ⚠️ Antes de qualquer código, 5 minutos: mandar a spec

Quem escreve o Terraform precisa saber **KEYS_ONLY** e os nomes das chaves **antes** de criar.
Projeção não se altera depois — se criarem com `ALL`, é apagar e recriar.

👉 [202608262115-spec-indice-para-quem-cria.md](202608262115-spec-indice-para-quem-cria.md) —
preencher 4 campos e enviar.

Depois do deploy, conferir em `infra-migration`: `IndexStatus` `ACTIVE`, `ProjectionType`
`KEYS_ONLY`, `ItemCount` `0`.

---

## Passo atual: ④ — a consulta ao índice

**Onde estamos:** primeira peça de código de verdade. Ela lê o índice e devolve as chaves dos
itens vencidos. Não depende do índice existir para ser escrita e revisada.

**Já sabemos do projeto:** usa `DynamoDbEnhancedClient` (por causa do `TableSchema.fromBean`),
então o acesso ao índice é `table.index(...)`, não `.indexName(...)`.

**Preencher antes:** `<TABELA>`, `<INDICE>`, `<PK_INDEX>`, `<SK_INDEX>`, `<PK_TABELA>`.

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat; eu aplico a mao.
- Voce pode LER qualquer arquivo, mas nao altere nenhum.
- Nunca reescreva um arquivo inteiro.
- Se algo te impedir de seguir (arquivo grande, ambiguidade, erro que voce nao resolve em
  duas tentativas), diga isso em UMA frase e PARE. Nao contorne em silencio.
- Limite de 40 linhas vale so para a parte descritiva. Codigo nao conta no limite.

Kotlin + AWS SDK v2 com DynamoDbEnhancedClient. Gere APENAS o metodo de consulta, seguindo
o estilo das classes que ja existem no projeto e reusando o cliente que ele ja configura.

Metodo que devolve as chaves (<PK_TABELA>) de todos os itens vencidos de um GSI esparso:
- tabela <TABELA>, indice <INDICE>, acessado por table.index("<INDICE>")
- particao do indice: <PK_INDEX>, String, valores "0" a "9" (10 shards)
- ordenacao do indice: <SK_INDEX>, String em ISO-8601
- para cada um dos 10 shards: consulta com igualdade no <PK_INDEX> e condicao <= :agora
  no <SK_INDEX>
- acumule os 10 shards numa lista unica de <PK_TABELA>

Requisitos obrigatorios, nao negociaveis:
1. Pagine e pare SOMENTE quando nao houver mais pagina. NUNCA pare porque uma pagina veio
   sem itens: pagina vazia com continuacao pendente significa que ha mais, e parar ali
   descarta o resto em silencio.
2. Nao use filtro (FilterExpression). Os dois recortes ja estao na chave.
3. O indice e KEYS_ONLY: a consulta devolve so as chaves do indice mais a chave da tabela.
   Nao tente ler campo de negocio a partir do resultado.
4. Falha em um shard nao interrompe os demais.
5. Nao use Scan em hipotese nenhuma.

Kotlin idiomatico, sem !!, sem comentarios explicativos no codigo.
```

⚠️ **`Limit` é tamanho de página, não teto de trabalho.** Tratá-lo como "processa N por dia"
faz a fila nunca drenar.

---

## Depois deste

1. Aplicar → compilar → **commit**
2. **⑥ o producer** — ETAPA 2 de
   [202608261217-sqs-producer-indice.md](202608261217-sqs-producer-indice.md)
3. Revisão dos dois juntos (ETAPA 3 da mesma receita)

Mapa das peças: [INVENTARIO.md](INVENTARIO.md)
