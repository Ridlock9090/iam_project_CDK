
# **Overview**


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

# Explanation
Property:Meaning
---

- Type:  Declares this is a Lambda function resource.  

- Handler:  The function inside the code that will be executed — here, index.handler (meaning the file “index.py” and the function “handler”).  

- Runtime:  Specifies the Python version (python3.11).  
 
- Role:  The IAM role that gives this Lambda permission to create/delete IAM users, groups, and Secrets Manager secrets.  

- Timeout:  90 seconds (the maximum time Lambda can run before being stopped).  

- Code.ZipFile:  Inline Python code (CloudFormation embeds the code directly instead of referencing an external ZIP).  



### Python Code Explanation

```py
import boto3, cfnresponse, secrets, string, random
iam = boto3.client('iam')
sm = boto3.client('secretsmanager')
```

boto3: AWS SDK for Python — used to call IAM and Secrets Manager.
cfnresponse: AWS-provided helper to send a success or failure signal back to CloudFormation.
iam client: Used to manage IAM users and groups.
sm client: Used to manage AWS Secrets Manager secrets.

### Password Generation Helper

```py
def generate_password(length=14):
    chars = string.ascii_letters + string.digits + "!@#$%^&*()-_"
    return ''.join(random.choice(chars) for _ in range(length))
```

Generates a random 14-character password using uppercase and lowercase letters, digits, and special symbols.

### Main Handler Function

```py
props = event['ResourceProperties']
groups = {
    "Dev": props.get("DevUsers", []),
    "OpsManager": props.get("OpsManagerUsers", []),
    "FinanceManager": props.get("FinanceManagerUsers", []),
    "DataAnalysts": props.get("DataAnalystUsers", [])
}
```

CloudFormation passes input parameters through: ResourceProperties.
The function reads user lists for each IAM group.

### Normalize Comma-Separated Strings

```py
for g in groups:
    if isinstance(groups[g], str):
        groups[g] = [u.strip() for u in groups[g].split(',') if u.strip()]
```

If the users were passed as comma-separated strings in the CloudFormation template, this converts them into Python lists for easier processing.

### Delete Event Handling

When CloudFormation deletes the stack, the Lambda function cleans up all created resources:

```py
if event['RequestType'] == 'Delete':
    for group, users in groups.items():
        for user in users:
            # Remove login profile (console password)
            iam.delete_login_profile(UserName=user)
            # Remove from group
            iam.remove_user_from_group(GroupName=group, UserName=user)
            # Delete IAM user
            iam.delete_user(UserName=user)
            # Delete stored secret
            sm.delete_secret(SecretId=f"IAM/{user}", ForceDeleteWithoutRecovery=True)
    cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
    return
```

***Purpose:*** Ensures that all IAM users and Secrets Manager entries created by the stack are deleted when the stack is removed.

### Create Event Handling

When the stack is ***created*** or ***updated***, it:

1. Creates users in IAM.
2. Creates login profiles (passwords).
3. Adds users to the correct IAM groups.
4. Stores their credentials in AWS Secrets Manager.

```py
for group, users in groups.items():
    for user in users:
        password = generate_password()
        iam.create_user(UserName=user)
        iam.create_login_profile(
            UserName=user,
            Password=password,
            PasswordResetRequired=True
        )
        iam.add_user_to_group(GroupName=group, UserName=user)
        sm.create_secret(
            Name=f"IAM/{user}",
            Description=f"Login credentials for IAM user {user}",
            SecretString=f'{{"username": "{user}", "password": "{password}"}}'
        )

```

If a user already exists, the function ***skips it*** using a try/except for EntityAlreadyExistsException.

Finally, it reports success back to CloudFormation:

```py
cfnresponse.send(event, context, cfnresponse.SUCCESS, {"UsersCreated": ", ".join(created)})

```
### Error Handling ###

If any step fails:

```py
except Exception as e:
    cfnresponse.send(event, context, cfnresponse.FAILED, {"Error": str(e)})

```
- This ensures CloudFormation knows that the custom resource failed.
- The stack deployment will then show an appropriate error message.

### In summary:
This inline Lambda function is part of a CloudFormation custom resource that automates IAM user creation, password management, and Secrets Manager storage, while ensuring proper cleanup on stack deletion.