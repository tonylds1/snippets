# ① O agendador — EventBridge Scheduler diário publicando na fila de gatilho

> Uma execução por dia, de madrugada, que **só publica uma mensagem vazia** numa fila SQS.
> O agendador não sabe nada dos dados: ele carrega apenas o fato de que chegou a hora.
>
> Receita completa: **quais variáveis criar, em que arquivo cada peça mora, o que vai em cada
> `tfvars`** e como ligar o agendador depois que o resto estiver de pé.

## 📍 Onde você está

Ver [INVENTARIO.md](INVENTARIO.md) para as 9 peças e a ordem de construção.

**Esta receita entrega ① (o agendador) e ② (a fila de gatilho)** — as duas andam juntas porque
o alvo do agendador é a fila, e a permissão de uma depende do ARN da outra.

🎯 **O agendador é a ÚLTIMA peça a ser LIGADA, não a última a ser escrita.** Escreva hoje com
`state = "DISABLED"`, commite sem risco, e ligue trocando **uma variável** quando ③④⑤⑥⑧
estiverem prontos. Ligar depois é editar um `tfvars`, não escrever infraestrutura sob pressão.

📌 **Quem garante que só uma réplica execute é a fila ②, não o agendador.** O Scheduler dispara
uma vez; se a aplicação tem 3 réplicas ouvindo, apenas uma recebe a mensagem. Sem a fila no meio
(agendador chamando a aplicação direto, ou `@Scheduled` no serviço), as 3 executam o dia inteiro.

## Preencher antes de colar

```
<PREFIXO>       = variável que já existe no repo e prefixa todo recurso (ex.: var.prefixo)
<FILA_GATILHO>  = nome curto da fila de gatilho (②)
<AGENDADOR>     = nome curto do schedule (①)
<slug>          = identificador do recurso no Terraform, sem hífen (ex.: gatilho_diario)
<ARQUIVO>       = nome do .tf novo desta feature (ex.: gatilho_diario.tf)
<pacote>        = pacote/módulo da aplicação que consome a fila
```

⚠️ **`name` de schedule e de fila força replacement.** Escolha o nome antes do primeiro apply;
renomear depois **destrói e recria** — e uma fila recriada perde as mensagens que estavam nela.

---

## 🗺️ Onde cada peça mora

A regra do repositório de infra: **`variable` não mora junto do recurso.** Um `.tf` de feature que
declara variável funciona, mas some da lista quando alguém procura o que é configurável.

| Peça | Arquivo | Por quê |
|---|---|---|
| as 5 `variable` | `variables.tf` | é onde se procura "o que dá para configurar aqui" |
| valores por ambiente | `inventories/<env>/terraform.tfvars` | um arquivo por ambiente: dev, hom, prod |
| fila ② + DLQ | `<ARQUIVO>` (novo) | a feature inteira num arquivo só, fácil de ler no PR |
| role + policies do agendador | `<ARQUIVO>` (mesmo) | a permissão referencia o ARN da fila; separar só espalha |
| schedule ① | `<ARQUIVO>` (mesmo) | — |
| `output` da URL/ARN da fila | `outputs.tf` | a aplicação precisa da URL para configurar o listener ③ |
| versão do provider | `versions.tf` | `aws_scheduler_schedule` exige provider AWS **≥ 4.56** |

📌 **Um arquivo novo, não um bloco no `main.tf`.** Feature em arquivo próprio é o que faz o PR
ser lido em cinco minutos — e é o que permite arrancar tudo depois com um `git rm`, se a story
morrer.

🔴 **Confirme a versão do provider antes de qualquer coisa:**

```bash
grep -A3 'required_providers' versions.tf
terraform providers
```

Provider abaixo de 4.56 não conhece `aws_scheduler_schedule` — o erro é
`Invalid resource type`, e a solução **não** é voltar para `aws_cloudwatch_event_rule`
(EventBridge Rules, o serviço antigo): é subir o provider. Se subir o provider não for possível
hoje, veja a seção "Se não der para usar o Scheduler" no fim.

