# Collection de Automatização de Requests

Esta collection contém funções reutilizáveis para automatizar chamadas de API relacionadas a eventos de autorização, envio de eventos para RabbitMQ, polling de status e consulta de transações, além de validação em sistemas externos (como Mambu). Utiliza variáveis de coleção para customização dinâmica e controle granular do fluxo.

---

## 1. Variáveis de Coleção

| Nome da Variável           | Tipo      | Valor Exemplo / Inicial              | Descrição / Objetivo                                                                    |
|---------------------------|-----------|------------------------------------|----------------------------------------------------------------------------------------|
| `rabbitToken`             | string    | (token de autenticação)              | Token para autenticação ao enviar eventos para a fila RabbitMQ                         |
| `correlationId`           | string    | (UUID ou string única)               | Identificador único para correlacionar requisições e eventos                          |
| `authorizationId`         | string    | ""                                 | ID da autorização para operações específicas                                          |
| `pismoSignToken`          | string    | ""                                 | Token para assinatura digital na integração com Pismo                                |
| `randomAmount`            | number    | ""                                 | Valor para simulação de transações financeiras                                        |
| `UrlRabbit`               | string    | "http://rabbitmq.local"              | URL base para API de eventos RabbitMQ                                                 |
| `UrlJudge`                | string    | "http://judge.api"                   | URL do serviço de consulta de transações Judge                                        |
| `byPassSecretSTG`         | string    | "1099a7..."                         | Segredo para bypass em ambiente de staging                                            |
| `authorizationCategory`   | array JSON| (exemplo: [{description: "desc"}, ...]) | Categoria(s) de autorização, usada para enviar eventos distintos                      |
| `pan`                     | string    | ""                                 | Número do cartão (PAN), opcional para eventos                                         |
| `debitProgramId`          | string    | ""                                 | ID do programa de débito usado em eventos                                             |
| `transactionType`         | string    | ""                                 | Tipo da transação                                                                    |
| `validateMambuFunction`   | string JS | Código da função para validação em Mambu | Função serializada guardada em variável para validação do resultado no Mambu          |
| `findAuthorizationTransaction` | string JS | Código da função para buscar transação | Função serializada para busca de transação no Judge                                  |
| `sendNetworkEvent`        | string JS | Código da função para envio de eventos | Função serializada para envio de eventos na fila RabbitMQ                            |
| (Outras variáveis...)     | ...       | ...                                | ...                                                                                    |

---

## 2. Flags de Controle do Fluxo

As flags abaixo permitem ativar ou desativar etapas específicas do fluxo de execução:

| Flag                             | Tipo    | Descrição                                             | Valor Padrão |
|---------------------------------|---------|-------------------------------------------------------|--------------|
| `sendFirstNetworkEventEnabled`  | boolean | Controla o envio do primeiro evento para RabbitMQ     | `true`       |
| `sendClearingCancelationEnabled`| boolean | Controla o envio do evento de clearing/cancelamento  | `true`       |
| `getTransactionEnabled`          | boolean | Controla se a busca da transação no Judge será realizada | `true`       |
| `validateMambuEnabled`           | boolean | Controla a validação do resultado no sistema Mambu    | `true`       |

---

## 3. Funções Reutilizáveis e Fluxo Principal

### 3.1 Funções Reutilizáveis

As funções abaixo são extraídas das variáveis de coleção e criadas dinamicamente para promover reutilização e flexibilidade.

- **`sendNetworkEvent(params)`**  
Envia eventos para RabbitMQ. Parâmetros incluem dados do evento, IDs, tokens e payload da transação.

- **`findAuthorizationTransaction(params)`**  
Realiza consultas no serviço Judge para recuperar dados e status atualizados da transação.

- **`validateMambu(params)`**  
Executa a validação da transação no sistema Mambu, garantindo consistência dos dados.

---

### 3.2 Fluxo Principal (`main`)

O fluxo principal executa as seguintes etapas:

1. **Validação inicial da resposta:**  
    Verifica se a resposta atual é aprovada (`approve === true`) e se o status HTTP é esperado.

2. **Primeiro envio de evento na fila RabbitMQ:**  
    Se ativado pela flag `sendFirstNetworkEventEnabled`, envia o evento inicial com detalhes da autorização.

