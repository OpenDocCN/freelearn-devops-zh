# 第九章：管理秘密

秘密管理是现代应用程序和系统的关键安全和运营要求。诸如用户名和密码之类的凭据通常用于验证对可能包含私人和敏感数据的资源的访问，因此非常重要的是，您能够实现一个能够以安全方式向您的应用程序提供这些凭据的秘密管理解决方案，而不会将它们暴露给未经授权的方。 

基于容器的应用程序的秘密管理具有挑战性，部分原因是容器的短暂性质以及在一次性和可重复基础设施上运行容器的基本要求。长期存在的服务器已经过去了，您可以在本地文件中存储秘密 - 现在您的服务器是可以来来去去的 ECS 容器实例，并且您需要一些机制能够在运行时动态地将秘密注入到您的应用程序中。我们迄今为止在本书中使用的一个天真的解决方案是使用环境变量直接将您的秘密注入到您的应用程序中；然而，这种方法被认为是不安全的，因为它经常会通过各种运营数据源以纯文本形式暴露您的秘密。一个更健壮的解决方案是实现一个安全的凭据存储，您的应用程序可以以安全的方式动态检索其秘密 - 然而，设置您自己的凭据存储可能会很昂贵、耗时，并引入重大的运营开销。

在本章中，您将实现一个简单而有效的秘密管理解决方案，由两个关键的 AWS 服务提供支持 - AWS Secrets Manager 和密钥管理服务或 KMS。这些服务将为您提供一个基于云的安全凭据存储，易于管理、成本效益，并且完全集成了标准的 AWS 安全控制，如 IAM 策略和角色。您将学习如何将支持通过环境变量进行配置的任何应用程序与您的秘密管理解决方案集成，方法是在您的 Docker 映像中创建一个入口脚本，该脚本使用 AWS CLI 动态地检索和安全地注入秘密到您的内部容器环境中，并且还将学习如何在使用 CloudFormation 部署您的环境时，将秘密暴露给 CloudFormation 堆栈中的其他资源。

以下主题将被涵盖：

+   创建 KMS 密钥

+   使用 AWS Secrets Manager 创建秘密

+   在容器启动时注入秘密

+   使用 CloudFormation 提供秘密

+   将秘密部署到 AWS

# 技术要求

以下列出了完成本章所需的技术要求：

+   对 AWS 帐户具有管理员访问权限

+   根据第三章的说明配置本地 AWS 配置文件

+   AWS CLI 版本 1.15.71 或更高版本

+   第八章需要完成，并成功部署示例应用程序到 AWS

