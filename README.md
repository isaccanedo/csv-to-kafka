# csv-to-kafka

Por qualquer motivo, o CSV ainda existe como um formato de intercâmbio de dados onipresente. Não fica muito mais simples: coloque algum texto simples com campos separados por vírgulas em um arquivo e coloque .csv no final. Se você se sentir útil, pode incluir uma linha de cabeçalho com os nomes dos campos.

```
order_id,customer_id,order_total_usd,make,model,delivery_city,delivery_company,delivery_address
1,535,190899.73,Dodge,Ram Wagon B350,Sheffield,DuBuque LLC,2810 Northland Avenue
2,671,33245.53,Volkswagen,Cabriolet,Edinburgh,Bechtelar-VonRueden,1 Macpherson Crossing
```

Neste artigo veremos como carregar esses dados CSV no Kafka, sem precisar escrever nenhum código

É importante ressaltar que não vamos reinventar a roda tentando escrever algum código para fazer isso sozinhos - o Kafka Connect (que faz parte do Apache Kafka) já existe para fazer tudo isso para nós; só precisamos do conector apropriado.

Esquemas?
Sim, esquemas. Os arquivos CSV podem não se importar muito com eles, mas os usuários de seus dados no Kafka sim. Idealmente, queremos uma maneira de definir o esquema dos dados que ingerimos para que possam ser armazenados e lidos por qualquer pessoa que queira usar os dados. Para entender por que isso é tão importante, confira:

```
- Microsserviços de Streaming: Contratos e Compatibilidade (InfoQ talk);
- Sim, Virginia, você realmente precisa de um registro de esquema (blog);
- Esquemas, Contratos e Compatibilidade (blog);
- Plataforma Confluent agora suporta Protobuf, JSON Schema e formatos personalizados (blog).
```

Se você for definir um esquema na ingestão (e espero que o faça), use Avro, Protobuf ou JSON Schema.

**Observação**
Você não precisa usar um esquema. Você pode simplesmente ingerir os dados CSV como estão, e eu abordo isso abaixo também

### Conector Kafka Connect SpoolDir
O conector Kafka Connect SpoolDir oferece suporte a vários formatos de arquivo simples, incluindo CSV. Obtenha-o no Confluent Hub. Depois de instalá-lo em seu trabalhador do Kafka Connect, certifique-se de reiniciá-lo para que ele possa buscá-lo. Você pode verificar executando:

```
$ curl -s localhost:8083/connector-plugins|jq '.[].class'|egrep 'SpoolDir'

"com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector"
"com.github.jcustenborder.kafka.connect.spooldir.SpoolDirJsonSourceConnector"
"com.github.jcustenborder.kafka.connect.spooldir.SpoolDirLineDelimitedSourceConnector"
"com.github.jcustenborder.kafka.connect.spooldir.SpoolDirSchemaLessJsonSourceConnector"
"com.github.jcustenborder.kafka.connect.spooldir.elf.SpoolDirELFSourceConnector"
```

### Carregando dados do CSV no Kafka e aplicando um esquema
Se você tiver uma linha de cabeçalho com nomes de campo, poderá aproveitá-los para definir o esquema no momento da ingestão (o que é uma boa ideia).

Crie o conector:

```
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/source-csv-spooldir-00/config \
    -d '{
        "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
        "topic": "orders_spooldir_00",
        "input.path": "/data/unprocessed",
        "finished.path": "/data/processed",
        "error.path": "/data/error",
        "input.file.pattern": ".*\\.csv",
        "schema.generation.enabled":"true",
        "csv.first.row.as.header":"true"
        }'
```

**Observação**
quando você cria o conector com esta configuração, você precisa executá-lo com "csv.first.row.as.header":"true" e um arquivo com cabeçalhos já colocados pendentes para serem lidos.

Agora vá até um consumidor Kafka e observe nossos dados. Aqui estou usando kafkacat porque é ótimo :)

```
$ docker exec kafkacat \
    kafkacat -b kafka:29092 -t orders_spooldir_00 \
             -C -o-1 -J \
             -s key=s -s value=avro -r http://schema-registry:8081 | \
             jq '.payload'
{
  "order_id": {
    "string": "500"
  },
  "customer_id": {
    "string": "424"
  },
  "order_total_usd": {
    "string": "160312.42"
  },
  "make": {
    "string": "Chevrolet"
  },
  "model": {
    "string": "Suburban 1500"
  },
  "delivery_city": {
    "string": "London"
  },
  "delivery_company": {
    "string": "Predovic LLC"
  },
  "delivery_address": {
    "string": "2 Sundown Drive"
  }
}
```

Além disso, no cabeçalho da mensagem Kafka estão os metadados do próprio arquivo:

```
$ docker exec kafkacat \
    kafkacat -b kafka:29092 -t orders_spooldir_00 \
             -C -o-1 -J \
             -s key=s -s value=avro -r http://schema-registry:8081 | \
             jq '.headers'
[
  "file.name",
  "orders.csv",
  "file.path",
  "/data/unprocessed/orders.csv",
  "file.length",
  "39102",
  "file.offset",
  "501",
  "file.last.modified",
  "2020-06-17T13:33:50.000Z"
]
```

### Configurando a chave de mensagem
Supondo que você tenha uma linha de cabeçalho para fornecer nomes de campo, você pode definir schema.generation.key.fields para o nome do(s) campo(s) que gostaria de usar para a chave de mensagem Kafka. Se você estiver executando isso após o primeiro exemplo acima, lembre-se de que o conector realoca seu arquivo, então você precisa movê-lo de volta para o local input.path para que ele seja processado novamente.

**Observação**
O nome do conector (aqui é source-csv-spooldir-01) é usado para rastrear quais arquivos foram processados e o deslocamento dentro deles, portanto, um conector com o mesmo nome não reprocessará um arquivo com o mesmo nome e deslocamento inferior já processado. Se você quiser forçá-lo a reprocessar um arquivo, dê um novo nome ao conector.

```
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/source-csv-spooldir-01/config \
    -d '{
        "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
        "topic": "orders_spooldir_01",
        "input.path": "/data/unprocessed",
        "finished.path": "/data/processed",
        "error.path": "/data/error",
        "input.file.pattern": ".*\\.csv",
        "schema.generation.enabled":"true",
        "schema.generation.key.fields":"order_id",
        "csv.first.row.as.header":"true"
        }'
```

A mensagem Kafka resultante tem o order_id definido como a chave da mensagem:

```
docker exec kafkacat \
    kafkacat -b kafka:29092 -t orders_spooldir_01 -o-1 \
             -C -J \
             -s key=s -s value=avro -r http://schema-registry:8081 | \
             jq '{"key":.key,"payload": .payload}'
{
  "key": "Struct{order_id=3}",
  "payload": {
    "order_id": {
      "string": "3"
    },
    "customer_id": {
      "string": "695"
    },
    "order_total_usd": {
      "string": "155664.90"
    },
    "make": {
      "string": "Toyota"
    },
    "model": {
      "string": "Avalon"
    },
    "delivery_city": {
      "string": "Brighton"
    },
    "delivery_company": {
      "string": "Jacobs, Ebert and Dooley"
    },
    "delivery_address": {
      "string": "4 Loomis Crossing"
    }
  }
}
```

### Alterando os tipos de campo do esquema
O conector faz um bom trabalho ao definir o esquema, mas talvez você queira substituí-lo. Você pode declarar tudo de antemão usando a configuração value.schema, mas talvez esteja satisfeito com ela inferindo todo o esquema, exceto por alguns campos. Aqui você pode usar o Single Message Transform para munge-lo:

```
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/source-csv-spooldir-02/config \
    -d '{
        "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirCsvSourceConnector",
        "topic": "orders_spooldir_02",
        "input.path": "/data/unprocessed",
        "finished.path": "/data/processed",
        "error.path": "/data/error",
        "input.file.pattern": ".*\\.csv",
        "schema.generation.enabled":"true",
        "schema.generation.key.fields":"order_id",
        "csv.first.row.as.header":"true",
        "transforms":"castTypes",
        "transforms.castTypes.type":"org.apache.kafka.connect.transforms.Cast$Value",
        "transforms.castTypes.spec":"order_id:int32,customer_id:int32,order_total_usd:float32"
        }'
```

Se você examinar o esquema que foi criado e armazenado no Registro de Esquema, poderá ver que os tipos de dados de campo foram definidos conforme especificado:

```
➜ curl --silent --location --request GET 'http://localhost:8081/subjects/orders_spooldir_02-value/versions/latest' |jq '.schema|fromjson'
{
  "type": "record", "name": "Value", "namespace": "com.github.jcustenborder.kafka.connect.model",
  "fields": [
    { "name": "order_id", "type": [ "null", "int" ], "default": null },
    { "name": "customer_id", "type": [ "null", "int" ], "default": null },
    { "name": "order_total_usd", "type": [ "null", "float" ], "default": null },
    { "name": "make", "type": [ "null", "string" ], "default": null },
    { "name": "model", "type": [ "null", "string" ], "default": null },
    { "name": "delivery_city", "type": [ "null", "string" ], "default": null },
    { "name": "delivery_company", "type": [ "null", "string" ], "default": null },
    { "name": "delivery_address", "type": [ "null", "string" ], "default": null }
  ],
  "connect.name": "com.github.jcustenborder.kafka.connect.model.Value"
}
```