---

## 1️⃣ `variables.tf` — as cinco variáveis

```hcl
variable "nome_fila_gatilho" {
  description = "Nome curto da fila de gatilho (②) que o TriggerHandler ③ consome."
  type        = string
  default     = "<FILA_GATILHO>"
}

variable "gatilho_cron" {
  description = <<-EOT
    Expressão do agendador (①). Padrão: todo dia às 03:00 (madrugada).
    O cron do EventBridge tem 6 campos e exige `?` em dia-do-mês OU dia-da-semana.
  EOT
  type        = string
  default     = "cron(0 3 * * ? *)"

  validation {
    condition     = can(regex("^(cron|rate|at)\\(", var.gatilho_cron))
    error_message = "Use cron(...), rate(...) ou at(...)."
  }
}

variable "gatilho_timezone" {
  description = "Fuso do agendador. SEM ISTO o cron roda em UTC e a madrugada vira meio da tarde."
  type        = string
  default     = "America/Sao_Paulo"
}

variable "gatilho_habilitado" {
  description = "Liga/desliga o agendador. Permite subir toda a infra com o disparo desligado."
  type        = bool
  default     = false
}

variable "gatilho_visibility_timeout" {
  description = <<-EOT
    Deve ser MAIOR que o tempo máximo do ciclo do handler ③.
    Se a mensagem reaparecer antes do fim, outra réplica pega e o dia processa duas vezes.
  EOT
  type        = number
  default     = 900
}
```

| Variável | Por que é variável, e não constante no recurso |
|---|---|
| `nome_fila_gatilho` | o nome muda por convenção de ambiente, e aparece em 4 lugares |
| `gatilho_cron` | 🎯 **dev/hom precisam disparar a cada 15 min para testar**; prod, 1×/dia |
| `gatilho_timezone` | o dia em que alguém pedir UTC, é uma linha de tfvars |
| `gatilho_habilitado` | 🔴 **é esta que liga o fluxo.** Ligar = 1 linha no PR, revisável em 10 segundos |
| `gatilho_visibility_timeout` | muda quando o ciclo cresce; ficar escondido no recurso esconde o risco |

🎯 **Só vira `variable` o que muda entre ambientes ou o que você quer poder mudar sem tocar na
lógica.** `input = jsonencode({})` e `flexible_time_window { mode = "OFF" }` não são variáveis:
são decisões de desenho, e transformá-las em parâmetro só cria a chance de alguém mudá-las.

## 2️⃣ `inventories/<env>/terraform.tfvars` — os valores

```hcl
# inventories/dev/terraform.tfvars
gatilho_cron       = "rate(15 minutes)"   # 🎯 testar não é esperar a madrugada
gatilho_habilitado = true

# inventories/hom/terraform.tfvars
gatilho_cron       = "cron(0 3 * * ? *)"
gatilho_habilitado = true

# inventories/prod/terraform.tfvars
gatilho_cron       = "cron(0 3 * * ? *)"
gatilho_habilitado = false                # 🔴 ligar só no PR que fecha a story
```

⚠️ **`rate(15 minutes)` em dev só é seguro se o ciclo for idempotente.** Se ainda não for, deixe
`gatilho_habilitado = false` até em dev e dispare na mão (seção "Testar sem esperar").

📌 **Não repita o valor default em todo tfvars.** Só declare ali o que **difere** do default —
tfvars cheio de valor igual ao default vira ruído e esconde a linha que importa.

---

## 3️⃣ `<ARQUIVO>` — ② a fila de gatilho + DLQ

A fila vem primeiro: a permissão do agendador referencia o ARN dela.

