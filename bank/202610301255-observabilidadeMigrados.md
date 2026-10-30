# Diagnóstico de observabilidade — Kotlin / Spring Boot

Atue como um engenheiro de software sênior especializado em Kotlin, Spring Boot, observabilidade, logs estruturados e sistemas distribuídos.

## Objetivo

Avaliar a observabilidade existente neste repositório antes da implementação de um novo fluxo de confirmação de go-live.

Este prompt é executado separadamente no backend e no BFF. Avalie apenas o papel do repositório atual:

- Backend: `PUT /migrados/{cnpj}/golive/acknowledgement`, que atualiza o atributo `notifiedAt` do cliente no DynamoDB.
- BFF: `PUT /migration/go-live/acknowledgement`, que identifica o cliente autenticado e chama o backend.

Neste primeiro momento, priorize o diagnóstico. Não implemente mudanças até apresentar os resultados e um plano.

## Escopo da análise

Examine apenas os componentes relevantes para:

1. Logging: SLF4J, Logback, JSON logging e campos estruturados.
2. Propagação de MDC, correlationId, traceId e demais identificadores existentes.
3. Tracing distribuído entre controller, service e clientes HTTP.
4. Integração existente com Datadog, Micrometer, OpenTelemetry ou outras ferramentas.
5. Tratamento de erros, exceções e logs.
6. Instrumentação de chamadas HTTP e acesso ao DynamoDB.
7. Compatibilidade de contexto com coroutines, caso utilizadas.

Não execute comandos Git, incluindo `git log`, `git status`, `git diff`, `git branch` ou `git merge-base`.

Não faça uma varredura completa do repositório. Comece pelas configurações e classes diretamente relacionadas.

## Avaliação obrigatória

Identifique:

- Quais ferramentas de observabilidade já existem.
- Quais campos são automaticamente adicionados aos logs.
- Como os IDs de correlação são recebidos e propagados.
- Se os logs podem ser correlacionados com traces distribuídos.
- Se existem logs duplicados ou informações sensíveis.
- Se a instrumentação existente já cobre os novos endpoints sem alterações adicionais.
- Quais lacunas realmente impediriam o diagnóstico de uma falha.

Não registre CNPJ completo, tokens, credenciais, dados bancários ou informações pessoais em logs.

O CNPJ permanece na URL do backend nesta entrega. Verifique se ele pode vazar em:

- Access logs.
- Tags de span, incluindo `http.url`.
- Resource name do APM, que deve usar o path templated.
- Logger do cliente HTTP.
- Mensagens de exceção, como `FeignException`.

Se houver vazamento, recomende a menor mitigação possível, sem alterar o path.

## Observabilidade do novo fluxo

Avalie como registrar, de maneira adequada:

- Início e conclusão da operação.
- Resultado da confirmação, quando útil.
- Duração da operação.
- Falhas de integração ou persistência.
- Identificadores de correlação e tracing já existentes.

Evite logs redundantes, principalmente mensagens que apenas repetem dados disponíveis no tracing automático.

Não registre cada chamada bem-sucedida em múltiplas camadas sem justificativa.

## Entrega do diagnóstico

Apresente uma tabela com:

- Componente analisado.
- Implementação existente.
- Lacuna identificada.
- Risco.
- Recomendação.
- Arquivos potencialmente afetados.

Classifique cada recomendação como:

- Necessária para o novo endpoint.
- Melhoria opcional.
- Não necessária porque já existe cobertura.

## Próxima etapa

Apresente um plano mínimo de implementação apenas para as lacunas necessárias ao novo fluxo.

Escreva o plano em um bloco autocontido, pronto para ser colado na seção de observabilidade do prompt de implementação deste repositório.

Não adicione dependências, interceptors, filtros, bibliotecas de logging ou configurações sem demonstrar que são necessários.

Não modifique o código nesta etapa.

Aguarde minha aprovação do plano antes de implementar.