Apenas gimme o texto simples! 😢
Todos esses esquemas parecem um monte de barulho, não é? Bem, na verdade não. Mas, se você absolutamente deve ter apenas CSV em seu tópico Kafka, veja como. Observe que estamos usando uma classe de conector diferente e estamos usando org.apache.kafka.connect.storage.StringConverter para escrever os valores. 

```
curl -i -X PUT -H "Accept:application/json" \
    -H  "Content-Type:application/json" http://localhost:8083/connectors/source-csv-spooldir-03/config \
    -d '{
        "connector.class": "com.github.jcustenborder.kafka.connect.spooldir.SpoolDirLineDelimitedSourceConnector",
        "value.converter":"org.apache.kafka.connect.storage.StringConverter",
        "topic": "orders_spooldir_03",
        "input.path": "/data/unprocessed",
        "finished.path": "/data/processed",
        "error.path": "/data/error",
        "input.file.pattern": ".*\\.csv"
        }'
```

O resultado? Apenas CSV.

```
➜ docker exec kafkacat \
    kafkacat -b kafka:29092 -t orders_spooldir_03 -o-5 -C -u -q
496,456,80466.80,Volkswagen,Touareg,Leeds,Hilpert-Williamson,96 Stang Junction
497,210,57743.67,Dodge,Neon,London,Christiansen Group,7442 Algoma Hill
498,88,211171.02,Nissan,370Z,York,"King, Yundt and Skiles",3 1st Plaza
499,343,126072.73,Chevrolet,Camaro,Sheffield,"Schiller, Ankunding and Schumm",8920 Hoffman Place
500,424,160312.42,Chevrolet,Suburban 1500,London,Predovic LLC,2 Sundown Drive
```

### Barra lateral: Esquemas em ação
Então, lemos alguns dados CSV no Kafka. Esse não é o fim de sua jornada. Vai ser usado para alguma coisa! Vamos fazer isso.

Aqui está o ksqlDB, no qual declaramos o tópico de pedidos no qual escrevemos com um esquema como um fluxo:

```
ksql> CREATE STREAM ORDERS_02 WITH (KAFKA_TOPIC='orders_spooldir_02',VALUE_FORMAT='AVRO');

 Message
----------------
 Stream created
----------------
```

Feito isso - e porque existe um esquema que foi criado no momento da ingestão - podemos ver todos os campos disponíveis para nós:

```
ksql> DESCRIBE ORDERS_02;

Name                 : ORDERS_02
 Field            | Type
-------------------------------------------
 ROWKEY           | VARCHAR(STRING)  (key)
 ORDER_ID         | INTEGER
 CUSTOMER_ID      | INTEGER
 ORDER_TOTAL_USD  | DOUBLE
 MAKE             | VARCHAR(STRING)
 MODEL            | VARCHAR(STRING)
 DELIVERY_CITY    | VARCHAR(STRING)
 DELIVERY_COMPANY | VARCHAR(STRING)
 DELIVERY_ADDRESS | VARCHAR(STRING)
-------------------------------------------
For runtime statistics and query details run: DESCRIBE EXTENDED <Stream,Table>;
ksql>
```

e execute consultas nos dados que estão no Kafka:

```
ksql> SELECT DELIVERY_CITY, COUNT(*) AS ORDER_COUNT, MAX(CAST(ORDER_TOTAL_USD AS DECIMAL(9,2))) AS BIGGEST_ORDER_USD FROM ORDERS_02 GROUP BY DELIVERY_CITY EMIT CHANGES;
+---------------+-------------+---------------------+
|DELIVERY_CITY  |ORDER_COUNT  |BIGGEST_ORDER_USD    |
+---------------+-------------+---------------------+
|Bradford       |13           |189924.47            |
|Edinburgh      |13           |199502.66            |
|Bristol        |16           |213830.34            |
|Sheffield      |74           |216233.98            |
|London         |160          |219736.06            |
```

E quanto aos nossos dados que acabamos de ingerir em um tópico diferente como CSV direto? Porque, tipo, esquemas não são importantes?

```
ksql> CREATE STREAM ORDERS_03 WITH (KAFKA_TOPIC='orders_spooldir_03',VALUE_FORMAT='DELIMITED');
No columns supplied.
```

Sim, sem colunas fornecidas. Sem esquema, sem bueno. Se você quiser trabalhar com os dados, seja para consultar em SQL, transmitir para um data lake ou fazer qualquer outra coisa, em algum momento você terá que declarar esse esquema. Por isso, o CSV, como método de serialização sem esquema, é uma maneira ruim de trocar dados entre sistemas.

Se você realmente deseja usar seus dados CSV no ksqlDB, você pode, basta inserir o esquema - que é propenso a erros e tedioso. Você o insere toda vez para usar os dados, todos os outros consumidores dos dados também o inserem a cada vez. Declará-lo uma vez na ingestão e estar disponível para todos usarem faz muito mais sentido.
