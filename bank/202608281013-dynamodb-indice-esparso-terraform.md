# Criar o índice esparso (GSI) por Terraform — quando a anotação não basta

> Use quando você **confirmou** que a anotação do Enhanced Client não cria o índice, e a
> criação passou a ser entrega de infraestrutura.
>
> Anotação em `data class` **descreve** o índice para a biblioteca; ela não fala com a AWS.
> Você sobe, não dá erro nenhum, e o índice simplesmente não existe. Foi o que o teste mostrou.

## 📍 Onde você está

Ver [INVENTARIO.md](INVENTARIO.md) para as 9 peças e a ordem de construção.

**Esta receita entrega ⑤ — o índice**, pelo caminho da infra. É a irmã de
[dynamodb-indice-esparso-programatico](202608261122-dynamodb-indice-esparso-programatico.md),
que resolve a mesma peça por código. **Escolha uma das duas, nunca as duas** — dois donos para
o mesmo índice é briga de drift a cada `apply`.

⚠️ **A anotação continua no código.** Ela não cria nada, mas é o que dá tipo à consulta ④
(`table.index(...)`). Apagar a anotação porque "não funcionou" quebra a peça seguinte.

## Preencher antes de colar

```
<TABELA>    = nome da tabela
<PK_TABELA> = chave de partição da tabela
<INDICE>    = nome do índice
<PK_INDEX>  = atributo do shard, String "0".."9" — é o que torna o índice esparso
<SK_INDEX>  = atributo de data, String ISO-8601
```

🔴 **O nome de `<PK_INDEX>` tem de ser o mesmo em três lugares:** o que cria o índice (aqui),
o que grava o atributo, e a consulta ④. Chave de GSI não se renomeia. Confira os três **antes**
do apply, não depois.

---

## A pergunta que decide tudo, antes de escrever uma linha

**A tabela está no estado deste Terraform?**

```bash
terraform state list | grep -i dynamodb
```

| Resposta | Caminho |
|---|---|
| **A.** aparece como `aws_dynamodb_table.<tabela>` | ✅ siga para "O código" — é só acrescentar dois blocos |
| **B.** não aparece, mas a tabela existe na AWS | ⚠️ **importar primeiro**, seção "Caso B". Nunca declare por cima sem importar |
| **C.** quem cria a tabela é outro repo, outro time ou a própria aplicação | ✋ pare e mande a spec: [spec-indice-para-quem-cria](202608262115-spec-indice-para-quem-cria.md) |

📌 O caso B é o que morde. Declarar um `aws_dynamodb_table` que já existe na AWS **sem importar**
faz o plan pedir *create* — e o apply falha com `ResourceInUseException`, na melhor das hipóteses.

---

## O código

Se a tabela já é declarada aqui (caso A), **não crie recurso novo**: acrescente os `attribute`
que faltam e o bloco `global_secondary_index` ao recurso que existe.

```hcl
resource "aws_dynamodb_table" "<tabela>" {
  name         = "${var.prefixo}-<TABELA>"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "<PK_TABELA>"

  # ⚠️ SÓ atributos que são chave — de tabela ou de índice. Campo de negócio não entra aqui.
  attribute {
    name = "<PK_TABELA>"
    type = "S"
  }

  # ↓ os dois que o índice acrescenta
  attribute {
    name = "<PK_INDEX>"     # shard: "0".."9"
    type = "S"
  }

  attribute {
    name = "<SK_INDEX>"     # data ISO-8601
    type = "S"
  }

  global_secondary_index {
    name            = "<INDICE>"
    hash_key        = "<PK_INDEX>"
    range_key       = "<SK_INDEX>"
    projection_type = "KEYS_ONLY"   # 🎯 decisão fechada, e não se altera depois
  }

  lifecycle {
    prevent_destroy = true          # 🔴 a rede de segurança desta mudança
  }

  timeouts {
    update = "60m"                  # criação de GSI é assíncrona e pode demorar
  }

  tags = var.tags
}
```

### Se a tabela for `PROVISIONED` em vez de on-demand

```hcl
  billing_mode   = "PROVISIONED"
  read_capacity  = 5
  write_capacity = 5

  global_secondary_index {
    name            = "<INDICE>"
    hash_key        = "<PK_INDEX>"
    range_key       = "<SK_INDEX>"
    projection_type = "KEYS_ONLY"
    read_capacity   = 5     # obrigatório em PROVISIONED
    write_capacity  = 5     # ⚠️ proibido em PAY_PER_REQUEST — erro de plan
  }
```

⚠️ **GSI tem capacidade própria, e autoscaling próprio.** Se a tabela tem
`aws_appautoscaling_target`, o índice precisa dos dele (`resource_id` termina em
`/index/<INDICE>`). Sem isso o índice fica travado na capacidade declarada e passa a
estrangular a escrita **da tabela inteira**.

---

## As cinco armadilhas do Terraform em GSI

