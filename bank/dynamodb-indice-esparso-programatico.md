# Criar índice esparso (GSI) por código, seguindo o padrão do projeto — prompt em 3 etapas

> Use quando os índices do projeto **não** são criados por Terraform, e sim programaticamente
> (classe de configuração, script de migração, SDK no startup). O prompt manda o modelo
> **descobrir o padrão que já existe** antes de escrever qualquer linha.
>
> **O erro que isso evita:** sem instrução de inspeção, o modelo gera a receita mais comum da
> internet — um `CreateTableRequest` novo, ou Terraform — e você descobre que não encaixa só
> depois de colar. Pior: ele escolhe a projeção do índice por conta própria.

## ⚠️ A decisão que vale revisar antes de rodar

**O nome das chaves do índice.** Chave de GSI não se renomeia: errou, é índice novo mais
migração — e o nome errado já estará espalhado pelo código e pelos dados gravados.
**Revise com alguém do time antes de criar.** São 5 minutos que compram irreversibilidade.

O resto é reversível. Inclusive a **projeção**: ela de fato não se altera, mas desfazer é
apagar e recriar o índice — e um índice esparso, que nasce vazio e cresce devagar, é rápido
de recriar. A tabela nunca é tocada, porque o índice é derivado dela.

## ⚠️ Criar VAZIO

Índice esparso só inclui itens que **têm** os atributos da chave. Se nenhum item tiver ainda,
o índice nasce instantâneo. Se você popular primeiro, vira backfill — caro e demorado.
**Ordem certa: criar o índice → só depois começar a gravar os atributos.**

## Preencher antes de colar

```
<TABELA>   = nome da tabela
<PK_INDEX> = atributo que só existe nos itens elegíveis (é o que torna o índice esparso)
<SK_INDEX> = atributo de data, string ISO-8601, usado para comparar "menor ou igual a agora"
<INDICE>   = nome do índice
```

## O prompt

No Copilot Chat, comece com `@workspace` — sem isso ele só enxerga os arquivos abertos.

```
Contexto: projeto Kotlin + Spring Boot que usa DynamoDB. Existe uma pasta chamada "dynamo"
com as definicoes de tabelas e indices. Os indices NAO sao criados por Terraform, sao
criados programaticamente. Preciso adicionar UM indice secundario global (GSI) esparso.

NAO gere codigo ainda. Trabalhe em 3 etapas e pare ao fim de cada uma, esperando eu responder.

=== ETAPA 1 - RELATORIO (sem codigo) ===
Inspecione a pasta "dynamo" e o resto do projeto. Responda citando arquivo e linha:
1. Como as tabelas e indices sao definidos? (classe de configuracao, script de migracao,
   chamada de SDK, anotacoes do Enhanced Client, outro)
2. Qual biblioteca e versao: AWS SDK v1, v2, ou DynamoDbEnhancedClient?
3. Em que momento a criacao roda: no startup da aplicacao, num script separado, num job?
4. Ja existe algum GSI criado nesse padrao? Se sim, mostre o trecho inteiro - ele e o
   molde a seguir.
5. A tabela usa capacidade on-demand (PAY_PER_REQUEST) ou provisionada? Se provisionada,
   como a capacidade dos indices existentes e definida?
6. Se a criacao roda no startup: o que acontece com varias replicas subindo ao mesmo tempo?
   Existe tratamento de ResourceInUseException ou de indice ja existente?
7. Como o projeto testa esse tipo de codigo hoje? (LocalStack, DynamoDBLocal, testcontainers)

Pare aqui.

=== ETAPA 2 - PROPOSTA (sem escrever arquivos) ===
Com base na ETAPA 1, me mostre:
- em que arquivo o indice novo entra e por que aquele arquivo
- o trecho exato que voce pretende adicionar
- se a criacao e assincrona e como o codigo saberia que o indice ficou ACTIVE
- qualquer premissa sua que a ETAPA 1 nao confirmou, listada como pergunta

Pare aqui.

=== ETAPA 3 - CODIGO ===
So depois que eu aprovar a ETAPA 2.

=== REQUISITOS DO INDICE (valem nas 3 etapas) ===
- Nome do indice: <INDICE>
- Chave de particao: <PK_INDEX>, tipo String. E um bucket de shard com 10 valores possiveis
  ("0" a "9"), para espalhar a escrita e permitir consulta em paralelo pelos 10 shards.
- Chave de ordenacao: <SK_INDEX>, tipo String, data em ISO-8601, para consulta por intervalo.
- O indice e ESPARSO de proposito: itens que nao tem esses dois atributos NAO entram nele.
  Nao adicione valor padrao para esses atributos em lugar nenhum.
- Projecao: KEYS_ONLY. Decisao fechada, nao proponha INCLUDE nem ALL e nao pergunte sobre
  isso. Quem consulta este indice so precisa descobrir QUAIS itens estao vencidos; os campos
  de negocio sao lidos direto da tabela, mais adiante no fluxo, por outro componente.
- Capacidade: exatamente o mesmo modo da tabela.

=== O QUE NAO FAZER ===
- Nao escreva producer, listener, use case, scheduler nem consulta ao indice. Escopo desta
  tarefa e SO a criacao do indice.
- Nao gere backfill, migracao de dados, nem qualquer codigo que popule os atributos.
- Nao altere a definicao da tabela alem de acrescentar o indice.
- Nao troque a biblioteca nem introduza dependencia nova.
- Nao use Scan em nenhuma hipotese.
- Nao renomeie nada que ja existe.
```

## Verificar em homologação depois do deploy

A criação de GSI é **assíncrona** — a chamada retorna antes de o índice existir de verdade.

```bash
aws dynamodb describe-table --table-name <TABELA> \
  --query 'Table.GlobalSecondaryIndexes[?IndexName==`<INDICE>`].[IndexName,IndexStatus,ItemCount]'
```

O que você quer ver:

| Campo | Esperado | Se vier diferente |
|---|---|---|
| `IndexStatus` | `ACTIVE` | `CREATING` = ainda subindo, espere e repita |
| `ItemCount` | `0` | diferente de 0 = **algum item já tinha os atributos**, o índice não nasceu vazio |
| `Backfilling` | ausente ou `false` | `true` = está copiando dados, ou seja, não estava vazio |

`ItemCount` é atualizado com atraso (a cada ~6h), então logo após a criação ele é confiável
para "nasceu vazio", mas não serve para acompanhar crescimento em tempo real.
