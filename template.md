AWSTemplateFormatVersion: '2010-09-09'
Description: 'Skalbar serverless miljö med DynamoDB - Steg 1'

Resources:
  # DynamoDB tabell för att lagra data
  MyDataTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ServerlessData
      BillingMode: PAY_PER_REQUEST  # Skalbar, betalar bara för vad du använder
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S  # S = String
      KeySchema:
        - AttributeName: id
          KeyType: HASH  # Primary key

  # IAM-roll för Lambda-funktionen
  LambdaExecutionRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: DynamoDBAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - dynamodb:PutItem
                  - dynamodb:GetItem
                  - dynamodb:UpdateItem
                  - dynamodb:DeleteItem
                  - dynamodb:Query
                  - dynamodb:Scan
                Resource: !GetAtt MyDataTable.Arn

  # Lambda-funktion för att hantera data
  DataHandlerFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: ServerlessDataHandler
      Runtime: python3.11
      Handler: index.lambda_handler
      Role: !GetAtt LambdaExecutionRole.Arn
      Environment:
        Variables:
          TABLE_NAME: !Ref MyDataTable
      Code:
        ZipFile: |
          import json
          import boto3
          import os
          from datetime import datetime
          
          dynamodb = boto3.resource('dynamodb')
          table = dynamodb.Table(os.environ['TABLE_NAME'])
          
          def lambda_handler(event, context):
              try:
                  # Hantera både direkta anrop och API Gateway anrop
                  http_method = event.get('httpMethod', 'GET')
                  
                  if http_method == 'GET':
                      # Läs data
                      response = table.scan()
                      return {
                          'statusCode': 200,
                          'headers': {
                              'Content-Type': 'application/json',
                              'Access-Control-Allow-Origin': '*'
                          },
                          'body': json.dumps(response['Items'])
                      }
                  
                  elif http_method == 'POST':
                      # Skriv data
                      body = json.loads(event['body'])
                      item = {
                          'id': body['id'],
                          'data': body.get('data', ''),
                          'timestamp': datetime.now().isoformat()
                      }
                      
                      table.put_item(Item=item)
                      
                      return {
                          'statusCode': 201,
                          'headers': {
                              'Content-Type': 'application/json',
                              'Access-Control-Allow-Origin': '*'
                          },
                          'body': json.dumps({'message': 'Data created successfully', 'item': item})
                      }
                  
                  else:
                      return {
                          'statusCode': 405,
                          'body': json.dumps({'error': 'Method not allowed'})
                      }
                      
              except Exception as e:
                  print(f"Error: {str(e)}")
                  return {
                      'statusCode': 500,
                      'body': json.dumps({'error': 'Internal server error'})
                  }

  # API Gateway REST API
  ServerlessAPI:
    Type: AWS::ApiGateway::RestApi
    Properties:
      Name: ServerlessDataAPI
      Description: API för att hantera serverless data
      EndpointConfiguration:
        Types:
          - REGIONAL

  # API Gateway Resource (/data)
  DataResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      RestApiId: !Ref ServerlessAPI
      ParentId: !GetAtt ServerlessAPI.RootResourceId
      PathPart: data

  # GET Method
  GetDataMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ServerlessAPI
      ResourceId: !Ref DataResource
      HttpMethod: GET
      AuthorizationType: NONE
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${DataHandlerFunction.Arn}/invocations'

  # POST Method
  PostDataMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      RestApiId: !Ref ServerlessAPI
      ResourceId: !Ref DataResource
      HttpMethod: POST
      AuthorizationType: NONE
      Integration:
        Type: AWS_PROXY
        IntegrationHttpMethod: POST
        Uri: !Sub 'arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${DataHandlerFunction.Arn}/invocations'

  # API Gateway Deployment
  ApiDeployment:
    Type: AWS::ApiGateway::Deployment
    DependsOn:
      - GetDataMethod
      - PostDataMethod
    Properties:
      RestApiId: !Ref ServerlessAPI
      StageName: prod

  # Lambda Permission för API Gateway
  LambdaApiGatewayPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref DataHandlerFunction
      Action: lambda:InvokeFunction
      Principal: apigateway.amazonaws.com
      SourceArn: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ServerlessAPI}/*/*'

Outputs:
  # Visa tabellnamnet när stacken är klar
  TableName:
    Description: 'Namnet på DynamoDB-tabellen'
    Value: !Ref MyDataTable
    Export:
      Name: !Sub '${AWS::StackName}-TableName'
  
  # Visa Lambda-funktionens namn
  LambdaFunctionName:
    Description: 'Namnet på Lambda-funktionen'
    Value: !Ref DataHandlerFunction
  
  # API Gateway URL
  ApiUrl:
    Description: 'URL till ditt API'
    Value: !Sub 'https://${ServerlessAPI}.execute-api.${AWS::Region}.amazonaws.com/prod/data'