```hcl
resource "aws_sqs_queue" "<slug>_dlq" {
  name                      = "${var.prefixo}-${var.nome_fila_gatilho}-dlq"
  message_retention_seconds = 1209600   # 14 dias — é o que dá tempo de investigar
  sqs_managed_sse_enabled   = true
  tags                      = var.tags
}

resource "aws_sqs_queue" "<slug>" {
  name                       = "${var.prefixo}-${var.nome_fila_gatilho}"
  visibility_timeout_seconds = var.gatilho_visibility_timeout
  receive_wait_time_seconds  = 20       # long polling: menos chamada vazia
  sqs_managed_sse_enabled    = true

  # 🎯 1 dia, não os 4 do padrão. Gatilho vencido não deve executar depois.
  message_retention_seconds = 86400

  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.<slug>_dlq.arn
    maxReceiveCount     = 3
  })

  tags = var.tags
}
```

📌 **Retenção de 1 dia é decisão de desenho, não economia.** Se o gatilho de hoje não foi
consumido, o de amanhã pega os mesmos itens — a consulta ④ é *"tudo que venceu até agora"*, não
*"o que venceu hoje"*. Guardar gatilho velho por 14 dias só produz execução dupla no dia em que
a fila destravar.

⚠️ **Se o projeto cria filas por módulo interno**, use o módulo da casa em vez de `aws_sqs_queue`
direto. O conteúdo é o mesmo; a forma muda — e um recurso solto no meio de um repo modularizado
é o tipo de coisa que trava revisão.

## 4️⃣ `<ARQUIVO>` — a permissão (a parte que as pessoas erram)

🔴 **O Scheduler não publica em fila sem uma role.** Diferente do EventBridge Rules antigo, que
usava *resource policy* na fila, o Scheduler **assume uma role sua** para chamar o alvo. Fila com
`Principal: scheduler.amazonaws.com` na policy dela **não** é o caminho aqui.

```hcl
data "aws_caller_identity" "atual" {}

# ⚠️ Se este data source já existir no repo, reaproveite — declarar duas vezes é erro.

data "aws_iam_policy_document" "<slug>_trust" {
  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRole"]

    principals {
      type        = "Service"
      identifiers = ["scheduler.amazonaws.com"]
    }

    # 🔴 Trava de confused deputy: só o Scheduler DESTA conta assume o papel.
    condition {
      test     = "StringEquals"
      variable = "aws:SourceAccount"
      values   = [data.aws_caller_identity.atual.account_id]
    }
  }
}

data "aws_iam_policy_document" "<slug>_envio" {
  statement {
    effect    = "Allow"
    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.<slug>.arn]   # 🎯 a fila, nunca "*"
  }
}

resource "aws_iam_role" "<slug>" {
  name               = "${var.prefixo}-${var.nome_fila_gatilho}-scheduler"
  assume_role_policy = data.aws_iam_policy_document.<slug>_trust.json
  tags               = var.tags
}

resource "aws_iam_role_policy" "<slug>" {
  name   = "envio-fila-gatilho"
  role   = aws_iam_role.<slug>.id
  policy = data.aws_iam_policy_document.<slug>_envio.json
}
```

🔴 **Se a fila usar chave KMS própria (CMK), `sqs:SendMessage` não basta.** Acrescente ao
documento de envio — sem isto a entrega falha em silêncio, e a única pista é a métrica
`TargetErrorCount`:

```hcl
  statement {
    effect    = "Allow"
    actions   = ["kms:GenerateDataKey", "kms:Decrypt"]
    resources = [aws_kms_key.<chave>.arn]
  }
```

📌 Com `sqs_managed_sse_enabled = true` (SSE-SQS, o do exemplo acima) **não é preciso nada disso**
— a criptografia é gerenciada e transparente. A pegadinha só existe com CMK.

## 5️⃣ `<ARQUIVO>` — ① o agendador

