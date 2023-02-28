# Monitorización Kafka 

[[_TOC_]]

## ToDo
- [x] Despliegue prometheus (modo Strimzi) en namespace monitoring-nop
- [ ] Exposición dashboards en monitoring-nop.itbatera.ejgv.eus
- [ ] Envio de Alerta Conector task state
- [ ] Revision de persistencia

## Prometheus

### Instalación del operador

Se descarga el fichero yaml con la descripción del operador de Prometheus:

```
curl -s https://raw.githubusercontent.com/coreos/prometheus-operator/master/bundle.yaml > prometheus-operator-deployment.yaml
```

En el fichero **[prometheus-operator-deployment.yaml](../files/prometheus-operator-deployment.yaml)** se substituye el namespace `default` por `monitoring` (o el namespace donde se vaya a desplegar Prometheus).

Se despliega el operador:

```
kubectl apply -f prometheus-operator-deployment.yaml
```
### Definición de los recursos necesarios

Los ficheros mencionados a continuación se extraen del directorio `examples/metrics` del operador [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator.git).

En el caso de que no se haya hecho previamente, para obtener esos directorios, se clona el repositorio git:

```
git clone https://github.com/strimzi/strimzi-kafka-operator.git
```

- **[prometheus-additional.yaml](../files/prometheus-additional.yaml)**

  Define un `Secret` con la configuración de Prometheus.

  Nota:
  El `job` `node-exporter` es necesario sólo si vamos a usar node-exporter para extraer métricas de los nodos del cluster.

- **[strimzi-pod-monitor.yaml](../files/strimzi-pod-monitor.yaml)**

  Debe modificarse `namespaceSelector.matchName` en los podMonitors:

  - cluster-operator-metrics
  - entity-operator-metrics
  - bridge-metrics
  - kafka-resource-metrics

  para que refleje el namespace de los pods a monitorizar (kafka-sandbox o el correspondiente en cada caso)


- **[prometheus-rules.yaml](../files/prometheus-rules.yaml)**

  Define las reglas para las diversas alertas.
- **[prometheus.yaml](../files/prometheus.yaml)**

  Define el servidor Prometheus junto con los correspondientes `ClusterRole`, `ClusterRoleBinding` y `ServiceAccount`.

  Se tiene que modificar el namespace a `monitoring` (o el namespace donde vaya a desplegarse Prometheus).



- **[alert-manager-config.yaml](../files/alert-manager-config.yaml)**
  Define un `Secret` con la configuración del Alert Manager.



- **[alert-manager.yaml](../files/alert-manager.yaml)**

  Define el recurso `Alertmanager` y el `Service` asociado.


### Despliegue
#### Manual
Se aplica la configuración de los recursos definidos en los ficheros anteriores:

```
kubectl apply -f prometheus-additional.yaml
kubectl apply -f strimzi-pod-monitor.yaml
kubectl apply -f prometheus-rules.yaml
kubectl apply -f prometheus.yaml
kubectl apply -f alert-manager-config.yaml
kubectl apply -f alert-manager.yaml
```
#### Automático
El despligue está automatizado mediante un [pipeline de Jenkins](https://builds.alm01.itbatera.euskadi.eus/job/platform/job/unix/job/j61-monitoring/job/33-deploy-nop/). 

Este pipeline utiliza un [repositorio de gitlab](https://src.alm01.itbatera.euskadi.eus/itbatera/gcp/gcp-project-templates/unix-gcp-templates) en el que están los ficheros yaml necesarios para el despliegue.


## Grafana

### Despliegue
Al igual que los ficheros yaml de prometheus, tanto el yaml de grafana como los json de dashboards se extraen del directorio `examples/metrics` del operador [strimzi-kafka-operator](https://github.com/strimzi/strimzi-kafka-operator.git).

Si se desea publicar el acceso a Grafana habría que revisar la definición del `service` en el fichero [grafana.yaml](../files/grafana.yaml).

Se despliega Grafana:

```
kubectl apply -f grafana.yaml
```

### Acceso

Si no se ha modificado el `service` se debe hacer un port forwarding:

```
kubectl port-forward svc/grafana 3000
```

En este caso, el acceso a Grafana se realiza mediante la url http://localhost:3000.



### Configuración inicial
#### Data source
Desde la aplicación, en `Configuration -> Data Sources` se añade Prometheus como `data source`.

En type se indica `Prometheus` y como server `http://prometheus-operated:9090`.

#### Dashboards
Los dashboards se importan desde los ficheros json ubicados en `examples/metrics/grafana-dashboards`.

Para importarlos, se utiliza el menú `Create(+) -> Import`.
## Monitorización de Kafka

Se generan los `ConfigMaps` con la configuración de métricas para `jmxPrometheusExporter` de kafka y zookeeper definidas en:

- [kafka-metrics-config.yaml](../files/kafka-metrics-config.yaml)
- [zookeeper-metrics-config.yaml](zookeeper-metrics-config.yaml)

```
kubectl apply -f kafka-metrics-config.yaml
kubectl apply -f zookeeper-metrics-config.yaml
```

El fichero yaml que define el recurso Kafka debe incluir:

```
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
…
spec:
  kafka:
  …
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: kafka-metrics

  …
  zookeeper:
  …
    metricsConfig:
      type: jmxPrometheusExporter
      valueFrom:
        configMapKeyRef:
          name: zookeeper-metrics
          key: zookeeper-metrics-config.yml
  …
  kafkaExporter:
    topicRegex: ".*"
    groupRegex: ".*"
```


Otros recursos que se desplieguen (KafkaConnect, KafkaBridge, KafkaMirrorMaker, …) también deberán incluir su configuración correspondiente.



## Monitorización de MongoDB

### mongodb-exporter

La instalación de mongodb-exporter se hace usando `helm`:

En un fichero, `values.yaml`, se incluye:
```
fullnameOverride: "mongodb-exporter"
podAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "9216" 
serviceMonitor:
  enabled: false
mongodb:
  uri: mongodb://user:pass@mongodb:27017/...
```

`prometheus.io/port` es el puerto en el que estarán disponibles las métricas

En el namespace monitoring (o el correspondiente para monitorización) se instala el exporter:
```
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter \
-f values.yaml
```
### serviceMonitor

El fichero `mongodb-servicemonitor.yaml` define el serviceMonitor para MongoDB:

```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  labels:
    serviceMonitorSelector: mongodb-exporter
  name: mongodb
spec:
  endpoints:
  - interval: 30s
    targetPort: 9216
    path: /metrics
  namespaceSelector:
    matchNames: []
  selector:
    matchLabels: {}
```
El despliegue se hace en el mismo namespace en el que esté el exporter (monitoring):

```
kubectl apply -f mongodb-servicemonitor.yaml -n monitoring
```

