# Módulo Start Stop

Realiza o start/stop dos recursos no horário especificado. É preciso incluir a tag definida no módulo nos recursos que devem ser incluídos na rotina.

Padrão CRON utilizado: https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-cron-expressions.html

## Parâmetros do Módulo

| Parâmetro         | Opcional | Descrição                                                                                                                                               |
|-------------------|----------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| `region`          | | Região AWS na qual as configurações se aplicarão (por exemplo, "us-east-1").                                                                              |
| `start_cron`      | | Expressão de cron para o horário de início dos recursos.                                                                                                  |
| `stop_cron`       | | Expressão de cron para o horário de parada dos recursos.                                                                                                  |
| `manual_endpoint` | Sim | Se for true, realiza a criação de um API Gateway com dois endpoints **/start** e **/stop.** Invocar esses endpoints via POST tem o mesmo efeito que as ações executadas nos horários das CRONs. Para invocar esses endpoints, é necessário incluir a API Key que é criada junto do API Gateway. Essa chave pode ser resgatada a partir do console ou CLI.                                                                            |
| `tag`             | | Bloco para especificar a tag que identifica quais recursos devem ser afetados pelo start-stop                                                                        |
| `ecs`             | Sim | Se for true, Services do ECS que possuírem a tag serão incluídos no start-stop. Isso é feito setando o Desired Count para 0.                                                                  |
| `rds`             | Sim | Se for true, instâncias RDS e Aurora que possuírem a tag serão incluídos no start-stop. Isso é feito a partir do desligamento das instâncias.                                                                  |
| `ec2`             | Sim | Se for true, instâncias EC2 que possuírem a tag serão incluídos no start-stop. Isso é feito a partir do desligamento das instâncias.                                                     |
| `asg`             | Sim | Se for true, ASG que possuírem a tag serão incluídos no start-stop. Não é necessário incluir a tag de start-stop nas instâncias EC2 que pertencerem ao ASG. Isso é feito setando o Desired Count para 0. |

## Como funciona

EventBridge/API Gateway => SNS => Lambda => EC2/ECS/RDS/ASG

O EventBridge dispara nos horários definidos pelas CRONs e publica em um dos tópicos SNS (`start-topic` ou `stop-topic`). Cada recurso habilitado possui um par de Lambdas inscritas nesses tópicos, que executam as ações correspondentes nos recursos que possuem a tag configurada.

## Uso manual via API Gateway

Quando `manual_endpoint = true`, um API Gateway é criado com dois endpoints que permitem acionar o start/stop fora do horário agendado.

### Obtendo a URL e a API Key

**URL base** (via console ou CLI):
```bash
aws apigateway get-stages --rest-api-id <api-id> --query 'item[0].{stage:stageName}'
# URL: https://<api-id>.execute-api.<region>.amazonaws.com/dev
```

**API Key** (via CLI):
```bash
aws apigateway get-api-keys --include-values --query 'items[?name==`start-stop-key`].value' --output text
```

### Chamando os endpoints

**Start:**
```bash
curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/dev/start \
  -H "x-api-key: <api-key>"
```

**Stop:**
```bash
curl -X POST https://<api-id>.execute-api.<region>.amazonaws.com/dev/stop \
  -H "x-api-key: <api-key>"
```