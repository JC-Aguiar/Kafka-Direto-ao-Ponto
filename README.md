# Kafka - Direto ao Ponto
Guia resumido criado por João Costal Aguiar à partir do livro: <i>"Kafka - the Definitive Guide 2º Edition"</i>,  escrito por Gwen Shapira, Todd Palino, Rajini Sivaram Krit Petty.\
Essa apresentação destaca os principais pontos dos tópicos mais relevantes nos capítulos deste  material disponibilizado pela Confluent, atual detentora da tecnologia.

# Sumário
- [Capítulo 01: Conhecendo Kafka](?)
- [Capítulo 02: Instalando Kafka](?)
- [Capítulo 03: pendente... ](link?)
- [Capítulo 04: pendente... ](link?)
- [Capítulo 05: pendente... ](link?)
- [Capítulo 06: pendente... ](link?)
- [Capítulo 07: pendente... ](link?)
- [Capítulo 08: pendente... ](link?)
- [Capítulo 09: pendente... ](link?)




# Capítulo 1: Conhecendo Kafka
Este resumo abrange as páginas 1 a 17 do livro e tem como objetivo fazer uma abordagem geral dos principais elementos da tecnologia.\
Outros capítulos irão se aprofundar e explorar em mais detalhes cada um dos pontos descritos à seguir. 


## Conceito
Kafka pode ser definido como `Distributing Streaming Platform` (plataforma de transmissão distribuída), criado para resolver problemas de  arquitetura onde um servidor central de mensageria em fila age como intermediário entre aplicações. Originalmente foi criado para atender uma demanda do LinkedIn para emitir informações do que os usuários faziam no website.

Kafka usa o padrão `Pub/Sub`, que se caracteriza por enviar pacotes de dados para um `Topic` ao invés de um destinatário em especial, categorizando as informações de forma a permitir que diferentes aplicações e servidores se inscrevam e recebam somente os dados do qual possuem assinatura.\
Todo evento trafegado dentro do Kafka consiste de uma chave e uma mensagem.
- A `chave`  é os metadados opcionais, usadas ao publicar eventos no `Broker`.
- A `mensagem` é a menor unidade de informação do Kafka e representa algum conteúdo.

Por uma questão de eficiência, as mensagens são escritas nos `Brokers` em lotes, os armazenando direto no disco (`Disk-Based Retention`). Assim, não é responsabilidade dos `Consumers` ou `Producers` gerenciar internamente a persistência ou retenção destes dados.

