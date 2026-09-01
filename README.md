# snippets

Trechos de referência para capturar rápido na máquina de trabalho, onde o ambiente é
travado e não dá para instalar nem experimentar à vontade.

> ## ⚠️ Repositório PÚBLICO — regra antes de commitar
>
> Nomes de tabela, campos, valores e volumes aqui são **fictícios**, sempre.
>
> **Nunca** commitar: regra de negócio real, número de produção, nome de sistema
> interno, nome de empregador, endpoint ou domínio corporativo.
>
> Material real fica fora deste repositório. Na dúvida, não commite — repositório
> público não se desfaz: clones e caches sobrevivem ao `git rm`.

## Começar aqui

**[bank/AGORA.md](bank/AGORA.md)** — sempre contém o único prompt a executar no momento.
Mesma URL toda vez:

```bash
curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
```

## Convenção de nome

Arquivos de receita começam com **`AAAAMMDDHHMM-`** (data de criação). Assim a listagem sai
em ordem cronológica e o último da lista é o mais recente.

`INVENTARIO.md` fica sem data de propósito: é o índice vivo, atualizado o tempo todo — uma
data ali seria mentira.

⚠️ Renomear muda a raw URL. Se você tinha um `curl` guardado com o nome antigo, pegue o novo
aqui embaixo.

## Conteúdo

| Pasta | Assunto |
|---|---|
| [bank/](bank/) | DynamoDB: exploração de tabela, índice esparso, escritas condicionais · SQS: producer/listener com falha por item · encaixar código gerado em projeto existente · remover código gerado errado · value objects de dinheiro (centavos) e percentual |

### Fluxo agendado (índice esparso → fila → consumidor)

Quatro receitas encadeadas. Comece pelo inventário — ele diz qual peça cada uma entrega:

0. [bank/copilot-contencao.md](bank/202608261829-copilot-contencao.md) — preâmbulo que impede o agente de travar calado
1. [bank/INVENTARIO.md](bank/INVENTARIO.md) — as 9 peças, a ordem de construção, as regras que atravessam tudo
2. [bank/dynamodb-indice-esparso-programatico.md](bank/202608261122-dynamodb-indice-esparso-programatico.md) — ⑤ o índice, por código
2b. [bank/dynamodb-indice-esparso-terraform.md](bank/202608281013-dynamodb-indice-esparso-terraform.md) — ⑤ o índice, por Terraform (quando a anotação não cria)
3. [bank/sqs-producer-indice.md](bank/202608261217-sqs-producer-indice.md) — ④ ⑥ consulta e producer
4. [bank/sqs-handler-mensagem-magra.md](bank/202608261425-sqs-handler-mensagem-magra.md) — ⑧ o consumidor

## Capturar um arquivo pelo terminal

```bash
curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/202608241731-dynamodb-queries.md
```