3. **Polling para atualização do status da transação:**  
    Executa tentativas iterativas (máximo de 30 tentativas, 2s entre cada) para buscar o status da transação na Judge, ignorando status pré-definidos.

4. **Envio de evento de clearing/cancelamento:**  
    Envia evento de clearing somente se o status da transação for válido (não ignorado) e se a flag `sendClearingCancelationEnabled` estiver habilitada.

5. **Busca final da transação:**  
    Obtém dados finais atualizados da transação usando `getTransaction`, se ativado.

6. **Validação no Mambu:**  
    Executa validação da transação no sistema Mambu, se a flag `validateMambuEnabled` estiver ativa.

---

## 4. Exemplos de Uso

### 4.1 Envio do Primeiro Evento na Fila

```javascript
const params = {
    pm,
    correlationId: correlationId, // gerado pelo uuid (ex: uuid.v4())
    destinationAddress: 'rabbitmq://localhost:0/Operational.Judge.Infrastructure.Strategies.Pismo.PayLoads:ProcessorAuthorizationEvent',
    messageNamespace: 'Operational.Judge.Infrastructure.Strategies.Pismo.PayLoads:ProcessorAuthorizationEvent',
    mti: '0100',
    conciliationType: 'DUAL_MESSAGE',
    authorizationCategory: authorizationCategory[1].description,
    authorizationResponseCode: '00',
    pan,
    requestPayload,
    exchangeName: 'pismo.authorization.events.judge.v2',
    caller: 'Mastercard',
    custom_response_code: ''
};
```

sendNetworkEvent(params)
.then(res => console.log('Evento enviado com sucesso:', res))
.catch(err => console.error('Erro ao enviar evento:', err));

---

## 5. Documentação dos Métodos Reutilizáveis

| Método | Parâmetros (`params`) | Retorno | Descrição Detalhada | Observações / Uso |
|--------|------------------------|---------|----------------------|--------------------|
| `sendNetworkEvent(params)` | - `pm`  <br> - `correlationId`  <br> - `destinationAddress`  <br> - `messageNamespace`  <br> - `mti`  <br> - `conciliationType`  <br> - `type`  <br> - `debitProgramCard`  <br> - `authorizationCategory`  <br> - `authorizationResponseCode`  <br> - `pan`  <br> - `requestPayload`  <br> - `exchangeName`  <br> - `caller`  <br> - `custom_response_code`  <br> - `adviceCode` | `Promise<string>` | Constrói e envia um evento de autorização para RabbitMQ. Utiliza dados da transação e prepara o payload conforme o esquema esperado pelo broker. Inclui metadados como `event_id`, `timestamp`, `org_id`. Envia via POST para a URL configurada em `environment`. | Pode ser invocada em qualquer lugar para disparar eventos para a fila RabbitMQ. Retorna uma `Promise` resolvida com confirmação ou rejeitada em erro. |
| `findAuthorizationTransaction(params)` | - `pm`  <br> - `fields` (com `nsu`, `authorization_code`, `retrieval_reference_number`)  <br> - `correlationId` | `Promise<object>` | Busca transação no serviço Judge usando dados da autorização e headers (`x-correlation-id`). Requisição GET para `/GetTransaction`. Retorna JSON com dados e status atualizados. | Usada no fluxo para polling do status da transação. Pode ser customizada conforme ambiente (Local, Sandbox, Staging). |
| `validateMambu(params)` | - `pm`  <br> - `aprovedOrReversedm`  <br> - `proxy` (cardId)  <br> - `authorizationId` | `Promise<object>` | Valida autorização consultando a API do Mambu via GET em `/api/cards/{proxy}/authorizationholds/{authorizationId}`. Inclui autenticação básica. Retorna dados da autorização para análise. | Usada para garantir que uma autorização foi registrada corretamente no sistema financeiro. |
| `sendJudgeTransactionRequest(params)` | - `pm`  <br> - `requestBody`  <br> - `correlationId` | `Promise<object>` | Envia POST para `/transactions` no serviço Judge, usado para reversões ou atualizações. Inclui headers de autenticação e bypass. Retorna resultado da operação. | Utilizada para envio de solicitações específicas como reversões ou atualizações no sistema Judge. |
