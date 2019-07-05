# Módulo 1: Configuração da IDE e Hospedagem de Website Estático

![Architecture](/images/module-1/architecture-module-1.png)

**Tempo estimado de duração:** 20 minutos

**Serviços usados:**
* [AWS Cloud9](https://aws.amazon.com/cloud9/)
* [Amazon Simple Storage Service (S3)](https://aws.amazon.com/s3/)

Neste módulo siga as instruções para criar seu IDE baseado na nuvem com o [AWS Cloud9](https://aws.amazon.com/cloud9/) e lance a primeira versão do website estático Mythical Mysfits. O [Amazon S3](https://aws.amazon.com/s3/) é um serviço de armazenamento de objetos barato e altamente durável e disponível que pode entregar objetos diretamente via HTTP. Isso o torna incrivelmente útil para entregar conteúdos web estáticos (html, js, css, conteúdos de mídia, etc.) diretamente para navegadores web directly to web browsers for sites on the Internet. Nós vamos utilizar o S3 para hospedar o conteúdo para o nosso website Mythical Mysfits.

### Começando

#### Faça login AWS Console
Para começar, faça login no [AWS Console](https://console.aws.amazon.com) com a conta AWS que você usará neste workshop.

Essa aplicação web pode ser implantada em qualquer região AWS que suporte todos os serviços utilizados nela. As regiões suportadas incluem:

* us-east-1 (N. Virginia)
* us-east-2 (Ohio)
* us-west-2 (Oregon)
* eu-west-1 (Ireland)

Selecione uma região do menu dropdown no canto superior direito do AWS Console.

### Criando sua IDE Mythical Mysifts

#### Crie um novo Ambiente AWS Cloud9

 Na página inicial do AWS Console, digite **Cloud9** na barra de pesquisa e selecione o serviço:
 ![aws-console-home](/images/module-1/cloud9-service.png)


Clique **Create Environment** na página inicial do Cloud9:
![cloud9-home](/images/module-1/cloud9-home.png)


Nomeie seu ambiente como **MythicalMysfitsIDE**, use qualquer descrição que você quiser e clique em **Next Step**:
![cloud9-name](/images/module-1/cloud9-name-ide.png)


Deixa as configurações do ambiante com seus valores padrão e clique em **Next Step**:
![cloud9-configure](/images/module-1/cloud9-configure-env.png)


Clique em **Create Environment**:
![cloud9-review](/images/module-1/cloud9-review.png)


Quando a criação da IDE tiver terminado, uma tela de boas vindas será apresentada para você. Ela se parece com essa:
![cloud9-welcome](/images/module-1/cloud9-welcome.png)

#### Clonando o repositório do Wokshop Mythical Mysfits

No painel inferior do seu novo IDE Cloud9 você verá um terminal de linha de comandos pronto para ser usado. Execute o seguinte comando git no terminal para clonar o código necessário para completar este tutorial:

```
git clone -b python https://github.com/aws-samples/aws-modern-application-workshop.git
```

Depois de clonar o repositório você verá que seus arquivos de projeto agora incluem os arquivos clonados:
![cloud9-explorer](/images/module-1/cloud9-explorer.png)


No terminal,mude o diretório para o diretório do repositório recém-clonado:

```
cd aws-modern-application-workshop
```

### Criando um Website Estático no Amazon S3

#### Crie um Bucket do S3 e Configure-o para hospedagem de website
Em seguida, vamos criar a componentes da infraestrutura necessária para hospedar um website estático no Amazon S3 utilizando o [AWS CLI](https://aws.amazon.com/cli/).

**Nota: Este workshop usa textos (placeholders) para indicar os locais em que você inserir nomes. Esses textos começam com o prefixo `SUBSTITUA_AQUI_` para que seja fácil encontrá-los usando CTRL-F no Windows ou ⌘-F no Mac.**

Primeiro, crie um [bucket do S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html), substitua *SUBSTITUA_AQUI_O_NOME_DO_BUCKET* com o nome único que você quiser dar para seu bucket, como descrito em [requirements for bucket names](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html#bucketnamingrules).** Coipie o nome que você escolheu e guarde-o, você usará ele em diversas partes desse workshop:

```
aws s3 mb s3://SUBSTITUA_AQUI_O_NOME_DO_BUCKET
```

Agora que nós criamos um bucket, nós precisamos configurar algumas opções para habilitar o bucket a ser usado para [hospedagem de websites estáticos](https://docs.aws.amazon.com/AmazonS3/latest/dev/WebsiteHosting.html).  Essa configuração habilita os objetos no bucket a serem requisitados usando um nome DNS público para o bucket, assim como direcionar requisições ao nome DNS para uma página específica do website (index.html, na maioria dos casos):

```
aws s3 website s3://SUBSTITUA_AQUI_O_NOME_DO_BUCKET --index-document index.html
```

#### Atualize as Políticas do Bucket S3

Todos os buckets criados no Amazon S3 são totalmente privados por padrão. Para que possam ser usados como um website público, nós precisamos criar uma [Politica](https://docs.aws.amazon.com/AmazonS3/latest/dev/example-bucket-policies.html) para o bucket S3 que indica que objetos armazenados dentro desse novo bucket podem ser acessados publicsamente por qualquer pessoa. Políticas de Bucket são representadas como documentos JSON que definam as *Ações* do S3 (chamadas API do S3) que possuem permissão (ou não possuem permissão) para serem feitas por diferentes *Principals* (no nosso caso o público, ou qualquer pessoa).

The JSON document for the necessary bucket policy is located at: `~/environment/aws-modern-application-workshop/module-1/aws-cli/website-bucket-policy.json`.  This file contains a string that needs to be replaced with the bucket name you've chosen (indicated with `SUBSTITUA_AQUI_O_NOME_DO_BUCKET`).  

Para **abrir um arquivo** no Cloud9, use o Explorador de Arquivos no painel esquerdo e clique duas vezes sobre `website-bucket-policy.json`:

![bucket-policy-image.png](/images/module-1/bucket-policy-image.png)

Isso abrirá o `bucket-policy.json` no painel de Edição de Arquivo. Substitua a string mostrada pelo nome do seu bucket usado nos comandos anteriores:

![replace-bucket-name.png](/images/module-1/replace-bucket-name.png)


Execute o sewguinte comando CLI command para adicionar uma política de acesso público ao bucket do seu website:

```
aws s3api put-bucket-policy --bucket SUBSTITUA_AQUI_O_NOME_DO_BUCKET --policy file://~/environment/aws-modern-application-workshop/module-1/aws-cli/website-bucket-policy.json
```

#### Publique o Cointeúdo do Website no S3

Agora que nosso novo bucket do website está apropriadamente configurado, vamos adicionar a primeira versão da homepage Mythical Mysfits homepage ao bucket. Use o seguinte comando S3 CLI que imita o comando do linux para copiar arquivos (**cp**) para copiar a página index.html fornecida do seu IDE para o seu novo bucket S3 (substituindo o nome  do bucket apropriadamente).

```
aws s3 cp ~/environment/aws-modern-application-workshop/module-1/web/index.html s3://SUBSTITUA_AQUI_O_NOME_DO_BUCKET/index.html
```

Agora, abra seu navegador favorito e insiora uma das URIs abaixo na barra de endereços. Uma das URIs abaixo tem um '.' antes do nome da região, e as outras têm '-'. Qual dela vocês deve usar vai depender da região que você estiver usando.

A string a ser substituida **SUBSTITUA_AQUI_SUA_REGIAO** deve estar de acordo com a região onde que você criou o seu bucket S3 (por exemplo: us-east-1):

Para a região us-east-1 (N. Virginia), us-west-2 (Oregon), eu-west-1 (Ireland) use:
```
http://SUBSTITUA_AQUI_O_NOME_DO_BUCKET.s3-website-SUBSTITUA_AQUI_SUA_REGIAO.amazonaws.com
```

Para a região us-east-2 (Ohio) use:
```
http://SUBSTITUA_AQUI_O_NOME_DO_BUCKET.s3-website.SUBSTITUA_AQUI_SUA_REGIAO.amazonaws.com
```

![mysfits-welcome](/images/module-1/mysfits-welcome.png)

Parabéns, você criou o site estático básico do Mythical Mysfits!

## Nota sobre o Amazon CloudFront - Melhor prática para entregar websites na AWS ##

Para que este workshop pudesse te permitir passar mais rapidamente pela parte de hospedar seu website estático Mythical Mysfits, nós te pedimos para tornar um bucket S3 publicamente acessível. Criar buckets S3 públicos é perfeitamente OK e normal para diversas aplicações... quando estiver criando um website na AWS que seja voltado para o público, a melhor prática é usar o [**Amazon CloudFront**](https://aws.amazon.com/cloudfront/) como a rede de entrega de conteúdo (Content Delivery Network (CDN)) global e endpoint voltado para o público em seu site. 

O Amazon CloudFront habilita o uso de diversas soluções que são benéficas para websites públicos (latência menor, redundância global, integração com AWS Web Application Firewall, etc.) e ainda reduz os custos de transferência de dados para um website quando comparado com casos em que os clientes fazem requisições de dsados diretamente do S3.

Mas, por causa de sua natureza global, a criação de uma nova distribuição CloudFront pode levar mais de 15 minutos em alguns casos antes que se faça disponível globalmente. Por conta disso, decidimos pular esse passo nesse tutorial para podermos fazê-lo de forma mais ágil. Mas se você estiver construindo um website público em outra ocasião, o uso do CloudFront deve ser um requisito para que se possa atender às melhores práticas. 

Para aprender mais sobre o CloudFront, [clique aqui.](https://aws.amazon.com/cloudfront/)

Isso conclui o Módulo 1.

[Ir para o Módulo](/module-2)


## [AWS Developer Center](https://developer.aws)