```hcl
resource "aws_scheduler_schedule" "<slug>" {
  name        = "${var.prefixo}-<AGENDADOR>"
  description = "Publica {} na fila de gatilho, uma vez por dia de madrugada."
  group_name  = "default"

  state = var.gatilho_habilitado ? "ENABLED" : "DISABLED"

  schedule_expression          = var.gatilho_cron
  schedule_expression_timezone = var.gatilho_timezone

  # OFF = hora exata. Com janela flexível a AWS escolhe um minuto qualquer dentro dela.
  flexible_time_window {
    mode = "OFF"
  }

  target {
    arn      = aws_sqs_queue.<slug>.arn
    role_arn = aws_iam_role.<slug>.arn

    # 🎯 O corpo é vazio de propósito: o handler ③ ignora o conteúdo.
    #    O sinal é a CHEGADA da mensagem, não o que ela diz.
    input = jsonencode({})

    retry_policy {
      maximum_retry_attempts       = 3
      maximum_event_age_in_seconds = 3600   # depois disso, gatilho velho não serve
    }

    # ⚠️ DLQ do AGENDADOR — falha em ENTREGAR na fila. É diferente da DLQ da fila,
    #    que guarda mensagem que o handler não conseguiu processar.
    dead_letter_config {
      arn = aws_sqs_queue.<slug>_dlq.arn
    }
  }
}
```

| Detalhe | Por quê |
|---|---|
| `state` vindo de variável | 🎯 ligar o fluxo é 1 linha de `tfvars`, revisável em 10 segundos |
| `schedule_expression_timezone` | **3h locais o ano inteiro.** Sem isto é UTC — 03:00 UTC = meia-noite em Brasília |
| `flexible_time_window { mode = "OFF" }` | bloco **obrigatório**; `OFF` = dispara na hora exata |
| `input = jsonencode({})` | corpo vazio, de propósito. Handler que lê o corpo passa a depender do agendador |
| `group_name = "default"` | explícito porque grupo **força replacement** se mudar depois |
| `maximum_event_age_in_seconds` | limita o retry: gatilho de 3h entregue às 9h é execução fora de hora |
| `dead_letter_config` | 🔴 **sem isto, gatilho que não foi entregue evapora sem rastro** |

---

## 📭 O corpo vazio — o que ele muda de verdade

O handler ③ não lê nada do que chega: o sinal é a **chegada** da mensagem. Isso é decisão de
desenho, não economia — e tem três consequências que valem estar escritas.

**1. Some a discussão de criptografia nesta fila.** Corpo vazio = nenhum dado trafegando. O
argumento de conformidade para chave KMS própria não existe aqui: `sqs_managed_sse_enabled` basta,
e a role dispensa `kms:GenerateDataKey` (armadilha 4). ⚠️ Continua valendo para a fila de
destino ⑦, que carrega identidade.

**2. "Vazio" não é string vazia.** SQS exige corpo com **pelo menos 1 caractere**.
`jsonencode({})` produz `{}` — o mínimo seguro. Não troque por `input = ""` nem remova o campo
achando que fica mais limpo.

**3. 🔴 O listener não pode desserializar em tipo.** Se o handler declarar um payload tipado, o
Jackson tenta montar o objeto a partir de `{}`. Em Kotlin, `data class` com campo não-nulo e sem
default → `MissingKotlinParameterException`, a mensagem vai para a DLQ, e o erro **parece problema
de infra quando é assinatura de método**. O handler recebe `String` (ou `Message<String>`) e ignora:

```kotlin
fun onGatilho(@Payload corpo: String) {   // ✅ ignora o corpo, de propósito
    producer.publicarVencidos()
}
```

### O que o corpo vazio custa — e o conserto de graça

Dois gatilhos idênticos no log são indistinguíveis, e uma mensagem `{}` na DLQ não diz de qual
execução veio. Com entrega *pelo menos uma vez* e `maximum_retry_attempts = 3`, isso não é
hipótese. O Scheduler substitui **atributos de contexto** dentro do `input`:

```hcl
input = jsonencode({
  execucao = "<aws.scheduler.execution-id>"
  horario  = "<aws.scheduler.scheduled-time>"
})
```

