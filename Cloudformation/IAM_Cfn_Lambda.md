
## **Overview**


This section of a CloudFormation template defines a Lambda function resource named CreateUsersFunction.
Its purpose is to automate IAM user creation, password generation, and Secrets Manager storage during CloudFormation stack creation — and clean up those resources when the stack is deleted.

The function also uses cfnresponse, which allows Lambda-backed custom resources to communicate success or failure back to CloudFormation.

```yaml
CreateUsersFunction:
  Type: AWS::Lambda::Function
  Properties:
    Handler: index.handler
    Runtime: python3.11
    Role: !GetAtt LambdaExecutionRole.Arn
    Timeout: 90
    Code:
      ZipFile: |
        # (Python code here)
```