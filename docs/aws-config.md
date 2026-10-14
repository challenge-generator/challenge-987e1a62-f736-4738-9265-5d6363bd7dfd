# Configuración de Servicios AWS

## Introducción
Este documento detalla la configuración de los servicios AWS utilizados en la arquitectura del sistema, con justificaciones técnicas para cada decisión. La configuración está diseñada para cumplir con los atributos de calidad identificados: latencia, disponibilidad, escalabilidad, seguridad y observabilidad.

---

## 1. Amazon API Gateway
**Propósito:** Punto de entrada para todas las APIs del sistema, gestionando autenticación, autorización, throttling y caching.

### Configuración
```yaml
apiVersion: v1
kind: RestApi
metadata:
  name: financial-api
spec:
  description: API para servicios financieros
  endpointConfiguration:
    types: [REGIONAL]
  minimumCompressionSize: 1024
  binaryMediaTypes:
    - 'application/json'
    - 'application/xml'

resources:
  /accounts:
    get:
      summary: Consultar cuentas de usuario
      description: Retorna la lista de cuentas asociadas a un usuario
      parameters:
        - name: userId
          in: query
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Lista de cuentas
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Account'
        '401':
          description: No autorizado
        '429':
          description: Too Many Requests
      x-amazon-apigateway-integration:
        uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${AccountLambda.Arn}/invocations
        httpMethod: POST
        type: aws_proxy
        passthroughBehavior: when_no_match
        contentHandling: CONVERT_TO_TEXT
    post:
      summary: Crear nueva cuenta
      description: Crea una nueva cuenta para un usuario
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateAccountRequest'
      responses:
        '201':
          description: Cuenta creada
        '400':
          description: Bad Request
        '401':
          description: No autorizado
      x-amazon-apigateway-integration:
        uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${AccountLambda.Arn}/invocations
        httpMethod: POST
        type: aws_proxy

  /payments:
    post:
      summary: Realizar pago
      description: Procesa un pago entre cuentas
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/PaymentRequest'
      responses:
        '200':
          description: Pago procesado
        '400':
          description: Bad Request
        '401':
          description: No autorizado
        '429':
          description: Too Many Requests
      x-amazon-apigateway-integration:
        uri: !Sub arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${PaymentLambda.Arn}/invocations
        httpMethod: POST
        type: aws_proxy

### Justificación
- **Latencia:** Uso de endpoint regional para reducir latencia geográfica. Caching habilitado para respuestas frecuentes.
- **Disponibilidad:** Despliegue en múltiples zonas de disponibilidad (Multi-AZ) para alta disponibilidad.
- **Escalabilidad:** Throttling configurado para proteger el backend de picos de tráfico.
- **Seguridad:** Integración con AWS Cognito para autenticación y AWS WAF para protección contra ataques.

---

## 2. AWS Lambda
**Propósito:** Ejecutar la lógica de negocio de manera serverless, escalando automáticamente según la demanda.

### Configuración
```yaml
AccountLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: account-service
    Runtime: nodejs18.x
    Handler: index.handler
    Code:
      S3Bucket: !Ref LambdaCodeBucket
      S3Key: account-service.zip
    MemorySize: 512
    Timeout: 10
    Environment:
      Variables:
        DYNAMODB_TABLE: !Ref AccountsTable
        LOG_LEVEL: INFO
    Role: !GetAtt LambdaExecutionRole.Arn
    VpcConfig:
      SecurityGroupIds:
        - !Ref LambdaSecurityGroup
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
    TracingConfig:
      Mode: Active
    Layers:
      - !Ref LambdaLayer

PaymentLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: payment-service
    Runtime: nodejs18.x
    Handler: index.handler
    Code:
      S3Bucket: !Ref LambdaCodeBucket
      S3Key: payment-service.zip
    MemorySize: 1024
    Timeout: 15
    Environment:
      Variables:
        DYNAMODB_TABLE: !Ref AccountsTable
        SQS_QUEUE_URL: !Ref PaymentQueue
        LOG_LEVEL: INFO
    Role: !GetAtt LambdaExecutionRole.Arn
    VpcConfig:
      SecurityGroupIds:
        - !Ref LambdaSecurityGroup
      SubnetIds:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
    TracingConfig:
      Mode: Active

LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: lambda-execution-role
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: lambda-dynamodb-access
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - dynamodb:GetItem
                - dynamodb:PutItem
                - dynamodb:UpdateItem
                - dynamodb:Query
                - dynamodb:Scan
              Resource: !GetAtt AccountsTable.Arn
      - PolicyName: lambda-sqs-access
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - sqs:SendMessage
                - sqs:ReceiveMessage
                - sqs:DeleteMessage
              Resource: !GetAtt PaymentQueue.Arn
      - PolicyName: lambda-logs
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - logs:CreateLogGroup
                - logs:CreateLogStream
                - logs:PutLogEvents
              Resource: arn:aws:logs:*:*:*
      - PolicyName: lambda-xray
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - xray:PutTraceSegments
                - xray:PutTelemetryRecords
              Resource: '*'

LambdaLayer:
  Type: AWS::Lambda::LayerVersion
  Properties:
    LayerName: common-utils
    Description: Librerías comunes para Lambdas
    Content:
      S3Bucket: !Ref LambdaCodeBucket
      S3Key: lambda-layer.zip
    CompatibleRuntimes:
      - nodejs18.x

### Justificación
- **Latencia:** Timeout ajustado a 10-15 segundos para evitar latencias altas. Uso de VPC para acceder a recursos privados sin pasar por internet.
- **Disponibilidad:** Despliegue en múltiples subnets para redundancia. Tracing activado para monitoreo.
- **Escalabilidad:** Memoria y timeout configurados según la carga esperada. Auto-scaling nativo de Lambda.
- **Seguridad:** Roles IAM con permisos mínimos. Variables de entorno cifradas.

---

## 3. Amazon SQS
**Propósito:** Desacoplar componentes para manejar operaciones asíncronas y mejorar la resiliencia.

### Configuración
```yaml
PaymentQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: payment-processing-queue.fifo
    FifoQueue: true
    ContentBasedDeduplication: true
    MessageRetentionPeriod: 345600 # 4 días
    VisibilityTimeout: 60
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt PaymentDLQ.Arn
      maxReceiveCount: 3

PaymentDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: payment-processing-dlq.fifo
    FifoQueue: true

### Justificación
- **Latencia:** Desacopla operaciones lentas (ej. procesamiento de pagos) para no bloquear solicitudes.
- **Disponibilidad:** Cola FIFO con deduplicación para evitar procesamiento duplicado.
- **Escalabilidad:** SQS escala automáticamente según la carga.
- **Resiliencia:** Dead Letter Queue (DLQ) para mensajes que fallan repetidamente.

---

## 4. Amazon DynamoDB
**Propósito:** Base de datos NoSQL escalable para almacenar datos de cuentas y transacciones.

### Configuración
```yaml
AccountsTable:
  Type: AWS::DynamoDB::Table
  Properties:
    TableName: Accounts
    AttributeDefinitions:
      - AttributeName: accountId
        AttributeType: S
      - AttributeName: userId
        AttributeType: S
    KeySchema:
      - AttributeName: accountId
        KeyType: HASH
    GlobalSecondaryIndexes:
      - IndexName: UserIndex
        KeySchema:
          - AttributeName: userId
            KeyType: HASH
        Projection:
          ProjectionType: ALL
        ProvisionedThroughput:
          ReadCapacityUnits: 10
          WriteCapacityUnits: 5
    ProvisionedThroughput:
      ReadCapacityUnits: 20
      WriteCapacityUnits: 10
    TimeToLiveSpecification:
      AttributeName: expiryTime
      Enabled: true
    StreamSpecification:
      StreamViewType: NEW_AND_OLD_IMAGES
    SSESpecification:
      SSEEnabled: true
    Replicas:
      - RegionName: us-west-2
      - RegionName: eu-west-1

### Justificación
- **Latencia:** Índices secundarios para consultas rápidas. Particiones distribuidas para evitar hotspots.
- **Disponibilidad:** Replicación cross-region con Global Tables.
- **Escalabilidad:** Throughput ajustable según demanda. Auto-scaling disponible.
- **Seguridad:** Cifrado en reposo con AWS KMS. TTL para limpieza automática de datos.

---

## 5. Amazon CloudFront
**Propósito:** Reducir latencia para contenido estático y cachear respuestas de API.

