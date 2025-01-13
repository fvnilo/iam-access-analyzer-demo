# iam-access-analyzer-demo

A demo of a CI/CD pipeline for IAM Access Analyzer

## Validating Identity-based and Resource-based Policies

Run this command to analyze the identity-based policy:

```sh
aws accessanalyzer validate-policy --policy-type IDENTITY_POLICY --policy-document file://iam-access-analyzer/identity-policy.json
```

There will be an issue. How can it be fixed?

<details>
  <summary>Click here for the answer</summary>
  
  The action name is invalid. The correct action name is `ec2:RunInstances`.
</details>

## Validating Service Control Policies (SCPs)

```sh
aws accessanalyzer validate-policy --policy-type SERVICE_CONTROL_POLICY --policy-document file://iam-access-analyzer/scp.json
```

There will be an issue. How can it be fixed?

<details>
  <summary>Click here for the answer</summary>
  
  Remove the line `"Principal": "*",`
</details>

## Previewing Access to AWS Resources

1. Create an analyzer:

    ```sh
    aws accessanalyzer create-analyzer --type ACCOUNT --analyzer-name AccessAnalyzerCICDWorkshop
    ANALYZER_ARN=$(aws accessanalyzer list-analyzers --query "analyzers[?status=='ACTIVE' && type=='ACCOUNT'].arn | [0]" --output text)
    ```

2. Create a queue policy file:

    ```sh
    ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
    cat << EOF > iam-access-analyzer/queue-policy.json
    {
        "Version": "2012-10-17",
        "Id": "MyQueuePolicy",
        "Statement": [{
            "Sid":"AllowSendMessage",
            "Effect": "Allow",
            "Principal": {
                "AWS": "111122223333"
            },
            "Action": "sqs:SendMessage",
            "Resource": "arn:aws:sqs:ca-central-1:${ACCOUNT_ID}:MyQueue"
        }]
    }
    EOF
    ```

3. Run the create-access-preview command

    ```sh
    QUEUE_POLICY=$(cat iam-access-analyzer/queue-policy.json | jq -c .)
    CONFIGURATIONS=$(jq -n -c --arg account_id "$ACCOUNT_ID" --arg queue_policy "$QUEUE_POLICY" '{"arn:aws:sqs:ca-central-1:\($account_id):MyQueue": {sqsQueue: {queuePolicy: $queue_policy}}}')
    PREVIEW_ID=$(aws accessanalyzer create-access-preview --configurations $CONFIGURATIONS --analyzer-arn $ANALYZER_ARN --output text)
    ```

4. Get the access preview:

    ```sh
    aws accessanalyzer get-access-preview --analyzer-arn $ANALYZER_ARN  --access-preview-id $PREVIEW_ID
    aws accessanalyzer list-access-preview-findings --analyzer-arn $ANALYZER_ARN --access-preview-id $PREVIEW_ID
    ```

    You should see a single finding that states:

    - an external account, `111122223333`, is able to perform the action sqs:SendMessage

## IAM Policy Validator for AWS CloudFormation

Install the validator:

```sh
pip3 install cfn-policy-validator
cfn-policy-validator --version
```

### Parsing

Parsing a template walks through each CloudFormation resource in a template,
pulling out identity-based and resource-based IAM policies.

Parse the template:

```sh
cfn-policy-validator parse --template-path cfn-policy-validator/parse-template.json --region ca-central-1
```

The parse command output the resource-based policy that was found in
the template and grouped the policy with the resource it is attached to.
Notice how the queue policy, MyQueuePolicy, from the CloudFormation
template is grouped with the queue, MyQueue, in the output.

### Validate

Validating:

- Parses a template
- Passes the identity-based and resource-based IAM policies to Access Analyzer
- Reports the findings returned from Access Analyzer

Validate the template:

```sh
cfn-policy-validator validate --template-path cfn-policy-validator/validate-template.json \
    --region ca-central-1
```

There will be a few blocking findings. What are they and how can be be fixed?

<details>
  <summary>Click here for the answer</summary>
  
- The action `sqs:ReceiveMessages` action does not exist.
  - Change `sqs:ReceiveMessages` to `sqs:ReceiveMessage` (remove `s` at the end).
- The `QueuePolicy` `Principal` should not be `*`.
  - Change the clause:

    ```json
    `"Principal": "*"`
    ```

    to:

    ```json
    "Principal": {
        "AWS": {
            "Fn::Sub": "arn:aws:iam::${AWS::AccountId}:root"
        }
    }
    ```

The final template should be:

```json
{
    "Resources": {
        "MyQueue": {
            "Type": "AWS::SQS::Queue"
        },
        "MyQueuePolicy": {
            "Type": "AWS::SQS::QueuePolicy",
            "Properties": {
                "PolicyDocument": {
                    "Version": "2012-10-17",
                    "Statement":[{
                        "Action": ["sqs:SendMessage", "sqs:ReceiveMessage"],
                        "Effect": "Allow",
                        "Resource": { "Fn::GetAtt": ["MyQueue", "Arn"] },
                        "Principal": {
                            "AWS": {
                                "Fn::Sub": "arn:aws:iam::${AWS::AccountId}:root"
                            }
                        }
                    }]
                },
                "Queues": [
                    { "Ref": "MyQueue" }
                ]
            }
        },
        "MyAdministrativeRole": {
            "Type": "AWS::IAM::Role",
            "Properties": {
                "AssumeRolePolicyDocument": {
                    "Statement": [
                        {
                            "Effect": "Allow",
                            "Principal": {
                                "AWS": [
                                    "111222333444",
                                    {"Ref": "AWS::AccountId"}
                                ]
                            },
                            "Action": [
                                "sts:AssumeRole"
                            ]
                        }
                    ]
                },
                "Policies": [
                    {
                        "PolicyName": "root",
                        "PolicyDocument": {
                            "Version": "2012-10-17",
                            "Statement": [
                                {
                                    "Effect": "Allow",
                                    "Action": "iam:PassRole",
                                    "Resource": "*"
                                }
                            ]
                        }
                    }
                ]
            }
        }
    }
}
```

You can run the validation again:

```sh
cfn-policy-validator validate --template-path cfn-policy-validator/validate-template.json \
    --region ca-central-1 \
    --allow-external-principals 111222333444 \
    --ignore-finding MyAdministrativeRole.PASS_ROLE_WITH_STAR_IN_RESOURCE
```

_Note_:

- `--allow-external-principals` can be used if an external principal is indeed trusted

</details>
