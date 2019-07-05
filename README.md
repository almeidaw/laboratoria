# Construa uma Aplicação Moderna na AWS (Python)

![mysfits-welcome](/images/module-1/mysfits-welcome.png)

### Bem-vinda(o) à versão em **Python** do Construa uma Aplicação Moderna no Workshop AWS!

**Experiência em AWS: Iniciante**

**Tempo para Completar: 3-4 horas**

**Custo para Completar: Muitos dos serviços usados estão inclusos no AWS Free Tier. Para aqueles que não estão, a aplicação de exemplo custará, no total, menos de US$1/dia.**

**Pré-requisitos do Tutorial:**

* **Uma conta AWS com acesso de Administrator**

Por favor, certifique-se de desligar todos os recursos criados durante esse workshop para garantir que não haja cobranças adicionais.

**Nota:**  Os custos estimados do workshop assumem que seu demo website criado como parte desse workshop servirá um tráfego nulo ou pequeno.

### Arquitetura da Aplicação

![Application Architecture](/images/arch-diagram.png)

O website Mythical Mysfits entrega seus conteúdo estático a partir do Amazon S3 com o Amazon CloudFront, possui uma API de micro-serviço no backend lançado como um container usando o AWS Fargate no Amazon ECS, armazena dados em uma banco de dados NoSQL gerenciado criado no Amazon DynamoDB, com autenticação e autorização para a aplicação feitas através do AWS API Gateway e sua integração com o Amazon Cognito.  Os cliques do usuário do website serão enviados como registros (records) para um fluxo de entrega do Amazon Kinesis Firehose, onde eles serão processados por funções AWS Lambda, que é serverless, e então armazenados no Amazon S3.

Você irá criar e fazer o deploy (implantar) de mudanças de sua aplicação de maneira totalmente programática. Você usará o AWS Command Line Interface para executar comandos para criar os componentes de infraestrutura necessários, incluindo um stack CI/CD totalmente gerenciado utilizando o AWS CodeCommit, CodeBuild, e CodePipeline. Finalmente você completará todas as tarefas de desenvolvimento exigidas dentro do seu próprio navegador usando a IDE baseada na nuvem AWS Cloud9.  

## Começar o Workshop de Aplicação Moderna

### [Ir para o Módulo 1](/module-1)


### Limpeza do Workshop (Uma vez que você tenha terminado)
Certifique-se de excluor todos os recursos criados durante o workshop como forma de garantir a cobrança por esses recursos não continue por um período maior do que o planejado. Nós recomendamos que você utilize o Console da AWS para explorar os recursos que voc
ê criou e excluí-los quando você estiver pronta(o).

Para os dois casos em que você provisionou recursos usando o AWS CloudFormation, você pode removê-los simplesmente rodando o seguinte comando do AWS CLI para cada stack do CloudFormation:

```
aws cloudformation delete-stack --stack-name NOME-DA-STACK-AQUI
```

Para remover todos os recursos criados você pode visitar os seguintes consoles da AWS, que contém os recursos que você criou durante o workshop Mythical Mysfits:
* [AWS Kinesis](https://console.aws.amazon.com/kinesis/home)
* [AWS Lambda](https://console.aws.amazon.com/lambda/home)
* [Amazon S3](https://console.aws.amazon.com/s3/home)
* [Amazon API Gateway](https://console.aws.amazon.com/apigateway/home)
* [Amazon Cognito](https://console.aws.amazon.com/cognito/home)
* [AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home)
* [AWS CodeBuild](https://console.aws.amazon.com/codebuild/home)
* [AWS CodeCommit](https://console.aws.amazon.com/codecommit/home)
* [Amazon DynamoDB](https://console.aws.amazon.com/dynamodb/home)
* [Amazon ECS](https://console.aws.amazon.com/ecs/home)
* [Amazon EC2](https://console.aws.amazon.com/ec2/home)
* [Amazon VPC](https://console.aws.amazon.com/vpc/home)
* [AWS IAM](https://console.aws.amazon.com/iam/home)
* [AWS CloudFormation](https://console.aws.amazon.com/cloudformation/home)
* [Amazon CloudFront](https://console.aws.amazon.com/cloudfront/home)


[Ir para o Módulo 1](/module-1)


## [Centro do desenvolvedor da AWS](https://developer.aws)
