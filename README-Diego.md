# Serverless Data Lake Framework Documentation

[Prescriptive Guidance Library: Deploy and manage a serverless data lake on the AWS Cloud by using infrastructure as code](https://apg-library.amazonaws.com/content/f4fc3ad2-1c4f-45ea-bc86-2db13105a173)


[Workshop: Serverless Data Lake Framework Workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/501cb14c-91b3-455c-a2a9-d0a21ce68114/en-US)


[Project documentation: AWS Serverless Data Lake Framework](https://sdlf.readthedocs.io/en/latest/constructs/)

[Github repository: data-lakes-on-aws](https://github.com/aws-solutions-library-samples/data-lakes-on-aws)


# Deploying SDLF

## 1. Initial setup

### Steps to Deploy

Log in to the AWS account console using an IAM role with Admin permissions and select an AWS region. Please choose an AWS region where all the services mentioned on the previous page's diagram are available (e.g. eu-west-1, us-east-1).

Click on the below button to launch a stack in CloudFormation. Accept the defaults and click on Create Stack on the final page:

<!-- 
[Launch Stack](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=sdlf-workshop-deployment&templateURL=https://sdlf-os-public-artifacts.s3-us-west-2.amazonaws.com/deploying-sdlf/initial-setup.yaml)
 -->


[Launch Stack from S3](https://console.aws.amazon.com/cloudformation/home?#/stacks/new?stackName=sdlf-workshop-deployment&templateURL=https://dl-demo-751196457174-us-east-1.s3.us-east-1.amazonaws.com/sdlf-files/initial-setup.yaml)

CodeBuild llama estos archivos:

- github/aws-solutions-library-samples/data-lakes-on-aws/sdlf-pipeline/deployspec.yaml
- github/aws-solutions-library-samples/data-lakes-on-aws/sdlf-pipeline/scr/pipeline.yaml

El archivo `deployspec.yaml` tiene referencias a estos 3 archivos:

"https://raw.githubusercontent.com/aws/serverless-application-model/develop/bin/sam-translate.py"
"https://github.com/aws/aws-sam-cli/releases/latest/download/aws-sam-cli-linux-x86_64.zip"
"https://raw.githubusercontent.com/awslabs/aws-serverless-data-lake-framework/main/sdlf-cicd/template-generic-cfn-module.yaml"

Los dos primeros hacen referencia a AWS SAM en tanto que el tercero es un archivo del proyecto de SDLF.

https://github.com/aws/serverless-application-model/tree/develop

https://github.com/aws/aws-sam-cli

Mi proyecto:

https://raw.githubusercontent.com/diegoperezh/data-lakes-on-aws/refs/heads/sdlf/sdlf-cicd/template-generic-cfn-module.yaml


- Stack creado: `sdlf-workshop-deployment`

Stacks creados a partir del stack `sdlf-workshop-deployment`:

```
sdlf-cfn-module-sdlf-stageglue                  Deploy a CloudFormation module
sdlf-cfn-module-sdlf-stagelambda                Deploy a CloudFormation module
sdlf-sdlf-stages-stage-glue-iam-policy          This template deploys a Module specific IAM permissions
sdlf-sdlf-stages-stage-lambda-iam-policy        This template deploys a Module specific IAM permissions
sdlf-cfn-module-sdlf-dataset                    Deploy a CloudFormation module
sdlf-cfn-module-sdlf-foundations                Deploy a CloudFormation module
sdlf-cfn-module-sdlf-team                       Deploy a CloudFormation module
sdlf-cfn-module-sdlf-pipeline                   Deploy a CloudFormation module
sdlf-lambdalayers-DatalakeLibrary               Deploy Lambda Layers
sdlf-sdlf-base-team-iam-policy                  This template deploys a Module specific IAM permissions
sdlf-sdlf-base-pipeline-iam-policy              This template deploys a Module specific IAM permissions
sdlf-sdlf-base-dataset-iam-policy               This template deploys a Module specific IAM permissions
sdlf-sdlf-base-datalakelibrary-iam-policy       This template deploys a Module specific IAM permissions
sdlf-sdlf-base-foundations-iam-policy           This template deploys a Module specific IAM permissions
sdlf-managed-policy                             This stack deploys the Policy for seed-farmer projects
seedfarmer-sdlf-artifacts                       This stack deploys the bucket for seed-farmer projects in each acccount/region
aws-codeseeder-sdlf                             AWS CodeSeeder - codeseeder-sdlf
seedfarmer-sdlf-deployment-role                 AWS CloudFormation for the SeedFarmer Deployment Role
seedfarmer-sdlf-toolchain-role                  AWS CloudFormation for the SeedFarmer Toolchain Role
```

### Description of `initial-setup.yaml`

This CloudFormation template sets up an automated deployment infrastructure for the AWS Serverless Data Lake Framework (SDLF).

1. IAM Roles (rCodeBuildServiceRole and rLambdaTriggerRole) :

- Creates two IAM roles: one for CodeBuild and one for Lambda
- The CodeBuild role has extensive permissions to work with various AWS services like CloudFormation, CodeCommit, S3, KMS, etc.
- The Lambda role has basic execution permissions plus the ability to interact with CodeBuild

2. CodeBuild Project (rCodeBuildProject) :

- Sets up a CodeBuild project that uses Amazon Linux 2 as the build environment
- Uses Python 3.12 as the runtime
- The buildspec contains several phases that:
    - Configure AWS credentials
    - Set up a Python virtual environment
    - Install and configure seed-farmer (a deployment tool)
    - Download and apply SDLF manifests
    - Points to the SDLF GitHub repository as the source

    - La lambda layer sdlf-DatalakeLibrary se crea en este proyecto de CodeBuild

3. Lambda Function (rLambdaTrigger) :

- Creates a Python 3.12 Lambda function
- Acts as a trigger for the CodeBuild project
- Handles CloudFormation custom resource requests
- When triggered with a 'Create' event, it starts the CodeBuild project
- Includes error handling and response sending back to CloudFormation

4. Custom Resource (rCustomLambdaTrigger) :

- Links the Lambda function as a custom resource in CloudFormation
- Allows the deployment process to be triggered when the stack is created

The overall purpose of this template is to automate the initial setup of SDLF by:

- Creating necessary IAM roles and permissions
- Setting up a CodeBuild project that will clone and deploy SDLF
- Using Lambda to trigger the deployment process
- Managing the entire lifecycle through CloudFormation


## 2. Deploying storage layers

```bash
mkdir sdlf-main
cd sdlf-main/
```



Crear los archivos `datadomain-datalake-dev.yaml`, `foundations-datalake-dev.yaml` y `tags.json` en el folder `sdlf-main`


### 2.1. Create a file called foundations-datalake-dev.yaml

The name of the organization owning the data lake (`pOrg`) can be updated, as well as the name of the role used to deploy SDLF (`pCicdRole`) with the name of the role used to follow this workshop.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: SDLF Foundations in datalake domain, dev environment

Parameters:
    pPipelineReference:
        Type: String
        Default: none

Resources:
    rAnycompany:
        Type: awslabs::sdlf::foundations::MODULE
        Properties:
            pPipelineReference: !Ref pPipelineReference
            pChildAccountId: !Ref AWS::AccountId
            pOrg: anycompany
            pDomain: datalake
            pDeploymentInstance: dev
            pCicdRole: Admin
```

### 2.2. Create a file called datadomain-datalake-dev.yaml

`foundations-datalake-dev.yaml` CloudFormation template is deployed as a nested stack, for that purpose, create another file called `datadomain-datalake-dev.yaml`

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: datalake data domain, dev environment

Parameters:
    pPipelineReference:
        Type: String
        Default: none

Resources:
    rAnycompany:
        Type: AWS::CloudFormation::Stack
        Properties:
            TemplateURL: ./foundations-datalake-dev.yaml
            Parameters:
                pPipelineReference: !Ref pPipelineReference
```

### 2.3. create a file called tags.json

```json
{
    "Tags" : {
        "Framework" : "sdlf"
    }
}
```
### 2.4. Deploy the newly created files using the AWS CLI

```bash
ARTIFACTS_BUCKET=$(aws cloudformation describe-stacks --query "Stacks[?StackName=='aws-codeseeder-sdlf-cba'][].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
aws cloudformation package --template-file ./datadomain-datalake-dev.yaml --s3-bucket "$ARTIFACTS_BUCKET" --s3-prefix sdlf --output-template-file output-template.yaml
aws cloudformation deploy --template-file output-template.yaml --stack-name sdlf-domain-datalake-dev --capabilities "CAPABILITY_NAMED_IAM" "CAPABILITY_AUTO_EXPAND"
```

- Stacks creados: 

- `sdlf-domain-datalake-dev`
- `sdlf-domain-datalake-dev-rAnycompany-XM1TXE7SOPAV`


## 3. Cataloging data layer

### 3.1. Create a file called datasets.yaml:

Crear el archivo `datasets.yaml` en el folder `sdlf-main`

- `pDatasetName` is the name of a specific dataset (e.g customers, orders, flights...).
- `pS3Prefix` specifies under which prefix relevant files will be stored in the buckets.

```yaml
AWSTemplateFormatVersion: 2010-09-09
Description: Example datasets

Parameters:
    pPipelineReference:
        Type: String
        Default: none

Resources:
    rLegislators:
        Type: awslabs::sdlf::dataset::MODULE
        Properties:
            pPipelineReference: !Ref pPipelineReference
            pDatasetName: legislators
            pS3Prefix: legislators
            pDeploymentInstance: dev
```


```bash
ARTIFACTS_BUCKET=$(aws cloudformation describe-stacks --query "Stacks[?StackName=='aws-codeseeder-sdlf-cba'][].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
aws cloudformation package --template-file ./datasets.yaml --s3-bucket "$ARTIFACTS_BUCKET" --s3-prefix sdlf --output-template-file output-datasets.yaml
aws cloudformation deploy --template-file output-datasets.yaml --stack-name sdlf-datalake-datasets-dev --capabilities "CAPABILITY_NAMED_IAM" "CAPABILITY_AUTO_EXPAND"
```

- Stack creado: `sdlf-datalake-datasets-dev`

## 4. Processing data

Copiar el archivo `pipeline-main.yaml` en el folder `sdlf-main`


`pPipeline` is the name given to the ETL pipeline encompassing the stage-lambda and stage-glue step functions. A team can create one or more pipelines within the lake (e.g. cdc, ml...), with any number of stages.


```bash
ARTIFACTS_BUCKET=$(aws cloudformation describe-stacks --query "Stacks[?StackName=='aws-codeseeder-sdlf'][].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
aws cloudformation package --template-file ./pipelines.yaml --s3-bucket "$ARTIFACTS_BUCKET" --s3-prefix sdlf --output-template-file output-pipelines.yaml
aws cloudformation deploy --template-file output-pipelines.yaml --stack-name sdlf-datalake-pipelines-dev --capabilities "CAPABILITY_NAMED_IAM" "CAPABILITY_AUTO_EXPAND"
```

- Stack creado: `sdlf-datalake-pipelines-dev` & `sdlf-datalake-pipelines-dev-rMain-WLCOL4BT2L6C`

### Hydrating the Data Lake

```bash
git clone --depth 1 --branch workshop-build https://github.com/awslabs/aws-serverless-data-lake-framework.git
cd sdlf-utils/workshop-examples/legislators/
```

```bash
./deploy.sh
```

- Stack creado: `sdlf-legislators-glue-job`

## 5. Consuming data

pTeamName refers to the SDLF team that will manage ETL pipelines and the associated application code and custom transformations within the lake. Change the pSNSNotificationsEmail parameter in the file to an email of your choice. Multiple teams (e.g. engineering, marketing, finance...) can be created by adding more resource blocks like this one.


```bash
ARTIFACTS_BUCKET=$(aws cloudformation describe-stacks --query "Stacks[?StackName=='aws-codeseeder-sdlf'][].Outputs[?OutputKey=='Bucket'].OutputValue" --output text)
aws cloudformation package --template-file ./team-datalake-engineering-dev.yaml --s3-bucket "$ARTIFACTS_BUCKET" --s3-prefix sdlf --output-template-file output-team.yaml
aws cloudformation deploy --template-file output-team.yaml --stack-name sdlf-engineering-team-dev --capabilities "CAPABILITY_NAMED_IAM" "CAPABILITY_AUTO_EXPAND"
```

- Stack creado: `sdlf-engineering-team-dev`