以下 GitHub URL 包含本章中使用的代码示例 - [`github.com/docker-in-aws/docker-in-aws/tree/master/ch9`](https://github.com/docker-in-aws/docker-in-aws/tree/master/ch9)。

查看以下视频以查看代码的实际操作：

[`bit.ly/2LzpEY2`](http://bit.ly/2LzpEY2)

# 创建 KMS 密钥

任何秘密管理解决方案的关键构建块是使用加密密钥加密您的凭据，这确保了您的凭据的隐私和保密性。AWS 密钥管理服务（KMS）是一项托管服务，允许您创建和控制加密密钥，并提供了一个简单、低成本的解决方案，消除了许多管理加密密钥的操作挑战。KMS 的关键功能包括集中式密钥管理、符合许多行业标准、内置审计和与其他 AWS 服务的集成。

在构建使用 AWS Secrets Manager 的秘密管理解决方案时，您应该在本地 AWS 帐户和区域中创建至少一个 KMS 密钥，用于加密您的秘密。AWS 确实提供了一个默认的 KMS 密钥，您可以在 AWS Secrets Manager 中使用，因此这不是一个严格的要求，但是一般来说，根据您的安全要求，您应该能够创建自己的 KMS 密钥。

您可以使用 AWS 控制台和 CLI 轻松创建 KMS 密钥，但是为了符合采用基础设施即代码的一般主题，我们将使用 CloudFormation 创建一个新的 KMS 密钥。

以下示例演示了在新的 CloudFormation 模板文件中创建 KMS 密钥和 KMS 别名，您可以将其放在 todobackend-aws 存储库的根目录下，我们将其称为`kms.yml`：

```
AWSTemplateFormatVersion: "2010-09-09"

Description: KMS Keys

Resources:
  KmsKey:
    Type: AWS::KMS::Key
    Properties:
      Description: Custom key for Secrets
      Enabled: true
      KeyPolicy:
        Version: "2012-10-17"
        Id: key-policy
        Statement: 
          - Sid: Allow root account access to key
            Effect: Allow
            Principal:
              AWS: !Sub arn:aws:iam::${AWS::AccountId}:root
            Action:
              - kms:*
            Resource: "*"
  KmsKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/secrets-key
      TargetKeyId: !Ref KmsKey

```

```
Outputs:
  KmsKey:
    Description: Secrets Key KMS Key ARN
    Value: !Sub ${KmsKey.Arn}
    Export:
      Name: secrets-key
```

使用 CloudFormation 创建 KMS 资源

在前面的例子中，您创建了两个资源——一个名为`KmsKey`的`AWS::KMS::Key`资源，用于创建新的 KMS 密钥，以及一个名为`KmsKeyAlias`的`AWS::KMS::Alias`资源，用于为密钥创建别名或友好名称。

`KmsKey`资源包括一个`KeyPolicy`属性，该属性定义了授予根帐户对密钥访问权限的资源策略。这是您创建的任何 KMS 密钥的要求，以确保您始终至少有一些方法访问密钥，您可能已经使用该密钥加密了有价值的数据，如果密钥不可访问，这将给业务带来相当大的成本。 

如果您通过 AWS 控制台或 CLI 创建 KMS 密钥，根帐户访问策略将自动为您创建。

在前面的示例中，CloudFormation 模板的一个有趣特性是创建了一个 CloudFormation 导出，每当您将`Export`属性添加到 CloudFormation 输出时就会创建。在前面的示例中，`KmsKey`输出将`Value`属性指定的`KmsKey`资源的 ARN 导出，而`Export`属性创建了一个 CloudFormation 导出，您可以在其他 CloudFormation 堆栈中引用它，以注入导出的值，而不必明确指定导出的值。稍后在本章中，您将看到如何利用这个 CloudFormation 导出，所以如果现在还不太明白，不用担心。

有了前面示例中的配置，假设您已经将此模板放在名为`kms.yml`的文件中，现在可以部署新的堆栈，这将导致创建新的 KMS 密钥和 KMS 资源：

```
> export AWS_PROFILE=docker-in-aws
> aws cloudformation deploy --template-file kms.yml --stack-name kms
Enter MFA code for arn:aws:iam::385605022855:mfa/justin.menga:

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - kms
> aws cloudformation list-exports
{
    "Exports": [
        {
            "ExportingStackId": "arn:aws:cloudformation:us-east-1:385605022855:stack/kms/be0a6d20-3bd4-11e8-bf63-50faeaabf0d1",
            "Name": "secrets-key",
            "Value": "arn:aws:kms:us-east-1:385605022855:key/ee08c380-153c-4f31-bf72-9133b41472ad"
        }
    ]
}
```

使用 CloudFormation 部署 KMS 密钥

在前面的例子中，在创建 CloudFormation 堆栈之后，请注意`aws cloudformation list-exports`命令现在列出了一个名为`secrets-key`的单个导出。此导出的值是您堆栈中 KMS 密钥资源的 ARN，您现在可以在其他 CloudFormation 堆栈中使用`Fn::ImportValue`内部函数来导入此值，只需简单地引用`secrets-key`的导出名称（例如，`Fn::ImportValue: secrets-key`）。

在使用 CloudFormation 导出时要小心。这些导出是用于引用静态资源的，您导出的值在未来永远不会改变。一旦另一个堆栈引用了 CloudFormation 导出，您就无法更改该导出的值，也无法删除导出所属的资源或堆栈。CloudFormation 导出对于诸如 IAM 角色、KMS 密钥和网络基础设施（例如 VPC 和子网）等静态资源非常有用，一旦实施后就不会改变。

# 使用 KMS 加密和解密数据

现在您已经创建了一个 KMS 密钥，您可以使用这个密钥来加密和解密数据。

以下示例演示了使用 AWS CLI 加密简单纯文本值：

```
> aws kms encrypt --key-id alias/secrets-key --plaintext "Hello World"
{
    "CiphertextBlob": "AQICAHifCoHWAYb859mOk+pmJ7WgRbhk58UL9mhuMIcVAKJ18gHN1/SRRhwQVoVJvDS6i7MoAAAAaTBnBgkqhkiG9w0BBwagWjBYAgEAMFMGCSqGSIb3DQEHATAeBglghkgBZQMEAS4wEQQMYm4au5zNZG9wa5ceAgEQgCZdADZyWKTcwDfTpw60kUI8aIAtrECRyW+/tu58bYrMaZFlwVYmdA==",
    "KeyId": "arn:aws:kms:us-east-1:385605022855:key/ee08c380-153c-4f31-bf72-9133b41472ad"
}
```

使用 KMS 密钥加密数据

在上面的示例中，请注意您必须使用`--key-id`标志指定 KMS 密钥 ID 或别名，并且每当使用 KMS 密钥别名时，您总是要使用`alias/<alias-name>`作为前缀。加密数据以 Base64 编码的二进制块形式返回到`CiphertextBlob`属性中，这也方便地将加密的 KMS 密钥 ID 编码到加密数据中，这意味着 KMS 服务可以解密密文块，而无需您明确指定加密的 KMS 密钥 ID：

```
> ciphertext=$(aws kms encrypt --key-id alias/secrets-key --plaintext "Hello World" --query CiphertextBlob --output text)
> aws kms decrypt --ciphertext-blob fileb://<(echo $ciphertext | base64 --decode)
{
    "KeyId": "arn:aws:kms:us-east-1:385605022855:key/ee08c380-153c-4f31-bf72-9133b41472ad",
    "Plaintext": "SGVsbG8gV29ybGQ="
}
```

使用 KMS 密钥解密数据

在上面的示例中，您加密了一些数据，这次使用 AWS CLI 查询和文本输出选项来捕获`CiphertextBlob`属性值，并将其存储在名为`ciphertext`的 bash 变量中。然后，您使用`aws kms decrypt`命令将密文作为二进制文件传递，使用 bash 进程替换将密文的 Base64 解码值传递到二进制文件 URI 指示器（`fileb://`）中。请注意，返回的`Plaintext`值不是您最初加密的`Hello World`值，这是因为`Plaintext`值是以 Base64 编码格式，下面的示例进一步使用`aws kms decrypt`命令返回原始明文值：

```
> aws kms decrypt --ciphertext-blob fileb://<(echo $ciphertext | base64 --decode) \
    --query Plaintext --output text | base64 --decode
Hello World
```

使用 KMS 密钥解密数据并返回明文值在前两个示例中，`base64 --decode`命令用于解码 MacOS 和大多数 Linux 平台上的 Base64 值。在一些 Linux 平台（如 Alpine Linux）上，`--decode`标志不被识别，您必须使用`base64 -d`命令。

# 使用 AWS Secrets Manager 创建秘密

您已经建立了一个可以用于加密和解密数据的 KMS 密钥，现在您可以将此密钥与 AWS Secrets Manager 服务集成，这是一个在 2018 年 3 月推出的托管服务，可以让您轻松且具有成本效益地将秘密管理集成到您的应用程序中。

# 使用 AWS 控制台创建秘密

尽管在过去的几章中我们专注于通过 CloudFormation 创建 AWS 资源，但不幸的是，在撰写本文时，CloudFormation 不支持 AWS Secrets Manager 资源，因此如果您使用 AWS 工具，您需要通过 AWS 控制台或 AWS CLI 来配置您的秘密。

要通过 AWS 控制台创建新秘密，请从服务列表中选择 AWS Secrets Manager，然后单击**存储新秘密**按钮。选择**其他类型的秘密**作为秘密类型，指定秘密键和值，并选择您在本章前面创建的`secrets-key` KMS 密钥，如下面的屏幕截图所示：

![](img/770663ce-e078-403a-ad44-11ed6bd3815f.png)

使用 AWS Secrets Manager 创建新秘密

在前面的示例中，请注意 AWS Secrets Manager 允许您在单个秘密中存储多个键/值对。这很重要，因为您经常希望将秘密注入为环境变量，因此以键/值格式存储秘密允许您将环境变量名称指定为键，将秘密指定为值。

单击下一步后，您可以配置秘密名称和可选描述：

![](img/3000a2a3-9520-40df-b976-da5096b821c8.png)配置秘密名称和描述

在前面的屏幕截图中，您配置了要称为`todobackend/credentials`的秘密，我们将在本章后面用于 todobackend 应用程序。一旦您配置了秘密名称和描述，您可以单击**下一步**，跳过**配置自动轮换**部分，最后单击**存储**按钮以完成秘密的创建。

# 使用 AWS CLI 创建秘密

您还可以使用`aws secretsmanager create-secret`命令通过 AWS CLI 创建秘密：

```
> aws secretsmanager create-secret --name test/credentials --kms-key-id alias/secrets-key \
 --secret-string '{"MYSQL_PASSWORD":"some-super-secret-password"}'
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:test/credentials-l3JdTI",
    "Name": "test/credentials",
    "VersionId": "beab75bd-e9bc-4ac8-913e-aca26f6e3940"
}
```

使用 AWS CLI 创建秘密

在前面的示例中，请注意您将秘密字符串指定为 JSON 对象，这提供了您之前看到的键/值格式。

# 使用 AWS CLI 检索秘密

您可以使用`aws secretsmanager get-secret-value`命令通过 AWS CLI 检索秘密：

```
> aws secretsmanager get-secret-value --secret-id test/credentials
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:test/credentials-l3JdTI",
    "Name": "test/credentials",
    "VersionId": "beab75bd-e9bc-4ac8-913e-aca26f6e3940",
    "SecretString": "{\"MYSQL_PASSWORD\":\"some-super-password\"}",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": 1523605423.133
}
```

使用 AWS CLI 获取秘密值

在本章后面，您将为示例应用程序容器创建一个自定义入口脚本，该脚本将使用上面示例中的命令在启动时将秘密注入到应用程序容器环境中。

# 使用 AWS CLI 更新秘密

回想一下第八章，驱动 todobackend 应用程序的 Django 框架需要配置一个名为`SECRET_KEY`的环境变量，用于各种加密操作。在本章早些时候，当您创建**todobackend/credentials**秘密时，您只为用于数据库密码的`MYSQL_PASSWORD`变量创建了一个键/值对。

让我们看看如何现在更新**todobackend/credentials**秘密以添加`SECRET_KEY`变量的值。您可以通过运行`aws secretsmanager update-secret`命令来更新秘密，引用秘密的 ID 并指定新的秘密值：

```
> aws secretsmanager get-random-password --password-length 50 --exclude-characters "'\""
{
    "RandomPassword": "E2]eTfO~8Z5)&amp;0SlR-&amp;XQf=yA:B(`,p.B#R6d]a~X-vf?%%/wY"
}
> aws secretsmanager update-secret --secret-id todobackend/credentials \
    --kms-key-id alias/secrets-key \
    --secret-string '{
 "MYSQL_PASSWORD":"some-super-secret-password",
 "SECRET_KEY": "E2]eTfO~8Z5)&amp;0SlR-&amp;XQf=yA:B(`,p.B#R6d]a~X-vf?%%/wY"
 }'
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:todobackend/credentials-f7AQlO",
    "Name": "todobackend/credentials",
    "VersionId": "cd258b90-d108-4a06-b0f2-849be15f9c33"
}
```

使用 AWS CLI 更新秘密值

在上面的例子中，请注意您可以使用`aws secretsmanager get-random-password`命令为您生成一个随机密码，这对于`SECRET_KEY`变量非常理想。重要的是，您要使用`--exclude-characters`排除引号和引号字符，因为这些字符通常会导致处理这些值的 bash 脚本出现问题。

然后运行`aws secretsmanager update-secret`命令，指定正确的 KMS 密钥 ID，并提供一个更新的 JSON 对象，其中包括`MYSQL_PASSWORD`和`SECRET_KEY`键/值对。

# 使用 AWS CLI 删除和恢复秘密

可以通过运行`aws secretsmanager delete-secret`命令来删除秘密，如下例所示：

```
> aws secretsmanager delete-secret --secret-id test/credentials
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:test/credentials-l3JdTI",
    "Name": "test/credentials",
    "DeletionDate": 1526198116.323
}
```

使用 AWS CLI 删除秘密值

请注意，AWS Secrets Manager 不会立即删除您的秘密，而是在 30 天内安排删除该秘密。在此期间，该秘密是不可访问的，但可以在安排的删除日期之前恢复，如下例所示：

```
> aws secretsmanager delete-secret --secret-id todobackend/credentials
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:todobackend/credentials-f7AQlO",
    "Name": "todobackend/credentials",
    "DeletionDate": 1526285256.951
}
> aws secretsmanager get-secret-value --secret-id todobackend/credentials
An error occurred (InvalidRequestException) when calling the GetSecretValue operation: You can’t perform this operation on the secret because it was deleted.

