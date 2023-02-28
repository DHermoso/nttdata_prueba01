# Kafka Conector mongodb

2021/06/01  v1.0: Primera versión

2021/06/25  v1.0: Activación de todos los conectores y pruebas de carga

2021/06/29  v1.0: Despliegue CI/CD



Se instalan los conectores desde https://www.confluent.io/hub/mongodb/kafka-connect-mongodb



The official MongoDB Kafka connector, providing both Sink and Source connectors.



## Caso de uso

Colecciones de mongodb Censos & Pacientes a consumir en cloud  por parte de la aplicación eHodei.



Streaming de collecciones mongodb mediante conectores kafka para lectura desde microservicio cloud.

```mermaid
graph LR;

Batera_Osaki_On_Prem(Mongo On Prem) -- MONGODB STREAM --> Google_Cloud_GKE( Google GKE);

    mongonprem(MongoDB OnPrem) --TLS--> source_conector( Kafka Source Conector );
    source_conector --TLS--> kafka_server( Kafka Server);
	kafka_server --TLS--> sink_conector( Kafka Sink Conector);
	sink_conector --TLS--> mongocloud(MongoDB Cloud);

style mongonprem fill:#ffcc00
style source_conector fill:#66ffff
style kafka_server fill:#66ffff
style sink_conector fill:#66ffff
style mongocloud fill:#66ffff

style Batera_Osaki_On_Prem fill:#ffcc00
style Google_Cloud_GKE fill:#66ffff


```

## Alcance