| Atributo | Serve para |
|---|---|
| `<aws.scheduler.execution-id>` | correlacionar log e mensagem na DLQ; detectar entrega repetida |
| `<aws.scheduler.scheduled-time>` | ver no log **quando era para ter rodado**, não quando rodou |
| `<aws.scheduler.attempt-number>` | distinguir primeira tentativa de retry |
| `<aws.scheduler.schedule-arn>` | qual agendador disparou, quando houver mais de um |

🔴 **A regra que mantém o desenho de pé: isso é para LOGAR, nunca para DECIDIR.** No dia em que
alguém escrever `if (horario == hoje)`, o handler passou a depender do agendador — e o gatilho
atrasado ou reprocessado mente. Quem decide o "quando" continua sendo a consulta ④, pelo relógio
dela. Se adotar, o handler segue recebendo `String`: o campo entra no log e morre ali.

---

📌 **Duas DLQs, dois problemas diferentes.** Reusar a mesma fila para as duas coisas funciona e
economiza um recurso — mas confunde o diagnóstico: mensagem lá dentro pode ser "o Scheduler não
conseguiu publicar" ou "o handler falhou 3 vezes". Se preferir separar, crie
`<slug>_scheduler_dlq` e dê `sqs:SendMessage` nela também para a role do agendador.

## 6️⃣ `outputs.tf` — o que a aplicação precisa saber

```hcl
output "<slug>_queue_url" {
  description = "URL da fila de gatilho — vai na configuração do listener ③."
  value       = aws_sqs_queue.<slug>.url
}

output "<slug>_queue_arn" {
  description = "ARN da fila de gatilho."
  value       = aws_sqs_queue.<slug>.arn
}

output "<slug>_schedule_arn" {
  description = "ARN do agendador — útil em alarme e em ticket de incidente."
  value       = aws_scheduler_schedule.<slug>.arn
}
```

📌 A URL é o que fecha o ciclo com o código: é o valor que entra na configuração do `<pacote>`
que ouve a fila. Sem output, alguém vai copiar a URL do console — e ela vira constante em
`application.yml`, que é como se perde o rastro entre infra e aplicação.

---

## ⏰ A expressão cron — o campo que morde

O cron do EventBridge tem **6 campos** (o cron do Unix tem 5), e **um dos campos de dia tem de
ser `?`** — não os dois `*`:

```
cron(minuto hora dia-do-mês mês dia-da-semana ano)
      0      3    *          *   ?             *
```

| Quero | Expressão | Nota |
|---|---|---|
| todo dia às 03:00 | `cron(0 3 * * ? *)` | ✅ o desta receita |
| todo dia às 02:30 | `cron(30 2 * * ? *)` | |
| só dias úteis, 03:00 | `cron(0 3 ? * MON-FRI *)` | agora o `?` troca de campo |
| a cada 15 min (dev) | `rate(15 minutes)` | `rate` ignora o timezone |
| uma vez, para testar | `at(2026-09-04T03:00:00)` | sem timezone no texto; use `schedule_expression_timezone` |

🔴 **`cron(0 3 * * * *)` não é erro de digitação — é erro de validação.** A AWS recusa: você não
pode restringir dia-do-mês **e** dia-da-semana ao mesmo tempo. Um dos dois é `?`.

⚠️ **Por que 03:00 e não 00:00:** meia-noite é o horário em que todo job de toda equipe dispara.
Madrugada mais tarde (2h–4h) evita a fila de contenção e ainda deixa o resultado pronto antes do
expediente. Se o ciclo depende de dado fechado do dia anterior, 03:00 dá margem para o batch que
fecha esse dado.

📌 **`schedule_expression_timezone` também resolve horário de verão** — o Scheduler usa a base de
fusos da IANA. Com o cron em UTC, o dia em que o fuso mudar, o job anda uma hora e ninguém liga
os pontos.

---

## 🔴 O plan que você lê linha a linha

```bash
terraform fmt <ARQUIVO>
terraform init
terraform validate
terraform plan -var-file=inventories/hom/terraform.tfvars -out=tfplan
terraform show tfplan | grep -E "will be|must be|forces replacement"
```

