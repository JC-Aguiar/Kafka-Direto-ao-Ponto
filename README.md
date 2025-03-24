![image](https://github.com/user-attachments/assets/6bcb78a8-4b3d-4daa-86d1-109a852438f9)


# Kafka - Direto ao Ponto
Guia resumido criado por João Costal Aguiar à partir do livro: <i>"Kafka - the Definitive Guide 2º Edition"</i>,  escrito por Gwen Shapira, Todd Palino, Rajini Sivaram Krit Petty. Essa apresentação destaca os principais pontos dos tópicos mais relevantes nos capítulos deste  material disponibilizado pela Confluent, atual detentora da tecnologia.


# Conhecendo Kafka
Este resumo abrange as páginas 1 a 17 do livro e tem como objetivo fazer uma abordagem geral dos principais elementos da tecnologia.\
Outros capítulos irão se aprofundar e explorar em mais detalhes cada um dos pontos descritos à seguir. 

## Conceito
Kafka pode ser definido como `Distributing Streaming Platform` (plataforma de transmissão distribuída), criado para resolver problemas de  arquitetura onde um servidor central de mensageria em fila age como intermediário entre aplicações. Originalmente foi criado para atender uma demanda do LinkedIn para emitir informações do que os usuários faziam no website.

Kafka usa o padrão `Pub/Sub`, que se caracteriza por enviar pacotes de dados para um `Topic` ao invés de um destinatário em especial, categorizando as informações de forma a permitir que diferentes aplicações e servidores se inscrevam e recebam somente os dados do qual possuem assinatura.\
Os eventos são registros imutáveis, similares a logs anexos no final de num arquivo físico. Eles são compostos por:
- `timestamp`: o momento em que a mensagem foi escrita no `Broker`
- `header`: contendo metadados do evento. 
- `key`: valor opcional para definir para qual `Partition` do `Topic` a mensagem será escrita (compactado como `binary array`).
- `value`: conteúdo da mensagem (compactado como `binary array`).

É recomendado usar o padrão `Protobuf` (Protocol Buffers) no conteúdo das mensagens para diminuir seu tamanho no `Broker`.\
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
As partições gerenciam sua fila através do `Offset`. Identificador único para cada mensagem. Vale reforçar que `Offser` é similar a um índice, mas sem acesso retroativo.\
Assim que o `Broker Controller` determina a alocação da `Partition` num `Broker`, temos 2 tipos de subcategorias possíveis: 
- `Partition Leader` fica como responsável por todas as operações de leitura e escrita para aquela partição.
- `Partition Follower` são replicadas existentes em diferentes `Brokers` para garantir segurança na persistência dos registros.

Consumo e publicação ocorrem apenas no `Partition Leader`. Caso um `Broker` venha a falhar, suas lideranças serão trocadas por uma das réplicas do tipo `Partition Follower`.\
As `Partitions`, por sua vez, também são divididos por um grupo de arquivos físicos chamados de `Segments`. Lá é onde ocorre o armazenamento das mensagens de fato. Cada `Segment` é nomeado com base no primeiro `Offset` a ser registrado nele (exemplo: `00000000000000000000.log`).\
Enquanto aberto, registram os `Events`. Após seu limite atingido, permanecem fechados até que os mais antigos sejam excluídos.


## Publicadores e Consumidores
Ambos são aplicações clientes do Kafka.

Publicadores são `Producers` e escrevem mensagens. Por padrão, balanceiam entre as `Partitions` de cada `Topic` durante o envio. É possível escolher uma partição específica usando os metadados da ‘chave’ ou criar seu próprio particionador personalizado.\
Múltiplos `Producers` podem atuar em 1:N `Topics`.

Uma analogia bem simplória para ilustrar a arquitetura `Pub/Sub` do Kafka seria imaginarmos o seguinte: quando você copia um texto para a área de transferência (`Publish`), o conteúdo dele (`Event`) é armazenado no final de um bloco de texto (`Partition`) existente num diretório específico de transferência (`Topic`) criado dentro de um dos seus HDs (`Broker`) disponibilizado pelo Sistema Operacional do computador (`Cluster`).

Consumidores são `Consumers` e leem mensagens nas respectivas ordens cronológicas de cada `Partition` através do `Offset`.\
Múltiplos `Consumers` podem atuar em 1:N `Topics`. Porém, só pode haver 1 `Consumer` por aplicação.\
Qualquer `Topic` com ao menos 1 consumidor irá eleger um `Consumer Group`. O grupo irá garantir integridade e distribuição no consumo. Desta forma 1 consumidor pode atuar em N `Partitions`, mas 1 `Partition` só pode ter no máximo 1 consumidor.\
Estes `Consumers Groups` são identificados com base no `group.id`. Permitindo que diferentes grupos de aplicações consumam a mesma mensagem em paralelo.
Caso algum membro do grupo pare de funcionar ou esteja ocioso por tempo demais, os demais membros irão se reorganizar para redistribuir as `Partitions` entre si.\
![image](https://github.com/user-attachments/assets/af922af4-6baa-4b59-a28e-287aac676bc7)

Exemplo 1:\
Se temos 1 `Topic` com 4 `Partitions` e 5 consumidores, então 1 consumidor ficará ocioso, sem previsão de obter e processar dados.

Exemplo 2:\
Se temos 1 `Topic` com 4 `Partitions` e 3 consumidores, então 1 consumidor atuará em 2 `Partitions` ao mesmo tempo (não gera sobrecarga, mas está sub-otimizado).


## Kafka Features
#### Kafka Connect 
Assistente para configurar coleta automática da base de dados para as publicar em um servidor Kafka ou vice-versa.

#### Kafka Streams
Provê API para facilitar desenvolver fluxos de processamento para aplicações aplicações.



# Instalando Kafka
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
O Kafka usa do ZooKeeper como `Cluster`, armazenando metadados do ambiente e dos `Brokers`. Recomenda-se instalar a versão 3.5 ou superior.\
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


# Criando Servidor Kafka
Este resumo abrange as páginas 19 a ?? do livro e tem como objetivo guiar as etapas para instalar e configurar um servidor Kafka.

Existe a feature `KRaft Mode` à partir do Kafka `version 3.3`, que permite independência completa do ZooKeeper. Além de ser mais performático, é recomendado para projetos que não envolvam sistemas Kafka legado. 

## Principais Requisitos de Hardware
- Velocidade de Transferência para o disco.
- Capacidade do disco.
- Velocidade da conexão de internet.
- Tamanho da memória disponível.
- Pode de CPU.

Para decisões técnicas de gerenciamento em serviços Cloud, consultar o livro `Kafka on Cloud` nas páginas 35 e 36. 

## Parâmetros do Servidor
As configurações descritas abaixo são os principais parâmetros a declarar no arquivo `server.properties`.\
Mas a lista completa pode ser encontrado na sessão [Broker Config](https://kafka.apache.org/documentation.html#brokerconfigs) na Documentação Oficial Apache Kafka.

`broker.id`
- Descrição: Identificador único (numérico) do servidor.
- Exemplo: para `host1.example.com` podemos usar ID `1`.

`zookeeper.connect`
- Descrição: Apontamento ao ZooKeeper no padrão `hostname:port/path`, onde `/path` é um caminho opcional, e recomendado, para o `chroot` (segmentação interna para cada aplicação `Cluster` usuária do ZooKeeper).
- Exemplo: `localhost:2181/kafka`.

`log.dirs`
- Descrição: Apontamento para mútliplos diretórios de log, separados por vírgula (`,`).
- Default: valor definido em `log.dir`.
- Exemplo: `/var/lib/kafka-logs1,/var/lib/kafka-logs2,/var/lib/kafka-logs3` (3 diretórios).

`num.recovery.threads.per.data.dir`
- Descrição: Configura o número de threads que serão usadas, para cada `log.dirs`, na recuperação de logs dos `Brokers` durante sua inicialização, encerramento ou recuperação pós-falha.
- Default: `1`.
- Exemplo: `3` (mas se temos 3 `log.dirs`, então são 9 threads no total).

`auto.create.topics.enable`
- Descrição: Se deve ou não criar tópicos automaticamente quando algum `Producer` ou `Consumer` solicitar publciação/consumo, ou se algum cliente requisitou metadados desse tópico.
- Default: `true`.

`auto.leader.rebalance.enable`
- Descrição: Para se certificar que nunca um `Cluster` fique desbalanceado por ter todos os tópicos num único `Broker`, essa configuração gera uma thread adicional de gerenciamento, com acionamento em intervalos definidos em `leader.imbalance.check.interval.seconds` e somente quando algum `Broker` exceder o limite de porcentagem de liderança declarado em `leader.imbalance.per.broker.percentage`.
- Defaults: 
	- Rebalanceamento = `true`.
	- Intervalo = `300` (5 minutos).
	- Porcentagem = `10` (%).

- `delete.topic.enable`
- Descrição: Ativa ou desabilita remoção de tópicos.
- Default: `false`.


## Parâmetros dos Tópicos
Aqui consta listados os principais parâmetros. Mas a lista completa pode ser encontrado na sessão [Topic Config](https://kafka.apache.org/documentation.html#topicconfigs) na Documentação Oficial Apache Kafka.

`num.partitions`
- Descrição: Número de partições quando um tópico é criado (geralmente quando tópico é gerado automaticamente). Uma vez definido, o número só pode ser incrementado e "nunca" reduzido. É uma boa prática definir o número de partições como múltiplo de `Brokers` disponíveis, para distribuir a liderança no `Cluster` (para considerações mais técnicas, consulte a página 28 do livro).
- Default: `1`.

`default.replication.factor`
- Descrição: Se a criação de tópicos automáticos está ativada, aqui se define o número das partições que serão criadas. O valor mínimo é `1`, indicando criação somente da `Parition Leader`. Acima disso, indicamos que serão geradas `N-1` `Partition Follower`. Importante: a real quantidade de partições criadas no Cluster será `num.partitions * default.replication.factor`, onde mais partições proporcionam maior tolerância a perda de dados. Recomendamos que seu valor mínimo seja `min.insync.replicas +1`.
- Default: `1`.

`log.retention.${unit}`
- Descrição: A duração em que o `Segment` mais antigo permanecerá armazenado no `Broker` é através do prazo limite ou de um tamanho máximo. Aqui definimos seu prazo limite, onde `${unit}` pode ser `hours`, `minutes` ou `ms` (menor unidade tem preferência em caso de múltiplas definições).
- Default: `168 hours` (7 dias).

`log.retention.bytes`
- Descrição: A duração em que o `Segment` mais antigo permanecerá armazenado no `Broker` é através do prazo limite ou de um tamanho máximo. Aqui definimos seu tamanho máximo, sendo que esse limite é por `Partition`. Portanto, 1Gb x 8 `Partitions` resultará em 8Gb.
- Default: `-1` (desabilitado).

`log.roll.ms`
- Descrição: A duração em que o `Segment` permanecerá aberto para receber `Events` é definido tanto por prazo limite quanto por tamanho máximo. Aqui definimos seu prazo limite. Evite que múltiplos `Segments` sejam fechados no mesmo momento usando também limite de capacidade.  
- Default: `null`.

`log.segment.bytes`
- Descrição: A duração em que o `Segment` permanecerá aberto para receber `Events` é definido tanto por prazo limite quanto por tamanho máximo. Aqui definimos seu tamanho máximo. 
- Default: `1Gb`.

`min.insync.replicas`
- Descrição: Define o número de réplicas (`Partition Follower`) quando um `Producer` publica um `Event`. Aumentar esse valor garante maior integridade nos dados, mas causa sobrecarga extra. Se um `Event` não for replicado acima do valor desse parâmetro, será rejeitado.
- Default: `1`.
- Valor recomendado: `2`.

`message.max.bytes`
- Descrição: Define o tamanho limite que um `Event` (após compactação) pode ser publicado pelo `Producer`. Alteração desse valor gera significativa alteração na performance do Kafka, pois impacta no gerenciamento entre as threads e o I/O do armazenamento.\
Atenção ao definir `fetch.message.max.bytes` (nos `Consumers`) abaixo desse parâmetro, pois pode gerar travamento permanente no consumo.    
- Default: `1000000` (1 Mb).



# Configurando Cluster Kafka
Páginas 36 até página 46.

## Considerações Bácias
É recomendado configurar mais de um `Broker` no seu `Cluster` Kafka a fim de melhorar e escalabilidade e prever perda de mensagens.

(imagem 2-2)

Mas para chegar no número exato, é importante identificar o peso total máximo das mensagens num `Cluster` contendo um único `Broker`. Com isso, uma opção simples seria dividir esse valor pela capacidade de armazenamento disponível.\
Vale salientar que algumas propriedades, sendo alteradas, escalonam com maior dramaticidade o cálculo e precisam ser levadas em considerações:
- `num.partitions`.
- `default.replication.factor`. 
- `log.retention.${unit}`.
- `log.retention.bytes`.

Lembrando que, para otimização do Kafka, seu ambiente deve ser capaz de tratar todas as `N` requisições. Onde `N` é o número total de `Partitions` nos seus `Clusters`.\
Portanto, se a conexão network não consegue dar conta desse número, procure um jeito de melhorar a conectividade ou diminuir as configurações dos `Brokers`.\
E mesmo que consiga tratar todas as conexões em seu limite, esse cenário pode causar estresse muito grande na CPU. Portanto, monitore sua saúde com base na quantidade única de clientes e `Consumers Groups`.\
Por fim, mesmo um ambiente muito bem configurado recomendamos não ultrapassar:
1. 14 mil `Partitions` por `Broker`.
2. 1 milhão de `Partitions` por `Cluster`.

### Exemplo de Cenário
Se o armazenamento é de 10TB e um único `Broker` com 10 tópicos guardou 1TB em média, então nosso `Cluster` pode ter até 10 `Brokers` desde que as configurações dos tópicos não sejam alteradas para valores maiores.


## Performance
Para melhorias na performance relacionadas ao Sistema Operacional, Garbage Collector e demais questões de infraestrutura, consulte as várias dicas expostas no livro dentro do tópico `Broker Configuration`. Indo da página 38 à página 46. 



# Kafka Producers
Páginas 47 à ...

## Configurações
Abaixo explicamos os principais parâmetros na configuração para clientes Kafka atuarem como `Producers`. 
A lista completa pode ser encontrado na sessão [Producers Config](https://kafka.apache.org/documentation.html#producerconfigs) na Documentação Oficial Apache Kafka.

`bootstrap.servers`
- Descrição: Lista dos apontamentos aos servidores Kafka no padrão `host:port`. Não precisa listar todos os `Brokers` de um `Cluster`, mas é recomendado listar ao menos dois.

`key.serializer`
- Descrição: Nome da classe Java que irá serializar a chave da mensagem (`key`). Este campo é obrigatório e caso não queira enviar nada, usar `VoidSerializer`.

`value.serializer`
- Descrição: Nome da classe Java que irá serializar o conteúdo da mensagem (`key`).

`buffer.memory`
- Descrição: Total de bytes da memória que o `Producer` pode armazenar num `Buffer` enquanto preparar as mensagens. Se as mensagens forem geradas mais rápido do que podem de fato ser enviadas ao servidor (`Broker`), o serviço será temporariamente bloqueado e uma exceção será lançada.
- Default:	`33554432` (ms).

`max.block.ms`
- Descrição: Tempo de boqueio que o `Producer` ficará aguardando.

`compression.type`
- Descrição: Modo de compactação das mensagens. As opções são: `none`, `gzip`, `snappy`, `lz4` ou `zstd`.
- Default: `none`.

`retries`
- Descrição: Uma das formas de realizar novas tentativas de publicação caso receba uma mensagem de erro durante. Recomendamos usar `delivery.timeout.ms` ao invés desse parâmetro para controlar o comportamento de re-tentativas. Defina um valor aqui maior do que `0` para ativar a re-tentativa. Se  `enable.idempotence` está desativado e `max.in.flight.requests.per.connection` possui valor acima de `1`, o `Producer` pode potencialmente mudar a ordenação das mensagens durante o envio. Exemplo: falha num lote durante o envio para a mesma partição de outro lote que conseguiu realizar a publicação.
- Default: `2147483647`.

`ssl.keystore.key`
- Descrição: Chave provada no formato definido em `ssl.keystore.type`. Mecanismo padrão SSL suporta apenas formatos `PEM` com `PKCS#8` chaves. Se a chave está encriptada, obrigatório definir a senha no `ssl.key.password`.

`ssl.key.password`
- Descrição: Senha da chave privada caso a chave de `ssl.keystore.key` esteja encriptada.

`ssl.keystore.certificate.chain`
- Descrição: Cadeia de certificados no formato especificado em `ssl.keystore.type`. Mecanismo padrão SSL suporta apenas formatos `PEM` com uma lista de certificados `X.509`.

`ssl.keystore.location`
- Descrição: Apontamento do local onde está o arquivo de certificado.

`ssl.keystore.password`
- Descrição: Senha para a chave do arquivo de certificado. Opcional. Somente se `ssl.keystore.location` fi configurado. Não aceita formato `PEM`.

`ssl.truststore.certificates`
- Descrição: Certificados confiáveis no formato definido em `ssl.truststore.type`. Mecanismo padrão SSL suporta apenas formatos `PEM` com certificados `X.509`.

`ssl.keystore.type`
- Descrição: O formato da chave do certificado. Os valores suportados por padrão em  `ssl.engine.factory.class` são: `JKS`, `PKCS12`, `PEM`.
- Default: `JKS`.

`ssl.truststore.type`
- Descrição: O formato do arquivo trust-store. Os valores suportados por padrão em  `ssl.engine.factory.class` são: `JKS`, `PKCS12`, `PEM`.
- Default: `JKS`.

`ssl.truststore.location`
- Descrição: Apontamento do local onde está o arquivo trust-store.

`ssl.truststore.password`
- Descrição: Senha do arquivo trust-store. Se uma senha não foi definida, o arquivo ainda será usado, mas será desabilitado. Esta senha não suporta formato for `PEM`.


## Formas de Publicar Mensagem
`Fire-and-forget`\
Envie mensagens sem se importar se chegaram com sucesso ou não. Na maioria das vezes, ela chegará com sucesso, já que o Kafka é altamente disponível.

`Synchronous send`\
Tecnicamente, o Kafka `Producer` é sempre assíncrono. Mas é possível torná-lo síncrono. O método `send()` retorna um objeto `Future` e se usarmos o método ` get()` à partir desse `Future`, iremos esperar para confirmar se o envio foi bem-sucedido ou não (antes de enviar o próximo registro).\
Permite capturar exceções quando o Kafka responde com um erro ou por time-out. Mas temos perda de desempenho, pois dependendo do quão ocupado o `Cluster` do Kafka está, os `Brokers` podem levar de 2 milissegundos a alguns segundos para responder às solicitações do `Producer`. Neste cenário, a thread de envio passará esse tempo esperando e não fazendo mais nada, nem mesmo enviando mensagens adicionais.

`Asynchronous send`\
Chamamos o método `send()` com uma função de retorno de chamada, que é acionada quando recebe uma resposta do `Broker` Kafka.



