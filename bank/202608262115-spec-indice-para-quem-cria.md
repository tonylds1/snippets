# Spec do índice para quem cria a infraestrutura

> Use quando **você não cria o índice** — quem cria é a pipeline, o time de infra ou o
> Terraform de outro repositório. Sem esta spec, a pessoa escolhe por você.
>
> ⚠️ **Duas coisas do índice não se alteram depois de criado: o nome das chaves e a projeção.**
> Corrigir qualquer uma exige apagar e recriar. Por isso a spec vai **antes**, não depois.

## Preencher e enviar

```
Spec do indice secundario global (GSI) -- por favor conferir antes de criar.

Tabela:            <TABELA>
Nome do indice:    <INDICE>

Chave de particao: <PK_INDEX>   tipo String
Chave de ordenacao:<SK_INDEX>   tipo String (data em ISO-8601)

Projecao:          KEYS_ONLY
Capacidade:        a mesma da tabela

Tres pontos que fazem diferenca:

1. PROJECAO KEYS_ONLY, nao ALL. Quem consulta este indice so precisa descobrir QUAIS itens
   estao vencidos; os campos de negocio sao lidos direto da tabela depois. Projetar campos
   que mudam com frequencia faria toda alteracao deles reescrever tambem o indice.
   Projecao nao pode ser alterada depois: mudar exige apagar e recriar o indice.

2. O INDICE E ESPARSO DE PROPOSITO. Os dois atributos de chave so existem nos itens que
   estao aguardando processamento. Itens sem esses atributos nao entram no indice -- e e
   isso que mantem o indice pequeno e a consulta barata. Nao definir valor padrao para
   esses atributos em lugar nenhum.

3. CRIAR ANTES DE QUALQUER ITEM TER OS ATRIBUTOS. Hoje nenhum item tem, entao o indice
   nasce vazio e a criacao e instantanea. Se for criado depois de os dados existirem,
   vira backfill: varredura da tabela inteira, demorada e cara.

Ambientes: homologacao primeiro. Producao so depois de validado.

Como eu confirmo que ficou certo:
  aws dynamodb describe-table --table-name <TABELA> \\
    --query 'Table.GlobalSecondaryIndexes[?IndexName==`<INDICE>`]'
  Esperado: IndexStatus ACTIVE, ItemCount 0, ProjectionType KEYS_ONLY.
```

## Depois que criarem

| Campo | Esperado | Se vier diferente |
|---|---|---|
| `IndexStatus` | `ACTIVE` | `CREATING` = ainda subindo |
| `ProjectionType` | `KEYS_ONLY` | ⚠️ **fale agora** — depois exige recriar |
| `ItemCount` | `0` | não nasceu vazio: algum item já tinha os atributos |
| `Backfilling` | ausente ou `false` | está varrendo a tabela |
