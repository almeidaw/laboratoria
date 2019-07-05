# Módulo 4 - Adicionando funcionalidades de Usuário e API com o Amazon API Gateway e AWS Cognito

![Architecture](/images/module-4/architecture-module-4.png)

**Tempo esperado:** 60 minutos

**Serviços usados:**
* [Amazon Cognito](http://aws.amazon.com/cognito/)
* [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
* [Amazon Simple Storage Service (S3)](https://aws.amazon.com/s3/)

### Resumo

Para adicionar alguns outros aspectos críticos para o site Mythical Mysfits, como permitir usuários a votar para os seu mysfit favorito e adotar um determinado mysfit, nós precisamos primeiro ter usuários registrados no site. Pra permitir registros e autenticações de usuários do site, nós vamos criar uma [**User Pool**](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-identity-pools.html) no [**AWS Cognito**](http://aws.amazon.com/cognito/) - um serviço de gerenciamento de usuários gerenciado. Então, para assegurar que apenas usuários registrados são autorizados a darem like ou adotarem mysfits no site, nós vamos criar um REST API com [**Amazon API Gateway**](https://aws.amazon.com/api-gateway/) para ficar na frente de nosso NLB. Amazon API Gateway é também um serviço gerenciado, ele provê terminação SSL, autorização de requisições, throttling, stages e versionamento de API, e muito mais.

### Adicionando uma User Pool no Website

#### Crie um User Pool no Cognito

Para criar o **Cognito User Pool** em que todos os visitantes do Mythical Mysfits vão estar armazenados, execute o seguinte comando CLI para criar a user pool *MysfitsUserPool* e indicar que todos os usuários registrados nessa pool devem, automaticamente, ter seus endereços de email verificados via confirmação de email antes de serem confirmados como usuários.  

```
aws cognito-idp create-user-pool --pool-name MysfitsUserPool --auto-verified-attributes email
```
Copie a resposta do comando anterior, ela inclui o ID único do seu user pool que você vai precisar usar em passos posteriores. Ex: `Id: us-east-1_ab12345YZ`

#### Crie o Client da Cognito User Pool

Agora, para integrar nosso frontend com o Cognito, nós precisamos criar um novo **User Pool Client** para essa user pool. Isso gera um identificador único que permitirá nosso site a ser autorizado para chamar as APIs não autenticadas no cognito onde os usuários do site podem se registrar e fazer log-in. Para criar um novo client usando o AWS CLI para a user pool anterior, rode o seguinte comando (substituindo o valor `--user-pool-id` com o que você copiou anteriormente):

```
aws cognito-idp create-user-pool-client --user-pool-id REPLACE_ME --client-name MysfitsUserPoolClient
```

### Adicionando uma nova REST API com Amazon API Gateway

#### Crie um API Gateway VPC Link

Agora, vamos tomar nossa atenção para criar uma nova RESTful API na frente do nosso serviço Flask existente, então nós podemos performar uma requisição de autorização antes de nosso NLB receber qualquer requisição. Nós faremos isso com o **Amazon API Gateway**, assim como dito no resumo do módulo. Para que o API Gateway integre-se de maneira privada com nosso NLB, nós iremos configurar um **API Gateway VPC Link** que permite APIs do API Gateway integrar diretamente com serviços do backend que estão hospedados dentro de uma VPC. **Obs:** Para o propósito desse workshop, nós criamos o NLB como *internet-facing* o que o deixou ser chamado diretamente nos módulos anteriores. Por causa disso, mesmo com nós requisitando tokens de autenticação após esse módulo, nosso NLB continuará estando aberto para acesso público atrás da API do API Gateway. Em um cenário de mundo real, você deverá criar seu NLB como *internal* desde o começo (ou criar um novo internal load balancer para substituir o já existente), sabendo isso o API Gateway seria sua estratégia para autorização de API Internet-facing. Mas por uma questão de tempo, nós usaremos o NLB que já criamos anteriormente e persiste acessível publicamente. 

Crie um VPC Link para nossa futura REST API usando o seguinte comando CLI (você vai precisar substituir o valor indicado com o ARN do Load Balancer que você salvou quando o NLB foi criado no Módulo 2): 

```
aws apigateway create-vpc-link --name MysfitsApiVpcLink --target-arns REPLACE_ME_NLB_ARN > ~/environment/api-gateway-link-output.json
```

O comando anterior vai criar um arquivo chamado `api-gateway-link-output.json` que contém o `id` para o VPC Link que está sendo criado. Isso também irá mostrar o status como `PENDING`, similar ao que está sendo mostrado abaixo. Isso vai tomar cerca de 5-10 minutos para terinar de ser criado, você pode copiar o `id` desse arquivo e proceder para o próximo passo.

```
{
    "status": "PENDING",
    "targetArns": [
        "YOUR_ARN_HERE"
    ],
    "id": "abcdef1",
    "name": "MysfitsApiVpcLink"
}
``` 

Com o VPC Link criado, nós podemos criar a REST API real usando o Amazon API Gateway.

#### Crie uma REST API usando Swagger

Sua REST API MythicalMysfits é definida usando **Swagger**, um popular framework open-source para descrição de APIs via JSON. Esse Swagger com a definição da API está localizada em `~/environment/aws-modern-applicaiton-workshop/module-4/aws-cli/api-swagger.json`. Abra esse arquivo, você verá a REST API e todos seus recursos, métodos e configurações.  

Existem diversos espaços em um arquivo JSON que precisam ser atualizados para incluir parametros específicos para o seu Cognito User Pool, assim como seu Network Load Balancer.    

O objeto `securityDefinitions` presente na definição da API indica que nós configuramos um mecanismo de autorização de apiKey usando o Authorization header. Você perceberá que a AWS provê extensões customizadas para Swagger usando o prefixo `x-amazon-api-gateway-`.

Pressione CTRL-F no arquivo para procurar os vários `REPLACE_ME` presentes. Substitua-los pelos parâmetros específicos condizentes. Uma vez terminado, salve o arquivo e execute o seguinte comando AWS CLI: 

```
aws apigateway import-rest-api --parameters endpointConfigurationTypes=REGIONAL --body file://~/environment/aws-modern-application-workshop/module-4/aws-cli/api-swagger.json --fail-on-warnings
```

Copie a resposta desse comando e salve o valor do `id` para o próximo passo:

```
{
    "name": "MysfitsApi",
    "endpointConfiguration": {
        "types": [
            "REGIONAL"
        ]
    },
    "id": "abcde12345",
    "createdDate": 1529613528
}
```

#### Deploy da API

Agora, nossa API já foi criada, mas ainda está para ser implantada em qualquer lugar. Para fazer o deploy de nossa API, nós precisamos primeiro criar o deployment e indicar qual **stage** desse deployment. Um Stage é uma referencia para o deployment, basicamente é uma snapshot da API. Um Stage é usado para gerenciar e otimizar um deployment particular. Por exemplo, você pode alterar as configurações de um stage para permitir caching, customizar throttling de requisições, configurar logging ou definir variáveis do stage. Vamos chamar nosso stage `prod`. Para criar o deployment para o prod stage, execute o seguinte comando CLI:

```
aws apigateway create-deployment --rest-api-id REPLACE_ME_WITH_API_ID --stage-name prod
```

Com isso, nossa REST API que é capaz de fazer a autorização do usuário está disponível e rodando... mas onde?! Sua API está disponível no seguinte endereço:

```
https://REPLACE_ME_WITH_API_ID.execute-api.REPLACE_ME_WITH_REGION.amazonaws.com/prod
```

Copie o endereço anterior, substituindo os valores apropriados, e adicione `/mysfits` no fim da URI. Cole isso em seu browser, você deve ver novamente sua resposta JSON dos Mysfits. Mas, nós adicionamos diversas funcionalidade como adoção e like de mysfits que nosso serviço Flask do backend ainda não implementa.

Vamos resolver isso agora.


### Atualizando o Site Mythical Mysfits

#### Atualizando o Flask Service Backend

Para acomodar as novas funcionalidades para ver os perfis dos Mysfits, likes e adoções, nós precisamos incluir o código python atualizado para nosso backend. Vamos sobrescrever sua base de código já existente com esses arquivos e colocalos no repositório:

```
cd ~/environment/MythicalMysfitsService-Repository/
```

```
cp -r ~/environment/aws-modern-application-workshop/module-4/app/* .
```

```
git add .
```

```
git commit -m "Update service code backend to enable additional website features."
```

```
git push
```

Enquanto essas atualizações no serviço estão sendo automaticamente depositadas no seu Pipeline de Ci/CD, continue para o próximo passo.


#### Atualize o site da Mythical Mysfits no S3

Abra a nova versão do index.html do Mythical Mysfits que nós colocaremos no S3 em breve, ela está localizada em: `~/environment/aws-modern-application-workshop/module-4/app/web/index.html`.
Nesse novo index.html, você irá perceber códigos HTML e JavaScript adicionais que estão sendo usados para adicionar as funcionalidades de registro e login de usuários. Esse código está interagindo com o SDK do JavaScript do AWS Cognito para ajudar a gerenciar o registro, autenticação, e autorização para todas as chamadas API que precisam disso. 

Nesse arquivo, substitua os **REPLACE_ME** com os OutputValues que você copiou anteriormente dentro de aspas simples e salve o arquivo:

![before-replace](/images/module-4/before-replace.png)

Também, para o processo de registro, você tem dois outros arquivos HTML adicionais para inserir esses valores dentro. `register.html` e `confirm.html`. Insira os valores copiados no **REPLACE_ME** nesses arquivos também.

Agora, vamos copiar esses arquivos HTML, assim como o JavaScript SDK do Cognito para o bucket S3 que está hospedando nosso site, então as novas funcionalidades serão publicadas online.

```
aws s3 cp --recursive ~/environment/aws-modern-application-workshop/module-4/web/ s3://YOUR-S3-BUCKET/
```

Recarregue o site do Mythical Mysfits no seu browser para ver as novas funcionalidades em ação!

Isso conclui o Módulo 4.

[Prossiga para o Módulo 5](/module-5)


## [Centro do desenvolvedor da AWS](https://developer.aws)