| O que aparecer | Significado |
|---|---|
| `+ aws_sqs_queue`, `+ aws_iam_role`, `+ aws_scheduler_schedule` | ✅ é isto: tudo novo, nada tocado |
| qualquer `~` num recurso que você não escreveu | 🔴 **PARE.** Você colidiu com nome de recurso existente |
| `must be replaced` numa fila | 🔴 mensagens em voo se perdem — confira se mudou `name` |
| `Invalid resource type` no validate | provider abaixo de 4.56 — ver seção 🗺️ |

**Homologação primeiro.** Prod só depois de ver o ciclo rodar inteiro em hom.

## ✅ Verificar depois do apply

```bash
# 1. o agendador existe e está no estado esperado
aws scheduler get-schedule --name <PREFIXO>-<AGENDADOR> \
  --query '{estado:State, cron:ScheduleExpression, tz:ScheduleExpressionTimezone, alvo:Target.Arn}'

# 2. a role realmente publica na fila (teste direto, sem esperar a madrugada)
aws sqs send-message --queue-url <URL> --message-body '{}'

# 3. chegou alguma coisa?
aws sqs get-queue-attributes --queue-url <URL> \
  --attribute-names ApproximateNumberOfMessages ApproximateNumberOfMessagesNotVisible
```

| Campo | Esperado | Se vier diferente |
|---|---|---|
| `State` | `DISABLED` até a story fechar | `ENABLED` antes da hora = gatilho disparando sem consumidor |
| `ScheduleExpressionTimezone` | `America/Sao_Paulo` | vazio = está rodando em UTC |
| `Target.Arn` | ARN da fila de gatilho | apontar para a fila **de destino** ⑦ por engano é o erro clássico |

### Testar sem esperar a madrugada

Três caminhos, do mais barato ao mais completo:

1. **Publicar na fila na mão** (`send-message` acima) — testa ③④⑤⑥, ignora o agendador.
2. **Um schedule descartável**, que se apaga sozinho depois de disparar:
   ```bash
   aws scheduler create-schedule --name teste-gatilho-manual \
     --schedule-expression "at(2026-09-04T14:05:00)" \
     --schedule-expression-timezone "America/Sao_Paulo" \
     --flexible-time-window Mode=OFF \
     --action-after-completion DELETE \
     --target "Arn=<ARN_FILA>,RoleArn=<ARN_ROLE>,Input={}"
   ```
   🎯 `--action-after-completion DELETE` é o que evita deixar lixo na conta.
3. **`gatilho_cron = "rate(15 minutes)"` em dev** — só quando o ciclo já for idempotente.

📌 O caminho 2 testa exatamente o que o caminho 1 não testa: **a role funciona?** Se a permissão
estiver errada, o `send-message` na mão passa (é você quem publica) e o agendador falha calado.

### Onde olhar quando não chegar nada

| Métrica (namespace `AWS/Scheduler`, dimensão `ScheduleName`) | Leitura |
|---|---|
| `InvocationAttemptCount` | 0 = **não disparou**: cron, timezone ou `State` |
| `TargetErrorCount` | > 0 = disparou e **o alvo recusou**: quase sempre IAM ou KMS |
| `InvocationDroppedCount` | > 0 = esgotou o retry — veja a DLQ do agendador |

## 🔌 Ligar o agendador (o PR que fecha a story)

```diff
  # inventories/prod/terraform.tfvars
- gatilho_habilitado = false
+ gatilho_habilitado = true
```

O plan tem de mostrar **exatamente uma** mudança:

```
~ resource "aws_scheduler_schedule" "<slug>" {
    ~ state = "DISABLED" -> "ENABLED"
```

⚠️ **Se aparecer mais alguma coisa, alguém mexeu na infra pelo console.** Resolva o drift antes
de ligar — ligar o agendador em cima de estado divergente é descobrir os dois problemas ao mesmo
tempo, às 3h da manhã.

---

## As armadilhas