Sincronización  mongoDB OnPrem>Cloud de las collecciones mongodb anonimizadas recogidas [Gitlab](https://src.alm01.itbatera.euskadi.eus/hodei/j61-app-backend-mongodb/-/tree/master/colecciones):


- [x] Bloque.json
- [x] Cargas.json
- [x] CensoHospitalizacionDomicilio.json
- [x] CensosBloqueQuirurgico.json
- [x] CensosCCEE.json
- [x] CensosHospitalDiaMedico.json
- [x] CensosHospitalDiaQuirurgico.json
- [x] CensosHospitalizacion.json
- [x] CensosPreadmisiones.json
- [x] CensosUrgencias.json
- [x] CentroArea.json
- [x] CentroUnidad.json
- [x] Consultor.json
- [x] DietarioPrimariaMongo.json
- [x] Dietools.json
- [x] Evolutivos.json
- [x] Onkobide.json
- [x] Osi.json
- [x] Pacientes.json
- [x] Profesional.json
- [x] ProfesionalArea.json
- [x] Seccion.json
- [x] Servicio.json	


**Elementos en NOP/PRE**: 

- [x] Mongodb Hodei2 OnPrem diferente al actual solo con datos de pruebas 
- [x] Kafka Source connector ( conectado a mongodb OnPrem)
- [x] Kafka Cluster
- [x] Kafka Sink connector ( conectado a mongodb Cloud)
- [x] MongoDB cluster (3 réplicas)

**Datos requeridos**: 

- Acceso mongodb OnPrem Hodei2: A desplegar e importar  las colecciones de datos de pruebas validadas
- Activar acceso de lectura a mongodb OnPrem Hodei2  desde kubernetes NOP

### !!!Importante a  nivel de seguridad!!!

Es necesario avanzar en la foto de seguridad antes de sincronizar en cloud una colección del actual entorno PRE (No está anonimizado). Trabajamos solamente con la colección de datos de prueba.

**Escenario a probar en NOP:**

  1. Despliegue de mongodb OnPrem Hodeiv2
  2. Activar Kafka conectores  Source&Sink
  3. Importar datos de prueba
  4. Validar sincronización den Mongodb Cloud

(Prueba previa a la activación de sincronizaciones desde la app en Osakidetza PRE)

Después de validarlo desactivar la sincronización y entregar a la app como entorno de vallidación Hodei2. A la espera de securización nop con nivel productivo.


### Volumetría mongodb conector

#### Cambios de colecciones

Respondiendo a tus dudas:

1. *¿Tenéis volumetria de cambios esperados por día en las colecciones de mongodb OnPrem?, lo comento para tener una idea de dimensionamiento completa del contector y cluster kafka para la foto completa de mongodb streaming.*

Esto que nos pides es difícil de calcular. Pero vamos a ver si damos unas fórmulas aproximadas para que os hagáis una idea de los cambios.

·    La base de datos es un 80% datos de paciente y un 20% datos de censo.

·    Los pacientes de Osakidetza son 3.000.000 y al día se cambian los datos de unos 100.000, esto viene a ser un 3,3% del total.

·    Los censos se regeneran 2 veces al día.

 

Por tanto, si tuvieramos xGb de MongoDB en pro:

·    ((x * 80 * 3.3/ 10.000) = Gb de pacientes que se trasladan diariamente.

·    (x*20/100)x2 = Gb de censos que se trasladan diariamente.  

 

Esto no es para nada exacto pero sirva de aproximación para la toma de decisiones.



####  Carga de Ficheros binarios

Archivos pdf que se cargan en el mongodb OnPrem (pendiente de desplegar) y se sincronizan con el mongodb cloud a traves del conector kafka.





1. *¿Estos archivos van a ir en colecciones diferentes? Si es así, podemos plantear un tratamiento diferente a nivel de sincronizaciones.*

Estos ficheros van en colecciones diferentes. De hecho, estamos pensando en que los ficheros de diettools tenga su colección y los de Onkobide otra colección propia.



 la volumetría que se ha detectado para las alicaciones Diettools y Onkobide.

A tener en cuenta que:

·    Estos ficheros tienen una vida de 48 horas, a partir de las cuales se sustituyen por sus versiones actualizadas.

·    No hay históricos.

 

**Dietools – Ficheros generados diariamente.**

 

| **Centro**                                 | **Total ficheros** | **Total ficheros Unificación Araba** |
| ------------------------------------------ | ------------------ | ------------------------------------ |
| TXAGO                                      | 4                  | 4                                    |
| SANTIAGO                                   | 4                  |                                      |
| PSIQ. ALAVA                                | 4                  |                                      |
| BASURTO                                    | 4                  | 4                                    |
| BERMEO                                     | 4                  | 4                                    |
| BIDASOA                                    | 4                  | 4                                    |
| CRUCES                                     | 4                  | 4                                    |
| DONOSTI                                    | 4                  | 4                                    |
| EIBAR                                      | 4                  | 4                                    |
| GALDAKAO                                   | 4                  | 4                                    |
| GORLIZ                                     | 4                  | 4                                    |
| LEZA                                       | 4                  | 4                                    |
| MENDARO                                    | 4                  | 4                                    |
| MONDRAGON                                  | 4                  | 4                                    |
| ONKOLOGIKO                                 | 4                  | 4                                    |
| SAN ELOY                                   | 4                  | 4                                    |
| STA. MARINA                                | 4                  | 4                                    |
| URDULIZ                                    | 4                  | 4                                    |
| ZALDIBAR                                   | 4                  | 4                                    |
| ZAMUDIO                                    | 4                  | 4                                    |
| ZUMARRAGA                                  | 4                  | 4                                    |
| **Total ficheros por día**                 | **84**             | **76**                               |
| Tamaño en Kb                               | 200                | 200                                  |
| Espacio ocupado en Kb                      | 16800              | 15200                                |
| **Espacio ocupado en Mb**                  | **16,8 Mb**        | **15,2 Mb**                          |
| **Previsión espacio ocupado de 48h en Mb** | **33,6 Mb**        | **30,4 Mb**                          |

 

**Onkobide – Ficheros generados diariamente**

 

| **Centro**                                 | **Total ficheros** |
| ------------------------------------------ | ------------------ |
| TXAGORRITXU H.                             | 100                |
| DONOSTIA H. UNIVERSITARIO                  | 100                |
| ONKOLOGIKOA                                | 100                |
| H. GALDAKAO-USANSOLO                       | 100                |
| CRUCES H. UNIVERSITARIO                    | 100                |
| HOSPITAL UNIVERSITARIO BASURTO             | 100                |
| **Total ficheros**                         | **600**            |
| Tamaño fichero aproximación en Kb          | 450                |
| Toal espacio ocupado en Kb                 | 270000             |
| **Total espacio ocupado en Mb**            | **270 Mb**         |
| **Previsión espacio ocupado de 48h en Mb** | **540 Mb**         |

 Pongo en copia a backend para que lo tengan en cuenta, en principio no parece un volumen problemático y el tamaño individual de archivo no supera el limite de documento en mongodb (16MB).



Me surgen dos dudas:

1. ¿Estos archivos van a ir en colecciones diferentes? Si es así, podemos plantear un tratamiento diferente a nivel de sincronizaciones.
2. ¿Tenéis volumetria de cambios esperados por día en las colecciones de mongodb OnPrem?, lo comento para tener una idea de dimensionamiento completa del contector y cluster kafka para la foto completa de mongodb streaming.



## Sincronización de una coleccion mongodb

Para activar una sincronización se requiere el alta de los siguientes elementos:

1. Kafka Topic
2. Permisos conector
3. Conector Source
4. Conector Sink



### 1. kafka topic



Para poder trabajar con colecciones con mayúsculas, es necesario seguir la siguiente nota de [Red Hat](https://access.redhat.com/solutions/5664921).

[deploy@V01SRH-007557 gke01nop]$ cat kafka_topic_Pacientes.yaml

```
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaTopic
metadata:
  name: mongotls.j61.pacientes
  labels:
    strimzi.io/cluster: kafka01nop-j61
spec:
  partitions: 1
  replicas: 3
  topicName: mongotls.j61.Pacientes

```



### 2. Permisos conector

cat kafkauser_mongodb-connect-community.yaml



```

apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaUser
metadata:
  name: mongodb-connect-community
  labels:
    strimzi.io/cluster: kafka01nop-j61
spec:
  authentication:
    type: tls
  authorization:
    type: simple
    acls:
    - resource:
        type: group
        name: connect-cluster
      operation: Read
    - resource:
        type: topic
        name: connect-cluster-configs
      operation: Read
    - resource:
        type: topic
        name: connect-cluster-configs
      operation: Describe
    - resource:
        type: topic
        name: connect-cluster-configs
      operation: Write
    - resource:
        type: topic
        name: connect-cluster-configs
      operation: Create
    - resource:
        type: topic
        name: connect-cluster-status
      operation: Read
    - resource:
        type: topic
        name: connect-cluster-status
      operation: Describe
    - resource:
        type: topic
        name: connect-cluster-status
      operation: Write
    - resource:
        type: topic
        name: connect-cluster-status
      operation: Create
    - resource:
        type: topic
        name: connect-cluster-offsets
      operation: Read
    - resource:
        type: topic
        name: connect-cluster-offsets
      operation: Write
    - resource:
        type: topic
        name: connect-cluster-offsets
      operation: Describe
    - resource:
        type: topic
        name: connect-cluster-offsets
      operation: Create
    - resource:
        type: group
        name: connect-cluster
      operation: Read
    - resource:
        type: topic
        name: mongotls.test.ships
      operation: Read
    - resource:
        type: topic
        name: mongotls.test.ships
      operation: Describe
    - resource:
        type: topic
        name: mongotls.test.ships
      operation: Write
    - resource:
        type: topic
        name: mongotls.test.ships
      operation: Create
    - resource:
        type: topic
        name: mongotls.test.ships2
      operation: Read
    - resource:
        type: topic
        name: mongotls.test.ships2
      operation: Describe
    - resource:
        type: topic
        name: mongotls.test.ships2
      operation: Write
    - resource:
        type: topic
        name: mongotls.test.ships2
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.dietarioprimariamongo
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.dietarioprimariamongo
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.dietarioprimariamongo
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.dietarioprimariamongo
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.DietarioPrimariaMongo
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.DietarioPrimariaMongo
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.DietarioPrimariaMongo
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.DietarioPrimariaMongo
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.censoshospitalizacion
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.censoshospitalizacion
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.censoshospitalizacion
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.censoshospitalizacion
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.CensosHospitalizacion
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.CensosHospitalizacion
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.CensosHospitalizacion
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.CensosHospitalizacion
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.pacientes
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.pacientes
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.pacientes
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.pacientes
      operation: Create
    - resource:
        type: topic
        name: mongotls.j61.Pacientes
      operation: Read
    - resource:
        type: topic
        name: mongotls.j61.Pacientes
      operation: Describe
    - resource:
        type: topic
        name: mongotls.j61.Pacientes
      operation: Write
    - resource:
        type: topic
        name: mongotls.j61.Pacientes
      operation: Create
    - resource:
        type: group
        name: connect-mongodb-sink-connector-community
      operation: Read
    - resource:
        type: group
        name: connect-mongodb-sink-connector-community2
      operation: Read
    - resource:
        type: group
        name: connect-mongodb-sink-dietarioprimariamongo
      operation: Read
    - resource:
        type: group
        name: connect-mongodb-sink-censoshospitalizacion
      operation: Read
    - resource:
        type: group
        name: connect-mongodb-sink-pacientes
      operation: Read


```



### 2. Conector Source

Para cada colección, un conector source.

```
[deploy@V01SRH-007557 kafkaconnector]$ cat mongodb-source-pacientes.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: mongodb-source-pacientes
  labels:
    strimzi.io/cluster: mongodb-connect-community
spec:
  class: com.mongodb.kafka.connect.MongoSourceConnector
  tasksMax: 1
  config:
    connection.uri: mongodb://mongoadmin:xxxxxx@10.65.96.178:27017,10.65.96.179:27017,10.65.96.180:27017/admin?authSource=admin&replicaSet=Replica_PRE_J61_FaseII
    topic.prefix: mongotls
    database: j61
    collection: Pacientes
    copy.existing: true
    key.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter.schemas.enable: false
    publish.full.document.only: true
    pipeline: >
      [{"$match":{"operationType":{"$in":["insert","update","replace"]}}},{"$project":{"_id":1,"fullDocument":1,"ns":1,"documentKey":1}}]

```



### 3. Conector Sink



Para cada colección, un conector sink.

```
[deploy@V01SRH-007557 kafkaconnector]$ cat mongodb-sink-pacientes.yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaConnector
metadata:
  name: mongodb-sink-pacientes
  labels:
    strimzi.io/cluster: mongodb-connect-community
spec:
  class: com.mongodb.kafka.connect.MongoSinkConnector
  tasksMax: 2
  config:
    topics: mongotls.j61.Pacientes
    connection.uri: mongodb://mongoadmin:xxxxxxxxx@mongodb-gke01nop-rs0-svc.backend-sandbox.svc.cluster.local:27017/?replicaSet=mongodb-gke01nop-rs0
    database: j61
    collection: Pacientes
    post.processor.chain: com.mongodb.kafka.connect.sink.processor.DocumentIdAdder,com.mongodb.kafka.connect.sink.processor.KafkaMetaAdder
    key.converter: org.apache.kafka.connect.json.JsonConverter
    key.converter.schemas.enable: false
    value.converter: org.apache.kafka.connect.json.JsonConverter
    value.converter.schemas.enable: false

```

## Despliegue CI/CD


Despliegue de  los objetos que sincronizan KAFKA GKE con los MONGODB ON PREMISE y los MONGODB GKE mediante pipelines.

Se han generado tres yaml por cada colección que hay en la BD MONGODB ON PREMISE, un kafkatopic, un kafkaconnector source (De mongodb onpremise a KAFKA) y un kafkaconnector sink (De Kafka a MONGODB GKE)

En total hay 23 colecciones, así que tenia pensado subir 69 yamls.

Se separa de la parte de KAFKA básica para facilitar su gestión. Para  poder gestionar los despliegues es mejor separar  propio de la instancia de KAFKA y por otro lado tener otro GITLAB/PIPELINE de los kafkaconnectors MONGODB.

 

Ahora tenemos un gitlab para el despliegue de KAFKA:

https://src.alm01.itbatera.euskadi.eus/itbatera/gcp/gcp-project-templates/unix-gcp-templates/-/tree/master/j61-kafka-sandbox

y un pipeline para sandbox

 https://builds.alm01.itbatera.euskadi.eus/job/platform/job/unix/job/j61-kafka/job/33-deploy-sandbox/

 

Pendiente la creación de otro gitlab con nombre por ejemplo kafaconnectors-sandbox y otro pipeline que aplique todos los yamls almacenados
