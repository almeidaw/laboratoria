# Module 7: Usando Machine Learning para Recomendar um Mysfit

![Architecture](/images/module-7/sagemaker-architecture.png)

**Tempo esperado:** 45 minutes

**Serviços usados:**
* [Amazon SageMaker](https://aws.amazon.com/sagemaker/)
* [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
* [Amazon S3](https://aws.amazon.com/s3/)
* [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
* [AWS Lambda](https://aws.amazon.com/lambda/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS Serverless Appliation Model (AWS SAM)](https://github.com/awslabs/serverless-application-model)
* [AWS SAM Command Line Interface (SAM CLI)](https://github.com/awslabs/aws-sam-cli)

## Visão Geral
Uma das áreas de tecnologia de crescimento mais rápido é machine learning. Como o custo para aproveitar os ambientes de computação de alto desempenho continuou a diminuir, o número de casos de uso nos quais os algoritmos de Machine Learning podem ser aplicados economicamente cresceu de forma astronômica. Podemos até usar machine learning para ajudar os visitantes do MythicalMysfits.com a descobrir qual Mysfit é perfeito para eles. E isso é exatamente o que você fará neste módulo.

Com o Amazon SageMaker, um serviço de machine learning totalmente gerenciado, você introduzirá um novo mecanismo de recomendação baseado em machine learning no site Mythical Mysfits. Isso permitirá que os visitantes do site nos forneçam detalhes sobre eles mesmos, e usaremos essas informações para invocar um modelo de machine learning e prever qual Mysfit seria mais adequado para eles. O Amazon SageMaker fornecerá as ferramentas necessárias para:
* Preparar dados para o treinamento do modelo (usando dados de amostra que criamos e disponibilizamos no S3)
* Treinar um modelo de aprendizado de máquina usando um dos muitos algoritmos de machine learning já implementados que o SageMaker fornece e avaliar a precisão do modelo.
* Armazene e implante o modelo criado para que ele possa ser invocado em escala para fornecer inferências à sua aplicação (o website Mythical Mysfits).

## Construindo um Modelo de Machine Learning

#### A Importância dos Dados
Um pré-requisito para iniciar qualquer jornada de machine learning é coletar dados. Os dados usados ​​definirão o entendimento do algoritmo sobre o caso de uso no qual está sendo chamado e sua capacidade de fazer previsões / inferências precisas. Se você introduzir o machine learning na sua aplicação usando dados insuficientes, irrelevantes ou imprecisos, você corre o risco de trazer mais danos do que benefícios para a sua aplicação.

No entanto, para o nosso website Mysfits, obviamente não temos grandes quantidades de dados de adoção precisos e históricos para fazer recomendações mysfit. Então, em vez disso, geramos muitos dados aleatoriamente. Isso significa que o modelo que você construirá fará previsões baseadas em dados aleatórios, e sua "precisão" será prejudicada porque os dados foram gerados aleatoriamente. Esse conjunto de dados ainda permitirá que você se familiarize com a forma de usar o SageMaker em uma aplicação real ... mas estaremos encobrindo todos os passos críticos exigidos no mundo real para identificar, reunir e organizar um conjunto de dados apropriado para ser usado em machine learning com sucesso.

### Criando um Hosted Notebook com o SageMaker
Cientistas de dados e desenvolvedores que desejam curar dados, definir e executar algoritmos, construir modelos e muito mais, tudo isso enquanto documentam completamente seu trabalho podem fazê-lo em um único lugar chamado **notebook**. Por meio do AWS SageMaker, você pode criar uma instância EC2 pré-configurada e otimizada para Machine Learning, e que já possui a aplicação [Jupyter Notebooks] (http://jupyter.org/) em execução, ela é chamada de **instância de notebook**. Para criar uma instância de notebook, primeiro precisamos criar alguns pré-requisitos que o notebook requer, ou seja, uma função do IAM que fornecerá à instância de notebook as permissões necessárias para executar tudo o que é necessário. Fornecemos outro template do CloudFormation para que você possa criar essa nova função do IAM. Execute o seguinte comando para criar a pilha:

```
aws cloudformation deploy --stack-name MythicalMysfits-SageMaker-NotebookRole --template ~/environment/aws-modern-application-workshop/module-7/cfn/notebook-role.yml --capabilities CAPABILITY_NAMED_IAM
```

Em seguida, execute o seguinte comando para criar uma instância notebook com o SageMaker (substituindo seu Account_Id no Arn da função):
```
aws sagemaker create-notebook-instance --notebook-instance-name MythicalMysfits-SageMaker-Notebook --instance-type ml.t2.medium --role arn:aws:iam::REPLACE_ME_ACCOUNT_ID:role/MysfitsNotbookRole
```

**Nota:** Demorará cerca de 10 minutos para a sua instância notebook passar do estado `Pending` para `InService`. Você pode prosseguir para as próximas etapas enquanto o notebook está sendo provisionado.

Finalmente, há um arquivo no seu repositório clonado que será usado nas próximas etapas e que você deve fazer o download. No File Explorer no Cloud9, localize `MythicalMysfitsIDE/aws-modern-application-workshop/module-7/sagemaker/mysfit_recommendations_knn.ipynb`. Clique com o botão direito do mouse e selecione Download. Salve este arquivo em sua estação de trabalho local e lembre-se de onde ele foi salvo.

### Usando o Amazon SageMaker

Em seguida, abra uma nova guia do navegador e visite o console do SageMaker (verifique se a região selecionada no canto superior direito do seu Console de Gerenciamento da AWS corresponde à região em que você está criando):
[Console do Amazon SageMaker](https://console.aws.amazon.com/sagemaker/home)

Então, clique em **Notebook Instances.**

Clique no radio button próximo à instância **MythicalMysfits-SageMaker-Notebook** que você acabou de criar por meio do CLI e clique em **Open Jupyter.**  Isso redirecionará você para a aplicação Jupyter Notebook em execução na sua instância de notebook.

![SageMaker Notebook Instances](/images/module-7/sagemaker-notebook-instances.png)

**NOTE**: que para este workshop, criamos a instância de notebook para permitir o acesso direto via Internet, executando em uma VPC gerenciada pelo serviço. Para obter mais detalhes sobre como acessar uma instância do notebook por meio de um VPC Interface Endpoint, caso deseje para caso de uso futuro, visite [esta página da documentação.](https://docs.aws.amazon.com/sagemaker/latest/dg/notebook-interface-endpoint.html).

Com o Jupyter aberto, você será apresentado à seguinte página inicial para sua instância de notebook:

![Jupyter Home](/images/module-7/jupyter-home.png)

Clique no botão **Upload**, e então encontre o arquivo que você baixou na seção anterior `mysfit_recommendations_knn.ipynb` e clique em **Upload** na linha do arquivo. Isso criará um novo documento do Notebook na instância de notebook dentro do Jupyter que usa o arquivo do notebook que você acabou de fazer upload.

Nós pré-escrevemos o notebook necessário que irá guiá-lo por meio do código necessário para construir um modelo.

#### Usando o Hosted Notebook para Construir, Treinar e Implantar um Modelo
Clique no nome do arquivo dentro da aplicação Jupyter, para o arquivo que você acabou de fazer upload, e uma nova guia do navegador será aberta para você trabalhar dentro do documento do notebook.

# PARE!
Siga as instruções no documento do notebook para implantar um endpoint do SageMaker para prever o melhor Mythical Mysfit para um usuário com base nas suas respostas do questionário.

Depois de concluir as etapas no notebook, retorne aqui para continuar com o workshop.

## Criando uma API REST serverless para Predições do Modelo

Agora que você tem um ponto de extremidade SageMaker implantado, vamos integrar a nossa própria API REST serverless do Mythical Mysfits ao endpoint. Isso nos permite definir a API exatamente de acordo com nossas especificações e para que o código frontend de nossa aplicação continue integrando-se às APIs que nós mesmos definimos, ao invés de APIs nativas de serviços da AWS. Construiremos o microsserviço para ser serverless usando o API Gateway e o AWS Lambda.

Vamos criar outro novo repositório do CodeCommit para onde podemos fazer o commit do código de serviço de recomendações:

```
aws codecommit create-repository --repository-name MythicalMysfitsRecommendationService
```

Em seguida, clone o novo repositório no seu ambiente Cloud9 usando o atributo `cloneUrlHttp` na resposta acima:

Primeiro, defina o diretório atual para o seu diretório inicial.
```
cd ~/environment
```

Em seguida, clone o repositório:
```
git clone REPLACE_ME_CLONE_URL
```

Agora copie o código do serviço e o template SAM para o novo repositório:
```
cp -r ~/environment/aws-modern-application-workshop/module-7/app/service/ ~/environment/MythicalMysfitsRecommendationService/
```

Há uma alteração de código que você deve fazer no código Python do serviço antes que possamos implantar a API. Abra `~/environment/MythicalMysfitsRecommendationService/service/recommendation.py` no Cloud9. Você verá uma única entrada que precisa ser substituída: `REPLACE_ME_SAGEMAKER_ENDPOINT`

Para recuperar o valor necessário, execute o seguinte comando do CLI para descrever seus endpoints do SageMaker:
```
aws sagemaker list-endpoints > ~/environment/sagemaker-endpoints.json
```

Abra `sagemaker-endpoints.json` e copie o valor de EndpointName que é prefixado com:
`knn-ml-m4-xlarge-` (é com isso que prefixamos o nome do nosso endpoint dentro do Jupyter notebook).

Cole o nome do EndpointValue no arquivo `recommendation.py` e salve o arquivo.

A última etapa de que precisamos antes de usar o SAM para implantar nosso microsserviço é um novo bucket do S3 que o SAM pode usar para empacotar nosso código-fonte como parte do pacote e da implantação. Escolha um novo nome para o bucket, como anteriormente, e execute o seguinte comando:

```
aws s3 mb s3://REPLACE_ME_RECOMMENDATION_NEW_BUCKET_NAME
```

Agora use o SAM para implantar seu microsserviço de recomendações, primeiro empacotando seu código Python para ser usado na função Lambda:
```
sam package --template-file ~/environment/MythicalMysfitsRecommendationService/service/cfn/recommendations-service.yml --output-template-file ~/environment/MythicalMysfitsRecommendationService/transformed-recommendations.yml --s3-bucket REPLACE_ME_RECOMMENDATION_BUCKET_NAME
```

Em seguida, implante a stack usando o CloudFormation:
```
aws cloudformation deploy --template-file ~/environment/MythicalMysfitsRecommendationService/transformed-recommendations.yml --stack-name MythicalMysfitsRecommendationsStack --capabilities CAPABILITY_NAMED_IAM
```

Quando esse comando for concluído, você terá implantado o wrapper de microsserviço da API REST para o endpoint do SageMaker que você criou por meio do Jupyter notebook.

Execute o seguinte comando para recuperar qual é o endpoint da API para o seu novo serviço e salve-o em um novo arquivo JSON `recommendation-endpoint.json`:
```
aws cloudformation describe-stacks --stack-name MythicalMysfitsRecommendationsStack > ~/environment/recommendation-endpoint.json
```

Vamos testar o novo serviço com o seguinte comando do CLI que usa o curl (uma ferramenta do Linux para fazer solicitações da web). Isso mostrará as recomendações em ação para um novo data point que corresponda às linhas do CSV que usamos para o treinamento de dados. Você usará o OutputValue do valor `recomendação-endpoint.json` acima para invocar sua própria API REST, certifique-se de anexar /recommendations após o endpoint, conforme mostrado abaixo:

```
curl -d '{"entry": [1,2,3,4,5]}' REPLACE_ME_RECOMMENDATION_API_ENDPOINT/recommendations -X POST
```

Você deve receber uma resposta como a seguinte:
```
{"recommendedMysfit": "c0684344-1eb7-40e7-b334-06d25ac9268c"}
```

Agora você está pronto para integrar essa nova funcionalidade de backend no website da Mythical Mysfits.

### Atualizando o website Mythical Mysfits

Um novo arquivo `index.html` foi incluído no Módulo 7, que contém o código necessário para apresentar aos usuários o questionário de Recomendação de Mysfit e apresentá-los ao Mysfit recomendado.

![Recommendation Button SS](/images/module-7/recommendation-button-ss.png)

Lembre-se de que você precisará copiar os valores existentes dos módulos anteriores e inseri-los conforme necessário para a funcionalidade dos websites existentes.

Abra `~/environment/aws-modern-application-workshop/module-7/web/index.html` para editar os valores `REPLACE_ME_` necessários, sendo o mais novo o `REPLACE_ME_RECOMMENDATION_API_ENDPOINT`, que pode ser recuperado como o valor de saída contido em `~/environment/recommendation-endpoint.json`

Depois de completar as edições necessárias em `index.html`, execute o seguinte comando novamente para atualizar o bucket do seu website Mythical Mysfits:

```
aws s3 cp ~/environment/aws-modern-application-workshop/module-7/web/index.html s3://YOUR-S3-BUCKET/
```

Agora você poderá ver um novo botão **Recomende um Mysfit** no website, que apresentará o questionário, capturará suas seleções e enviará essas seleções para nosso microsserviço de recomendações.

Parabéns, você completou o módulo 7!


### Limpeza do Workshop

#### Limpeza do Módulo 7:
Adicionamos código ao SageMaker notebook para excluir o endpoint implantado. Se você retornar ao notebook e prosseguir para as próximas células de código, poderá executá-las para desativar o endpoint a fim de parar de pagar por ele.

#### Limpeza Geral do Workshop:
Certifique-se de excluir todos os recursos criados durante o workshop para garantir que o faturamento dos recursos não continue por mais tempo do que você pretende. Recomendamos que você utilize o Console da AWS para explorar os recursos criados e excluí-los quando você estiver pronto.

Nos dois casos em que você provisionou recursos usando o AWS CloudFormation, é possível remover esses recursos simplesmente executando o seguinte comando do CLI para cada stack:

```
aws cloudformation delete-stack --stack-name STACK-NAME-HERE
```

Para remover todos os recursos criados, você pode visitar os seguintes Consoles da AWS, que contêm recursos que você criou durante o workshop Mythical Mysfits:
* [Amazon SageMaker](https://console.aws.amazon.com/sagemaker/home)
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

# Conclusão

Essa experiência foi criada para lhe dar uma ideia de como é ser um desenvolvedor projetando e construindo arquiteturas de aplicações modernas em cima da AWS. Os desenvolvedores na AWS podem provisionar recursos programaticamente usando o AWS CLI, reutilizar as definições de infraestrutura por meio do AWS CloudFormation, criar e implantar automaticamente alterações de código usando o conjunto de ferramentas de desenvolvedor de serviços de código da AWS e aproveitar vários recursos de serviços de computação e aplicativos que não exigem que você provisione ou gerencie quaisquer servidores!

Como próximo passo, para saber mais sobre o funcionamento interno do site Mythical Mysfits que você criou, mergulhe nos modelos CloudFormation fornecidos e nos recursos declarados neles.

Esperamos que você tenha aproveitado o Workshop de Aplicações Modernas da AWS! Se você encontrar algum problema ou tiver comentários/perguntas, não hesite em abrir um issue ou enviar um e-mail para andbaird@amazon.com.

Obrigado!


## [Centro do desenvolvedor da AWS](https://developer.aws)
