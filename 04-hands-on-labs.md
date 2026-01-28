# Hands-On Labs for Cloud Architects

Practical exercises to build experience with both AWS and Azure, leveraging your existing AWS knowledge.

## Lab 1: Deploy a Highly Available Web Application

### Objective
Deploy a web application with load balancing, auto-scaling, and database backend.

### AWS Implementation

```bash
# 1. Create VPC with public/private subnets
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# 2. Create Application Load Balancer
aws elbv2 create-load-balancer \
    --name my-alb \
    --subnets subnet-xxx subnet-yyy \
    --security-groups sg-xxx

# 3. Create Launch Template
aws ec2 create-launch-template \
    --launch-template-name web-template \
    --launch-template-data '{
        "ImageId": "ami-xxx",
        "InstanceType": "t3.micro",
        "UserData": "base64-encoded-script"
    }'

# 4. Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name web-asg \
    --launch-template LaunchTemplateName=web-template \
    --min-size 2 --max-size 6 --desired-capacity 2 \
    --vpc-zone-identifier "subnet-xxx,subnet-yyy" \
    --target-group-arns arn:aws:elasticloadbalancing:...

# 5. Create RDS Multi-AZ
aws rds create-db-instance \
    --db-instance-identifier mydb \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --multi-az \
    --allocated-storage 20
```

### Azure Implementation

```bash
# 1. Create Resource Group
az group create --name myResourceGroup --location eastus

# 2. Create Virtual Network
az network vnet create \
    --resource-group myResourceGroup \
    --name myVNet \
    --address-prefix 10.0.0.0/16 \
    --subnet-name webSubnet \
    --subnet-prefix 10.0.1.0/24

# 3. Create VM Scale Set with Load Balancer
az vmss create \
    --resource-group myResourceGroup \
    --name myScaleSet \
    --image Ubuntu2204 \
    --upgrade-policy-mode automatic \
    --instance-count 2 \
    --admin-username azureuser \
    --generate-ssh-keys \
    --load-balancer myLoadBalancer

# 4. Configure Auto-scale Rules
az monitor autoscale create \
    --resource-group myResourceGroup \
    --resource myScaleSet \
    --resource-type Microsoft.Compute/virtualMachineScaleSets \
    --name autoscale \
    --min-count 2 \
    --max-count 6 \
    --count 2

az monitor autoscale rule create \
    --resource-group myResourceGroup \
    --autoscale-name autoscale \
    --condition "Percentage CPU > 70 avg 5m" \
    --scale out 2

# 5. Create Azure SQL Database
az sql server create \
    --name myserver \
    --resource-group myResourceGroup \
    --location eastus \
    --admin-user sqladmin \
    --admin-password <password>

az sql db create \
    --resource-group myResourceGroup \
    --server myserver \
    --name mydb \
    --service-objective S0 \
    --zone-redundant true
```

### Verification Checklist
- [ ] Load balancer distributes traffic across instances
- [ ] Auto-scaling triggers on CPU threshold
- [ ] Database failover works (simulate failure)
- [ ] Application recovers from instance termination

---

## Lab 2: Serverless API with Database

### Objective
Build a REST API using serverless compute and managed database.

### AWS Implementation (Lambda + API Gateway + DynamoDB)

**Lambda Function (Python):**
```python
# lambda_function.py
import json
import boto3
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Items')

def lambda_handler(event, context):
    http_method = event['httpMethod']

    if http_method == 'GET':
        item_id = event['pathParameters']['id']
        response = table.get_item(Key={'id': item_id})
        return {
            'statusCode': 200,
            'body': json.dumps(response.get('Item', {}), default=str)
        }

    elif http_method == 'POST':
        body = json.loads(event['body'])
        table.put_item(Item=body)
        return {
            'statusCode': 201,
            'body': json.dumps({'message': 'Created'})
        }

    return {'statusCode': 400, 'body': 'Bad Request'}
```

**Deploy with SAM:**
```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  ItemsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: lambda_function.lambda_handler
      Runtime: python3.11
      Events:
        GetItem:
          Type: Api
          Properties:
            Path: /items/{id}
            Method: get
        CreateItem:
          Type: Api
          Properties:
            Path: /items
            Method: post
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref ItemsTable

  ItemsTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: Items
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
      BillingMode: PAY_PER_REQUEST
```

### Azure Implementation (Functions + Cosmos DB)

