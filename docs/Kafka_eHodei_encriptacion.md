# <img src="https://upload.wikimedia.org/wikipedia/commons/0/0a/Apache_kafka-icon.svg" width="60">Kafka eHodei: Encriptación 

v 1.0 2021/07/14 sergio-perez@ejie.eus  Primera versión



[TOC]


## Encriptación TLS

[Strimzi](https://strimzi.io/) Kafka soporta comunicaciones encriptadas entre los diferentes componentes del cluster kafka y los streams de aplicaciones mediante el protocolo TLS.  Las comunicaciones entre los brokers kafka (interbroker), entre los nodos ZooKeeper (internodal), y entre éstos y los operadores  son siempre encriptadas.  La encriptación de las comunicaciones entre clientes y brokers kafka se activan en la configuración TLS de los listeners del cluster. Los certificados TLS también son utilizados para autenticación.



![](C:\Users\sperezbu\OneDrive - ELKARLAN\IL\Productos\eHodei\imgs\secure_communication.png)



## Certificados


Para soportar la encriptación a nivel de cluster, cada componente Kafka requiere certificados de clave privada y pública. Todos los certificados de componentes son firmados por una Autoridad Certificadora(CA) interna, denominada `cluster CA`. 

De forma similar, cada aplicación cliente Kafka se conecta al mediante autenticación TLS, necesita clave privada y certificados. Una segunda CA, llamada `clients CA`, se emplea para firmar los certificados de los clientes Kafka.

Ambos  `cluster CA` y `clients CA` tiene un certificado de clave pública auto-firmado.

Los brokers kafka son configurados para confiar en los certificados firmados por la `cluster CA` o `clients CA`. Los componentes que no requieren conexión desde cliente, como los ZooKeeper,  solamente confian en los certificados firmado por la `cluster CA`.

Los listeners Kafka se configuran con encriptación TLS activada, y certificados auto-firmados por la  CA . Cuando los clientes intentan conectar con un listenter securizado con dicho certificado, no confían por defecto. Es necesario configurar a nivel de cliente la clave pública de la CA con la que se firmaron los certificados de servidor.

Por lo tanto, se requiere la distribución de la clave publica a todos los clientes y activar su configuración. Por otro lado, es necesario actualizar la clave pública CA cada vez que se cambia.

Los periodos de validez de certificados CA, definido como número de días transcurridos desde la generación del certificado, se configuran en `Kafka.spec.clusterCa.validityDays` y `Kafka.spec.clientsCa.validityDays`.



````
Not Before                                     Not After
    |                                              |
    |<--------------- validityDays --------------->|
                              <--- renewalDays --->|
````

El periodo de validez para los dos certificados son  365 días. Por defecto, el operador genera y renueva automáticamente los certificados CA emitidos por `cluster CA` o `clients CA`. Los certificados proporcionados por usuarios no son renovados.

El periodo de validez de los certificados CA se puede comprobar mediante el comando openssl:

````
[gke01nop]$ kubectl -n kafka-sandbox get secret kafka01nop-j61-cluster-ca-cert  -o 'jsonpath={.data.ca\.crt}' |  base64 -d | openssl x509 -subject -issuer -startdate -enddate -noout
subject= /O=io.strimzi/CN=cluster-ca v0
issuer= /O=io.strimzi/CN=cluster-ca v0
notBefore=Jul 15 09:32:11 2021 GMT
notAfter=Jul 15 09:32:11 2022 GMT
[gke01nop]$ kubectl -n kafka-sandbox get secret kafka01nop-j61-clients-ca-cert  -o 'jsonpath={.data.ca\.crt}' |  base64 -d | openssl x509 -subject -issuer -startdate -enddate -noout
subject= /O=io.strimzi/CN=clients-ca v0
issuer= /O=io.strimzi/CN=clients-ca v0
notBefore=Jul 15 09:32:13 2021 GMT
notAfter=Jul 15 09:32:13 2022 GMT
````

El comando devuelve las fechas  `notBefore` y `notAfter` date, que es el periodo de validez para cada certificado CA.



### Secretos

[Strimzi](https://strimzi.io/) Kafka utiliza Secretos kubernetes para almacenar claves privadas y certificados de los diferentes componentes del cluster kafka y clientes. Los Secretos se utilizan para establecer conexiones TLS encriptadas entre kafka brkers, y entre brokers y clientes. Tamibién se emplean para autenticación mutual TLS.

-  Un *Cluster Secret* contiene el certificado `cluster CA` para firmar los certifidados de brokers kafka, y la emplea el cliente para validar la identidad del broker en la conexión encriptada TLS al cluster kafka.
- Un *Client Secret* contiene el certifidado `client CA`  para que un usuario firem sus propios certificados cliente que permite la autenticación Mutual TLS contra el cluster Kafka. El broker valida la identidad del cliente mediante el mismo certificado `clinet CA`.
- Un  *User Secret*  contiene la clave privada y certificado que es generado y firmado por el `client CA`  en la creación de un nuevo usuario. La clave y el certificado se emplean en el acceso al cluster para autenticación y autorización.

Los Secretos almacenan claves privadas y certificados en formato PEM y PKCS #12.  El uso de claves y certifiados en formato PEM  implica que el usuario tiene que obtenerlos del  Secreto, y generar el correspondiente truststore (o keystore) para utilizar en sus aplicaciones  Java. El almacenamiento en formato PKCS #12 permite un uso directo desde  truststore (o keystore).

El tamaño de todas las claves es de 2048 bits. 



A continuación de detallan los secretos en el entorno sandbox .

- **Cluster CA Secrets:**



| Secret                         | Field                      | Descripción                |
| ------------------------------ | -------------------------- | -------------------------- |
| kafka01nop-j61-cluster-ca      | ca.key                     | Clave privada cluster CA.  |
| kafka01nop-j61-cluster-ca-cert | ca.p12                     | Fichero PKCS #12 con  claves y certificados.                 |
|                                      | ca.password                         | Protección password de fichero PKCS #12. |
|                                | ca.crt                     | Certificado cluster CA. |
| kafka01nop-j61-kafka-brokers   | kafka01nop-j61-kafka-#.p12 | Fichero PKCS #12 con  claves y certificados. |
|                                | kafka01nop-j61-kafka-#.password                           | Protección password de fichero PKCS #12. |
|                                | kafka01nop-j61-kafka-#.crt                           | Certificado de pod kafka broker #  firmado por clave privada de Cluster CA. |
|                                | kafka01nop-j61-kafka-#.key                         | Clave privada de pod kafka broker # . |
| kafka01nop-j61-zookeeper-nodes | kafka01nop-j61-zookeeper-#.p12 |Fichero PKCS #12 con  claves y certificados.|
|                                | kafka01nop-j61-zookeeper-#.password |Protección password de fichero PKCS #12.|
|                                | kafka01nop-j61-zookeeper-#.crt |Certificado de pod ZooKeeper #  firmado por clave privada de Cluster CA.|
|                                | kafka01nop-j61-zookeeper-#.key |Clave privada del  pod Zookeeper # .|
| kafka01nop-j61-entity-operator-certs | entity-operator.p12 |Fichero PKCS #12 con  claves y certificados.|
|                                | entity-operator.password |Protección password de fichero PKCS #12.|
|                                | entity-operator.crt                         |Certificado  para comunicaciones TLS entre el Operador Entity y Kafka o Zookeeper, firmado por clave privada de Cluster CA.|
|                                | entity-operator.key |Clave privada para comunicaciones TLS entre el Operador Entity y Kafka o Zookeeper.|


- **Client CA Secrets:**



- **User Secrets:**




**Certificados y secretos de  conectores MongoDB Kafka**

Secreto en kubernetes  para el acceso de conectores :

```
[gke01nop]$ kubectl describe  secret/mongodb-connect-community -n kafka-sandbox
Name:         mongodb-connect-community
Namespace:    kafka-sandbox
Labels:       app.kubernetes.io/instance=mongodb-connect-community
              app.kubernetes.io/managed-by=strimzi-user-operator
              app.kubernetes.io/name=strimzi-user-operator
              app.kubernetes.io/part-of=strimzi-mongodb-connect-community
              strimzi.io/cluster=kafka01nop-j61
              strimzi.io/kind=KafkaUser
Annotations:  <none>

Type:  Opaque

Data
====
ca.crt:         1854 bytes
user.crt:       1489 bytes
user.key:       1704 bytes
user.p12:       2792 bytes
user.password:  12 bytes

```



En el  pod  kafkaconnect de origen,  están activados  en la ruta `/opt/kafka/connect-certs`.

 ````
/opt/kafka/connect-certs/kafka01nop-j61-cluster-ca-cert:
total 0
lrwxrwxrwx 1 root root 13 Jul 15 09:50 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 13 Jul 15 09:50 ca.p12 -> ..data/ca.p12
lrwxrwxrwx 1 root root 18 Jul 15 09:50 ca.password -> ..data/ca.password

/opt/kafka/connect-certs/mongodb-connect-community:
total 0
lrwxrwxrwx 1 root root 13 Jul 15 09:50 ca.crt -> ..data/ca.crt
lrwxrwxrwx 1 root root 15 Jul 15 09:50 user.crt -> ..data/user.crt
lrwxrwxrwx 1 root root 15 Jul 15 09:50 user.key -> ..data/user.key
lrwxrwxrwx 1 root root 15 Jul 15 09:50 user.p12 -> ..data/user.p12
lrwxrwxrwx 1 root root 20 Jul 15 09:50 user.password -> ..data/user.password

 ````
Estos certificados se renuevan mediante el redespliegue de conectores.

**Certificados y secretos de Aplicación**

Secreto en kubernetes  para el acceso de aplicación:

````
[gke01nop]$ kubectl describe secret/j61-auditoria
Name:         j61-auditoria
Namespace:    kafka-sandbox
Labels:       app.kubernetes.io/instance=j61-auditoria
              app.kubernetes.io/managed-by=strimzi-user-operator
              app.kubernetes.io/name=strimzi-user-operator
              app.kubernetes.io/part-of=strimzi-j61-auditoria
              strimzi.io/cluster=kafka01nop-j61
              strimzi.io/kind=KafkaUser
Annotations:  <none>

Type:  Opaque

Data
====
ca.crt:         1854 bytes
user.crt:       1472 bytes
user.key:       1704 bytes
user.p12:       2736 bytes
user.password:  12 bytes
````



Y se entregan a DevOps(Everis) las credenciales del usuario en un tar.gz   y la ca del cluster que son necesarias para que se incluya en el  keystore de aplicación.



Desde el microservicio (Spring Boot) se debe cargar las credenciales a nivel de keystore, como se muestra en el [ejemplo](https://itnext.io/kafka-on-kubernetes-the-strimzi-way-part-3-19cfdfe86660).



## Enlaces de interes

- [Red Hat AMQ- Managing TLS certificates](https://access.redhat.com/documentation/en-us/red_hat_amq/2020.q4/html/using_amq_streams_on_openshift/security-str)
- [Certificados firmados por CA de confianza](https://strimzi.io/blog/2021/05/07/deploying-kafka-with-lets-encrypt-certificates/)
