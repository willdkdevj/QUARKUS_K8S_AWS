# Implementando serviços Quarkus em uma arquitetura CQRS em um ambiente de transações bancárias
> Este projeto apresenta um cenário onde são realizadas transações bancárias, executando eventos assíncronos guiados por uma arquitetura CQRS em um serviço disponibilizado em cloud.

[![Quarkus Badge](https://img.shields.io/badge/-Quarkus-purple?style=flat-square&logo=QUARKUS&logoColor=white&link=https://quarkus.io/)](https://quarkus.io/)
[![Kafka Badge](https://img.shields.io/badge/-Kafka-000?style=flat-square&logo=Apache-Kafka&logoColor=white&link=https://kafka.apache.org/)](https://kafka.apache.org/)
[![K8S Badge](https://img.shields.io/badge/-K8S-blue?style=flat-square&logo=Kubernetes&logoColor=white&link=https://kubernetes.io/pt-br/)](https://kubernetes.io/pt-br/)


## Descrição da Aplicação
A aplicação simula uma conta bancária em que um usuário final realiza transações de depósito e saques de modo assíncrono, na qual se faz necessário recalcular o saldo da conta bancária do usuário para manter sua atomicidade, desta forma, garantindo que ao consultar seu saldo, este apresente o real estado da conta. Para isto, a aplicação é consistituída por dois serviços: *Transaction-Service* que recebe as transações de receita ou despesas, no caso, depósitos ou saques realizadas na conta de um determinado cliente, e ao finalizar o processamento destas transações, são retornadas suas conclusões para um outro serviço, que é representado pelo *Balance-Service* que será responsável por ter as informações restritas ao ``extrato`` da conta do cliente, enquanto o outro banco possuirá cada interação bancária.

<img align="center" width="400" height="300" src="/images/cqrs_architecture.png">

Desta forma, temos dois serviços que executam atribuições distintas de operações bancárias utilizando da tecnologia Quarkus Java, sendo estruturados utilizando os conceitos da arquitetura em CQRS, na qual se obtem uma aplicação que trafega as informações entre bancos de dados relacionais PostgreSQL utilizando do barramento assíncrono da plataforma do Apache Kafka.

## Sobre a Arquitetura CQRS
O *Command Query Responsibility Segregation* (CQRS) é um padrão arquitetural que propõem a separação das responsabilidades por modulos entre as consultas e os comandos para manipulação de dados, ele não é um estilo arquitetural como desenvolvimento em camadas, modelo client-server, REST, entre outros. Ele  não se trata de um padrão arquitetural de alto-nível, mas sim, de um modo de "componentizar" partes da aplicação, podendo ser parte ou ela por inteiro.

Este tipo de arquitetura tem o conceito que a informação obtida, não necessariamente será de um único banco de dados, no caso, pode haver mais de um banco trafegando dados do cliente ou utilizar-se de um componente para armazenar um dado específico sobre o mesmo. Desta forma, a informação a ser retornada não será a informação geral de um dado usuário, na qual podem existir inúmeros atributos referente a tupla, mas um fragmento da qual o serviço necessitará para retornar o valor aguardado, assim sendo, não havendo a necessidade de ter sua performance afetada devido a gravação, locks ou disponibilidade do banco de dados.

Para cada transação efetuada pelo usuário, o serviço (Service Interfaces) correspondente pode interagir com módulos diferentes, sendo no cenário um para a consulta de extrado (Query Model) e outra para transações bancárias (Command Model), que por sua vez compartilham dados através de um barramento assíncrono com o Apache Kafka.

Referente a *QueryStack* é mais simples do que a *CommandStack*, pois sua responsabilidade é de obter os dados formatados para a exibição. Desta forma, podemos dizer que a QueryStack é uma camada síncrona que recupera os dados de um banco de leitura desnormalizado. Este banco desnormalizado pode ser um NoSQL como MongoDB, Redis, RavenDB etc, já neste projeto será utilizado o MongoDB. O conceito de desnormalizado pode ser aplicado com *One Table Per View* ou seja uma consulta ``flat`` que retorna todos os dados necessários para ser exibido em uma view (tela) específica.
O uso de consultas “flats” em um banco desnormalizado evita a necessidade de joins, tornando as consultas muito mais rápidas. 
> É preciso ter a ciência que haverá a duplicidade de dados para compreender este modelo.

Já o *CommandStack* possuí o contexto assíncrono. É aqui que encontramos as entidades, regras de negócio, processos e etc. O CommandStack segue uma abordagem ``behavior-centric`` onde toda intenção de negócio é inicialmente disparada pela UI como um caso de uso. Utilizamos o conceito de Commands para representar uma intenção de negócio. Os Commands são declarados de forma imperativa e são disparados assincronamente no formato de eventos, são interpretados pelos CommandHandlers e retornam um evento de sucesso ou falha.
> Toda vez que um Command é disparado será alterado o estado de uma entidade no banco de gravação, desta forma, um processo tem que ser disparado para os agentes que irão atualizar os dados necessários sejam invocados no banco de leitura.

### Vantagens de implementar o CQRS

A implementação do CQRS quebra o conceito monolítico clássico de uma implementação de arquitetura em N camadas, onde todo o processo de escrita e leitura passa pelas mesmas camadas e concorre entre si no processamento de regras de negócio e uso de banco de dados. Este tipo de abordagem aumenta a disponibilidade e escalabilidade da aplicação e a melhoria na performance surge principalmente pelos aspectos:

* Todos comandos são assíncronos e processados em fila, assim diminui-se o tempo de espera.
* Os processos que envolvem regras de negócio existem apenas no sentido da inclusão ou alteração do estado das informações.
* As consultas na QueryStack são feitas de forma separada e independente e não dependem do processamento da CommandStack.
* É possível escalar separadamente os processos da CommandStack e da QueryStack.

Uma outra vantagem de utilizar o CQRS é que toda representação do seu domínio será mais expressiva e reforçará a utilização da linguagem ubíqua nas intenções de negócio.


## Sobre o Framework Quarkus

É mais um projeto da comunidade Open Source que consite em um framework Java nativo em Kubernetes [K8S] projetado para funcionar com padrões, estruturas e bibliotecas mais "enxutas" que a da própria Java Virtual Machine [JVM], sendo mais uma boa opção para o deploy de tecnologias em *cloud*, como micro services e serverless.

Um dos benefícios do framework é a econônica de recursos, como o baixo consumo de memória e o baixo tempo de inicialização, que permite uma implantação de pods mais veloz. Outra característica relevante é a agilidade na produtividade que encurtar as fases de testes devido ao ``live coding`` (código vivo), ao permitir visualizar as atualizações no momento da sua execução. Além disso, o Quarkus permite unificar a programação imperativa e reativa fornecendo mais possibilidades aos desenvolvedores.

Quanto ao *deloy*, como ele necessita de menos recursos é possível realizar o *spin up* um pouco maior que os frameworks tradicionais, isto significa que, com o Quarkus JVM pode-se fazer mais com a mesma quantidade de recursos com mais aplicações escalonadas. Isto faz com que a aplicação seja mais responsiva à mudança de cargas e mais confiável durante operações com grande necessidade de escala.

Sobre os serviços implementados, temos o *Transaction-Service* que como já vimos será responsável por realizar todas as transações bancárias, onde em seu *endpoint* TransactionResource temos um método que recebe requisições POST que recebe um objeto Transaction a fim de persistí-lo, enviá-lo para o Kafka Server e retornar o ID da transação caso tudo ocorrá sem problemas.
```sh
	@POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    @Transactional
    public Response save(Transaction transaction) {
        transaction.persist();
        emitter.send(transaction);
        return Response.created(URI.create("/transactions/" + transaction.id)).build();
    }
```

Já a classe que representa a entidade no contexto relacional do banco do dados estende a classe [PanacheEntityBase] que é um padrão declarar seus atributos de modo público (Public) por utilizar o conceito de *active-record*, que é você ter a classe entidade tendo os métodos de manipulação no banco de dados, por isso, não se faz necessário haver um repositório JPA para utilizar métodos de persistência, como o método ``ṕersist()``.

Outra classe importante para o serviço é a TransactionOutgoingProducer que é uma classe de escopo do Jakarta EE que produz sempre que ocorre um evento de mensagem do resource (TransactionResource) é injetado ao contexto da aplicação um *emitter* (emissor) para o canal do tópico ("transaction"), na qual está presente no objeto Transaction as informações sobre a transação a serem serializadas para envio como registro no barramento Kafka.


Agora no outro lado da arquitetura encontra-se o serviço *Balance-Service* injeta no contexto da aplicação um Balances e um método GET que permite encontrar uma transação através do accountId gerado pelo registro da transação em Transaction-Service.
```sh
	@Inject
    public BalanceResource(Balances balances) {
        this.balances = balances;
    }

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    @Transactional
    public Response findByAccountId(@QueryParam("accountId") String accountId) {
        return balances.findByAccountId(accountId).map(balance -> Response.ok(balance).build())
                .orElseThrow(IllegalArgumentException::new);
    }
```
Já a classe que representa a entidade também estende a classe [PanacheEntityBase], mas que tem o diferencial de possuir métodos como o recalculateBalance, que verifica o tipo de transação efetuada para realizar em tempo de execução a atualização do valor saldo em conta ao incrementar (receita/depósito) ou decrementar (despesa/saque) o atributo ``value``.
```sh
	public Balance recalculateBalance(Transaction transaction) {
        if (transaction.isIncome()) {
            this.value += transaction.getValue();
        } else {
            this.value -= transaction.getValue();
        }
        return this;
    }
```

Sua comunicação com o Kafka Server é possível através da classe TransactionIncomeProducer que produz para o CDI um objeto Topology que fica "escutando" o tópico [transaction] a fim de averiguar alguma atualização sobre interação entre os serviços com o objetivo de obter estas transactions e recalculdar o valor da transaction.
```sh
	@Inject
    public TransactionIncomeProducer(Balances balances) {
        this.balances = balances;
    }

    @Produces
    public Topology onTransactionTopology() {
        var builder = new StreamsBuilder();
        builder.stream("transactions",
                Consumed.with(Serdes.String(),
                        Serdes.serdeFrom(new JsonObjectSerializer(), new JsonObjectDeserializer())))
                .foreach((__, value) -> {
                    log.info("Receiving transaction " + value);
                    var transaction = Transaction.ofMap(value.getMap());
                    log.info("Receiving transaction with description " + transaction.getDescription());
                    balances.recalculate(transaction);
                });
        return builder.build();
    }
```
Para finalizar, a construção do artefato é realizado de forma nativa com a própria API do Quarkus através do seu plugin Maven e a Oracle plugin Maven VM que está configurada na plataforma, na qual é executado em um container Docker e será exclusivo dele onde o próprio Quarkus fornece os scripts para sua execução. Este script é gerado pela pelo scarfold do Quarkus no diretório Docker com o arquivo Docker.native.
```sh
mvn package -Pnative -Dquarkus.native.container-build=true
```
Desta forma, o Quarkus utiliza o Docker que está localmente instalado e através de uma imagem ele realiza um building da aplicação utilizando a [GraalVM], que é uma machine virtual Quarkus, na qual criará uma artefato nativo que não precisará da JVM para ser executado. A GraalVM é uma versão que possui somente os módulos padrões.

Ao concluir a construção do artefato se faz necessário criar uma Docker image executamos o seguinte comando:
```sh
docker build -f src/main/docker/Dockerfile.native -t moduscreate/quarkus-transaction-service:v1 .
```

Para enviar a imagem para o Docker Hub para ser consumida pelos serviços da Amazon Web Services (AWS), posteriormente.
```sh
docker push moduscreate/quarkus-transaction-service:v1
```

 

Estas etapas devem ser feitas em todos os serviços que criamos com o Quarkus, neste caso, o Transaction-Service e o Balance-Service.


## Sobre as Necessidades de Ferramentas Externas
Esta aplicação foi realizada seu deploy no serviço Elastic Kubernetes Service [EKS] da Amazon Web Services [AWS], utilizando dois serviços do framework Quarkus que se comunicam através de um barramento assíncrono usando a plataforma Kafka para processamento de streams (mensageria). Desta forma, foram desenvolvidos manifestos do Kubernetes para implantação no EKS e parametrização das configurações necessárias para obter um ambiente como o de produção.

* Banco de Dados - RDS que roda uma instancia do PostgreSQL (Class - db.t2.micro);
* Cluster - EKS que contem o serviço de Kubernetes na qual no EC2 Experience estão presentes 3 máquinas rodando a aplicação, 1 máquina rodando um Kafka Server e 1 máquina rodando um K6 Performance Server para testes da aplicação.

<img align="center" width="400" height="300" src="/images/settings_panel.png">

Kubernetes (K8s) é um produto Open Source utilizado para automatizar a implantação, o dimensionamento e o gerenciamento de aplicativos em contêiner.
> Este texto foi obtido da própria documentação da plataforma

Para implantar os serviços externos:
```
docker-compose up -d --build
```
Ele implantará quatro contêineres docker em seu ambiente com MongoDB, PostgreSQL, Kafka e Zookepper (exigido por Kafka)

Depois de implantar o Kafka, você precisará [criar o tópico no cluster Kafka] (https://kafka.apache.org/quickstart).


### Construindo a Estrutura de Configurações do K8S
No Kubernetes (K8S) em um mesmo cluster podem ser exportados inúmeras aplicações, onde se faz necessário criar uma estrutura através da parametrização de um perfil e local onde encontramos os containers responsáveis por diponibilizarem as aplicações. 

Em cada serviço são criados scripts para configuração do ambiente em cloud, na qual será orquestrado pelo K8S. Desta forma, nos diretórios ``kubefiles/`` estão todos os scripts necessários para configuração, como:
* Namespace - cria um local específico para administração dos serviços em execução; 
* Service - cria um proxy a fim de orientar qual replica de serviço deve atender a requisição;
* Deployment - ele especifica em qual local (namespace) deve ser criado a replica do serviço (a aplicação), quantas replicas devem ser executadas, qual o modelo que deve ser seguido, a especificação do container com a imagem e porta para acesso e saída, entre outras configurações;

Desta forma, para realizar a configuração do cluster fornecido pela AWS utilizando o comando ``kubectl apply -f "file_name.yml"`` para automatizá-la.


## Testando a Aplicação

#### Executando uma requisição CURL para criar uma transação *income*
```
curl -X POST -H "Content-Type: application/json" -d @income-transaction.json http://localhost:8080/transactions
```
#### Executando uma requisição CURL para criar uma transação *expense*
```
curl -X POST -H "Content-Type: application/json" -d @expense-transaction.json http://localhost:8080/transactions
```
#### Executando uma requisição CURL para obter um balanciamento
```
curl http://localhost:8081/balance\?accountId\=william | json_pp
```
#### Executando o [K6's](https://k6.io) para um teste simples de performance
````
k6 run --vus 10 --duration 60s performance-tests/income.js
k6 run --vus 10 --duration 60s performance-tests/expense.js
````

## Agradecimentos

Obrigado por ter acompanhado aos meus esforços para desenvolver este Projeto utilizando o Quarkus e o padrão arquitetural CQRS para criação de uma aplicação em *cloud*! :octocat:

Como diria um antigo mestre:
> *"Cedo ou tarde, você vai aprender, assim como eu aprendi, que existe uma diferença entre CONHECER o caminho e TRILHAR o caminho."*
>
> *Morpheus - The Matrix*