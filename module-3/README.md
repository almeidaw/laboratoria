# Módulo 3 - Adicionando uma camada para dados com o Amazon DynamoDB

![Architecture](/images/module-3/architecture-module-3.png)

**Tempo esperado:** 20 minutos

**Serviços usados:**
* [Amazon DynamoDB](https://aws.amazon.com/dynamodb/)

### Resumo

Agora que você já tem funcionando um pipeline de CI/CD para entregar as atualizações feitas em seu serviço de maneira automática sempre que seu código for atualizado em seu repositório, possibilitando mover rapidamente novas funcionalidades da aplicação da fase de concepção até estarem disponíveis para os clientes da Mythical Mysfits. Junto com esse ganho de agilidade podemos adicionar outra parte fundamental na arquitetura do site da Mythical Mysfits, uma camada de Dados. Nesse módulo você vai criar uma tabela no [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), um serviço de banco de dados gerenciado e escalável da AWS de super performance. Em vez de ter todos os Mysfits armazenados em um arquivo JSON estático, nós armazenaremos eles em um banco de dados, para assim fazer o site ser futuramente mais escalável.

### Adicionando um Banco de Dados NoSQL no Mythical Mysfits

#### Criando uma tabela no DynamoDB

Para adicionar uma tabela do DynamoDB na arquitetura nós precisamos incluir outro arquivo de entrada JSON que define a tabela chamada **MysfitsTable**. Essa tabela terá um índice primário definido por um atributo hash key chamado **MysfitId**, e outros dois índices secundários. O primeiro índice secundário vai ter a hash key **GoodEvil** e a range key **MysfitId**, por outro lado o segundo índice secundário terá a hash key **LawChaos** e a range key **MysfitId**. Esses dois índices secundários vão nos permitir executar consultas nessa tabela para retornar todos os Mysfits que são de uma determinada Espécie ou Linhagem para permitir que a funcionalidade de filtragem que ainda não está funcionando no site. Você pode ver esse arquivo em 
`~/environment/aws-modern-application-workshop/module-3/aws-cli/dynamodb-table.json`. Não é necessária nenhuma mudança nesse arquivo. Para aprender mais sobre índices no DynamoDB e outros conceitos chave visite [this page](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html).

Para criar a tabela utilizando o AWS CLI, execute o seguinte comando no terminal do Cloud9:

```
aws dynamodb create-table --cli-input-json file://~/environment/aws-modern-application-workshop/module-3/aws-cli/dynamodb-table.json
```

Depois que esse comando rodar, você pode visualizar os detalhes da tabela criada executando o seguinte comando do AWS CLI no terminal:

```
aws dynamodb describe-table --table-name MysfitsTable
```

Se executarmos o seguinte comando para recuperar todos os itens armazenadas na tabela, você verá que sua tabela encontra-se vazia:

```
aws dynamodb scan --table-name MysfitsTable
```

```
{
    "Count": 0,
    "Items": [],
    "ScannedCount": 0,
    "ConsumedCapacity": null
}
```

#### Adicionando Itens na Tabela do DynamoDB

Também é disponibilizado um arquivo JSON que pode ser usado para inserir em batch um número de itens Mysfit dentro da tabela. Isso pode ser realizado utilizando a API do DynamoDB **BatchWriteItem.** Para chamar essa API usando a arquivo JSON, execute o seguinte comando no terminal (a resposta do serviço deve reportar que não há itens que não foram processados):  

```
aws dynamodb batch-write-item --request-items file://~/environment/aws-modern-application-workshop/module-3/aws-cli/populate-dynamodb.json
```

Agora, se você rodou o mesmo comando para escanear todo o conteúdo da tabela, você achará os itens que foram carregados para a tabela:

```
aws dynamodb scan --table-name MysfitsTable
```

### Fazendo a Primeira Mudança *Real* no Código

#### Copiando o Flask Service Code Atualizado

Agora que nós já possuímos nossos dados incluidos na tabela, vamos modificar nosso código da aplicação para ler dessa tabela em vez de retornar o arquivo JSON estático que foi usado no Módulo 2. Nós incluímos um novo conjunto de arquivos python para nosso microsserviço Flask, mas agora em vez de ler o arquivo JSON estático nós faremos uma requisição para o DynamoDB.

A requisição é formada usando o SDK da AWS para Python chamado **boto3**. Esse SDK é um jeito simples porém poderoso de interagir com os serviços da AWS via códigos em Python. Ele permite você a usar funções dos serviços que são compatíveis com as APIs da AWS e com os comandos CLI que você vem executando ao longo desse workshop. Traduzir esses comandos para que eles funcionem como um código Python é simples quando se está utilizando o **boto3**. Para copiar os novos arquivos dentro do seu repositório do CodeCommit, execute o seguinte comando no terminal:

```
cp ~/environment/aws-modern-application-workshop/module-3/app/service/* ~/environment/MythicalMysfitsService-Repository/service/
```

#### Envie o Código Atualizado para se Pipeline de CI/CD

Agora, nós precisamos cadastrar essas mudanças de código para o CodeCommit usando linhas de comando git. Rode os seguintes comandos para dar entrada as mudanças de código e dar início ao Pipeline de CI/CD:

```
cd ~/environment/MythicalMysfitsService-Repository
```

```
git add .
```

```
git commit -m "Add new integration to DynamoDB."
```

```
git push
```

Agora, nos próximos 5-10 minutos você verá suas mudanças no código alcançarem todo seu Pipeline CI/CD no CodePipeline e chegar até o AWS Fargate no Amazon ECS. Sinta-se livre para explorar o console do AWS CodePipeline e ver as mudanças surtindo efeito ao longo do seu Pipeline.

#### Atualizando o Conteúdo do Website no S3

Finalmente, nós precisamos publicar uma nova página index.html no seu bucket S3 para assim a nova funcionalidade que utiliza as requisições para filtrar as resposta seja utilizada. O novo arquivo index.html está presente em `~/environment/aws-modern-application-workshop/module-3/web/index.html`. Abra esse arquivo na sua IDE do Cloud9 e substitua a string “REPLACE_ME” assim como você fez no Módulo 2, com o endpoint do NLB aprorpiado. Lembre-se de não incluir o path /mysfits. Veja o arquivo que você editou no diretório /module-2/ se você precisar. Depois de substituir o endpoint para apontar para o seu NLB, faça o upload do novo index.html rodando o seguinte comando (substitua com o nome do bucket que você criou no Módulo 1):

```
aws s3 cp --recursive ~/environment/aws-modern-application-workshop/module-3/web/ s3://REPLACE_ME_WEBSITE_BUCKET_NAME/
```

Visite novamente seu site Mythical Mysfits para ver a nova população de Mysfits sendo carregada da sua tabela do DynamoDB e como a funcionalidade de Filtro está funcionando.

Isso conclui o Módulo 3.

[Prossiga para o Módulo 4](/module-4)


## [Centro do desenvolvedor da AWS](https://developer.aws)
