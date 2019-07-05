# Módulo 5 - Capturando o Comportamento do Usuário

![Architecture](/images/module-5/architecture-module-5.png)

**Tempo esperado:** 30 minutos

**Serviços usados:**
* [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
* [AWS Kinesis Data Firehose](https://aws.amazon.com/kinesis/data-firehose/)
* [Amazon S3](https://aws.amazon.com/s3/)
* [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
* [AWS Lambda](https://aws.amazon.com/lambda/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS Serverless Appliation Model (AWS SAM)](https://github.com/awslabs/serverless-application-model)
* [AWS SAM Command Line Interface (SAM CLI)](https://github.com/awslabs/aws-sam-cli)

### Resumo

Agora que seu site Mythical Mysfits está rodando, vamos criar uma maneira de entender melhor como usuários estão interagindo com o site e seus Mysfits. Seria bem fácil para nós analizar ações que usuários tomaram no site que impactaram em mudanças de dados no backend - quando um mysfit é adotado ou recebe um like. Mas entender as ações que os usuários estão tomando no site *antes* que a decisões de dar um like ou adotar um mysfit pode ajudar a desenhar uma melhor experiência futura que pode impactar em mais mysfits sendo adotados. Para nos ajudar a reunir esses insights, nós implementaremos a habilididade para o frontend do site de submeter um tiny request, toda vez que um perfil de um mysfit é clicado por um usuário, para um novo API de microsserviço que criaremos. Esses registros serão processados em tempo real por uma função serverless, agregados e armazenados para qualquer análise futura que você queira fazer.

Padrões de Design de aplicações modernas preferem serviços focados, desacoplados e modulares. Então em vez de criar métodos adicionais no já existente serviço que temos funcionando, nós vamos criar um novo e desacoplado serviço com o propósito de receber eventos de cliques de usuários provenientes do site. A estrutura completa descrita está representada por um template do CloudFormation.

A estrutura serverless de processamento em tempo real que você irá criar incluí os seguintes serviços AWS:

* Um [**AWS Kinesis Data Firehose delivery stream**](https://aws.amazon.com/kinesis/data-firehose/): Kinesis Firehose é um serviço altamente disponível e gerenciado para streaming em tempo real que aceita registros de dados e os recebe automaticamente dentro de diversos possíveis destinos de armazenamento na AWS, como por exemplo Amazon S3, ou o Amazon Redshift. Kinesis Firehose também permite que todos os registros recebidos por uma stream sejam automaticamente entregues a funções criadas com **AWS Lambda**. Isso significa que códigos que você escreveu podem performar qualquer processamento adicional ou transformações dos registros antes deles serem agregados e armazenados no destino configurado.

* Um [**Amazon buicket S3**](https://aws.amazon.com/s3/): Um novo bucket será criado no S3 onde todos os registros dos eventos de cliques serão agregados em arquivos e armazenados como objetos.

* Uma [**AWS Lambda function**](https://aws.amazon.com/lambda/): AWS Lambda permite desenvolvedores a escrever códigos que apenas contêm o que sua lógica requer. Aqui, uma função Serverless é definida usando AWS SAM. Esse código será colocado no AWS Lambda, escrita em Python, e processará os registros de cliques que são recebidos pela delivery stream. A função recupera atributos adicionais sobre os cliques nos Mysfits para tornar os registros com mais informações relevantes. Mas, para o propósito desse workshop, o código tem o intúito de demonstrar as possibilidades arquiteturais de incluir uma função serverless para performar qualquer processamento adicional nos dados. Uma vez que a função Lambda é criada e o Kinesis Firehose delivery stream é configurada como uma fonte de eventos para a função, o delivery stream vai automaticamente entregar os registros dos cliques como eventos para a função que nós criamos. Essa função vai entregar os registros atualizados para o Bucket S3 configurado.

* Um [**Amazon API Gateway REST API**](https://aws.amazon.com/api-gateway/): AWS Kinesis Firehose provê um serviço de API assim como outros serviços AWS, e nesse caso nós estamos usando operações PutRecord para colocar os registros de eventos dos usuários no delivery stream.

* [**IAM Roles**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html): Kinesis Firehose requer uma role que o permita entregar os registros recebidos como eventos para a função Lambda criada assim como os registros processados para o S3 bucket de destino. A API do Amazon API Gateway também requer uma nova role que permite a API a invocar a API PutRecord do Kinesis Firehose para cada requisição recebida.

Antes de lançarmos o template do CloudFormation descrito anteriormente, nós precisamos atualizar e modificar o código da função Lambda.

### Copiar o Código do Serviço de Streaming

#### Criar um novo Repositório no CodeCommit

Esse novo stack que você lançará usando o CloudFormation irá não só conter os recursos do ambiente da infraestrutura, mas também o código da aplicação em si. O código da função do AWS Lambda é entregue para o serviço ao fazer o upload do pacote .zip para o Amazon S3 bucket. O SAM CLI automatiza esse processo para você. Usando isso, nós podemos criar um template do CloudFormation que referencia localmente no sistema de arquivos em que todos os códigos para nosso Lambda estão armazenados. Então, o SAM CLI vai empacotar isso dentro de um arquivo .zip, fazer o upload disso para um Amazon S3 bucket configurado. Nós podemos então fazer o deploy desse template CloudFormation gerado pelo SAM CLI para a AWS e ver o ambiente ser criado. 

Primeiro, vamos criar um novo repositório CodeCommit onde o código do serviço de streaming irá permanecer:
```
aws codecommit create-repository --repository-name MythicalMysfitsStreamingService-Repository
```

Em resposta a esse comando, copie o valor de `"cloneUrlHttp"`. Isso deve ter a seguinte forma:
`https://git-codecommit.REPLACE_ME_REGION.amazonaws.com/v1/repos/MythicalMysfitsStreamingService-Repository`

Depois, vamos clonar esse novo e vazio repositório em nossa IDE:
```
cd ~/environment/
```

```
git clone REPLACE_ME_WITH_ABOVE_CLONE_URL
```

#### Copiar a Base de Códigos do Serviço de Streaming

Agora, vamos mover o nosso diretório de trabalho para esse novo repositório:
```
cd ~/environment/MythicalMysfitsStreamingService-Repository/
```

Então, copie a aplicação do módulo 5 para esse novo repositório:
```
cp -r ~/environment/aws-modern-application-workshop/module-5/app/streaming/* .
```

E vamos copiar o template do CloudFormation para esse módulo também:
```
cp ~/environment/aws-modern-application-workshop/module-5/cfn/* .
```

### Atualizar o Código e o Pacote da Função Lambda

#### Use o pip para Instalar as Dependencias da Função Lambda

Agora, nós temos o repositório com todos os artefatos providos:
* Um template CFN para a criação do stack todo.
* Um arquivo Python que contêm o código para nossa função Lambda: `streamProcessor.py`

Essa é uma abordagem comum que clientes da AWS tomam - armazenar os templates do CloudFormation junto com os códigos da aplicação em um único repositório. Desse jeito, você tem apenas um lugar onde todas as mudanças na aplicação e no ambiente podem ser mapeadas juntas.

Mas, se você olhar o código dentro do arquivo `streamProcessor.py`, você perceberá que ele está usando o pacote `requests` do Python para fazer uma API para o serviço Mythical Mysfits criado previamente. Bibliotecas externas não são automaticamente incluidas no ambiente da AWS Lambda. Você precisará empacotar todas as dependências juntas com o código da função Lambda. Nós usaremos o gerenciador de pacotes do Python `pip`. No terminal do Cloud9, rode os seguintes comandos para instalar o pacote `requests`:

```
pip install requests -t .
```
Uma vez que o comando completa, você vai ver vários pacotes adicionais armazenado no seu repositório.

#### Atualize o Código da Função Lambda

Agora, nós temos uma mudança no código para tornar a função Lambda pronta para o deploy. Existe uma linha no arquivo `streamProcessor.py` que precisa ser substituída com o ApiEndpoint do seu serviço - o mesmo ApiEndpoint que você criou no módulo 4 e foi usado no frontend do site. Assegure-se que você salvou o arquivo que você copiou no seu repositório. 

![replace me](/images/module-5/replace-api-endpoint.png)

Esse serviço é responsável por integrar com o MysfitsTable no DynamoDB, assim mesmo que nós escrevêssemos uma função Lambda que integra diretamente com a tabela do DynamoDB. Nós vamos integrar a tabela com o serviço existente e ter uma arquitetura de aplicação muito mais modular e desacoplada.

#### Coloque seu Código no CodeCommit

Vamos colocar nossas mudanças no código no novo repositório e então salva-las no CodeCommit:

```
git add .
```

```
git commit -m "New stream processing service."
```

```
git push
```

### Criando um Stack de Serviço de Streaming

#### Crie um Bucket S3 para a função Lambda e do Pacotes

Com o arquivo Python com a linha mudada, nós estamos prontos para usar o AWS SAM CLI para empacotar todos os códigos de suas funções, fazer o upload para o S3, e criar o template do CloudFormation para criar nosso streaming stack.

Primeiro, use o AWS CLI para criar um novo bucket S3 onde sua função Lambda vai ser uploaded. Nomes de buckets S3 precisam ser globalmente únicos ao longo de todos os clientes da AWS, então substitua o fim desse nome de bucket com uma string que é única para você:

```
aws s3 mb s3://REPLACE_ME_YOUR_BUCKET_NAME/
```

#### Use the SAM CLI to Package your Code for Lambda

Com nosso bucket criado, nós estamos prontos para usar o SAM CLI para empacotar e fazer o upload do nosso código e transformar o template do CloudFormation, lembre-se de substituir o último parametro de comando com o nome do bucket que você criou anteriormente:

```
sam package --template-file ./real-time-streaming.yml --output-template-file ./transformed-streaming.yml --s3-bucket REPLACE_ME_YOUR_BUCKET_NAME
```

Se com sucesso, você verá um novo arquivo `transformed-streaming.yml` no diretório `./MythicalMysfitsStreamingService-Repository/`, se você checar os conteúdos, verá que o paramentro CodeUri da função Lambda foi atualizada com a locação do objeto em que o SAM CLI foi uploaded.

#### Faça o Deploy do Stack usando o AWS CloudFormation

Execute o seguinte comando para fazer o deploy do stack de streaming:

```
aws cloudformation deploy --template-file /home/ec2-user/environment/MythicalMysfitsStreamingService-Repository/transformed-streaming.yml --stack-name MythicalMysfitsStreamingStack --capabilities CAPABILITY_IAM
```

Uma vez que a criação do stack estiver completa, o microsserviço de processamento em tempo real será criado.


### Enviando Cliques de Perfis de Mysfit para o Serviço

#### Atualizando o Conteúdo do Site

Com o stack de stremaing rodando, nós agora precisamos publicar uma nova versão do frontend do Mythical Mysfits que inclui o JavaScript que envia eventos para nosso serviço sempre que um perfil de mysfit é clicado por um usuário.

O novo arquivo index.html está incluso em: `~/environment/aws-modern-application-workshop/module-5/web/index.html`

Para os valores anteriores de variáveis, você pode referir ao arquivo `index.html` anterior que você atualizou no módulo 4.

Performe o seguinte comando para o novo stack de streaming para recuperar o novo endpoint do API Gateway:

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsStreamingStack
```

#### Coloque a nova versão do Site no S3

Substitua o valor final dentro do arquivo `index.html` com o streamingApiEndpoint. Agora você está pronto para publicar a atualização final da home page do Mythical Mysfits:

```
aws s3 cp ~/environment/aws-modern-application-workshop/module-5/web/index.html s3://YOUR-S3-BUCKET/
```

Recarregue o site Mythical Mysfits em seu browser uma outra vez e você agora terá um site que grava e publica cada vez que um usuário clica em um perfil de mysfit! 

Para ver as gravações que foram processados, eles chegarão ao bucket S3 de destino como parte do seu MythicalMysfitsStreamingStack. Visite o console do S3 e explore o bucket que você criou para os registros de streaming (ele terá o prefixo `mythicalmysfitsstreamings-clicksdestinationbucket`):
[Amazon S3 Console](https://s3.console.aws.amazon.com/s3/home)

Isso concluí o Módulo 5.

### [Prossiga para o Módulo 6](/module-6)


#### [Centro do desenvolvedor da AWS](https://developer.aws)
