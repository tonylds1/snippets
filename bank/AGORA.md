# AGORA — o prompt do momento

> Arquivo vivo, sem data no nome: **sempre contém o único prompt a executar agora.**
> Mesma URL toda vez.
>
> ```bash
> curl -O https://raw.githubusercontent.com/tonylds1/snippets/main/bank/AGORA.md
> ```

---

## Passo atual: ⑤ índice — **descobrir quem cria índices neste projeto**

**Onde estamos:** a revisão voltou com 5 itens NÃO CONFORME — e os quatro últimos são sintoma
do primeiro. **Ninguém cria o índice.** Projeção, capacidade, corrida entre réplicas e status
assíncrono são todos propriedades da chamada de criação, que não existe.

**O que já está certo:** a constante tem dono único, os tipos das chaves estão corretos, os
atributos são nuláveis (nasce esparso) e nada os grava ainda (nasce vazio). O modelo está bom.

**Por que este prompt e não "escreva a criação":** escrever do zero é adivinhar. O que criou a
**tabela** é quase certamente o que deve criar o **índice** — e a resposta decide se isto é
trabalho de código ou dependência de outro repositório.

**O que esperar:** cinco respostas com arquivo e linha, no chat, em segundos. Nenhum código.

### Colar isto inteiro

```
Regras desta resposta, sem excecao:
- NAO edite nenhum arquivo. Responda no chat.
- NAO gere codigo nesta resposta.
- Se nao encontrar algo, escreva "nao encontrei" em vez de supor ou propor.
- Limite de 40 linhas vale SO para a parte descritiva (listas, respostas, analise).
  Diff de correcao NAO conta no limite: mostre o diff inteiro sempre.

Tarefa: descobrir COMO este projeto cria tabelas e indices no DynamoDB hoje. Nada alem disso.

Responda citando arquivo e linha:

1. Onde a tabela do DynamoDB e criada? Procure por CreateTable, createTable, UpdateTable,
   aws_dynamodb_table, migration, bootstrap, script de inicializacao. Se nao houver NADA
   no repositorio que crie a tabela, diga isso explicitamente.
2. Existe algum GSI ja criado neste projeto? Se sim, mostre o trecho INTEIRO que o cria --
   ele e o molde a seguir.
3. Existe pasta ou arquivo de Terraform neste repositorio? Liste o que tem dentro.
4. O projeto usa DynamoDbEnhancedClient. Existe alguma chamada a createTable() em qualquer
   lugar: startup, migration, teste de integracao, configuracao de LocalStack? Mostre.
5. Nos testes, como a tabela e criada para os testes de integracao? Mostre o trecho inteiro.
```

O item **5** costuma ser o mais revelador: a montagem da tabela nos testes de integração é
onde o time descreve tabela e índices por extenso.

---

## O que fazer com a resposta

| Resposta | Significa | Próximo passo |
|---|---|---|
| achou código que cria tabela/índice | **(a)** o padrão existe | escrever a criação seguindo aquele molde — é código, sai hoje |
| não achou nada, e há Terraform | **(b)** o índice é do repo de infra | não é código: é dependência de outro repositório e de outro dono |
| achou só nos testes | ambíguo | o molde serve de referência, mas alguém tem de criar em hom |

⚠️ **Se for (b)**, isto deixa de caber no seu dia: outro repositório, outro fluxo de aprovação.
Descobrir hoje é melhor que descobrir na hora do deploy.

---

## Depois deste

1. Criar o índice pelo caminho que a resposta indicar → compilar → **commit**
2. Subir para homologação: `IndexStatus` = `ACTIVE`, `ItemCount` = `0`
3. Próxima peça: **④ + ⑥**, em [202608261217-sqs-producer-indice.md](202608261217-sqs-producer-indice.md)

Mapa das peças: [INVENTARIO.md](INVENTARIO.md)