> aws secretsmanager restore-secret --secret-id todobackend/credentials
{
    "ARN": "arn:aws:secretsmanager:us-east-1:385605022855:secret:todobackend/credentials-f7AQlO",
    "Name": "todobackend/credentials"
}

> aws secretsmanager get-secret-value --secret-id todobackend/credentials \
 --query SecretString --output text
```

```
{
  "MYSQL_PASSWORD":"some-super-secret-password",
  "SECRET_KEY": "E2]eTfO~8Z5)&amp;0SlR-&amp;XQf=yA:B(`,p.B#R6d]a~X-vf?%%/wY"
}
```

使用 AWS CLI 恢复秘密值

您可以看到，在删除秘密后，您无法访问该秘密，但是一旦使用`aws secretsmanager restore-secret`命令恢复秘密，您就可以再次访问您的秘密。

# 在容器启动时注入秘密

在 Docker 中管理秘密的一个挑战是以安全的方式将秘密传递给容器。

下图说明了一种有些天真但可以理解的方法，即使用环境变量直接注入你的秘密作为明文值，这是我们在第八章中采取的方法：

![](img/b8598acf-a39f-4589-a201-c349a97e31bd.png)通过环境变量注入密码

这种方法简单易配置和理解，但从安全角度来看并不被认为是最佳实践。当你采用这种方法时，你可以通过检查 ECS 任务定义来以明文查看你的凭据，如果你在 ECS 容器实例上运行`docker inspect`命令，你也可以以明文查看你的凭据。你也可能无意中使用这种方法记录你的秘密，这可能会无意中与未经授权的第三方共享，因此显然这种方法并不被认为是良好的实践。

另一种被认为更安全的替代方法是将你的秘密存储在安全的凭据存储中，并在应用程序启动时或在需要秘密时检索秘密。AWS Secrets Manager 就是一个提供这种能力的安全凭据存储的示例，显然这是我们在本章将重点关注的解决方案。

当你将你的秘密存储在安全的凭据存储中，比如 AWS Secrets Manager 时，你有两种一般的方法来获取你的秘密，如下图所示：

+   **应用程序注入秘密：** 采用这种方法，你的应用程序包括直接与凭据存储进行接口的支持。在这里，你的应用程序可能会寻找一个静态名称的秘密，或者可能会通过环境变量注入秘密名称。在 AWS Secrets Manager 的示例中，这意味着你的应用代码将使用 AWS SDK 来进行适当的 API 调用，以从 AWS Secrets Manager 检索秘密值。

+   **Entrypoint 脚本注入秘密：**使用这种方法，您可以将应用程序需要的秘密的名称配置为标准环境变量，然后在应用程序之前运行 entrypoint 脚本，从 AWS Secrets Manager 中检索秘密，并将它们作为环境变量注入到内部容器环境中。尽管这听起来与在 ECS 任务定义级别配置环境变量的方法类似，但不同之处在于这发生在容器内部，而外部配置的环境变量应用后，这意味着它们不会暴露给 ECS 控制台或`docker inspect`命令：

![](img/1f2b0532-b097-4637-861e-d3492194ff46.png)使用凭据存储存储和检索密码

应用程序注入秘密的方法通常从安全角度被认为是最佳方法，但这需要应用程序明确支持与您使用的凭据存储进行交互，这意味着需要额外的开发和成本来支持这种方法。

entrypoint 脚本方法被认为不太安全，因为您在应用程序外部暴露了一个秘密，但秘密的可见性仅限于容器本身，不会在外部可见。使用 entrypoint 脚本确实提供了一个好处，即不需要应用程序专门支持与凭据存储进行交互，使其成为为大多数组织提供运行时秘密的更通用解决方案，而且足够安全，这是我们现在将要关注的方法。

# 创建一个 entrypoint 脚本

Docker 的`ENTRYPOINT`指令配置了容器执行的第一个命令或脚本。当与`CMD`指令一起配置时，`ENTRYPOINT`命令或脚本被执行，`CMD`命令作为参数传递给`entrypoint`脚本。这建立了一个非常常见的模式，即 entrypoint 执行初始化任务，例如将秘密注入到环境中，然后根据传递给脚本的命令参数调用应用程序。

以下示例演示了为 todobackend 示例应用程序创建 entrypoint 脚本，您应该将其放在 todobackend 存储库的根目录中：

```
> pwd
/Users/jmenga/Source/docker-in-aws/todobackend
> touch entrypoint.sh > tree -L 1 .
├── Dockerfile
├── Makefile
├── docker-compose.yml
├── entrypoint.sh
└── src