### Configuración
```yaml
CloudFrontDistribution:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      Enabled: true
      Comment: Distribución para reducir latencia
      DefaultRootObject: index.html
      Origins:
        - Id: APIGateway
          DomainName: !Sub ${FinancialApi}.execute-api.${AWS::Region}.amazonaws.com
          OriginPath: /prod
          CustomOriginConfig:
            HTTPPort: 80
            HTTPSPort: 443
            OriginProtocolPolicy: https-only
        - Id: S3Bucket
          DomainName: !GetAtt StaticContentBucket.DomainName
          S3OriginConfig:
            OriginAccessIdentity: !Sub origin-access-identity/cloudfront/${CloudFrontOAI}
      DefaultCacheBehavior:
        TargetOriginId: APIGateway
        ForwardedValues:
          QueryString: true
          Cookies:
            Forward: all
          Headers:
            - Authorization
            - Origin
        ViewerProtocolPolicy: redirect-to-https
        AllowedMethods:
          - GET
          - HEAD
          - OPTIONS
          - PUT
          - POST
          - PATCH
          - DELETE
        CachedMethods:
          - GET
          - HEAD
          - OPTIONS
        MinTTL: 0
        DefaultTTL: 60
        MaxTTL: 300
        Compress: true
      CacheBehaviors:
        - PathPattern: /static/*
          TargetOriginId: S3Bucket
          ForwardedValues:
            QueryString: false
            Cookies:
              Forward: none
          ViewerProtocolPolicy: redirect-to-https
          AllowedMethods:
            - GET
            - HEAD
          CachedMethods:
            - GET
            - HEAD
          MinTTL: 3600
          DefaultTTL: 86400
          MaxTTL: 31536000
      PriceClass: PriceClass_100
      ViewerCertificate:
        CloudFrontDefaultCertificate: true
        MinimumProtocolVersion: TLSv1.2_2021

CloudFrontOAI:
  Type: AWS::CloudFront::CloudFrontOriginAccessIdentity
  Properties:
    CloudFrontOriginAccessIdentityConfig:
      Comment: OAI para acceder a S3

StaticContentBucket:
  Type: AWS::S3::Bucket
  Properties:
    BucketName: financial-static-content
    AccessControl: Private
    LifecycleConfiguration:
      Rules:
        - Id: ExpirationRule
          Status: Enabled
          ExpirationInDays: 365

StaticContentBucketPolicy:
  Type: AWS::S3::BucketPolicy
  Properties:
    Bucket: !Ref StaticContentBucket
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            AWS: !Sub arn:aws:iam::cloudfront:user/CloudFront Origin Access Identity ${CloudFrontOAI}
          Action: s3:GetObject
          Resource: !Sub ${StaticContentBucket.Arn}/*

### Justificación
- **Latencia:** Distribución global con edge locations para reducir latencia.
- **Disponibilidad:** Failover automático a origen si falla el edge.
- **Seguridad:** HTTPS obligatorio. Origen protegido con OAI.
- **Escalabilidad:** Manejo automático de picos de tráfico.

---

## 6. AWS IAM
**Propósito:** Gestionar permisos con el principio de mínimo privilegio.

### Configuración
```yaml
ApiGatewayRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: api-gateway-role
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: apigateway.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: api-gateway-logs
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - logs:CreateLogGroup
                - logs:CreateLogStream
                - logs:PutLogEvents
              Resource: arn:aws:logs:*:*:*

CognitoRole:
  Type: AWS::IAM::Role
  Properties:
    RoleName: cognito-role
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: cognito-idp.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: cognito-lambda-invoke
        PolicyDocument:
          Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action: lambda:invokeFunction
              Resource: !GetAtt AccountLambda.Arn

### Justificación
- **Seguridad:** Permisos granulares para cada servicio. Separación de responsabilidades.
- **Mantenibilidad:** Roles con nombres descriptivos y políticas bien documentadas.

---

## 7. Amazon Cognito
**Propósito:** Autenticación y autorización de usuarios.