**Azure Function (Python):**
```python
# function_app.py
import azure.functions as func
import json
from azure.cosmos import CosmosClient

app = func.FunctionApp()

# Connection via managed identity
client = CosmosClient.from_connection_string(os.environ["COSMOS_CONNECTION"])
database = client.get_database_client("ItemsDB")
container = database.get_container_client("Items")

@app.route(route="items/{id}", methods=["GET"])
def get_item(req: func.HttpRequest) -> func.HttpResponse:
    item_id = req.route_params.get('id')
    try:
        item = container.read_item(item=item_id, partition_key=item_id)
        return func.HttpResponse(json.dumps(item), status_code=200)
    except:
        return func.HttpResponse("Not found", status_code=404)

@app.route(route="items", methods=["POST"])
def create_item(req: func.HttpRequest) -> func.HttpResponse:
    body = req.get_json()
    container.create_item(body)
    return func.HttpResponse("Created", status_code=201)
```

**Deploy with Azure CLI:**
```bash
# Create Cosmos DB
az cosmosdb create --name mycosmosdb --resource-group myResourceGroup

az cosmosdb sql database create \
    --account-name mycosmosdb \
    --resource-group myResourceGroup \
    --name ItemsDB

az cosmosdb sql container create \
    --account-name mycosmosdb \
    --resource-group myResourceGroup \
    --database-name ItemsDB \
    --name Items \
    --partition-key-path /id

# Create Function App
az functionapp create \
    --resource-group myResourceGroup \
    --consumption-plan-location eastus \
    --runtime python \
    --functions-version 4 \
    --name myFunctionApp \
    --storage-account mystorageaccount
```

---

## Lab 3: CI/CD Pipeline for Infrastructure

### Objective
Set up Infrastructure as Code with automated deployment.

### AWS (CloudFormation + CodePipeline)

**infrastructure.yaml:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Three-tier application infrastructure'

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        - Key: Environment
          Value: !Ref Environment

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Web server security group
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0

Outputs:
  VPCId:
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VPCId'
```

**buildspec.yml for CodeBuild:**
```yaml
version: 0.2
phases:
  install:
    commands:
      - pip install cfn-lint
  pre_build:
    commands:
      - cfn-lint infrastructure.yaml
  build:
    commands:
      - aws cloudformation package --template-file infrastructure.yaml --s3-bucket $ARTIFACT_BUCKET --output-template-file packaged.yaml
artifacts:
  files:
    - packaged.yaml
```

### Azure (Bicep + Azure DevOps)

**main.bicep:**
```bicep
@allowed(['dev', 'staging', 'prod'])
param environment string

param location string = resourceGroup().location

resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'vnet-${environment}'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'web-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: 'db-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
        }
      }
    ]
  }
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'nsg-web-${environment}'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceAddressPrefix: '*'
          destinationAddressPrefix: '*'
        }
      }
    ]
  }
}

output vnetId string = vnet.id
```

**azure-pipelines.yml:**
```yaml
trigger:
  branches:
    include:
      - main

stages:
  - stage: Validate
    jobs:
      - job: ValidateBicep
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'MyServiceConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file main.bicep

  - stage: DeployDev
    dependsOn: Validate
    jobs:
      - deployment: DeployToDev
        environment: 'dev'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'MyServiceConnection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment group create \
                        --resource-group rg-dev \
                        --template-file main.bicep \
                        --parameters environment=dev
```

---

## Lab 4: Kubernetes Deployment

### Objective
Deploy a containerized application to managed Kubernetes.

### Common Kubernetes Manifests

**deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: myregistry/web-app:v1
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### AWS EKS Setup

```bash
# Create EKS cluster
eksctl create cluster \
    --name my-cluster \
    --region us-west-2 \
    --nodegroup-name standard-nodes \
    --node-type t3.medium \
    --nodes 3 \
    --nodes-min 2 \
    --nodes-max 5

# Configure kubectl
aws eks update-kubeconfig --name my-cluster --region us-west-2

# Deploy application
kubectl apply -f deployment.yaml

# Set up AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=my-cluster \
    -n kube-system
```

### Azure AKS Setup

```bash
# Create AKS cluster
az aks create \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --node-count 3 \
    --enable-addons monitoring \
    --generate-ssh-keys \
    --enable-cluster-autoscaler \
    --min-count 2 \
    --max-count 5

# Get credentials
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster

# Deploy application
kubectl apply -f deployment.yaml

# Enable Application Gateway Ingress Controller (optional)
az aks enable-addons \
    --resource-group myResourceGroup \
    --name myAKSCluster \
    --addons ingress-appgw \
    --appgw-name myApplicationGateway \
    --appgw-subnet-cidr "10.2.0.0/16"
```

---

## Lab 5: Monitoring and Observability

### Objective
Set up comprehensive monitoring, logging, and alerting.

### AWS Implementation

```bash
# Create CloudWatch Dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "ApplicationDashboard" \
    --dashboard-body '{
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "metrics": [
                        ["AWS/EC2", "CPUUtilization", "AutoScalingGroupName", "web-asg"]
                    ],
                    "title": "EC2 CPU Utilization"
                }
            }
        ]
    }'

