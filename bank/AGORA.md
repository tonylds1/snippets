# AGORA — o prompt do momento

> Arquivo vivo, sem data no nome: **sempre contém o único prompt a executar agora.**
> Mesma URL toda vez, sem procurar seção em arquivo longo.
>
> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## Passo atual: ⑤ índice esparso — **revisão antes de subir**

**Onde estamos:** o objeto com os nomes de índice e a entity estão prontos. Falta confirmar
que alguém de fato cria o índice na AWS e que ele vai nascer vazio.

**Antes de colar:** preencha a linha `Arquivos em que voce pode propor alteracao`.

**O que esperar de volta:** uma lista de 10 respostas CONFORME/NÃO CONFORME, no chat, em
segundos — ele não vai editar arquivo nenhum. Se demorar mais de 5 minutos, pare.

**Os dois itens que decidem o deploy:** o **1** (alguém cria o índice de verdade) e o **7**
(nasce vazio, sem backfill).

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat.
- Voce pode LER qualquer arquivo, mas so pode PROPOR alteracao nos que eu listar.
- Nunca reescreva um arquivo inteiro. Mostre so as linhas que mudam, com 3 de contexto.
- Se algo te impedir de seguir (arquivo grande, permissao, ambiguidade, erro que voce nao
  resolve em duas tentativas), diga isso em UMA frase e PARE. Nao contorne em silencio.
- Se a resposta passar de 40 linhas, pare e diga o que ficou faltando.

Arquivos em que voce pode propor alteracao:
<liste aqui: o object com os nomes de indice, a entity, e o que criar o indice>

Tarefa: revisar o codigo do indice esparso ja existente. NAO gere nada do zero.

Para cada item responda apenas CONFORME ou NAO CONFORME, citando arquivo e linha:

1. Existe uma chamada real a AWS que cria o indice (CreateTable ou UpdateTable). Mostre a
   linha. Anotacao em data class NAO cria indice: se so houver anotacao, marque NAO CONFORME.
2. O nome do indice existe em UM unico lugar, como `const val`. Nenhum literal repetido.
   (A consulta ao indice ainda nao foi escrita: ignore a parte deste item que fala dela.)
3. A chave de particao do indice e String e a de ordenacao e String em ISO-8601.
4. A projecao e KEYS_ONLY.
5. O modo de capacidade do indice e o mesmo da tabela.
6. Os dois atributos de chave do indice sao NULAVEIS no modelo, com null como padrao. Se
   algum for nao-nulavel ou tiver default, o indice deixa de ser esparso.
7. Nenhum codigo grava esses atributos ainda, e nao existe backfill nem migracao de dados.
   O indice tem de nascer vazio.
8. Se a criacao roda no startup: varias replicas subindo juntas nao quebram. Diga como
   ResourceInUseException ou indice ja existente e tratado.
9. A criacao de GSI e assincrona. Diga o que o codigo faz enquanto o status e CREATING.
10. Nada mais na definicao da tabela foi alterado.

Depois da lista, mostre o diff das correcoes dos itens NAO CONFORME. Nao acrescente
funcionalidade, configuracao nem propriedade nova que nao esteja nesta lista.
```

---

## Depois deste

1. Aplicar as correções dos NÃO CONFORME → compilar → **commit**
2. Subir para homologação e conferir: `IndexStatus` = `ACTIVE`, `ItemCount` = `0`
3. Próxima peça: **④ + ⑥**, em [202608261217-sqs-producer-indice.md](202608261217-sqs-producer-indice.md)

Mapa completo das peças: [INVENTARIO.md](INVENTARIO.md)