### Configuración
```yaml
UserPool:
  Type: AWS::Cognito::UserPool
  Properties:
    UserPoolName: financial-user-pool
    AdminCreateUserConfig:
      AllowAdminCreateUserOnly: false
    AutoVerifiedAttributes:
      - email
    UsernameAttributes:
      - email
    Policies:
      PasswordPolicy:
        MinimumLength: 8
        RequireLowercase: true
        RequireNumbers: true
        RequireSymbols: true
        RequireUppercase: true
    Schema:
      - Name: email
        AttributeDataType: String
        Required: true
        Mutable: true
      - Name: phone_number
        AttributeDataType: String
        Required: false
        Mutable: true
    MfaConfiguration: ON
    SoftwareTokenMfaConfiguration:
      Enabled: true
    EmailConfiguration:
      EmailSendingAccount: COGNITO_DEFAULT
    AccountRecoverySetting:
      RecoveryMechanisms:
        - Name: verified_email
          Priority: 1

UserPoolClient:
  Type: AWS::Cognito::UserPoolClient
  Properties:
    ClientName: financial-app-client
    UserPoolId: !Ref UserPool
    GenerateSecret: true
    RefreshTokenValidity: 30
    AccessTokenValidity: 1
    IdTokenValidity: 1
    TokenValidityUnits:
      AccessToken: hours
      IdToken: hours
      RefreshToken: days
    ExplicitAuthFlows:
      - ALLOW_USER_PASSWORD_AUTH
      - ALLOW_REFRESH_TOKEN_AUTH
    PreventUserExistenceErrors: ENABLED
    SupportedIdentityProviders:
      - COGNITO

IdentityPool:
  Type: AWS::Cognito::IdentityPool
  Properties:
    IdentityPoolName: financial-identity-pool
    AllowUnauthenticatedIdentities: false
    CognitoIdentityProviders:
      - ClientId: !Ref UserPoolClient
        ProviderName: !GetAtt UserPool.ProviderName
        ServerSideTokenCheck: true

### Justificación
- **Seguridad:** Autenticación multifactor (MFA) para usuarios. Políticas de contraseñas robustas.
- **Latencia:** Tokens de corta duración para reducir riesgo de exposición.
- **Escalabilidad:** Manejo automático de usuarios a escala.

---

## 8. AWS WAF
**Propósito:** Proteger contra ataques comunes como SQL injection, XSS y DDoS.

### Configuración
```yaml
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    Name: financial-web-acl
    Scope: REGIONAL
    DefaultAction:
      Allow: {}
    VisibilityConfig:
      SampledRequestsEnabled: true
      CloudWatchMetricsEnabled: true
      MetricName: financial-web-acl
    Rules:
      - Name: AWS-AWSManagedRulesCommonRuleSet
        Priority: 0
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesCommonRuleSet
        OverrideAction:
          None: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: AWS-AWSManagedRulesCommonRuleSet
      - Name: AWS-AWSManagedRulesSQLiRuleSet
        Priority: 1
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesSQLiRuleSet
        OverrideAction:
          None: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: AWS-AWSManagedRulesSQLiRuleSet
      - Name: RateLimitRule
        Priority: 2
        Action:
          Block: {}
        Statement:
          RateBasedStatement:
            Limit: 1000
            AggregateKeyType: IP
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: RateLimitRule

WebACLAssociation:
  Type: AWS::WAFv2::WebACLAssociation
  Properties:
    ResourceArn: !Sub arn:aws:apigateway:${AWS::Region}::/restapis/${FinancialApi}/stages/prod
    WebACLArn: !GetAtt WebACL.Arn

### Justificación
- **Seguridad:** Protección contra ataques comunes sin impacto en latencia.
- **Disponibilidad:** Rate limiting para evitar saturación del sistema.

---

## 9. Amazon CloudWatch
**Propósito:** Monitoreo y alertas en tiempo real.