1 directory, 4 files
```

在 Todobackend 存储库中创建一个 entrypoint 脚本

以下示例显示了入口脚本的内容，该脚本将从 AWS Secrets Manager 中注入秘密到环境中：

```
#!/bin/bash
set -e -o pipefail

# Inject AWS Secrets Manager Secrets
# Read space delimited list of secret names from SECRETS environment variable
echo "Processing secrets [${SECRETS}]..."
read -r -a secrets <<< "$SECRETS"
for secret in "${secrets[@]}"
do
  vars=$(aws secretsmanager get-secret-value --secret-id $secret \
    --query SecretString --output text \
    | jq -r 'to_entries[] | "export \(.key)='\''\(.value)'\''"')
  eval "$vars"
done

# Run application
exec "$@"
```

定义一个将秘密注入到环境中的入口脚本

在前面的例子中，从`SECRETS`环境变量创建了一个名为`secrets`的数组，该数组预计以空格分隔的格式包含一个或多个秘密的名称，这些秘密应该被处理。例如，您可以通过在示例中演示的方式设置`SECRETS`环境变量来处理名为`db/credentials`和`app/credentials`的两个秘密：

```
> export SECRETS="db/credentials app/credentials"
```

定义多个秘密

回顾前面的例子，然后脚本通过循环遍历数组中的每个秘密，使用`aws secretsmanager get-secret-value`命令获取每个秘密的`SecretString`值，然后将每个值传递给`jq`实用程序，将`SecretString`值解析为 JSON 对象，并生成一个 shell 表达式，将每个秘密键和值导出为环境变量。请注意，`jq`表达式涉及大量的转义，以确保特殊字符被解释为文字，但这个表达式的本质是为凭据中的每个键值对输出`export *key*='*value*'`。

为了进一步理解这一点，您可以在命令行上使用您之前创建的`todobackend/credentials`秘钥运行相同的命令：

```
> aws secretsmanager get-secret-value --secret-id todobackend/credentials \
 --query SecretString --output text \
 | jq -r 'to_entries[] | "export \(.key)='\''\(.value)'\''"'
export MYSQL_PASSWORD='some-super-secret-password'
export SECRET_KEY='E2]eTfO~8Z5)&amp;0SlR-&amp;XQf=yA:B(`,p.B#R6d]a~X-vf?%%/wY'
```

生成一个将秘钥导出到环境中的 Shell 表达式

在前面的例子中，请注意输出是您将执行的单独的`export`命令，以将秘密键值对注入到环境中。每个环境变量值也被单引号引起来，以确保 bash 将所有特殊字符视为文字值。

回顾前面的例子，在 for 循环中的`eval $vars`语句简单地将生成的导出语句作为 shell 命令进行评估，这导致每个键值对被注入到本地环境中。

在单独的变量中捕获`aws secretsmanager ...`命令替换的输出，可以确保任何在此命令替换中发生的错误将被传递回您的入口脚本。您可能会尝试在 for 循环中只运行一个`eval $(aws secretsmanager ..)`语句，但采用这种方法意味着如果`aws secretsmanager ...`命令替换退出并出现错误，您的入口脚本将不会意识到这个错误，并且将继续执行，这可能会导致应用程序出现奇怪的行为。

循环完成后，最终的`exec "$@"`语句将控制权交给传递给入口脚本的参数，这些参数由特殊的`$@` shell 变量表示。例如，如果您的入口脚本被调用为`entrypoint python3 manage.py migrate --noinput`，那么`$@` shell 变量将保存参数`python3 manage.py migrate --noinput`，最终的`exec`命令将启动并将控制权交给`python3 manage.py migrate --noinput`命令。

在容器入口脚本中使用`exec "$@"`方法非常重要，因为`exec`确保容器的父进程成为传递给入口点的命令参数。如果您没有使用`exec`，只是运行命令，那么运行脚本的父 bash 进程将保持为容器的父进程，并且在停止容器时，bash 进程（而不是您的应用程序）将接收到后续的信号以终止容器。通常希望您的应用程序接收这些信号，以便在终止之前优雅地清理。

# 向 Dockerfile 添加入口脚本

现在，您已经在 todobackend 存储库中建立了一个入口脚本，您需要将此脚本添加到现有的 Dockerfile，并确保使用`ENTRYPOINT`指令指定脚本作为入口点：

```
...
...
# Release stage
FROM alpine
LABEL=todobackend

# Install operating system dependencies
RUN apk add --no-cache python3 mariadb-client bash curl bats jq && \
 pip3 --no-cache-dir install awscli

# Create app user
RUN addgroup -g 1000 app && \
    adduser -u 1000 -G app -D app

# Copy and install application source and pre-built dependencies
COPY --from=test --chown=app:app /build /build
COPY --from=test --chown=app:app /app /app
RUN pip3 install -r /build/requirements.txt -f /build --no-index --no-cache-dir
RUN rm -rf /build

# Create public volume
RUN mkdir /public
RUN chown app:app /public
VOLUME /public

# Entrypoint script
COPY entrypoint.sh /usr/bin/entrypoint
RUN chmod +x /usr/bin/entrypoint
ENTRYPOINT ["/usr/bin/entrypoint"]

# Set working directory and application user
WORKDIR /app
USER app
```

向 Dockerfile 添加入口脚本

在前面的例子中，请注意您修改第一个`RUN`指令以确保安装了 AWS CLI，方法是添加`pip3 --no-cache install awscli`命令。

最后，您将入口脚本复制到`/usr/bin/entrypoint`，确保脚本具有可执行标志，并将脚本指定为镜像的入口点。请注意，您必须以 exec 样式格式配置`ENTRYPOINT`指令，以确保您在容器中运行的命令作为参数传递给入口脚本（请参阅[`docs.docker.com/engine/reference/builder/#cmd`](https://docs.docker.com/engine/reference/builder/#cmd)中的第一个注释）。

现在您的 Dockerfile 已更新，您需要提交更改，重新构建并发布 Docker 镜像更改，如下例所示：

```
> git add -A
> git commit -a -m "Add entrypoint script"
[master 5fdbe62] Add entrypoint script
 4 files changed, 31 insertions(+), 7 deletions(-)
 create mode 100644 entrypoint.sh
> export AWS_PROFILE=docker-in-aws
> make login
$(aws ecr get-login --no-include-email)
Login Succeeded
> make test && make release docker-compose build --pull release
Building release
Step 1/28 : FROM alpine AS test
latest: Pulling from library/alpine...
...
docker-compose run app bats acceptance.bats
Starting todobackend_db_1 ... done
Processing secrets []...
1..4
ok 1 todobackend root
ok 2 todo items returns empty list
ok 3 create todo item
ok 4 delete todo item
App running at http://localhost:32784
> make publish docker-compose push release
Pushing release (385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:latest)...
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
fdc98d6948f6: Pushed
9f33f154b3fa: Pushed
d8aedb2407c9: Pushed
f778da37eed6: Pushed
05e5971d2995: Pushed
4932bb9f39a5: Pushed
fa63544c9f7e: Pushed
fd3b38ee8bd6: Pushed
cd7100a72410: Layer already exists
latest: digest: sha256:5d456c61dd23728ec79c281fe5a3c700370382812e75931b45f0f5dd1a8fc150 size: 2201
Pushing app (385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend:5fdbe62)...
The push refers to repository [385605022855.dkr.ecr.us-east-1.amazonaws.com/docker-in-aws/todobackend]
fdc98d6948f6: Layer already exists
9f33f154b3fa: Layer already exists
d8aedb2407c9: Layer already exists
f778da37eed6: Layer already exists
05e5971d2995: Layer already exists
4932bb9f39a5: Layer already exists
fa63544c9f7e: Layer already exists
fd3b38ee8bd6: Layer already exists
cd7100a72410: Layer already exists
34d86eb: digest: sha256:5d456c61dd23728ec79c281fe5a3c700370382812e75931b45f0f5dd1a8fc150 size: 2201
```

发布更新的 Docker 镜像

在上面的示例中，当 Docker 镜像发布时，请注意应用程序服务的 Docker 标签（在我的示例中为`5fdbe62`，实际哈希值会因人而异），您可以从第一章中回忆起，它指定了源代码库的 Git 提交哈希。您将在本章后面需要此标签，以确保您可以部署您的更改到在 AWS 中运行的 todobackend 应用程序。

# 使用 CloudFormation 提供秘密

您已在 AWS Secrets Manager 中创建了一个秘密，并已添加了支持使用入口脚本将秘密安全地注入到容器中的功能。请记住，入口脚本会查找一个名为`SECRETS`的环境变量，而您 CloudFormation 模板中的`ApplicationTaskDefinition`和`MigrateTaskDefinition`资源目前正在直接注入应用程序数据库。为了支持在您的堆栈中使用秘密，您需要配置 ECS 任务定义，以包括`SECRETS`环境变量，并配置其名称为您的秘密名称，并且您还需要确保您的容器具有适当的 IAM 权限来检索和解密您的秘密。

另一个考虑因素是您的`ApplicationDatabase`资源的密码是如何配置的——目前配置为使用堆栈参数输入的密码；但是，您的数据库现在需要能够以某种方式从您新创建的秘密中获取其密码。

# 配置 ECS 任务定义以使用秘密

首先要处理重新配置 ECS 任务定义以使用您新创建的秘密。您的容器现在包括一个入口脚本，该脚本将从 AWS Secrets Manager 中检索秘密，并且在更新各种 ECS 任务定义以将您的秘密名称导入为环境变量之前，您需要确保您的容器具有执行此操作的正确权限。虽然您可以将此类权限添加到应用于 EC2 实例级别的 ECS 容器实例角色，但更安全的方法是创建特定的 IAM 角色，您可以将其分配给您的容器，因为您可能会与多个应用程序共享 ECS 集群，并且不希望从在集群上运行的任何容器中授予对您秘密的访问权限。

ECS 包括一个名为 IAM 任务角色的功能（[`docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html`](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)），它允许您在 ECS 任务定义级别授予 IAM 权限，并且在我们只想要将对 todobackend 秘密的访问权限授予 todobackend 应用程序的情况下非常有用。以下示例演示了创建授予这些特权的 IAM 角色：

```
...
...
Resources:
  ...
  ...
  ApplicationTaskRole:
 Type: AWS::IAM::Role
 Properties:
 AssumeRolePolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Effect: Allow
 Principal:
 Service: ecs-tasks.amazonaws.com
 Action:
 - sts:AssumeRole
 Policies:
 - PolicyName: SecretsManagerPermissions
 PolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Sid: GetSecrets
 Effect: Allow
 Action:
 - secretsmanager:GetSecretValue
 Resource: !Sub arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:todobackend/*
 - Sid: DecryptSecrets
 Effect: Allow
 Action:
 - kms:Decrypt
 Resource: !ImportValue secrets-key
  ApplicationTaskDefinition:
    Type: AWS::ECS::TaskDefinition
...
...
```

创建 IAM 任务角色

在前面的示例中，您创建了一个名为`ApplicationTaskRole`的新资源，其中包括一个`AssumeRolePolicyDocument`属性，该属性定义了可以承担该角色的受信任实体。请注意，这里的主体是`ecs-tasks.amazonaws.com`服务，这是您的容器在尝试使用 IAM 角色授予的权限访问 AWS 资源时所假定的服务上下文。该角色包括一个授予`secretsmanager:GetSecretValue`权限的策略，这允许您检索秘密值，这个权限被限制为所有以`todobackend/`为前缀命名的秘密的 ARN。如果您回顾一下之前的示例，当您通过 AWS CLI 创建了一个测试秘密时，您会发现秘密的 ARN 包括 ARN 末尾的随机值，因此您需要在 ARN 中使用通配符，以确保您具有权限，而不考虑这个随机后缀。请注意，该角色还包括对`secrets-key` KMS 密钥的`Decrypt`权限，并且您使用`!ImportValue`或`Fn::ImportValue`内部函数来导入您在第一个示例中导出的 KMS 密钥的 ARN。

有了`ApplicationTaskRole`资源，以下示例演示了如何重新配置`stack.yml`文件中的`todobackend-aws`存储库中的`ApplicationTaskDefinition`和`MigrateTaskDefinition`资源：

```
Parameters:
  ...
  ...
  ApplicationSubnets:
    Type: List<AWS::EC2::Subnet::Id>
    Description: Target subnets for EC2 instances
 # The DatabasePassword parameter has been removed
  VpcId:
    Type: AWS::EC2::VPC::Id
    Description: Target VPC
 ...
  ... 
Resources:
  ...
  ...
  MigrateTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: todobackend-migrate
 TaskRoleArn: !Sub ${ApplicationTaskRole.Arn}
      ContainerDefinitions:
        - Name: migrate
          Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/docker-in-aws/todobackend:${ApplicationImageTag}
          MemoryReservation: 5
          Cpu: 5
          Environment:
            - Name: DJANGO_SETTINGS_MODULE
              Value: todobackend.settings_release
            - Name: MYSQL_HOST
              Value: !Sub ${ApplicationDatabase.Endpoint.Address}
            - Name: MYSQL_USER
              Value: todobackend
            - Name: MYSQL_DATABASE
              Value: todobackend
            # The MYSQL_PASSWORD variable has been removed
 - Name: SECRETS
 Value: todobackend/credentials
            - Name: AWS_DEFAULT_REGION
              Value: !Ref AWS::Region  ...
  ...
  ApplicationTaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Family: todobackend
 TaskRoleArn: !Sub ${ApplicationTaskRole.Arn}
      Volumes:
        - Name: public
      ContainerDefinitions:
        - Name: todobackend
          Image: !Sub ${AWS::AccountId}.dkr.ecr.${AWS::Region}.amazonaws.com/docker-in-aws/todobackend:${ApplicationImageTag}
          MemoryReservation: 395
          Cpu: 245
          MountPoints:
            - SourceVolume: public
              ContainerPath: /public
          Environment:- Name: DJANGO_SETTINGS_MODULE
              Value: todobackend.settings_release
            - Name: MYSQL_HOST
              Value: !Sub ${ApplicationDatabase.Endpoint.Address}
            - Name: MYSQL_USER
              Value: todobackend
            - Name: MYSQL_DATABASE
              Value: todobackend
 # The MYSQL_PASSWORD and SECRET_KEY variables have been removed            - Name: SECRETS
 Value: todobackend/credentials
            - Name: AWS_DEFAULT_REGION
              Value: !Ref AWS::Region
...
...
```

配置 ECS 任务定义以使用秘密

在上面的示例中，您配置每个任务定义使用 IAM 任务角色通过`TaskRoleArn`属性，该属性引用了您在上一个示例中创建的`ApplicationTaskRole`资源。接下来，您添加新入口脚本在您的 Docker 镜像中期望的`SECRETS`环境变量，并删除先前从 AWS Secrets Manager 服务中检索的`MYSQL_PASSWORD`和`SECRET_KEY`变量。请注意，您需要包括一个名为`AWS_DEFAULT_REGION`的环境变量，因为这是 AWS CLI 所需的，以确定您所在的区域。

因为您不再将数据库密码作为参数注入到堆栈中，您还需要更新 todobackend-aws 存储库中的`dev.cfg`文件，并且还要指定您在之前示例中发布的更新的 Docker 镜像标记：

```
ApplicationDesiredCount=1
ApplicationImageId=ami-ec957491
ApplicationImageTag=5fdbe62
ApplicationSubnets=subnet-a5d3ecee,subnet-324e246f
VpcId=vpc-f8233a80
```

更新输入参数

在上面的示例中，`DatabasePassword=my-super-secret-password`行已被删除，并且`ApplicationImageTag`参数的值已被更新，引用了您新更新的 Docker 镜像上标记的提交哈希。

# 向其他资源公开秘密

您已更新了 ECS 任务定义，使您的应用容器现在将从 AWS Secrets Manager 中提取秘密并将它们注入为环境变量。这对于您的 Docker 镜像效果很好，因为您可以完全控制您的镜像的行为，并且可以添加诸如入口脚本之类的功能来适当地注入秘密。对于依赖这些秘密的其他资源，您没有这样的能力，例如，您堆栈中的`ApplicationDatabase`资源定义了一个 RDS 实例，截至撰写本文时，它不包括对 AWS Secrets Manager 的本地支持。

解决这个问题的一个方法是创建一个 CloudFormation 自定义资源，其工作是查询 AWS Secrets Manager 服务并返回与给定秘密相关的秘密值。因为自定义资源可以附加数据属性，所以您可以在其他资源中引用这些属性，提供一个简单的机制将您的秘密注入到任何不原生支持 AWS Secrets Manager 的 CloudFormation 资源中。如果您对这种方法的安全性有疑问，CloudFormation 自定义资源响应规范（[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-responses.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/crpg-ref-responses.html)）包括一个名为`NoEcho`的属性，该属性指示 CloudFormation 不通过控制台或日志信息公开数据属性。通过设置此属性，您可以确保您的秘密不会因查询 CloudFormation API 或审查 CloudFormation 日志而无意中暴露。

# 创建一个 Secrets Manager Lambda 函数

以下示例演示了向您的 CloudFormation 堆栈添加一个 Lambda 函数资源，该函数查询 AWS Secrets Manager 服务，并返回给定秘密名称和秘密值内键/值对中的目标键的秘密值：

```
...
...
Resources:
  SecretsManager:
 Type: AWS::Lambda::Function
 DependsOn:
 - SecretsManagerLogGroup
 Properties:
 FunctionName: !Sub ${AWS::StackName}-secretsManager
 Description: !Sub ${AWS::StackName} Secrets Manager
 Handler: index.handler
 MemorySize: 128
 Runtime: python3.6
 Timeout: 300
 Role: !Sub ${SecretsManagerRole.Arn}
 Code:
 ZipFile: |
 import cfnresponse, json, sys, os
 import boto3

 client = boto3.client('secretsmanager')

 def handler(event, context):
            sys.stdout = sys.__stdout__
 try:
 print("Received event %s" % event)
 if event['RequestType'] == 'Delete':
 cfnresponse.send(event, context, cfnresponse.SUCCESS, {}, event['PhysicalResourceId'])
 return
 secret = client.get_secret_value(
 SecretId=event['ResourceProperties']['SecretId'],
 )
 credentials = json.loads(secret['SecretString'])
              # Suppress logging output to ensure credential values are kept secure
              with open(os.devnull, "w") as devnull:
                sys.stdout = devnull
                cfnresponse.send(
                  event, 
                  context, 
                  cfnresponse.SUCCESS,
                  credentials, # This dictionary will be exposed to CloudFormation resources
                  secret['VersionId'], # Physical ID of the custom resource
                  noEcho=True
                )
 except Exception as e:
 print("A failure occurred with exception %s" % e)
 cfnresponse.send(event, context, cfnresponse.FAILED, {})
 SecretsManagerRole:
 Type: AWS::IAM::Role
 Properties:
 AssumeRolePolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Effect: Allow
 Principal:
 Service: lambda.amazonaws.com
 Action:
 - sts:AssumeRole
 Policies:
 - PolicyName: SecretsManagerPermissions
 PolicyDocument:
 Version: "2012-10-17"
 Statement:
 - Sid: GetSecrets
 Effect: Allow
 Action:
 - secretsmanager:GetSecretValue
 Resource: !Sub arn:aws:secretsmanager:${AWS::Region}:${AWS::AccountId}:secret:todobackend/*
            - Sid: DecryptSecrets
              Effect: Allow
              Action:
 - kms:Decrypt
 Resource: !ImportValue secrets-key
- Sid: ManageLambdaLogs
 Effect: Allow
 Action:
 - logs:CreateLogStream
 - logs:PutLogEvents
 Resource: !Sub ${SecretsManagerLogGroup.Arn}
```

```
SecretsManagerLogGroup:
 Type: AWS::Logs::LogGroup
 Properties:
 LogGroupName: !Sub /aws/lambda/${AWS::StackName}-secretsManager
 RetentionInDays: 7...
  ...
```

添加一个 Secrets Manager CloudFormation 自定义资源函数

前面示例的配置与您在第八章中执行的配置非常相似，当时您创建了`EcsTaskRunner`自定义资源函数。在这里，您创建了一个`SecretsManager` Lambda 函数，配有一个关联的`SecretsManagerRole` IAM 角色，该角色授予了从 AWS Secrets Manager 检索和解密密钥的能力，类似于之前创建的`ApplicationTaskRole`，以及一个`SecretsManagerLogGroup`资源，用于收集来自 Lambda 函数的日志。

函数代码比 ECS 任务运行器代码更简单，期望传递一个名为 `SecretId` 的属性给自定义资源，该属性指定秘密的 ID 或名称。函数从 AWS Secrets Manager 获取秘密，然后使用 `json.loads` 方法将秘密键值对加载为名为 `credentials` 的 JSON 对象变量。然后，函数将 `credentials` 变量返回给 CloudFormation，这意味着每个凭据都可以被堆栈中的其他资源访问。请注意，您使用 `with` 语句来确保由 `cfnresponse.send` 方法打印的响应数据被抑制，通过将 `sys.stdout` 属性设置为 `/dev/null`，因为响应数据包含您不希望以明文形式暴露的秘密值。这种方法需要一些小心，您需要在 `handler` 方法的开头将 `sys.stdout` 属性恢复到其默认状态（由 `sys.__stdout__` 属性表示），因为您的 Lambda 函数运行时可能会在多次调用之间被缓存。

自定义资源函数代码可以扩展到将秘密部署到 AWS Secrets Manager。例如，您可以将预期的秘密值的 KMS 加密值作为输入，甚至生成一个随机的秘密值，然后部署和公开此凭据给其他资源。

# 创建一个秘密自定义资源

现在您已经为自定义资源准备了一个 Lambda 函数，您可以创建实际的自定义资源，该资源将提供对存储在 AWS Secrets Manager 中的秘密的访问。以下示例演示了在本章前面创建的 **todobackend/credentials** 密钥的自定义资源，然后从您的 `ApplicationDatabase` 资源中访问该密钥：

```
...
...
Resources:
  Secrets:
 Type: AWS::CloudFormation::CustomResource
 Properties:
 ServiceToken: !Sub ${SecretsManager.Arn}
 SecretId: todobackend/credentials
  SecretsManager:
    Type: AWS::Lambda::FunctionResources:
  ...
  ...
  ApplicationDatabase:
    Type: AWS::RDS::DBInstance
    Properties:
      Engine: MySQL
      EngineVersion: 5.7
      DBInstanceClass: db.t2.micro
      AllocatedStorage: 10
      StorageType: gp2
      MasterUsername: todobackend
 MasterUserPassword: !Sub ${Secrets.MYSQL_PASSWORD} ...
  ...
```

添加一个 Secrets Manager 自定义资源

在前面的示例中，您创建了一个名为 `Secrets` 的自定义资源，它通过 `ServiceToken` 属性引用 `SecretsManager` 函数，然后通过 `SecretId` 属性传递要检索的凭据的名称。然后，现有的 `ApplicationDatabase` 资源上的 `MasterUserPassword` 属性被更新为引用通过 `Secrets` 资源可访问的 `MYSQL_PASSWORD` 键，该键返回存储在 **todobackend/credentials** 密钥中的正确密码值。

# 将秘密部署到 AWS

此时，您已准备好部署对 CloudFormation 堆栈的更改，您可以使用我们在过去几章中使用的`aws cloudformation deploy`命令来执行：

```
> aws cloudformation deploy --template-file stack.yml \
 --stack-name todobackend --parameter-overrides $(cat dev.cfg) \
 --capabilities CAPABILITY_NAMED_IAM

Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - todobackend
```

部署 CloudFormation 堆栈更改

部署将影响以下资源：

+   支持自定义资源的资源将首先被创建，同时将应用于 ECS 任务定义的更改。

+   名为`Secrets`的自定义资源将被创建，一旦创建，将公开**todobackend/credentials**密钥的键/值对给其他 CloudFormation 资源。

+   `ApplicationDatabase`资源将被更新，`MasterPassword`属性将根据**todobackend/credentials**密钥中`MYSQL_PASSWORD`变量的值进行更新。

+   `MigrateTask`自定义资源将根据与关联的`MigrateTaskDefinition`的更改进行更新，并运行一个新任务，该任务使用更新后的 todobackend 镜像中的入口脚本将**todobackend/credentials**密钥中的每个键/值对导出到环境中，其中包括访问应用程序数据库所需的`MYSQL_PASSWORD`变量。

+   `ApplicationService`资源将根据与关联的`ApplicationTaskDefinition`的更改进行更新，并且类似于`MigrateTask`，每个应用程序实例现在在启动时将注入与**todobackend/credentials**密钥相关的环境变量。更新将触发`ApplicationService`的滚动部署，这将使新版本的应用程序投入使用，然后排空和移除旧版本的应用程序，而不会造成任何中断。

假设部署成功，您应该能够验证应用程序仍然成功运行，并且可以列出、添加和删除待办事项。

您还应该验证您的`SecretsManagerFunction`资源未记录秘密的明文值—以下屏幕截图显示了此功能的日志输出，并且您可以看到它抑制了发送回 CloudFormation 的成功响应的日志记录：

![](img/25145a22-a7df-45d4-bf25-3c5f3a9aa41b.png)查看 Secrets Manager 功能的日志输出

# 摘要

秘密管理对于短暂的 Docker 应用程序来说是一个挑战，其中预先配置的长时间运行的服务器并不再是一个选项，因为凭据存储在配置文件中，直接将密码作为外部配置的环境变量注入被认为是一种糟糕的安全实践。这需要一个秘密管理解决方案，使您的应用程序可以动态地从安全凭据存储中获取秘密，在本章中，您成功地使用 AWS Secrets Manager 和 KMS 服务实现了这样的解决方案。

您学会了如何创建 KMS 密钥，用于加密和解密机密信息，并由 AWS Secrets Manager 使用，以确保其存储的秘密的隐私和保密性。接下来，您将介绍 AWS Secrets Manager，并学习如何使用 AWS 控制台和 AWS CLI 创建秘密。您学会了如何在秘密中存储多个键/值对，并介绍了诸如删除保护之类的功能，其中 AWS Secrets Manager 允许您在 30 天内恢复先前删除的秘密。

有了样本应用程序的凭据存储位置，您学会了如何在容器中使用入口点脚本，在容器启动时动态获取和注入秘密值，使用简单的 bash 脚本与 AWS CLI 结合，将一个或多个秘密值作为变量注入到内部容器环境中。尽管这种方法被认为比应用程序直接获取秘密不太安全，但它的优势在于可以应用于支持环境变量配置的任何应用程序，使其成为一个更加通用的解决方案。

在为您的应用程序发布更新的 Docker 镜像后，您更新了 ECS 任务定义，以注入每个容器应检索的秘密的名称，然后创建了一个简单的自定义资源，能够将您的秘密暴露给不支持 AWS Secrets Manager 的其他类型的 AWS 资源，并且没有机制（如容器入口点脚本）来检索秘密。您确保配置了此自定义资源，以便它不会通过日志或其他形式的操作事件透露您的凭据，并更新了应用程序数据库资源，以通过此自定义资源检索应用程序的数据库密码。

有了一个安全管理解决方案，您已经解决了前几章的核心安全问题，在下一章中，您将学习如何解决应用程序的另一个安全问题，即能够独立隔离网络访问并在每个容器或 ECS 任务定义基础上应用网络访问规则。

# 问题

1.  真/假：KMS 服务要求您提供自己的私钥信息。

1.  KMS 的哪个特性允许您为密钥指定逻辑名称，而不是基于 UUID 的标识符？

1.  您想避免手动配置在多个 CloudFormation 堆栈中使用的 KMS 密钥的 ARN。假设您在单独的 CloudFormation 堆栈中定义了 KMS 密钥，您可以使用哪个 CloudFormation 功能来解决这个问题？

1.  真/假：当您从 AWS Secrets Manager 中删除一个秘密时，您永远无法恢复该秘密。

1.  在入口脚本中，您通常会使用哪些工具来从 AWS Secrets Manager 检索秘密并将秘密中的键/值对转换为适合导出到容器环境的形式？

1.  在容器入口脚本中收到一个错误，指示您没有足够的权限访问一个秘密。您检查了 IAM 角色，并确认它对该秘密允许了一个单一的`secretsmanager:GetSecretValue`权限。您需要授予哪些其他权限来解决这个问题？

1.  在处理不应公开为明文值的敏感数据时，应设置哪个 CloudFormation 自定义资源属性？

1.  在访问 AWS 资源的容器入口脚本中收到错误消息“您必须配置区域”。您应该向容器添加哪个环境变量？

# 进一步阅读

您可以查看以下链接，了解本章涵盖的主题的更多信息：

+   CloudFormation KMS 密钥资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-kms-key.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-kms-key.html)

+   CloudFormation KMS 别名资源参考：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-kms-alias.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-kms-alias.html)

+   AWS KMS 开发人员指南：[`docs.aws.amazon.com/kms/latest/developerguide/overview.html`](https://docs.aws.amazon.com/kms/latest/developerguide/overview.html)

+   AWS CLI KMS 参考：[`docs.aws.amazon.com/cli/latest/reference/kms/index.html`](https://docs.aws.amazon.com/cli/latest/reference/kms/index.html)

+   AWS Secrets Manager 用户指南：[`docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html`](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)

+   AWS CLI Secrets Manager 参考：[`docs.aws.amazon.com/cli/latest/reference/secretsmanager/index.html`](https://docs.aws.amazon.com/cli/latest/reference/secretsmanager/index.html)

+   AWS Python SDK Secrets Manager 参考：[`boto3.readthedocs.io/en/latest/reference/services/secretsmanager.html`](http://boto3.readthedocs.io/en/latest/reference/services/secretsmanager.html)

+   CloudFormation 导出：[`docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html)

+   Docker Secrets Management 的一般讨论：[`github.com/moby/moby/issues/13490`](https://github.com/moby/moby/issues/13490)
