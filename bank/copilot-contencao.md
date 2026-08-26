# Preâmbulo de contenção — impedir que o agente trave calado

> Cole **no começo** de qualquer prompt em modo agente. Resolve o caso em que ele fica 20
> minutos "editing X.kt" sem escrever nada e sem dizer o que está tentando fazer.
>
> **A causa quase sempre é a mesma:** ele resolveu reescrever um arquivo grande inteiro, erra
> um delimitador, o verificador de erros acusa, ele tenta de novo — e o laço se fecha sem
> nada aparecer na tela.

## O preâmbulo

```
Regras desta sessao. Valem para todas as respostas, sem excecao.

1. NAO edite arquivos. Mostre o diff no chat; eu aplico a mao.
2. Trabalhe SOMENTE nestes arquivos: <liste os arquivos>. Se precisar de qualquer outro,
   PARE e me peca pelo nome. Nao procure sozinho.
3. Antes de cada acao, escreva UMA linha dizendo o que vai fazer e por que.
4. Uma alteracao por vez. Termine, mostre o resultado, e espere minha resposta.
5. Se qualquer coisa te impedir de seguir -- arquivo grande demais, permissao, ambiguidade,
   erro que voce nao consegue resolver em duas tentativas -- diga isso em UMA frase e PARE.
   Nao tente contornar em silencio.
6. Nunca reescreva um arquivo inteiro. Mostre apenas as linhas que mudam, com 3 linhas de
   contexto antes e depois.
7. Se a resposta passar de 40 linhas, pare e me diga o que ficou faltando.
```

As regras que mais pesam são a **1**, a **5** e a **6**. A 6 sozinha já elimina a maior parte
dos travamentos.

## Quanto esperar

| Situação | Normal |
|---|---|
| edição pontual | segundos a ~1 min |
| arquivo grande reescrito | 1–3 min |
| verificador de erros iterando | **é sintoma, não etapa** |

**Teto: 5 minutos.** Pare antes disso se ele voltar ao mesmo arquivo pela terceira vez, se o
verificador de erros disparar em sequência, ou se nada aparecer por ~1 minuto.

## Quando travar

```bash
git diff --stat      # o que ele mexeu
git diff             # como mexeu
```

Fique com o que presta; o resto, `git checkout -- <arquivo>`.

⚠️ **Commit antes de cada execução.** É o que transforma "será que ele quebrou algo?" numa
verificação de dez segundos. Commit é local — não vai para o servidor, não vira PR, e dá
para esmagar tudo num só antes de subir.