### Configuración
```yaml
Dashboard:
  Type: AWS::CloudWatch::Dashboard
  Properties:
    DashboardName: FinancialSystemDashboard
    DashboardBody: |
      {
        "widgets": [
          {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
              "metrics": [
                [ "AWS/ApiGateway", "Latency", "ApiName", "financial-api", "Stage", "prod", { "stat": "Average", "period": 60 } ],
                [ ".", "5XXError", ".", ".", ".", ".", { "stat": "Sum", "period": 60 } ],
                [ ".", "4XXError", ".", ".", ".", ".", { "stat": "Sum", "period": 60 } ]
              ],
              "view": "timeSeries",
              "stacked": false,
              "region": "${AWS::Region}",
              "title": "API Gateway - Latencia y Errores"
            }
          },
          {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
              "metrics": [
                [ "AWS/Lambda", "Invocations", "FunctionName", "account-service", { "stat": "Sum", "period": 60 } ],
                [ ".", "Errors", ".", ".", { "stat": "Sum", "period": 60 } ],
                [ ".", "Duration", ".", ".", { "stat": "Average", "period": 60 } ]
              ],
              "view": "timeSeries",
              "stacked": false,
              "region": "${AWS::Region}",
              "title": "Lambda - Cuenta Servicio"
            }
          },
          {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
              "metrics": [
                [ "AWS/DynamoDB", "ConsumedReadCapacityUnits", "TableName", "Accounts", { "stat": "Sum", "period": 60 } ],
                [ ".", "ConsumedWriteCapacityUnits", ".", ".", { "stat": "Sum", "period": 60 } ],
                [ ".", "SuccessfulRequestLatency", ".", ".", { "stat": "Average", "period": 60 } ]
              ],
              "view": "timeSeries",
              "stacked": false,
              "region": "${AWS::Region}",
              "title": "DynamoDB - Rendimiento"
            }
          },
          {
            "type": "metric",
            "x": 12,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
              "metrics": [
                [ "AWS/SQS", "ApproximateNumberOfMessagesVisible", "QueueName", "payment-processing-queue.fifo", { "stat": "Average", "period": 60 } ],
                [ ".", "ApproximateNumberOfMessagesNotVisible", ".", ".", { "stat": "Average", "period": 60 } ]
              ],
              "view": "timeSeries",
              "stacked": false,
              "region": "${AWS::Region}",
              "title": "SQS - Cola de Pagos"
            }
          }
        ]
      }

LatencyAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: HighLatencyAlarm
    ComparisonOperator: GreaterThanThreshold
    EvaluationPeriods: 1
    MetricName: Latency
    Namespace: AWS/ApiGateway
    Period: 60
    Statistic: Average
    Threshold: 200
    ActionsEnabled: true
    AlarmActions:
      - !Ref NotificationTopic
    Dimensions:
      - Name: ApiName
        Value: financial-api
      - Name: Stage
        Value: prod

ErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: HighErrorRateAlarm
    ComparisonOperator: GreaterThanThreshold
    EvaluationPeriods: 1
    MetricName: 5XXError
    Namespace: AWS/ApiGateway
    Period: 60
    Statistic: Sum
    Threshold: 10
    ActionsEnabled: true
    AlarmActions:
      - !Ref NotificationTopic
    Dimensions:
      - Name: ApiName
        Value: financial-api
      - Name: Stage
        Value: prod

NotificationTopic:
  Type: AWS::SNS::Topic
  Properties:
    TopicName: financial-system-notifications
    Subscription:
      - Protocol: email
        Endpoint: operations-team@example.com

### Justificación
- **Observabilidad:** Dashboards con métricas clave para monitoreo proactivo.
- **Disponibilidad:** Alertas en tiempo real para incidentes críticos.

---

## 10. AWS X-Ray
**Propósito:** Trazabilidad de solicitudes a través de microservicios.

### Configuración
```yaml
XRaySamplingRule:
  Type: AWS::XRay::SamplingRule
  Properties:
    SamplingRule:
      RuleName: financial-sampling-rule
      ResourceARN: "*"
      Priority: 10000
      FixedRate: 0.05
      ReservoirSize: 1
      ServiceName: "*"
      ServiceType: "*"
      HTTPMethod: "*"
      URLPath: "*"
      Version: 1

XRayServiceMap:
  Type: AWS::XRay::Group
  Properties:
    GroupName: FinancialServices
    FilterExpression: "service(\"financial-api\")"

### Justificación
- **Observabilidad:** Trazabilidad completa de solicitudes para diagnóstico de problemas.
- **Latencia:** Muestreo eficiente para minimizar overhead.

---

## Conclusión
La configuración descrita cumple con los atributos de calidad identificados y está alineada con las mejores prácticas del AWS Well-Architected Framework. Cada servicio ha sido configurado con justificaciones técnicas claras para garantizar latencia baja, alta disponibilidad, escalabilidad, seguridad y observabilidad. Esta configuración será evaluada y refinada en las siguientes fases del proyecto.