Diferneça entre uma arquitetura sem Kafka (imagem superior) para uma arquitetura com Kafka (imagem inferior).
![image](https://github.com/user-attachments/assets/c99988c1-9fce-4289-bfc7-efb8ba797006)\
![image](https://github.com/user-attachments/assets/29a2c7a2-6210-4b75-9f03-bf8b52120082)



## Ninhos e Servidores 
Servidores Kafka são chamados de `Broker`, sendo instanciados dentro de um ninho denominado de `Cluster`.\
Os servidores são responsáveis por realizar a ponte entre `Consumers` e `Producers`. Armazenando e gerenciados as mensagens.\
Todo ninho tem ao menos 1 `Broker` e 1 `Broker Controller`. Este que será o administrador do `Cluster`, sendo responsável por:
- Gerenciar qual `Partitions` será inscrita para qual membro.
- Monitorar a saúde de cada um dos demais membros.

![image](https://github.com/user-attachments/assets/3310c55b-5166-4dbe-87d0-102475b84b86)

Quando se possui mais de 1 `Cluster` é comum a necessidade de sincronizar os dados usando o mecanismos de replicação. Entretanto, esse recurso do Kafka é projetado para funcionar apenas dentro de um único `Cluster`, não sendo compartilhável entre os demais.\
Um alternativa é a ferramenta `MirrorMaker`, para `Clusters Replication`. Funciona exatamente como um `Consumers` e `Producers` entre os `Clusters`, os conectando por um sistema de fila próprio.\
![image](https://github.com/user-attachments/assets/4443523f-8245-401f-8f1b-79fdb05a413f)


## Tópicos e Partições
Mensagens são classificadas por `Topics` e estes, por sua vez, podem conter 1:N `Partitions`. O Kafka não garante ordenação (cronológica) das mensagens por `Topic`, mas garante por partição individual.\
![image](https://github.com/user-attachments/assets/802888cc-cd95-481a-87b1-3c87ed71a2e5)

`Partitions` são unidades de armazenamento paralelo dentro de um `Topic`, podendo ser hospedadas em diferentes `Brokers` para permitir escalonamento horizontal. Portanto, embora 1 `Broker` possa ter N `Topics`, não necessariamente este terá os mesmos N `Partitions` consigo.\
As partições gerenciam sua fila através do `Offset` (índice). Identificador único para cada mensagem.\
Assim que o `Broker Controller` determina a alocação da `Partition` num `Broker`, temos 2 tipos de subcategorias possíveis: 
- `Partition Leader` fica como responsável por todas as operações de leitura e escrita para aquela partição.
- `Partition Follower` são replicadas existentes em diferentes `Brokers` para garantir segurança na persistência dos registros. 


## Publicadores e Consumidores
Ambos são aplicações clientes do Kafka.

Publicadores são `Producers` e escrevem mensagens. Por padrão, balanceiam entre as `Partitions` de cada `Topic` durante o envio. É possível escolher uma partição específica usando os metadados da ‘chave’ ou criar seu próprio particionador personalizado.\
Múltiplos `Producers` podem atuar em 1:N `Topics`.

Uma analogia bem simplória para ilustrar a arquitetura `Pub/Sub` do Kafka seria imaginarmos o seguinte: quando você copia um texto para a área de transferência (`Publish`), o conteúdo dele (`Event`) é armazenado no final de um bloco de texto (`Partition`) existente num diretório específico de transferência (`Topic`) criado dentro de um dos seus HDs (`Broker`) disponibilizado pelo Sistema Operacional do computador (`Cluster`).

Consumidores são `Consumers` e leem mensagens nas respectivas ordens cronológicas de cada `Partition` através do `Offset`.\
Múltiplos `Consumers` podem atuar em 1:N `Topics`. Porém, só pode haver 1 `Consumer` por aplicação.\
Qualquer `Topic` com ao menos 1 consumidor irá eleger um `Consumer Group`. O grupo irá garantir integridade e distribuição no consumo. Desta forma 1 consumidor pode atuar em N `Partitions`, mas 1 `Partition` só pode ter no máximo 1 consumidor.\
Caso algum membro do grupo pare de funcionar ou esteja ocioso por tempo demais, os demais irão se reorganizar para redistribuir as `Partitions` entre si.\
![image](https://github.com/user-attachments/assets/af922af4-6baa-4b59-a28e-287aac676bc7)

- Exemplo 1: Se temos 1 `Topic` com 4 `Partitions` e 5 consumidores, então 1 consumidor ficará ocioso, sem previsão de obter e processar dados.
- Exemplo 2: se temos 1 `Topic` com 4 `Partitions` e 3 consumidores, então 1 consumidor atuará em 2 `Partitions` ao mesmo tempo (não gera sobrecarga, mas está sub-otimizado).


## Kafka Features
#### Kafka Connect 
Assistente para configurar coleta automática da base de dados para as publicar em um servidor Kafka ou vice-versa.

#### Kafka Streams
Provê API para facilitar desenvolver fluxos de processamento para aplicações aplicações.



# Capítulo 02: Instalando Kafka
Este resumo abrange as páginas 19 a ?? do livro e tem como objetivo guiar as etapas para instalar e configurar um servidor Kafka.\

## Sistema Operacional
Não há restrições para nenhum Sistema Operacional, mas existe uma melhor performance em ambientes Linux.

## Requisitos
1. [Instalar Oracle JDK 8+](https://www.oracle.com/java)
2. [Instalar ZooKeeper 3.5+](https://oreil.ly/iMZjR)
3. [Instalar Apache Kafka](https://oreil.ly/xLopS)

#### Oracle JDK 8+
Kafka é uma aplicação Java e necessita de um JDK instalado no ambiente. Recomenda-se versão Java 8 ou superior.\
Caso não existam restrições impostas, sempre instalar a versão mais recente.

#### ZooKeeper
O Kafka usa do ZooKeeper como `Cluster`, armazenando metadados do ambiente e dos `Consumers`. Recomenda-se instalar a versão 3.5 ou superior.\
![image](https://github.com/user-attachments/assets/f7f2d207-f23f-4e3b-b12c-ee026eda8db2)

Assim que instalado, já existe exemplos de configuração padrão para se usar em `/usr/local/zookeeper/config/zoo_sample.cfg`.\
Mas o livro concede uma instalação com configurações próprias, tendo como base ZooKeeper 3.5.9 e JDK 11:
~~~bash
# Descompacta arquio zipado
tar -zxf apache-zookeeper-3.5.9-bin.tar.gz

# Move executável para diretório de configuração
mv apache-zookeeper-3.5.9-bin /usr/local/zookeeper

# Cria diretório de dados
mkdir -p /var/lib/zookeeper

# Cria arquivo de configuração 'zoo.cfg'
cp > /usr/local/zookeeper/conf/zoo.cfg << EOF
	tickTime=2000
	dataDir=/var/lib/zookeeper
	clientPort=2181
EOF

# Define atalho para executar Java
export JAVA_HOME=/usr/java/jdk-11.0.10

# Inicia ZooKeeper
/usr/local/zookeeper/bin/zkServer.sh start
~~~

##### Múltiplos ZooKeepers
Chamados de ZooKeeper Ensemble quando existe um grupo de servidores ZooKeepers (nós) operando em conjunto como um único serviço de coordenação distribuída.\
Para que uma operação de escrita seja confirmada, a maioria dos nós deve concordar com a operação (`quorum`). Esse cálculo usa algoritmos de consenso e consistência, onde a maioria simples define a conclusão. <b>Portanto, mantenha os nós sempre em número ímpar</b>.

Para configurar um ZooKeeper Ensemble é preciso:
1. Que cada nó tenha seu próprio diretório com seu próprio `config/zoo.cfg`.
2. Que todas as configurações listem os mesmos servidores par que a comunicação funcione.
3. Que cada configuração tenha seu próprio `dataDir` e `clientPort`.
4. Que cada nó tenha um arquivo `data/myid` contendo dentro seu respectivo ID.


Exemplo de diretório com 3 nós:
~~~md
/usr/local/zookeeper/
	├── node1/
	│   ├── conf/zoo.cfg
	│   └── data/myid
	├── node2/
	│   ├── conf/zoo.cfg
	│   └── data/myid
	└── node3/
	    ├── conf/zoo.cfg
	    └── data/myid
~~~
	
Exemplo de arquivo `zoo.cfg` para node1:
~~~bash
# Unidade de tempo de 2000 milissegundos
tickTime=2000

# Repositório localizado em `/var/lib/zookeeper` (não pode repetir nos demais nós)
dataDir=/var/lib/zookeeper-1

# Porta de acesso dos clientes é `2181` (não pode repetir nos demais nós)
clientPort=2181

# Prazo limite de conexão entre `Partitions Followers` x `Partitions Leader` é de 40 segundos (20 x 2000 / 1000)
initLimit=20

# Prazo limite de dessincronia entre `Partitions Followers` x `Partitions Leader` é de 10 segundos (5 x 2000 / 1000)
syncLimit=5

# Servidor `ID 1` com hostname `zoo1.example.com`, porta de comunicação geral `2888` e a porta de comunicação do líder `3888`
server.1=zoo1.example.com:2888:3888	

# Servidor `ID 2` com hostname `zoo2.example.com`, porta de comunicação geral `2888` e a porta de comunicação do líder `3888`
server.2=zoo2.example.com:2888:3888

# Servidor `ID 3` com hostname `zoo3.example.com`, porta de comunicação geral `2888` e a porta de comunicação do líder `3888`
server.3=zoo3.example.com:2888:3888

# Limite de clientes conetados é de 60
maxClientCnxns=60
~~~

Exemplo de arquivo `myid` para node1:
~~~bash
1
~~~


#### Criando Kafka Broker
Recomendá-se usar a última versão estável, mas vale pena sempre conferir questões de compatibilidade com as respectivas versões Java e ZooKeeper previamente instaladas.\
Inicie o Kafka apenas após já ter algum Zookeper rodando e com acesso.

Exemplo de configuração:
~~~bash
# Descompata arquivo zipado 
tar -zxf kafka_2.13-2.7.0.tgz

# Instalando Apache Kafka no diretório alvo
mv kafka_2.13-2.7.0 /usr/local/kafka

# Cria diretório de logs dentro do novo diretório
mkdir /tmp/kafka-logs

# Define atalho para executar Java
export JAVA_HOME=/usr/java/jdk-11.0.10

# Inicia Apache Kafka com determinadas configurações (está já vem pronto para uso) 
/usr/local/kafka/bin/kafka-server-start.sh -daemon /usr/local/kafka/config/server.properties
~~~

#### Testando Kafka Broker
Uma vez que o Kafka estiver rodando, podemos testar algumas operações básicas.

Criando tópico de teste:
~~~bash
# Criando tópico no Broker `localhost:9092` sob o nome `test`
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --replication-factor 1 --partitions 1 --topic test

# Exibindo tópico alvo
/usr/local/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --describe --topic test
~~~

Publicando mensagens:
~~~bash
# Publicando 2 mensagens no Broker `localhost:9092` para o tópico `test`
/usr/local/kafka/bin/kafka-console-producer.sh --bootstrap-server localhost:9092 --topic test
Test Message 1
Test Message 2
# Use o comando ` Ctrl+C` para sair do `Producer` a qualquer momento
~~~

Consumindo mensagens:
~~~bash
# Consome as mensagens, desde o início, no Broker `localhost:9092` para o tópico `test`
/usr/local/kafka/bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
~~~


