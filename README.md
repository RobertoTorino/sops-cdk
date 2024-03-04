# sops-cdk

**Getting Started with SOPS (Secrets OPerationS) using AWS KMS and the AWS CDK**

---

[SOPS Demo](https://www.youtube.com/watch?v=V2PRhxphH2w)                    
[CDK SOPS SECRETS](https://constructs.dev/packages/cdk-sops-secrets/v/1.5.48?lang=typescript)               
[SOPS with CDK](https://repost.aws/articles/ARjfHwDesbQ02aLibYwRJ9pQ/manage-secrets-in-cdk-via-mozilla-secretoperations-sops)           
[SOPS Bug](https://constructs.dev/packages/cdk-sops-secrets/v/1.5.61?lang=typescript#error-getting-data-key-0-successful-groups-required-got-0)

> **SOPS = Secrets OPerationS, is a command-line tool that encrypts and decrypts YAML/JSON files using a symmetric key. It supports encryption algorithms and integrates seamlessly with AWS. AWS KMS (Key Management Service) is a cloud-based service that helps create and control cryptographic keys.**

### Instructions

> **Install SOPS: `brew install sops`**

#### Create an AWS KMS Key and IAM Alias:

**Important! At the bottom you can find the CDK templates.**

**Info: It is recommended to use at least two master keys in different regions.**

```shell
aws kms create-key \
--region "eu-west-1" \
--description "SOPS KMS Key EU-West-1" \
--tags TagKey='Foo',TagValue='bar' TagKey='Example',TagValue='sops'
```

```shell
aws kms create-key \
--region "us-east-1" \
--description "SOPS KMS Key US-East-1" \
--tags TagKey='Foo',TagValue='bar' TagKey='Example',TagValue='sops'
```

**Optional, a key alias can only be created separate:**

```shell
aws kms create-alias \
--alias-name alias/sops-dev-key-eu \
--target-key-id your_key_id
```

```shell
aws kms create-alias \
--alias-name alias/sops-dev-key-us \
--target-key-id your_key_id
```

**From the output, copy the key ARN, and save it as an environment variable:**
`export SOPS_KMS_ARN="arn:aws:kms:us-east-1:AWS_ACCOUNT_ID:key/your_key_id"`

#### AWS IAM Role:

**We use roles to indicate that a user of the AWS account is allowed to make use of KMS master keys. You can use keys in various accounts by tying each KMS key to a role that the user is allowed to assume in each account.**

**If you want to use a role, change your environment variable like this:**                            
`export SOPS_KMS_ARN="arn:aws:kms:us-east-1:AWS_ACCOUNT_ID:key/your_key_id+arn:aws:iam::AWS_ACCOUNT_ID:role/sops-dev-role"`

#### Create an AWS IAM Role:

```shell
#!/bin/sh
# https://awscli.amazonaws.com/v2/documentation/api/latest/reference/iam/create-role.html
# SOPS (Secrets OPerationS)

aws iam create-role \
    --role-name sops-dev-role \
    --assume-role-policy-document file://sops-dev-role.json \
    --description "The role for using sops." \
    --tags '{"Key": "Application", "Value": "my-application"}' '{"Key": "Stage", "Value": "dev"}
```

**The role must have permission to call Encrypt and Decrypt using KMS. An example policy is shown below:**

```json
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Action": [
      "kms:Encrypt",
      "kms:Decrypt",
      "kms:ReEncrypt*",
      "kms:GenerateDataKey*",
      "kms:DescribeKey"
    ],
    "Principal": {
      "AWS": [
        "arn:aws:iam:AWS_ACCOUNT_ID:role/sops-dev-role"
      ]
    }
  }
}
```

**The role also needs a trust relationship. Update an AWS IAM role with a trust policy:**
```shell
aws iam update-assume-role-policy \
--role-name sops-dev-role \
--policy-document file://sops-dev-role-trust-relationships.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::AWS_ACCOUNT_ID:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```
---

#### Assuming roles and using KMS in various AWS accounts

**In SOPS You only need to specify the role a KMS key must assume alongside its ARN, as follows:**

```json
{
  "secret": "value",
  "sops": {
    "kms": [
      {
        "arn": "arn: aws: kms: us-east-1: AWS_ACCOUNT_ID: key/your_key_id",
        "role": "arn:aws: iam:: AWS_ACCOUNT_ID: role/sops-dev-xyz",
        "created_at": "",
        "enc": "",
        "aws_profile": ""
      }
    ],
    "gcp_kms": null,
    "azure_kv": null,
    "hc_vault": null,
    "age": null,
    "lastmodified": "",
    "mac": "",
    "pgp": null,
    "unencrypted_suffix": "_unencrypted",
    "version": "3.8.1"

  }
}
```

### Manage your secrets
- Create a "sops" folder under the CDK project.
- Inside the "sops" folder run:
```shell
sops -kms "$SOPS_KMS_ARN" mymanagedsecrets.sops.json
```
This command will open the .json file in your editor, and you can enter the secrets in that file:
```json
{
  "my_first_secret": "my_first_token",
  "my_second_secret": "my_second_token"
}
```

#### Using SOPS in the AWS CDK:

To use the SOPS file in the CDK, we will import the SOPS secrets construct:  [`cdk-sops-secrets`](https://constructs.dev/packages/cdk-sops-secrets/v/1.5.61?lang=typescript)                           
This construct library replaces CDK SecretsManager secrets and allows storing secrets encrypted with SOPS in AWS Secrets Manager.            
It uses the `mymanagedsecrets.sops.json` file to store the secrets in secretsmanager.          
Install the construct with npm: `npm i cdk-sops-secrets@latest` and update your `package.json` file.

**Example CDK code:**

```typescript
import * as cdk from 'aws-cdk-lib';
import { RemovalPolicy } from 'aws-cdk-lib';
import { SopsSecret } from 'cdk-sops-secrets';

export class SopsSecretsManager extends cdk.Stack {
    constructor (scope: cdk.App, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        new SopsSecret(this, 'SopsCredentials', {
            secretName: 'sops_credentials',
            sopsFilePath: 'sops/mymanagedsecrets.sops.json',
        });
    }
}
```

```typescript
import * as cdk from 'aws-cdk-lib';
import { aws_iam, aws_kms } from 'aws-cdk-lib';
import { Effect } from 'aws-cdk-lib/aws-iam';


export class SopsDevRole extends cdk.Stack {
    constructor (scope: cdk.App, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        const mySopsDevRole = new aws_iam.Role(this, 'sops-dev-role', {
            assumedBy: new aws_iam.AccountPrincipal(this.account),
        });

        mySopsDevRole.addToPolicy(new aws_iam.PolicyStatement({
            sid: 'SOPSDevActions',
            effect: Effect.ALLOW,
            actions: [
                'kms:Encrypt',
                'kms:Decrypt',
                'kms:ReEncrypt*',
                'kms:GenerateDataKey*',
                'kms:DescribeKey'
            ],
            resources: [
                '*'
            ],
        }));

        new aws_kms.Key(this, 'sops-dev-key-eu', {
            removalPolicy: cdk.RemovalPolicy.DESTROY,
            pendingWindow: cdk.Duration.days(7),
            alias: 'alias/sops-dev-key-eu',
            description: '"SOPS KMS Key EU-West-1"',
            enableKeyRotation: false,
        });
    }
}
```

```typescript
import * as cdk from 'aws-cdk-lib';
import { aws_kms } from 'aws-cdk-lib';

export class SopsKmsKeyUs extends cdk.Stack {
    constructor (scope: cdk.App, id: string, props?: cdk.StackProps) {
        super(scope, id, props);

        new aws_kms.Key(this, 'sops-dev-key-us', {
            removalPolicy: cdk.RemovalPolicy.DESTROY,
            pendingWindow: cdk.Duration.days(7),
            alias: 'alias/sops-dev-key-us',
            description: '"SOPS KMS Key US-East-1"',
            enableKeyRotation: false,
        });
    }
}
```

**The app stack:**
```typescript
#!/usr/bin/env node
import { App, Stack, Tags } from 'aws-cdk-lib';
import 'source-map-support/register';
import { SopsSecretsManager } from '../lib/sops-secrets';

const app = new App();

const env = { account: 'AWS_ACCOUNT_ID', region: 'eu-west-1' }

enum Environment {
    dev = 'dev'
}
const construct = 'test-stack';

const addTags = (stack: Stack, environment: Environment) => {
    Tags.of(stack).add('Application', construct, {
        applyToLaunchedInstances: true,
        includeResourceTypes: [],
    });
    Tags.of(stack).add('Stage', environment, {
        applyToLaunchedInstances: true,
        includeResourceTypes: [],
    });
};

const sopsDevRoleStack = new SopsDevRole(app, 'SopsDevRoleStack', {
    stackName: 'SopsDevRoleStack',
    description: 'Stack for the sops dev role.',
    env,
});
addTags(sopsDevRoleStack, Environment.dev);

const sopsSecretsStack = new SopsSecretsManager(app, 'SopsSecretsManagerStack', {
    stackName: 'SopsSecretsManagerStack',
    description: 'Stack for the sops secrets.',
    env,
});
addTags(sopsSecretsStack, Environment.dev);

const sopsKmsKeyUsStack = new SopsKmsKeyUs(app, 'SopsKmsKeyUsStack', {
    stackName: 'SopsDeVKeyStackUS',
    description: 'Stack for the sops kms key in us-east-1.',
    env: { account: 'AWS_ACCOUNT_ID', region: 'us-east-1' }
});
addTags(sopsKmsKeyUsStack, Environment.dev);
```

**Pipeline example:**
```typescript
environmentVariables: {
    FIRST_TOKEN: {
        type: BuildEnvironmentVariableType.SECRETS_MANAGER,
            value: 'sops_credentials:my_api_key',
    },
    SECOND_TOKEN: {
        type: BuildEnvironmentVariableType.SECRETS_MANAGER,
            value: 'sops_credentials::my_secret_value',
    },
},
```    

**Deploy the stacks with `cdk deploy <NAME_OF_YOUR_STACK>`**
```shell
cdk deploy SopsDevRoleStack
cdk deploy SopsKmsKeyUsStack
cdk deploy SopsSecretsManagerStack
```

> **To view the decrypted file, use the -d flag: `sops -d mymanagedsecrets.sops.json`.**            
> **To add or remove KMS or PGP keys under the sops section use the -s flag: `sops -s mymanagedsecrets.sops.json`.**                    
> **Always make sure you are programmatically logged in to your AWS account to let SOPS find your KMS key.**                 

> #### [GitHub](https://github.com/RobertoTorino)
> ![GitHub](images/github.png) 
> 
