# Módulo 6: Rastreando Requisições da Aplicação

![Architecture](/images/module-6/x-ray-arch-diagram.png)

**Tempo esperado:** 45 minutos

**Serviços usados:**
* [AWS CloudFormation](https://aws.amazon.com/cloudformation/)
* [AWS X-Ray](https://aws.amazon.com/x-ray/)
* [Amazon DynamoDB](https://aws.amazon.com/dynamodb/)
* [Amazon Simple Notification Service (AWS SNS)](https://aws.amazon.com/sns/)
* [Amazon S3](https://aws.amazon.com/s3/)
* [Amazon API Gateway](https://aws.amazon.com/api-gateway/)
* [AWS Lambda](https://aws.amazon.com/lambda/)
* [AWS CodeCommit](https://aws.amazon.com/codecommit/)
* [AWS Serverless Appliation Model (AWS SAM)](https://github.com/awslabs/serverless-application-model)
* [AWS SAM Command Line Interface (SAM CLI)](https://github.com/awslabs/aws-sam-cli)

### Visão geral
Agora, nós vamos mostrar para vocês como inspecionar e analisar de forma profunda o comportamento de requisições em novas funcionalidades para o website dos Mythical Mysfits, usando o [**AWS X-Ray**](https://aws.amazon.com/kinesis/data-firehose/).  A nova funcionalidade irá habilitar usuários a contatarem a equipe dos Mythical Mysfits, por meio de um botão **Fale Conosco** que colocaremos no website.  Como muitos dos passos necessários para criar um novo microsserviço para lidar com recebimento de questões de usuários refletem atividades que vocês realizaram anteriormente neste workshop, nós preparamos um template do CloudFormation que irá criar o novo serviço programaticamente usando o AWS SAM.

O template do CloudFormation inclui:
* Uma **API do API Gateway**: Um novo microsserviço será criado e terá um único recurso REST, `/questions`.
Essa API receberá o texto de uma pergunta de um usuário e o endereço de e-mail do usuário que a enviou.
* Uma **Tabela do DynamoDB **: Uma nova tabela do DynamoDB onde as perguntas do usuário serão armazenadas e persistidas.  Esta tabela do DynamoDB será criada com um [**DynamoDB Stream**](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html) habilitado.  O stream fornecerá um stream de eventos em tempo real para todas as novas perguntas armazenadas no banco de dados, para que possam ser processadas imediatamente.
* Um **Tópico do AWS SNS**: O AWS SNS permite que as aplicações publiquem mensagens e assinem tópicos de mensagens.  Usaremos um novo tópico como forma de enviar notificações a um endereço de e-mail inscrito para um endereço de e-mail.
* Duas **Funções AWS Lambda**: Uma função do AWS Lambda será usada como o back-end do serviço para as requisições de API de perguntas. A outra função do AWS Lambda receberá eventos da tabela do DynamoDB de perguntas e publicará uma mensagem para cada um deles no tópico do SNS acima.  Se você visualizar as definições de recurso do CloudFormation para essas funções no arquivo `~/environment/aws-modern-application-workshop/module-6/app/cfn/customer-question.yml`, você verá uma Propriedade listada que indica `Tracing: Active`.  Isso significa que todas as invocações da função Lambda serão automaticamente rastreadas pelo **AWS X-Ray**.
* **Funções do IAM** necessárias para cada um dos recursos e ações acima.

Volte para o seu diretório raiz do ambiente Cloud9 no seu terminal de linha de comando para que nossos comandos subsequentes sejam executados do mesmo local:

```
cd ~/environment/
```

Primeiro, vamos criar outro novo repositório no **AWS CodeCommit** para o novo microsserviço:
```
aws codecommit create-repository --repository-name MythicalMysfitsQuestionsService-Repository
```

Em seguida, clone o novo repositório em seu IDE usando o valor `cloneUrlHttp` obtido da resposta no comando `create-repository` acima que você acabou de executar:
```
git clone REPLACE_ME_WITH_ABOVE_CLONE_URL
```

Depois, mova seu terminal para o novo diretório do repositório:
```
cd ~/environment/MythicalMysfitsQuestionsService-Repository
```

Agora, copie o novo código da aplicação QuestionsService no diretório do repositório, seguido pelo template do CloudFormation necessário para implantar a infraestrutura requerida pelo QuestionsService:
```
cp -r ~/environment/aws-modern-application-workshop/module-6/app/* .
```

```
 cp -r ~/environment/aws-modern-application-workshop/module-6/cfn/* .
```

Para este novo microsserviço, incluímos todos os pacotes necessários para que as funções do AWS Lambda sejam implantadas e chamadas. Antes de implantá-lo, é necessário criar outro bucket S3 a ser usado pelo AWS SAM como um destino para o código empacotado do QuestionService (lembre-se de que todos os nomes de buckets do S3 precisam ser únicos e têm restrições de nomenclatura):
```
aws s3 mb s3://REPLACE_ME_NEW_QUESTIONS_SERVICE_CODE_BUCKET_NAME
```
e execute o seguinte comando para transformar o template do SAM no CloudFormation ...

```
sam package --template-file ~/environment/MythicalMysfitsQuestionsService-Repository/customer-questions.yml --output-template-file ~/environment/MythicalMysfitsQuestionsService-Repository/transformed-questions.yml --s3-bucket REPLACE_ME_NEW_QUESTIONS_SERVICE_CODE_BUCKET_NAME
```

... e implemente a stack do CloudFormation. **Note: forneça um endereço de e-mail ao qual você tenha acesso como o parâmetro REPLACE_ME_EMAIL_ADDRESS (substitua-o antes de colar esse comando, a criação da stack falhará se você executar o comando sem fornecer um endereço de e-mail válido). Este será o endereço de e-mail do qual as perguntas de usuários serão publicadas pelo tópico do SNS **:
```
aws cloudformation deploy --template-file /home/ec2-user/environment/MythicalMysfitsQuestionsService-Repository/transformed-questions.yml --stack-name MythicalMysfitsQuestionsService-Stack --capabilities CAPABILITY_IAM --parameter-overrides AdministratorEmailAddress=REPLACE_ME_YOUR_EMAIL_ADDRESS
```

Quando este comando for concluído, vamos capturar o output da stack para que possamos referenciar seus valores nas etapas subsequentes (criaremos um arquivo em seu IDE chamado `questions-service-output.json`):

```
aws cloudformation describe-stacks --stack-name MythicalMysfitsQuestionsService-Stack > ~/environment/questions-service-output.json
```

Em seguida, visite o endereço de e-mail fornecido e CONFIRME sua inscrição no tópico do SNS:
![SNS Confirm](/images/module-6/confirm-sns.png)


Agora, com o novo serviço de backend em execução, vamos fazer as alterações necessárias no `index.html`, para que o frontend possa incluir o novo botão *Fale Conosco*. Abra `~ / environment / aws-modern-application-workshop / module-6 / web / index.html` e insira o endpoint da API para a nova API de Perguntas, recupere o valor do output de` REPLACE_ME_QUESTIONS_API_ENDPOINT` da stack do CloudFormation acima (localizada em `~ / environment / questions-service-output.json`). ** Lembre-se de que você também precisará colar os mesmos valores usados ​​anteriormente a deste módulo para os outros endpoints e pool de usuários dos microsserviços do Mysfits. **



Depois de fazer a alteração necessária no `index.html`, execute o seguinte comando para copiá-lo para o bucket de seu website no S3.

```
aws s3 cp ~/environment/aws-modern-application-workshop/module-6/web/index.html s3://YOUR-S3-BUCKET/
```

Agora que a nova funcionalidade Fale Conosco está implantada, visite o website e submeta uma ou duas perguntas. Se você confirmou a inscrição no SNS na etapa acima, você começará a ver essas perguntas chegando na sua caixa de entrada. Quando você ver esse email chegar, você pode seguir para explorar e analisar o ciclo de vida da requisição.

Agora, para começar a ver o comportamento da requisição para este microsserviço, visite o console do AWS X-Ray para explorar:

[AWS X-Ray Console](https://console.aws.amazon.com/xray/home)

Ao visitar o Console do X-Ray, você visualizará imediatamente um **mapa de serviços**, que mostra a relação de dependência entre todos os componentes que o X-Ray para a qual recebe **segmentos de rastreio**:

![X-Ray Lambda Only](/images/module-6/lambda-only-x-ray.png)

No início, este mapa de serviços inclui apenas as nossas funções do AWS Lambda. Sinta-se à vontade para explorar o console do X-Ray e aprender mais sobre como detalhar os dados que se tornaram automaticamente visíveis apenas listando a propriedade `Tracing: Active` no template do CloudFormation que você implantou.

Em seguida, vamos instrumentalizar mais a stack do microsserviço para que todas as dependências de serviço sejam incluídas no mapa de serviço e nos segmentos de rastreio gravados.

Primeiro, vamos instrumentalizar a API REST do API Gateway. Emita o seguinte comando inserindo o valor para ``REPLACE_ME_QUESTIONS_REST_API_ID`` que está localizado no arquivo ``questions-service-output.json`` criado pelo comando `describe-stacks` do CloudFormation mais recente que você acabou de executar. O comando abaixo permitirá que o rastreio inicie no nível do API gateway da stack de serviços:

```
aws apigateway update-stage --rest-api-id REPLACE_ME_QUESTIONS_REST_API_ID --stage-name prod --patch-operations op=replace,path=/tracingEnabled,value=true
```

Agora, envie outra pergunta para o website Mythical Mysfits e você verá que a API REST também está incluída no mapa de serviço!

![API Gateway Traced](/images/module-6/api-x-ray.png)

Depois, você usará o [SDK do AWS X-Ray para Python](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html) para que os serviços que estão sendo chamados pelas duas funções do Lambda como parte da stack de perguntas também sejam representados no mapa de serviços do X-Ray. O código já foi escrito para fazer isso, você só precisa descomentar as linhas relevantes (a remoção de comentários é realizada excluindo o `#` anterior em uma linha de código python). No código da função Lambda, você verá comentários que indicam `#UNCOMMENT_BEFORE_2ND_DEPLOYMENT` ou `#UNCOMMENT_BEFORE_3RD_DEPLOYMENT`. 

Você já concluiu a primeira implantação dessas funções usando o CloudFormation, então essa será sua **2ª implantação**. Descomente cada uma das linhas indicadas abaixo de todos os casos de `UNCOMMENT_BEFORE_2ND_DEPLOYMENT` nos seguintes arquivos e salve os arquivos depois de fazer as alterações necessárias:
* `~/environment/MythicalMysfitsQuestionsStack-Repository/PostQuestionsService/mysfitsPostQuestion.py`
* `~/environment/MythicalMysfitsQuestionsStack-Repository/ProcessQuestionsStream/mysfitsProcessStream.py`

**Note: As alterações que você descomentou permitem que o SDK do AWS X-Ray instrumentalize o SDK de Python da AWS (boto3) para capturar dados de rastreamento e registrá-los no serviço Lambda sempre que uma chamada da API da AWS é feita. Essas poucas linhas de código são tudo o que é necessário para que o X-Ray rastreie automaticamente seu mapa de serviço da AWS em uma aplicação serverless usando o AWS Lambda!**

Com essas alterações feitas, implemente uma atualização no código da função Lambda emitindo os dois comandos a seguir:

Primeiro, use o SAM para criar novos pacotes de código da função Lambda e faça o upload do código empacotado no S3:
```
sam package --template-file ~/environment/MythicalMysfitsQuestionsService-Repository/customer-questions.yml --output-template-file ~/environment/MythicalMysfitsQuestionsService-Repository/transformed-questions.yml --s3-bucket REPLACE_ME_NEW_QUESTIONS_SERVICE_CODE_BUCKET_NAME
```

Depois, use o CloudFormation para implantar as alterações na stack em execução:

```
aws cloudformation deploy --template-file /home/ec2-user/environment/MythicalMysfitsQuestionsService-Repository/transformed-questions.yml --stack-name MythicalMysfitsQuestionsService-Stack --capabilities CAPABILITY_IAM --parameter-overrides AdministratorEmailAddress=REPLACE_ME_YOUR_EMAIL_ADDRESS
```

Depois que o comando for concluído, envie uma pergunta adicional ao website Mythical Mysfits e dê uma olhada no console do X-Ray novamente. Agora você pode rastrear como o Lambda está interagindo com o DynamoDB e com o SNS!

![Services X-Ray](/images/module-6/services-x-ray.png)

A etapa final deste módulo é familiarizar-se com o uso do AWS X-Ray para triagem de problemas em seu aplicativo. Para conseguir isso, vamos *desajustar* as coisas nós mesmos e adicionar um código terrível à sua aplicação. Tudo o que esse código fará é fazer com que seu serviço web adicione 5 segundos de latência e lance uma exceção para requisições aleatórias :).

Volte para o seguinte arquivo e remova os comentários indicados por `# UNCOMMENT_BEFORE_3RD_DEPLOYMENT`:  
* `~/environment/MythicalMysfitsQuestionsStack-Repository/PostQuestionsService/mysfitsPostQuestion.py`

Este é o código que fará com que sua função Lambda lance uma exceção. Além disso, você pode notar acima da função `hangingException ()` que estamos usando a funcionalidade pronta para uso do **SDK do AWS X-Ray** para gravar um subsegmento de rastreio toda vez que a função for chamada.  Agora, quando você analisar o Rastreamento de uma requisição específica, poderá ver que todas as requisições estão presas nessa função por pelo menos cinco segundos antes de lançarem a exceção.
O uso dessa funcionalidade em suas próprias aplicações ajudará você a identificar gargalos de latência semelhantes em seu código ou locais onde exceções estão sendo lançadas.

Depois de fazer as alterações necessárias no código e salvar o arquivo `mysfitsPostQuestion.py`, execute os mesmos dois comandos de antes para empacotar e implantar suas alterações no CloudFormation. ** Use a tecla de seta PARA CIMA dentro do seu terminal Cloud9 para ver os comandos anteriores e primeiro execute o comando `sam package` no seu histórico, e depois o comando` aws cloudformation deploy`, subseqüentemente.**

Depois de ter emitido esses dois comandos, envie mais algumas perguntas em seu Website Mysfits. Algumas dessas perguntas não serão exibidas na sua caixa de entrada, pois seu código novo e terrível gerou um erro!

Se você visitar o console do X-Ray novamente, perceberá que o mapa de serviço da função Lambda MysfitPostQuestionsFunction tem um anel ao seu redor que não é mais apenas Verde. Isso porque respostas de erro foram geradas lá. O X-Ray fornecerá a você uma representação visual da integridade geral do serviço em todos os serviços instrumentados em seu mapa de serviço:

![X-Ray Errors](/images/module-6/x-ray-errors.png)

Se você clicar nesse serviço no mapa de serviço, perceberá que no lado direito do console do X-Ray você pode visualizar os rastreamentos que correspondem à latência geral em destaque mostrada no gráfico de latência do serviço e/ou ao código de status em que você está interessado.  Aumente o zoom do gráfico de latência para que o sinal em torno de 5 segundos esteja dentro do gráfico e / ou selecione a caixa de seleção de Erro e clique em **View traces**:

![View Traces](/images/module-6/view-traces.png)

Isso levará você ao painel de Rastreamento, onde você poderá explorar o ciclo de vida de solicitações específicas, ver o gasto de latência em cada segmento do serviço e visualizar a exceção reportada e o rastreamento da stack associada. Clique em qualquer um dos IDs de Rastreamentos em que a resposta é reportada como 502, depois na página subseqüente **Detalhes do Rastreamento**, clique em **hangingException** para visualizar esse subsegmento específico onde a exceção foi lançada em nosso código:

![Exception](/images/module-6/exception.png)

Parabéns, você completou o módulo 6!

### [Prossiga para o Módulo 7](/module-7)


#### [Centro do desenvolvedor da AWS](https://developer.aws)