| # | Armadilha | O que acontece |
|---|---|---|
| 1 | `*` nos dois campos de dia | erro de validação na AWS. Um dos dois é `?` |
| 2 | esquecer `schedule_expression_timezone` | roda em UTC; 03:00 vira meia-noite local |
| 3 | achar que resource policy na fila basta | não basta: o Scheduler **assume role**, e sem ela nunca publica |
| 4 | fila com CMK sem `kms:GenerateDataKey` | falha silenciosa; só `TargetErrorCount` denuncia |
| 5 | `visibility_timeout` menor que o ciclo | mensagem reaparece com o handler rodando → **dia processado duas vezes** |
| 6 | agendador ligado antes do consumidor | mensagem acumula, expira em 1 dia, e o primeiro dia "some" |
| 7 | apontar o `target` para a fila de destino ⑦ | o producer nunca roda, e mensagens vazias vão para o consumidor final |
| 8 | sem `dead_letter_config` no target | gatilho não entregue evapora sem rastro |
| 8b | handler com payload **tipado** | `{}` não desserializa; falha vira DLQ e parece erro de infra |
| 9 | renomear `name` depois | replacement: fila recriada perde mensagens em voo |
| 10 | `flexible_time_window` com janela | dispara em minuto imprevisível — inútil quando o horário importa |

## O que não fazer

- ❌ **`@Scheduled` na aplicação** em vez do agendador. Com N réplicas, N execuções por dia — e o
  bug só aparece quando alguém escala o serviço.
- ❌ **Agendador chamando a aplicação direto** (Lambda, HTTP, ECS task). Some o desacoplamento, o
  retry e o lock que a fila dá de graça.
- ❌ **Ler o corpo para decidir qualquer coisa.** Atributo de contexto do Scheduler entra no log
  e morre ali. Ver "O corpo vazio".
- ❌ **Corpo de mensagem com data ou parâmetro.** No dia em que o gatilho atrasar ou for
  reprocessado, esse corpo mente. A consulta ④ decide o "quando" a partir do relógio dela.
- ❌ **Criar o schedule pelo console "só para testar em hom".** Vira drift, e o próximo apply o
  apaga — ou pior, deixa os dois disparando.
- ❌ **`resources = ["*"]`** na policy de envio. É uma role que só existe para publicar numa fila.
- ❌ **Ligar em prod no mesmo PR que cria a infra.** Separar em dois PRs custa cinco minutos e é o
  que permite reverter só o disparo.

## Se não der para usar o Scheduler

Provider travado abaixo de 4.56, ou conta sem o serviço liberado: dá para fazer com **EventBridge
Rules** (o serviço antigo), com duas diferenças que importam:

```hcl
resource "aws_cloudwatch_event_rule" "<slug>" {
  name                = "${var.prefixo}-<AGENDADOR>"
  schedule_expression = "cron(0 6 * * ? *)"   # 🔴 SÓ UTC — 6h UTC = 3h em Brasília
  state               = var.gatilho_habilitado ? "ENABLED" : "DISABLED"
}

resource "aws_cloudwatch_event_target" "<slug>" {
  rule  = aws_cloudwatch_event_rule.<slug>.name
  arn   = aws_sqs_queue.<slug>.arn
  input = jsonencode({})
}

# 🔴 Aqui a permissão é resource policy NA FILA, não role — o inverso do Scheduler
resource "aws_sqs_queue_policy" "<slug>" {
  queue_url = aws_sqs_queue.<slug>.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "events.amazonaws.com" }
      Action    = "sqs:SendMessage"
      Resource  = aws_sqs_queue.<slug>.arn
      Condition = { ArnEquals = { "aws:SourceArn" = aws_cloudwatch_event_rule.<slug>.arn } }
    }]
  })
}
```

⚠️ **Rules não tem fuso horário.** Você passa a converter na mão e a lembrar do horário de verão —
é exatamente o problema que o Scheduler resolve. Use como plano B, e deixe anotado no PR que a
migração para `aws_scheduler_schedule` fica pendente.
