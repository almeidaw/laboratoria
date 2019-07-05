# Módule 2: Criando um Serviço com o AWS Fargate

![Architecture](/images/module-2/architecture-module-2.png)

**Tempo esperado:** 60 minutos

**Serviços usados:**
* [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
* [AWS Identity and Access Management (IAM)](https://aws.amazon.com/iam/)
* [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/)
* [Amazon Elastic Load Balancing](https://aws.amazon.com/elasticloadbalancing/)
* [Amazon Elastic Container Service (ECS)](https://aws.amazon.com/ecs/)
* [AWS Fargate](https://aws.amazon.com/fargate/)
* [AWS Elastic Container Registry (ECR)](https://aws.amazon.com/ecr/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS CodePipeline](https://aws.amazon.com/codepipeline/)
* [AWS CodeDeploy](https://aws.amazon.com/codedeploy/)
* [AWS CodeBuild](https://aws.amazon.com/codebuild/)


### Resumo

No modulo 2, você vai hospedar um micro serviço novo usando [AWS Fargate](https://aws.amazon.com/fargate/) no [Amazon Elastic Container Service](https://aws.amazon.com/ecs/) para que o seu site “Mythical Mysthical” possa ter um back-end integrado. O AWS Fargate é uma opção para a implementação no Amazon ECS, que permite implantar containers sem precisar gerenciar clusters ou servidores. Para o back-end do site, usaremos python e criaremos um aplicativo flask em um container docker, este ficará atrás de um Network load balancer (balanceador de carga por rede). Estes iram formar o micro serviço de back-end que irá se integrar com o seu site.

### Criando a infraestrutura central com o AWS CloudFormation

Antes de criar nosso serviço, precisamos criar um ambiente de infraestrutura que sustentará o serviço, incluindo a infraestrutura de rede via [Amazon VPC](https://aws.amazon.com/vpc/), and the [AWS Identity and Access Management](https://aws.amazon.com/iam/) VPC e as funções de gerenciamento de acesso e identidade [(IAM)]( https://aws.amazon.com/iam/) da AWS que definirão as permissões que o ECS e o nossos containers terão acesso. Usaremos o [AWS CloudFormation](https://aws.amazon.com/cloudformation/) para isso, este é um serviço da AWS que permite que os clientes gerem infraestrutura como código, em dois formatos de arquivo suportados, JSON ou YAML. Esses recursos estão disponíveis em /module-2/cfn/core.yml. Este template irá criar os seguintes recursos:

* [**An Amazon VPC**](https://aws.amazon.com/vpc/) - Um ambiente de rede que é formado por 4 subnets (duas publicas e duas privadas) no espaço IP privada de 10.0.0.0/16, bem como todas as configurações de route tables. As subnets são criadas em zonas de disponibilidade da AWS separadas (AZ) para permitir alta disponibilidade em várias instalações físicas em uma região da AWS. [here](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.RegionsAndAvailabilityZones.html).
* [**Dois NAT Gateways**](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html) : (uma para cada subnet publica, também espalhadas em múltiplas AZs) – permite que os containers que nós vamos eventualmente lançar nas nossa subnets privadas possam se comunicar com a internet para baixar os pacotes necessários.
* [**Um VPC Endpoint para o DynamoDB**](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/vpc-endpoints-dynamodb.html) - O nosso micro serviço de back-end vai se comunicar com o DynamoDB [Amazon DynamoDB](https://aws.amazon.com/dynamodb/) para ter persistência (parte do modulo 3)
* [**Um Security Group**](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html) - permite que o container Docker receba trafego da internet via porta 8080 via o load balancer
* [**Regras do IAM**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) - Serão usadas para permitir que serviços da AWS possam se comunicar e interagir com outros serviços como o DynamoDB, S3 e outros.
Para criar esses recursos, rode o comando a seguir no terminal do Cloud9 (vai demorar aproximadamente 10 minutos para a stack ser criada):

```
aws cloudformation create-stack --stack-name MythicalMysfitsCoreStack --capabilities CAPABILITY_NAMED_IAM --template-body file://~/environment/aws-modern-application-workshop/module-2/cfn/core.yml   
```

Você pode checar os status da sua stack via console da AWS por esse comando:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack
```

Rode o comando “DESCRIBE-STACKS”, até você ver o status "StackStatus": "CREATE_COMPLETE" 
![cfn-complete.png](/images/module-2/cfn-complete.png)


Quando você receber essa resposta, significa que o CloudFormation terminou a criação dos recursos, como os recursos de rede e segurança. Aguarde até mostrar o `CREATE_COMPLETE` antes de prosseguir.

**Você estará usando valores de saída desde comando durante o restante do workshop. Você pode executar o seguinte comando para enviar diretamente o comando `discribe-stacks` acima para um novo arquivo no seu IDE que será armazenado como `cloudformation-core-output.json`::**

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsCoreStack > ~/environment/cloudformation-core-output.json
```

## Módulo 2a: Lançando um serviço com o AWs Fargate

### Criando um serviço de container flask:

#### Construindo a imagem Docker:

A seguir, você vai criar uma imagem para os seus containers Docker, a imagem contem todo o código e as configurações necessárias para rodar o “Mythical Maysfits” back-end como um micro serviço de API criada com Flask. Nós vamos construir a imagem do docker com o Cloud9 e depois enviar para o Amazon Elastic Container Registry, onde estará disponível para que nós possamos usar quando formos criar o serviço usando o Fargate.

Todo o código necessário para rodar o back-end do site está disponível em `/module-2/app`, repositório esse que você já clonou dentro do Cloud9 IDE. Se você desejar rever o código python que cria a API usando flask, veja no arquivo `/module-2/app/service/mythicalMysfitsService.py`

O Docker já vem instalado com com o cloud9 IDE que você já criou, então ao invés de criar a imagem localmente, devemos rodar o seguinte comando no terminal do Cloud9

* Navegue até `~/environment/module-2/app`

```
cd ~/environment/aws-modern-application-workshop/module-2/app
```

* Você pode obter o account ID e a região default no output do CloudFormation que foi previamente executado

* -	Substitua REPLACE_ME_ACCOUNT_ID por o seu ID e REPLACE-ME-REGION pela região padrão na linha de comando abaixo, para que possamos gerar a imagem do Docker usando o Docker file que contem as instruções. O -t é um comando para tags da imagem Docker, possibilitando que está possa ser encontrada depois no [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) .

```
docker build . -t REPLACE_ME_ACCOUNT_ID.dkr.ecr.REPLACE_ME_REGION.amazonaws.com/mythicalmysfits/service:latest
```

Você verá o download e a instalação do docker e todas as dependências necessárias que a aplicação precisa, e o output da criação da imagem.   **Copie a tag dessa imagem para ser usado mais tarde. No exemplo abaixo mostra a tag: 111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest**

```
Successfully built 8bxxxxxxxxab
Successfully tagged 111111111111.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest
```

#### Testando o serviço localmente

Vamos testar a imagem localmente com o Cloud9 para termos certeza que tudo está funcionando como esperado. Copie a tag da imagem que resultou dos comandos anteriores e rode os comandos para lançar o container “localmente” (que na verdade é dentro da nossa instancia com o Cloud9 na AWS !):

```
docker run -p 8080:8080 REPLACE_ME_WITH_DOCKER_IMAGE_TAG
```

como resultado, você verá um aviso do docker, falando que o mesmo está rodando localmente:

```
 * Running on http://0.0.0.0:8080/ (Press CTRL+C to quit)
```

Para testar nosso serviço com um request local, nós vamos abrir o navegador que existe dentro do Cloud9 IDE que você pode ver as aplicações que estão rodando localmente. Para abrir o navegador preview, selecione **preview > preview running applications** no menu bar do Cloud9:

![preview-menu](/images/module-2/preview-menu.png)

Isso vai abrir um novo painel na IDE onde o navegador web vai estar disponível. Adicione /mysfits no fim da URL que está no navegador, e depois pressione enter:

![preview-menu](/images/module-2/address-bar.png)

Se você ver uma uma resposta do serviço que retornou um documento JSON armazenado em `/aws-modern-application-workshop/module-2/app/service/mysfits-response.json`

Quando o teste terminar, você pode terminar o serviço pressionando control + c no PC ou MacOS. 

#### Mande a imagem Docker para o Amazon ECS

Com o teste “local” tendo sido de sucesso, vamos subir a imagem do Docker em um repositório no [Amazon Elastic Container Registry](https://aws.amazon.com/ecr/) (Amazon ECR) Ao invés de criar um registro, rode o seguinte comando, este vai criar um repositório no registro padrão do ECS na sua conta.

```
aws ecr create-repository --repository-name mythicalmysfits/service
```

A resposta para esse comando deve conter um metadata sobre a criação do repositório. Ao invés de enviar a imagem para o nosso novo repositório, nós vamos obter as credenciais para o nosso Docker para o repositório. Rode o comando a seguir, que irá retornar um comando de login para que possamos obter as credenciais do nosso Docker e executa-lo automaticamente (inclua todo o comando abaixo, até mesmo com o $). Caso retorne ‘login succeded’ é o aviso de que o comando foi executado com sucesso.

```
$(aws ecr get-login --no-include-email)
```

Agora envie a imagem para o repositório ECR que você criou usando a tag que você tinha copiado anteriormente. Com o comando abaixo será enviada toda a imagem e os componentes necessários para o ECR:

```
docker push REPLACE_ME_WITH_DOCKER_IMAGE_TAG
```

Execute o seguinte comando para ver as imagens armazenadas no repositório do ECR

```
aws ecr describe-images --repository-name mythicalmysfits/service
```

### Configurando os serviços que são pré-requisitos no Amazon ECS

#### Crie um cluster no Amazon ECS

Agora, nós já temos a imagem disponível no amazon ECR para que nós possamos lançar via ECS usando o AWS Fargate. O mesmo serviço que você testou localmente com o Cloud9 via terminal, agora lançaremos ele na nuvem e deixaremos ele disponível com um Network Load Balancer (balanceador de carga por rede).

Primeiro, crie um **Cluster** no **Amazon Elastic Container Service** (Amazon ECS). Isso representará um cluster de “servidores” onde seu serviço de container será lançado. Os servidores estão “clonadas” por que você vai usar o **AWS Fargate**. O fargate permite que você escolha um cluster onde seus container serão lançados, sem que você tenha que se preocupar em provisionar ou gerenciar nenhum servidor.

Para criar um novo cluster ECS, execute o seguinte comando:

```
aws ecs create-cluster --cluster-name MythicalMysfits-Cluster
```

#### Crie um grupo de logs no AWS CloudWatch

Proximo, nós iremos criar um grupo para logs no **AWS CloudWatch Logs**. AWS CloudWatch Logs é um serviço para armazenamento de logs e analise dos mesmos. Todos os logs que forem gerados pelos seus containers serão enviados automaticamente para o CloudWatch logs como parte desse grupo especifico. Por isso é importante usar o Fargat, já que você não vai ter acesso aos servidores que estão sustentando o seu serviço.

Para criar o grupo de logs no CloudWatch, execute o seguinte comando:

```
aws logs create-log-group --log-group-name mythicalmysfits-logs
```

#### Registre uma definição de tarefa no ECS

Agora que nós temos um cluster registrado e um grupo de logs definido para onde os logs dos nossos containers irão, nós estamos prontos para criar uma **definição de tarefa (task definition)**. Uma definição de tarefa organiza um conjunto de containers que serão provisionados de uma única vez, e quais as configurações que são necessárias para elas. Você vai usar o AWS CLI para criar a definição de tarefas.

Um arquivo JSON foi provisionado que vai servir para gerar as configurações necessárias, para tal execute os seguintes comandos:

Abra o arquivo  `~/environment/aws-modern-application-workshop/module-2/aws-cli/task-definition.json`.

Substitua os valores indicados pelos os verdadeiros que correspondem a seu cluster.

Os valores serão tirados da resposta do CloudFormation que você usou para provisionar a estrutura, por exemplo: `REPLACE_ME_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/mythicalmysfits/service:latest`

Uma vez que você substituiu os valores no `task-definition.json`, salve o arquivo. Execute o seguinte comando e registre uma nova definição de tarefa no ECS: 

```
aws ecs register-task-definition --cli-input-json file://~/environment/aws-modern-application-workshop/module-2/aws-cli/task-definition.json
```

### Ative o Load Balancver Fargate Service

#### Crie um network load balancer 

Com a nova definição de tarefa registrada, nós estamos prontos para provisionar a infraestrutura necessária para o sustentar o nosso serviço. Ao invés de expor o serviço diretamente a internet, nós vamos provisionar um [**Network Load Balancer (NLB)**](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html) (balanceador de carga por rede) para ficar na frente do nosso serviço. Isso vai possibilitar que os usuários possam se comunicar com o nosso serviço usando um único DNS, tornando possível que o nosso serviço possa escalar para mais ou menos de acordo com a demanda.

Para provisionar um novo NLB (network load balancer), execute os seguintes comandos no CLI do seu Cloud9 (substitua os IDs das subnets pelos que estão na saída do seu CloudFormation):

```
aws elbv2 create-load-balancer --name mysfits-nlb --scheme internet-facing --type network --subnets REPLACE_ME_PUBLIC_SUBNET_ONE REPLACE_ME_PUBLIC_SUBNET_TWO > ~/environment/nlb-output.json
```

Depois que esse Código for executado com sucesso, um novo arquivo vai ser criado na sua IDE chamado `nlb-output.json`. Você vai usar o `DNSName`, `VPCId` e `LoadBalancerArn` em passos no futuro

#### Crie um Load Balancer Target Group

O próximo passo use o CLI para cria um target group (grupo de alvos) no NLB (network load balancer). Um target group permite que os recursos da AWS se auto registrem como alvos, para que possam receber requestes do load balancer, O comando inclui um valor que vai precisar ser substituido, seu `VPC-Id`que pode ser encontrado no output do CloudFormation que foi executado anteriormente.

```
aws elbv2 create-target-group --name MythicalMysfits-TargetGroup --port 8080 --protocol TCP --target-type ip --vpc-id REPLACE_ME_VPC_ID --health-check-interval-seconds 10 --health-check-path / --health-check-protocol HTTP --healthy-threshold-count 3 --unhealthy-threshold-count 3 > ~/environment/target-group-output.json
```
Este comando estiver completo, o seu output vai ser salvo em `target-group-output.json`. Você vai referenciar o `TargetGroupArn`como passo subsequente.

#### Crie um Load Balancer Listener

Próximo passo, use o CLI para criar um **Load Balancer Listener** (ouvinte do balanceador de carga). Isso faz com que o load balancer saiba quais são as portas especificas das requisições. Lembre-se de substituir os valores indicados pelos valores que você salvou anteriormente:

```
aws elbv2 create-listener --default-actions TargetGroupArn=REPLACE_ME_NLB_TARGET_GROUP_ARN,Type=forward --load-balancer-arn REPLACE_ME_NLB_ARN --port 80 --protocol TCP
```

### Criando um serviço com o Fargate

#### Criando um serviço com as regras certas para o ECS

Se você já usou o ECS no passado você pode pular esse passo e ir para o próximo. Se você nunca usou o ECS antes, nós precisaremos criar um service linked role (conjunto de regras para um certo serviço da AWS, para que este possa se comunicar com outros serviços).

Sem a criação destas regras, o ECS não poderá se comunicar com outros serviços e realizar sua funções. Execute o seguinte comando no terminal:

```
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com
```

Se o comando a cima retornar um erro sobre a regra já existir, você pode ignorar, significa que a regra já foi criada automaticamente em algum momento no passado.


#### Crie o serviço

Com o Network load balancer criado e configurado, e o ECS com a permissões necessárias já configuradas, nós já configuramos ECS onde nossos containers vão rodar e receber trafego automaticamente. Nós também incluímos um arquivo JSON para o input do CLI que está localizado em: `~/environment/aws-modern-application-workshop/module-2/aws-cli/service-definition.json`.  O arquivo inclui toda a configuração e detalhes dos serviços que serão criados, incluindo notas de que esse serviço deve ser lançado via **AWS Fargate** - que significa que você não precisará se preocupar em provisionar servidores para o cluster que você vai criar. Tudo estará em sincronia e será criado e gerenciado pela AWS

Abra ```~/environment/aws-modern-application-workshop/module-2/aws-cli/service-definition.json``` Na IDE e substitua os valores indicados por `REPLACE-ME`. Salve o arquivo, e execute o seguinte comando no terminal:

```
aws ecs create-service --cli-input-json file://~/environment/aws-modern-application-workshop/module-2/aws-cli/service-definition.json
```

Após o serviço ser criado, ECS vai provisionar uma nova tarefa que vai rodar o container que você enviou e que está registrado no NLB (network load balancer).

#### Teste o serviço

Copie o DNS que você salvou quando criou o NLB e mande uma requisição para ele através do browser do Cloud9 (ou simplesmente cole no navegador de sua preferência). Tente enviar a requisição para o recurso Mysfits:

```
http://mysfits-nlb-123456789-abc123456.elb.us-east-1.amazonaws.com/mysfits
```

A resposta a requisição deve ser o mesmo JSON de quando você testou localmente, caso isso aconteceu significa que o Flask API está rodando via AWS Fargate


### Atualize Mytrical Mysfits para chamar o NLB

#### Substitua o endpoint da API
Próximo passo, nós precisamos integrar nosso website com a API do backend ao invés de utilizarmos o que está escrito diretamente no código que nós armazenamos no S3. Você precisa atualizar o seguinte arquivo para que este use a mesma URL NLB para as chamadas APIs (não inclua o /mysfits no caminho):  `/module-2/web/index.html`

Abra o arquivo no Cloud9 e substitua o que está em destaque na imagem abaixo para a URL do NLB:

![before replace](/images/module-2/before-replace.png)

Após a substituição, deve ficar parecido com isso:

![after replace](/images/module-2/after-replace.png)

#### Atualize o S3
Para atualizar o arquivo no S3, utilize o nome do bucket que utilizamos no modulo 1, e execute o seguinte comando:

```
aws s3 cp ~/environment/aws-modern-application-workshop/module-2/web/index.html s3://INSERT-YOUR-BUCKET-NAME/index.html
```

Abra o seu website usando a mesma URL que foi utilizada no final do modulo 1, você deve ser capaz de ver o novo site Mythical Mysfits, onde esse está retornando as informações JSON que estão retornando os dados da sua API flask, que está rodando em um container lançado pelo AWS Fargate


## Módulo 2b: Automatizando os deploys usando os serviços de código da AWS

![Architecture](/images/module-2/architecture-module-2b.png)


### Criando o pipeline de CI/CD

#### Criando um bucker S3 para armazenar o pipeline de programação

Agora que você possui o serviço criado e rodando na nuvem, você possivelmente pode pensar em alterações no código que gostaria de fazer no serviço Flask. Seria um engasgo de produtividade muito grande, toda vez que você quisesse fazer uma alteração no código, ser obrigado a realizar todos os passos que já fizemos previamente. Aqui entra o nosso Continuous Integration e Continuous Deployment ou CI/CD!

Nesse modulo, nós vamos criar um CI/CD completamente auto-gerenciavel, que levará automaticamente toda a alteração de código a sua arquitetura.

O primeiro passo que iremos fazer é criar um novo bucket no S3 para armazenar temporariamente as alterações que vamos fazer no código. Escolha um novo nome para o seu novo Bucket S3, e execute o comando a seguir no terminal:

```
aws s3 mb s3://REPLACE_ME_CHOOSE_ARTIFACTS_BUCKET_NAME
```

O próximo passo, este bucket precisa de uma polituca propria, para que sejam definidas permissões de uso e acesso ao mesmo, somente o nosso pipeline de CI/CD deve ter acesso a esse bucket, para isso já temos uma politica em forma de arquivo JSON que está localizada em: `~/environment/aws-modern-application-workshop/module-2/aws-cli/artifacts-bucket-policy.json`.  Abra esse arquivo, nele você precisará alterar uma série de parâmetros, incluindo ARNs que nós criamos como parte do MythicalMysfitsCoreStack anteriormente, como o nome do bucket que estaremos usando para o CI/CD.

Uma vez que o arquivo JSON foi modificado, salve-o. Em seguida execute o comando abaixo no terminal:

```
aws s3api put-bucket-policy --bucket REPLACE_ME_ARTIFACTS_BUCKET_NAME --policy file://~/environment/aws-modern-application-workshop/module-2/aws-cli/artifacts-bucket-policy.json
```

#### Crie um repositório no CodeCommit

Nós precisaremos de um lugar para enviar e armazenar os códigos. Crie um repositório no [**AWS CodeCommit Repository**](https://aws.amazon.com/codecommit/) usado o CLI para isso:

```
aws codecommit create-repository --repository-name MythicalMysfitsService-Repository
```

#### Crie um projeto no Codebuild

Com o repositório criado para armazenar os nossos códigos. Precisamos de algo que pegue os novos códigos e leve aos serviços aos quais estes pertençam. Por isso vamos criar um [**AWS CodeBuild Project**](https://aws.amazon.com/codebuild/). A qualquer momento que o trigger for ativado, ele provisionará toda a infraestrutura necessária para que o serviço possa estar em funcionamento. Todos os passo para a criação e provisionamento do Docker está no documento: `~/environment/aws-modern-application-workshop/module-2/app/buildspec.yml`. O **buildspec.yml** é o arquivo que possui todo o passo a passo para que o CodeBuild saiba como provisionar a arquitetura e a nova imagem Docker com os códigos atualizados.

Para criar um projeto no CodeBuild, outro arquivo de input é necessário ser atualizado. Este está localizado em `~/environment/aws-modern-application-workshop/module-2/aws-cli/code-build-project.json`.  Parecido com o que já fizemos anteriormente na edição dos passos. Após as alterações, salve o arquivo, e execute o comando abaixo para criar o projeto:

```
aws codebuild create-project --cli-input-json file://~/environment/aws-modern-application-workshop/module-2/aws-cli/code-build-project.json
```

#### Cria um pipeline do CodePipeline

Finalmente, nós precisamos de um jeito de criar a integração continua (continuously integration) do nosso repositório do CodeCommit com o projeto no CodeBuild, este precisa construir a aplicação no ECS. O [**AWS CodePipeline**](https://aws.amazon.com/codepipeline/) é o serviço que faz a junção das coisas criando um **pipeline** que manterá sua aplicação sempre atualizada.

Seu pipeline vai fazer exatamente o que foi disse acima. A qualquer momento que seu código for alterado seu repositório será atualizado, o CodePipeline vai receber os códigos mais atualizados e enviará para o CodeBuild, para que a construção ocorrá. Quando o CodeBuild terminar, o Pipeline manda para o ECS para a construção e execução do cluster

Todos os passos estão definidos em um arquivo JSON que você verá o input do AWS CLI para criar o pipeline. Este arquivo está localizado em: `~/environment/aws-modern-application-workshop/module-2/aws-cli/code-pipeline.json`, abra o arquivo e faça as substituições necessária no arquivo, depois salve-o.

Uma vez salvo, crie o pipeline com o seguinte comando:

```
aws codepipeline create-pipeline --cli-input-json file://~/environment/aws-modern-application-workshop/module-2/aws-cli/code-pipeline.json
```

#### Ative o acesso automatizado ao repositório com imagem para o ECS

Nós temos uma ultima parada antes do nosso CI/CD pipeline possa executar automaticamente as atualizações nas nossas aplicações. Com o CI/CD pipeline no lugar, você não vai mais precisar fazer o envio manual das atualizações nos códigos. Nós precisamos colocar a politica n o serviço para que faça as funções desejadas por nós. Esta politica está localizada em: `~/environment/aws-modern-application-workshop/module-2/aws-cli/ecr-policy.json`. Atualize com as informações dos seus serviços, salve as alterações e execute o comando abaixo:

```
aws ecr set-repository-policy --repository-name mythicalmysfits/service --policy-text file://~/environment/aws-modern-application-workshop/module-2/aws-cli/ecr-policy.json
```

Quando você receber uma mensagem dizendo que os comandos foram executados com sucesso, significa que seu pipeline foi criado com sucesso.

### Teste o Pipeline CI/CD

#### Usando o Git com AWS CodeCommit
Para testar seu novo pipeline, nós precisamos configurar o git no seu Cloud9 e integrar com o repositório CodeCommit 

O CodeCommit gera credenciais de acesso para fazer com que a integração seja mais fácil. Execute os comandos a seguir para que se faça a integração git- CodeCommit

```
git config --global user.name "REPLACE_ME_WITH_YOUR_NAME"
```

```
git config --global user.email REPLACE_ME_WITH_YOUR_EMAIL@example.com
```

```
git config --global credential.helper '!aws codecommit credential-helper $@'
```

```
git config --global credential.UseHttpPath true
```

Agora altere o diretório no ambiente da sua IDE usando o terminal:

```
cd ~/environment/
```

Agora, clone o repositório usando o terminal:

```
git clone https://git-codecommit.REPLACE_REGION.amazonaws.com/v1/repos/MythicalMysfitsService-Repository
```

O comando irá dizer que o nosso repositório está vazio, vamos corrigir, mandando para ele os dados da aplicação para o repositório:

```
cp -r ~/environment/aws-modern-application-workshop/module-2/app/* ~/environment/MythicalMysfitsService-Repository/
```

#### Enviando uma alteração de código

Agora temos o nosso pipeline funcionando, junto com o nosso serviço que foi lançado via Fargate, os arquivos da aplicação estão armazenado no Cloud9. Para mostrarmos que o pipeline está funcionando, vamos abrir o arquivo: `~/environment/MythicalMysfitsService-Repository/service/mysfits-response.json` E altere a idade de um dos mysfits para outro valor, e salve o arquivo.

Depois de salvar, mande o arquivo para o novo repositório, com o comando abaixo:

```
cd ~/environment/MythicalMysfitsService-Repository/
```

Após isso, execute os seguintes comandos para publicar as alterações na sua aplicação:

```
git add .
git commit -m "I changed the age of one of the mysfits."
git push
```

Após a publicação, abra o codePipeline e veja o seu pipeline criado em funciando, e recriando a arquitetura com o novo código. Essa alteração pode demorar de 5 a 10 minutos, para que suas alterações estejam disponíveis no Fargate. Durante esse tempo o CodePipeline vai ativar o trigger e reconstruir a arquitetura da sua aplicação com os novos códigos. Após o tempo, abra o DNS do seu serviço no navegador e veja as alterações.

Você pode ver o progresso da ação no console do CodePipeline (Não precisa de nenhuma ação, só aprecie a automação!):
[AWS CodePipeline](https://console.aws.amazon.com/codepipeline/home)

Isso conclui o Módulo 2.

[Prossiga para o Módulo 3](/module-3)


## [Centro do desenvolvedor da AWS](https://developer.aws)