| # | Armadilha | O que acontece |
|---|---|---|
| 1 | listar atributo que **não é chave** em `attribute` | erro `all attributes must be indexed` ou diff perpétuo. Só chave entra ali |
| 2 | achar que o `attribute` cria o campo nos itens | não cria nada. É só declaração de tipo para o esquema de chaves |
| 3 | acrescentar **dois** GSIs no mesmo apply | `UpdateTable` cria um índice por vez. Um por PR |
| 4 | mexer em `projection_type` depois | não é alteração in-place: destrói e recria o índice |
| 5 | `read_capacity` em tabela on-demand | erro no plan — e o contrário também (falta dele em `PROVISIONED`) |

🎯 **Terraform não tem como expressar "esparso".** O índice nasce esparso porque **nenhum item
tem os dois atributos ainda** — não por causa de nada escrito no `.tf`. Por isso a ordem é:
criar o índice **primeiro**, começar a gravar os atributos **depois**. Invertido, vira backfill:
varredura da tabela inteira, demorada e cara.

---

## 🔴 O plan que você tem de ler linha a linha

Este é o passo que separa "acrescentei um índice" de "recriei a tabela de produção".

```bash
terraform fmt <arquivo>.tf
terraform init
terraform validate
terraform plan -var-file=inventories/hom/terraform.tfvars -out=tfplan
terraform show tfplan | grep -E "will be|must be|forces replacement"
```

| O que aparecer | Significado |
|---|---|
| `~ update in-place` | ✅ **é este.** O índice é acrescentado, a tabela não é tocada |
| `+ global_secondary_index` dentro do `~` | ✅ confirma que a mudança é só o índice |
| `must be replaced` / `forces replacement` | 🔴 **PARE.** Isso apaga a tabela e todos os dados |
| `Error: Instance cannot be destroyed` | ✅ o `prevent_destroy` fez o trabalho dele — algo acima ia destruir a tabela |

Só forçam substituição: `name`, `hash_key`, `range_key` e `billing_mode` de `PROVISIONED` para
on-demand em algumas versões do provider. Se o plan pede replacement, **você mexeu numa dessas
sem perceber** — provavelmente digitou o nome da tabela diferente do que está na AWS.

**Homologação primeiro. Produção só depois de verificado.**

---

## Depois do apply — verificar que nasceu certo

```bash
aws dynamodb describe-table --table-name <TABELA> \
  --query 'Table.GlobalSecondaryIndexes[?IndexName==`<INDICE>`]'
```

| Campo | Esperado | Se vier diferente |
|---|---|---|
| `IndexStatus` | `ACTIVE` | `CREATING` = ainda subindo, espere e repita |
| `ProjectionType` | `KEYS_ONLY` | ⚠️ **fale agora** — corrigir exige apagar e recriar |
| `ItemCount` | `0` | ≠ 0 = algum item já tinha os atributos; não nasceu vazio |
| `Backfilling` | ausente ou `false` | `true` = está varrendo a tabela |

`ItemCount` é atualizado a cada ~6h: serve para "nasceu vazio", não para acompanhar crescimento.

---

## Caso B — a tabela existe na AWS, mas não no estado

Ordem obrigatória. **Não pule o passo 3.**

```bash
# 1. declare o recurso exatamente como a tabela está hoje, SEM o índice novo
# 2. importe
terraform import aws_dynamodb_table.<tabela> <TABELA>

# 3. 🔴 plan tem de sair "No changes" ANTES de você acrescentar qualquer coisa
terraform plan -var-file=inventories/hom/terraform.tfvars
```

Enquanto o plan não sair limpo, sua declaração diverge da tabela real — e o apply vai "corrigir"
a tabela para bater com o que você escreveu. É aí que se perde TTL, stream ou point-in-time
recovery que ninguém tinha declarado.

Só depois do `No changes`, acrescente os blocos do índice e volte para a leitura do plan.

### Alternativa de uma vez só, sem Terraform

Se o índice é pontual e a tabela não é gerenciada por Terraform em lugar nenhum:

```bash
aws dynamodb update-table \
  --table-name <TABELA> \
  --attribute-definitions \
      AttributeName=<PK_INDEX>,AttributeType=S \
      AttributeName=<SK_INDEX>,AttributeType=S \
  --global-secondary-index-updates '[{
    "Create": {
      "IndexName": "<INDICE>",
      "KeySchema": [
        {"AttributeName": "<PK_INDEX>", "KeyType": "HASH"},
        {"AttributeName": "<SK_INDEX>", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "KEYS_ONLY"}
    }
  }]'
```

⚠️ Isto cria o índice **fora** de qualquer estado. Vale para homologação e para destravar teste;
para produção, o índice precisa ter dono declarado — senão o próximo `apply` de quem gerencia a
tabela pode removê-lo.

---

## O que não fazer

- ❌ **`ignore_changes = [global_secondary_index]`** para "resolver" drift. Isso não conserta
  nada: só faz o Terraform parar de contar a verdade sobre o índice.
- ❌ **Valor padrão** para `<PK_INDEX>` ou `<SK_INDEX>` em qualquer lugar do código. Um default
  põe todo item no índice e o esparso morre — o índice vira cópia da tabela.
- ❌ **`projection_type = "ALL"`** "por segurança". Cada alteração de campo de negócio passaria a
  reescrever o índice também.
- ❌ **Gravar os atributos antes do índice existir.** Vira backfill.
- ❌ **Manter a criação programática junto.** Escolhida a infra, o código que criava o índice sai.
