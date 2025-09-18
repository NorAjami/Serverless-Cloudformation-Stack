# Serverless CloudFormation Stack

En skalbar serverless applikation byggd med AWS CloudFormation som Infrastructure as Code (IaC). Projektet demonstrerar en komplett REST API med automatiserad datahantering utan serverhantering.

## 🏗️ Arkitektur

```
Internet → API Gateway → Lambda Function → DynamoDB
```

**Komponenter:**
- **API Gateway**: REST API endpoint för HTTP-requests
- **Lambda**: Serverless Python-funktion för affärslogik
- **DynamoDB**: NoSQL-databas för datalagring
- **IAM Roller**: Säkra behörigheter mellan tjänster
- **CloudFormation**: Infrastructure as Code för reproducerbar deployment

## 🚀 Funktioner

- ✅ **GET /data**: Hämta all lagrad data
- ✅ **POST /data**: Skapa ny data med automatisk timestamp
- ✅ **Skalbar**: Hanterar från 0 till miljontals requests automatiskt
- ✅ **Säker**: IAM-roller med minimal behörighet
- ✅ **CORS-aktiverat**: Kan anropas från webbapplikationer
- ✅ **Felhantering**: Robusta felmeddelanden

## 📦 Installation & Deployment

### Förutsättningar
```bash
# AWS CLI installerat och konfigurerat
aws configure

# Verifiera att du har rätt konto
aws sts get-caller-identity
```

### Deploy Stack
```bash
# Klona projektet
git clone https://github.com/NorAjami/Serverless-Cloudformation-Stack.git
cd Serverless-Cloudformation-Stack

# Skapa CloudFormation stack
aws cloudformation create-stack \
  --stack-name serverless-stack \
  --template-body file://serverless-stack.yaml \
  --capabilities CAPABILITY_IAM

# Vänta på att stacken skapas (2-3 minuter)
aws cloudformation describe-stacks \
  --stack-name serverless-stack \
  --query 'Stacks[0].StackStatus'

# Hämta API URL när klar
aws cloudformation describe-stacks \
  --stack-name serverless-stack \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiUrl`].OutputValue' \
  --output text
```

## 🧪 Testning

### Hämta data (GET)
```bash
curl https://YOUR-API-ID.execute-api.eu-west-1.amazonaws.com/prod/data
```

### Skapa data (POST)
```bash
curl -X POST https://YOUR-API-ID.execute-api.eu-west-1.amazonaws.com/prod/data \
  -H "Content-Type: application/json" \
  -d '{"id": "user123", "data": "Hello Serverless World!"}'
```

### Exempel Response
```json
{
  "message": "Data created successfully",
  "item": {
    "id": "user123",
    "data": "Hello Serverless World!",
    "timestamp": "2025-09-18T15:30:45.123456"
  }
}
```

## 🔧 CloudFormation Resurser

| Resurs | Typ | Beskrivning |
|--------|-----|-------------|
| `MyDataTable` | DynamoDB::Table | Huvuddatabas med pay-per-request billing |
| `LambdaExecutionRole` | IAM::Role | Säkra behörigheter för Lambda |
| `DataHandlerFunction` | Lambda::Function | Python 3.11 funktion för datahantering |
| `ServerlessAPI` | ApiGateway::RestApi | REST API endpoint |
| `DataResource` | ApiGateway::Resource | /data route |
| `GetDataMethod` | ApiGateway::Method | GET metod |
| `PostDataMethod` | ApiGateway::Method | POST metod |
| `ApiDeployment` | ApiGateway::Deployment | Publicerar API till prod stage |
| `LambdaApiGatewayPermission` | Lambda::Permission | Tillåter API Gateway anropa Lambda |

## 🏷️ Outputs

Efter successful deployment får du:
- **TableName**: Namnet på DynamoDB-tabellen
- **LambdaFunctionName**: Namnet på Lambda-funktionen  
- **ApiUrl**: Den publika URL:en för ditt REST API

## 🗑️ Rensa upp

```bash
# Ta bort hela stacken (sparar pengar!)
aws cloudformation delete-stack --stack-name serverless-stack

# Verifiera att den är borttagen
aws cloudformation describe-stacks --stack-name serverless-stack 2>/dev/null || echo "Stack deleted successfully"
```

## 💰 Kostnader

- **DynamoDB**: Pay-per-request (gratis för små volymer)
- **Lambda**: Första 1M requests/månad gratis
- **API Gateway**: Första 1M API calls/månad gratis
- **Perfekt för utveckling och testning!**

## 🔍 Troubleshooting

### Stack creation failed?
```bash
# Se detaljerade fel
aws cloudformation describe-stack-events \
  --stack-name serverless-stack \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`]'
```

### API returnerar 500-fel?
```bash
# Kolla Lambda logs
aws logs tail /aws/lambda/ServerlessDataHandler --follow
```

### Permissions errors?
- Kontrollera att `--capabilities CAPABILITY_IAM` används
- Verifiera AWS credentials med `aws sts get-caller-identity`

## 📚 Lärdomar

Detta projekt demonstrerar:
- **Infrastructure as Code** med CloudFormation
- **Serverless arkitektur** med AWS Lambda
- **REST API design** med API Gateway
- **NoSQL datamodellering** med DynamoDB
- **Säkerhet** med IAM roller och policies
- **Cost optimization** med pay-per-use modeller

## 🤝 Utveckling

Projektet är strukturerat för enkel vidareutveckling:
- Lägg till fler HTTP-metoder (PUT, DELETE)
- Implementera autentisering med API Keys
- Lägg till datavalidering och business logic
- Integrera med andra AWS-tjänster

## 📝 Teknisk Stack

- **Infrastructure**: AWS CloudFormation (YAML)
- **Compute**: AWS Lambda (Python 3.11)
- **Database**: AWS DynamoDB (NoSQL)
- **API**: AWS API Gateway (REST)
- **Security**: AWS IAM
- **Version Control**: Git + GitHub

---