# Create Alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "HighCPU" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-east-1:123456789:alerts

# Enable X-Ray tracing (in application)
# pip install aws-xray-sdk
```

**X-Ray Integration (Python):**
```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

patch_all()

@xray_recorder.capture('my_function')
def my_function():
    # Your code here
    pass
```

### Azure Implementation

```bash
# Create Log Analytics Workspace
az monitor log-analytics workspace create \
    --resource-group myResourceGroup \
    --workspace-name myWorkspace

# Create Application Insights
az monitor app-insights component create \
    --app myAppInsights \
    --location eastus \
    --resource-group myResourceGroup \
    --workspace myWorkspace

# Create Alert Rule
az monitor metrics alert create \
    --name "HighCPU" \
    --resource-group myResourceGroup \
    --scopes /subscriptions/.../resourceGroups/myResourceGroup/providers/Microsoft.Compute/virtualMachineScaleSets/myScaleSet \
    --condition "avg Percentage CPU > 80" \
    --window-size 5m \
    --evaluation-frequency 1m \
    --action myActionGroup
```

**Application Insights Integration (Python):**
```python
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer

tracer = Tracer(
    exporter=AzureExporter(connection_string='InstrumentationKey=...'),
    sampler=ProbabilitySampler(1.0)
)

with tracer.span(name='my_function'):
    # Your code here
    pass
```

---

## Lab 6: Security Implementation

### Objective
Implement security best practices including encryption, secrets management, and network security.

### AWS Implementation

```bash
# Create KMS Key
aws kms create-key --description "Application encryption key"

# Store secret in Secrets Manager
aws secretsmanager create-secret \
    --name "prod/db/password" \
    --secret-string '{"username":"admin","password":"secret123"}'

# Enable VPC Flow Logs
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-xxx \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name VPCFlowLogs

# Create WAF Web ACL
aws wafv2 create-web-acl \
    --name "WebACL" \
    --scope REGIONAL \
    --default-action '{"Allow":{}}' \
    --rules '[
        {
            "Name": "RateLimitRule",
            "Priority": 1,
            "Statement": {
                "RateBasedStatement": {
                    "Limit": 1000,
                    "AggregateKeyType": "IP"
                }
            },
            "Action": {"Block":{}},
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "RateLimitRule"
            }
        }
    ]'
```

### Azure Implementation

```bash
# Create Key Vault
az keyvault create \
    --name myKeyVault \
    --resource-group myResourceGroup \
    --location eastus

# Store Secret
az keyvault secret set \
    --vault-name myKeyVault \
    --name "db-password" \
    --value "secret123"

# Enable diagnostic logging
az monitor diagnostic-settings create \
    --resource /subscriptions/.../myVNet \
    --name "VNetDiagnostics" \
    --logs '[{"category":"NetworkSecurityGroupEvent","enabled":true}]' \
    --workspace myWorkspace

# Create WAF Policy
az network application-gateway waf-policy create \
    --name myWAFPolicy \
    --resource-group myResourceGroup

az network application-gateway waf-policy rule create \
    --policy-name myWAFPolicy \
    --resource-group myResourceGroup \
    --name RateLimitRule \
    --priority 1 \
    --rule-type RateLimitRule \
    --rate-limit-threshold 1000
```

---

## Practice Challenges

### Challenge 1: Disaster Recovery
Design and implement a DR solution with:
- RPO < 1 hour
- RTO < 4 hours
- Automated failover testing

### Challenge 2: Cost Optimization
Analyze a running environment and:
- Identify cost savings opportunities
- Implement reserved instances
- Set up cost alerts and budgets

### Challenge 3: Zero-Downtime Deployment
Implement a deployment strategy that:
- Supports blue/green deployments
- Includes automated rollback
- Has health check validation

### Challenge 4: Multi-Region Active-Active
Build an application that:
- Runs in two regions simultaneously
- Uses global database replication
- Implements intelligent traffic routing

---

## Study Resources

### AWS Certifications
- AWS Solutions Architect Associate/Professional
- AWS DevOps Engineer Professional

### Azure Certifications
- AZ-305: Azure Solutions Architect Expert
- AZ-400: Azure DevOps Engineer Expert

### Recommended Practice
1. **AWS Free Tier:** 12 months of free services
2. **Azure Free Account:** $200 credit + 12 months free
3. **Cloud Playground:** A Cloud Guru, Pluralsight sandboxes
4. **GitHub:** Practice IaC with public repos

---

*Return to [README.md](README.md) for the complete guide overview*
