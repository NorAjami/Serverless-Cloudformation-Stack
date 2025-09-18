mitt-serverless-projekt/
├── serverless-stack.yaml
├── lambda/
└── README.md


1. Skapa en .yaml fil, lägg in kod för att DynamoDB tabell.

2. skapa en stack, via AWS CLI:
aws cloudformation create-stack \
  --stack-name min-serverless-stack \
  --template-body file://serverless-stack.yaml

3. verifiera att stacken skapades:
aws cloudformation describe-stacks --stack-name min-serverless-stack

Du ska se:
StackStatus: "CREATE_COMPLETE" (om den är klar)
StackStatus: "CREATE_IN_PROGRESS" (om den fortfarande skapas)

4. Se vad som händer i realtid (via CLI)

aws cloudformation describe-stack-events --stack-name min-serverless-stack

Detta visar alla händelser, senaste först.

4. Kolla Outputs (via CLI)
aws cloudformation describe-stacks --stack-name min-serverless-stack --query 'Stacks[0].Outputs'

5. Verifiera DynamoDB-tabellen (via CLI)
aws dynamodb list-tables

## lägg till lambda funktionen 
- iam roll, DataHandlerFunction (python, get post)

Låt oss verifiera att uppdateringen gick bra:
Kolla stackens status:
bashaws cloudformation describe-stacks --stack-name min-serverless-stack --query 'Stacks[0].StackStatus'
Du vill se: "UPDATE_COMPLETE"
Kolla att Lambda-funktionen skapades:
bashaws lambda list-functions --query 'Functions[?FunctionName==`ServerlessDataHandler`]'
Eller kolla alla outputs från din stack:
bashaws cloudformation describe-stacks --stack-name min-serverless-stack --query 'Stacks[0].Outputs'
Du ska nu se både TableName OCH LambdaFunctionName i outputs.
Testa Lambda-funktionen direkt:
När uppdateringen är klar kan vi testa om Lambda-funktionen fungerar:
bashaws lambda invoke \
  --function-name ServerlessDataHandler \
  --payload '{"httpMethod":"GET"}' \
  response.json && cat response.json
Detta borde returnera en tom lista [] eftersom tabellen är tom än så länge.

## API Gateway:
- Detta kommer göra det möjligt att anropa din Lambda via HTTP från webben, inte bara via AWS CLI.

1. API Gateway REST API: Skapar huvudAPI:et
2. Resource (/data): Skapar endpoint /data
3. GET & POST Methods: Hanterar HTTP GET och POST requests
4. Deployment: Publicerar API:et på "prod" stage
5. Lambda Permission: Låter API Gateway anropa din Lambda
6. API URL Output: Visar den publika URL:en du kan använda