---
title: Le tracing dans Grafana
created: 2021-10-14
author:
  - mikamakusa
category:
  - Cloud
tags:
  - Elasticsearch
  - Grafana
  - Loki
  - Tempo
  - OpenTelemetry
---
# Le tracing dans Grafana

Au départ, il est vrai que **Grafana** ne gère pas les logs, il y a de nombreuses applications qui le font très bien, notamment **Kibana** qui, en plus des logs, gère métriques et journaux grâce aux **beats** (**metricbeat**, **filebeat**, **journalbeat**, **packetbeat**).

Cependant cet article n'est pas centré sur **Kibana**, mais bien sur **Grafana** ainsi que deux autres composants, à savoir : **Loki** et **Tempo**.

Oui...nous avons la seconde *Disney Princess* dans le monde de l'informatique après **Thanos**.

## Le besoin

Imaginons...L'Apocalypse (celle avec un grand A)...l'application sur laquelle vous travaillez en tant que développeur, composée d'une multitude de micro-services, déjà déployée en production (sinon, c'est beaucoup moins amusant) commence à avoir un comportement pour le moins exotique...à la veille d'une démonstration.

J'avais évoqué l'Apocalypse, n'est-ce pas ?

Bon, je sais que ça pourrait être bien pire, mais ce n'est pas terminé...parce que l'application en question, au bout de quelques heures de comportement ***TRES*** étrange, finit par s'arrêter de manière tout aussi inexplicable et que notre rôle, dans l'histoire, consiste à trouver pourquoi ?

En toute logique, puisqu'il s'agit de microservices, autant commencer par analyser les logs de chacuns d'entre eux en commençant par le premier : <*code>kubectl logs <pod1> -c <container> -n <namespace1> --sin*ce=3h</code>

Et...rien d'alarmant, aucun warning, aucune erreur, tout semble sous contrôle...frustrant, mais vous vous y étiez préparé...après tout, on ne peut pas forcément tomber sur le coupable dès le premier coup.

Allons jeter un oeil aux logs du second : <*code>kubectl logs <pod2> -c <container> -n <namespace2> --sin*ce=3h</code>

Et...toujours rien, même pas un petit faux positif...ceci dit, il y a d'autres microservice pour lesquels il faut analyser les logs.

L'analyse des logs et le dépannage de l'application risque d'être extrêmement chronophage à l'aide de la méthode présentée au-dessus.

## Les ingrédients

Comme vous pouvez le deviner, nous allons faire appel à **Grafana**, **Loki** et **Tempo**...ainsi qu'a **Filebeat**, **Logstash** et **OpenTelemetry**.

Vous trouverez partout sur Internet des tutoriels et articles sur le même sujet mais abordant une technologie différente du couple **Filebeat/Logstash**, à savoir **Fluentd**, mais le but ici est d'ajouter un peu de challenge.

Le tout à l'aide de **Kubernetes** et **Helm**.

Le seul composant qui ne sera pas déployé via **Helm** sera **Istio** avec lequel nous allons activer le *tracing *(le proxy I**stio** inclue un *TraceID*) en injectant la configuration suivante :

```yaml
apiVersion: install.istio.io/v1alpha1

kind: IstioOperator

metadata:

name: istio-operator

namespace: istio-system

spec:

profile: default

meshConfig:

accessLogFile: /dev/stdout

accessLogEncoding: JSON

accessLogFormat: |

[%START_TIME%] "%REQ(:METHOD)% %REQ(X-ENVOY-ORIGINAL-PATH?:PATH)% %PROTOCOL%" %RESPONSE_CODE% %RESPONSE_FLAGS% %RESPONSE_CODE_DETAILS% %CONNECTION_TERMINATION_DETAILS% "%UPSTREAM_TRANSPORT_FAILURE_REASON%" %BYTES_RECEIVED% %BYTES_SENT% %DURATION% %RESP(X-ENVOY-UPSTREAM-SERVICE-TIME)% "%REQ(X-FORWARDED-FOR)%" "%REQ(USER-AGENT)%" "%REQ(X-REQUEST-ID)%" "%REQ(:AUTHORITY)%" "%UPSTREAM_HOST%" %UPSTREAM_CLUSTER% %UPSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_LOCAL_ADDRESS% %DOWNSTREAM_REMOTE_ADDRESS% %REQUESTED_SERVER_NAME% %ROUTE_NAME% traceID=%REQ(x-b3-traceid)%

enableTracing: true

defaultConfig:

tracing:

sampling: 100

max_path_tag_length: 99999

zipkin:

address: otel-collector.namespace.svc:9411
```

### Loki

Effectivement, je n'ai pas encore présenté la nouvelle Disney Princess du monde de l'informatique, à savoir Loki.

Contrairement à d'autres systèmes de journalisation, Loki a été pensé autour de l'indexation de métadonnées des journaux de logs : les labels. Ces mêmes données qui sont ensuite compressées et stockées sur des espaces de stockage de type S3 (AWS S3, Google Cloud Storage, Openstack Swift, Azure Blob Storage, etc...) ou bien localement.

Le déploiement de **Loki** via **Helm** est assez rapide au vu du nombre d'options peu élevé :

- activation de **Loki**,

- activation de **Promtail**,

- activation de **Fluent-bit**,

- activation de **Grafana**,

- activation de **Prometheus**,

- activation et configuration de **Filebeat**,

- activation et configuration de **Logstash**.

Parmi les différentes options, je déconseille fortement d'activer **Filebeat**, **Logstash**, **Grafana**, **Fluent-bit** et **Prometheus**...Les deux derniers ne seront pas du tout utilisés par la suite et pour les trois autres, il est préférable de privilégier des charts **Helm** dédiés.

```yaml
fluent-bit:
  enabled: false
promtail:
  enabled: true
prometheus:
  enabled: false
alertmanager:
  persistentVolume:
    enabled: false
  server:
    persistentVolume:
      enabled: false
filebeat:
  enabled: false
logstash:
  enabled: false
```

### Promtail

Mais pourquoi ai-je activé **promtail **?

Ce composant est responsable de l’envoi des logs vers **Loki**., tout simplement. Il s’avère que même si **Logstash** est bien configuré, **Loki** ne recevra ni n'indexe les logs que si P**romtail** est présent pour l’aider dans sa tâche.

Par contre, pas d’inquiétude...c’est le composant par défaut, donc il s’acquittera de sa tâche sans configuration particulière et est totalement invisible, néanmoins cela implique de stocker localement les logs (ce qui est suffisant pour notre usage actuel). Si vous souhaitez déporter le stockage sur S3/Object/Blob Storage, il faudra certainement creuser un peu du côté des options les options de configuration.

### Tempo

C'est une technologie assez récente venant également de **Grafana Labs** (Un peu moins d'un an, la première release - la 0.2.0 - date de fin Octobre 2020) de traçage distribué, économique car ne nécessitant que le stockage objet déjà utilisé par **Loki** et intégré à **Grafana**, **Prometheus** et **Loki**. De plus, il peut être utilisé avec n'importe quel protocole de traçage tels que **OpenTelemetry**, **Zipkin**, **Jaeger** ou **OpenTracing**.

Le déploiement de **Tempo** via **Helm** est également très rapide, bien qu'il y ait deux conteneurs différents, ce sera dans **Tempo** (le premier conteneur) qu'il faudra définir les *receivers* (à savoir **zipkin** et **opentelemetry** ) ainsi que des arguments particuliers pour la ligne de commande.

```yaml
tempo:
  extraArgs:
    "distributor.log-received-traces": true
receivers:
  zipkin:
  otlp:
protocols:
  http:
  grpc:
```

### OpenTelemetry

**OpenTelemetry** est un ensemble d'API, de SDK, d'outils et d'intégrations créés dans le seul but de gérer des données télémétriques telles que des *traces*, des *métriques* et des *logs*. Le projet est agnostique et n'est pas attaché à un matériel (constructeur ou fournisseur de services Cloud) et est configurable pour transmettre des données vers le backend de notre choix (tels que **jeager**, **prometheus** ou, dans notre cas, **Tempo**).

Dans notre cas, nous n'allons configurer que les *recievers*, les *exporters* et les *pipelines* :

```yaml
config:
  recievers:
    zipkin:
      endpoint: 0.0.0.0:9411
  exporters:
    otlp:
      endpoint: tempo.namespace.svc.cluster.local;55680
      insecure: true
    service:
      pipelines:
      traces:
        recievers: [zipkin]
        exporters: [otlp]
```

Faites bien attention, le chart **OpenTelemetry** inclue de nombreux *recievers*, *exporters* et *pipelines* qui ne vous seront peut être pas utile (**Prometheus** par exemple).

### Filebeat

Plus récent que **Logstash**, mais sortie des mêmes labos d'**Elastic**, et proposant une fonctionnalité de base similaire à **Logstash**, à savoir l'envoi de logs d'une sources basée sur des fichiers vers une destination. Dans certains cas, il est possible de se passer de **Logstash** et d'injecter directement les logs collectés par **Filebeat** dans **Elasticsearch**.

Cependant nous n'avons pas besoin de ce dernier, **Filebeat** ne propose pas de plugin pour **Loki** contrairement à **Logstash**.

Nous allons toujours le déployer via **Helm**, le chart propose le choix entre *daemonset* et *deployment*...Sachez que le fonctionnement est assez identique, à ceci près que le *daemonset* sera déployé sur tous les nodes du cluster avec la configuration suivante :

```yaml
filebeat.inputs:
  - type: container
paths:
  - /var/log/containers/.log
processors:
  - add_kubernetes_metadata:
host: ${NODE_NAME}
matchers:
  - logs_path:
logs_path: "/var/log/containers/"
output.logstash:
  hosts: ["logstash-headless.namespace.svc:5044"]
```

L'utilisation du *processors / add_kubernetes_metadata* permet d'ajouter de nombreux labels à destination de **Loki** afin d'affiner la gestion des logs dans **Grafana**.

### Logstash

Comme évoqué plus tôt, la collecte des métriques pouvait être assurée par **Logstash** avant la création de collecteur plus léger (**Filebeat**, **Metricbeat**, **Packetbeat**, **Journalbeat**). Bien que la collecte de données soit toujours possible, **Logstash** est souvent de plus en plus utilisé en tant que *forwarder* ayant la particularité de pouvoir transformer ou de filtrer les logs (via les différents filtres ou l'utilisation de **Grok**).

**<span style="text - decoration: underline;">Attention**</span> : Le chart Helm permet de déployer **Logstash** de manière assez rapide, cependant si vous n'éditez pas le fichier *values.yaml*, vous risquez de passer à côté de l'update de l'image **Docker**.

Celle qui est proposée est une image officielle ne prenant pas en compte le plugin **Loki**...il est préférable de la remplacer par *grafana/logstash-output-loki:1.0.1*.

Il reste la configuration qui reste assez classique avec les keywords **input**, **output** et **filter** (même si le dernier ne sera peut-être pas utile) :


```yaml
image:
  registry: docker.io
  repository: grafana/logstash-output-loki
  tag: 1.0.1
input: |-
  beats {
    port => 5044
  }
filter: ""
output: |-
  loki {
    url => "http://loki-headless:3100/loki/api/v1/push"
    batch_size => 112640 112.64 kilobytes
    retries => 5
    min_delay => 3
    max_delay => 500
    message_field => "message"
  }

```

### Grafana

Il n'y a plus vraiment besoin de le présenter...Le déploiement de **Grafana** via **Helm** va être l'occasion d'automatiser la configuration des datasources **Loki** et **Tempo** :

```yaml
datasources:
  datasources.yaml:
    apiVersion: 1
datasources:
  - name: Tempo
    type: tempo
    access: browser
    orgId: 1
    uid: tempo
    url: http://tempo.namespace.svc:3100
    isDefault: true
    editable: true
  - name: Loki
    type: loki
    access: browser
    orgId: 1
    uid: loki
    url: http://loki.namespace.svc:3100
    isDefault: false
    editable: true
    jsonData:
      derivedFields:
        - datasourceName: Tempo
          matcherRegex: "X-B3-TraceId: (\\w+)"
          name: TraceID
          url: "$${__value.raw}"
          datasourceUid: tempo
          env:
            JAEGER_AGENT_PORT: 6831
            adminUser: <ADMIN>
            adminPassword: <PASSWORD>
service:
  type: LoadBalancer

```
## Passons en cuisine

Maintenant que tous les ingrédients sont réunis, passons au déploiement proprement dit.

Il y a un certain ordre à respecter afin de ne pas rencontrer d'erreurs au niveau d'un composant ou d'un autre.

Par exemple :

- **Logstash** est à déployer avant **Filebeat**,

- **Tempo** est à déployer avant **OpenTelemetry**,

- **Grafana** peut être déployé à n'importe quel moment.

Puisque tout va être déployé à l'aide de **Helm**, il conviendra d'exécuter les commandes suivantes :

```shell
kubectl create ns <YOUR-NAMESPACE>

helm repo add grafana https://grafana.github.io/helm-charts

helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts

helm repo add bitnami https://charts.bitnami.com/bitnami

helm repo add elastic https://helm.elastic.co

helm repo update

helm upgrade --install tempo grafana/tempo --version 0.7.4 -n <YOUR-NAMESPACE> -f tempo/values.yaml

helm upgrade --install loki grafana/loki-stack --version 2.4.1 -n <YOUR-NAMESPACE> -f loki-stack/values.yaml
helm upgrade --install opentelemetry open-telemetry/opentelemetry-collector -n <YOUR-NAMESPACE> -f open-telemetry/values.yaml
helm upgrade --install logstash bitnami/logstash -n <YOUR-NAMESPACE> -f logstash/values.yaml
helm upgrade --install filebeat elastic/filebeat -n <YOUR-NAMESPACE> -f filebeat/values.yaml
helm upgrade --install grafana grafana/grafana -n <YOUR-NAMESPACE> -f grafana/values.yaml
kubectl get all -n <YOUR-NAMESPACE>

```

Une fois que tout est *ready* et opérationnel - je vous invite même à faire un tour dans les logs de chaque composant, pour éviter toute surprise - vous allez pouvoir passer à la phase de test.

## La dégustation

Maintenant que tout à été déployé, allons voir dans **Grafana** :

```shell
kubectl port-forward svc/grafana -n <YOUR-NAMESPACE> 8081:3000
```
Ouvrez votre browser sur l'URL `http://localhost:8081/explore` avec les credentials définies dans grafana/values.yaml puis sélectionnez Loki dans le menu déroulant en haut :


Puis dans le champs *Log Browser*, ajoutez le filtre correspondant à votre application (Ici, ce sera *productpage*) afin d'afficher les logs:

Cliquez sur l'une des lignes de logs afin que les labels et les champs détectés soient visible...et cliquez sur le bouton **Tempo** :

Immédiatement, la fenêtre de **Grafana** va se diviser en deux, d'un côté **Loki** et **Tempo** de l'autre...et dans cette seconde partie, **Tempo** va afficher la trace complète relative au log sélectionné :


## Finalement...

La stack **Grafana/Loki/Tempo** ainsi que les autres composants présentés ici, en plus de présenter des graphiques aussi beaux que pratiques, permettront de debugger relativement efficacement les situations plus ou moins apocalyptique comme celle présentée en introduction.

Ca ne fera pas tout, ce n'est pas la stack en elle même qui va vous débloquer, le reste des actions ne dépend plus que de vous.

## Mon retour sur le sujet :

- Après avoir goûté à **ElasticSearch**, je trouve **Loki** (beaucoup) plus léger. Dans une optique de réduction des coûts (aussi bien pour du FinOps que pour du GreenIT), ce n’est pas négligeable,

- Vu que **Grafana** peut gérer Traces, Logs et Métriques, il n’y a plus besoin d’autres outils,

- **Loki** et **Tempo** sont facilement scalable,

- Contrairement à **Elasticsearch**, pas d’*index pattern*, d’*index policy management* ou autres actions à effectuer post-déploiement.

Malgré toutes ces bonnes idées derrière **Loki/Tempo/Logstash**, il y a quelques inconvénients :

- Un manque de documentation qui pourrait être assez décourageant. Lorsque l’on se lance sur la tracing, la documentation disponible est très souvent centrée sur l’association entre **F****luentd** et **Loki** ou sur **ElasticSearch/Fluentd** et **Loki**. La seule page d’informations relative à **Logstash/Loki** se trouve sur le site officiel de **Grafana** et n’est pas complètement fonctionnelle.

- Il manque un opérateur **Loki** car il semblerait que la configuration ou le processus de mise-à-jour soient un peu compliqués.

- Le déploiement et la compréhension de **Loki** risquent de le rendre difficile à maintenir...et peut-être d’en dégoûter quelques-uns au passage.

- Bien que le chart Helm **loki-stack** soit composé de plusieurs chart bien distinct, il ne semble pas possible, à l’heure actuelle, de modifier la configuration de l’un d’entre eux et de le déployer tel quel.

- Le plugin **loki** pour **logstash** n’est pas officiel du côté d’Elastic, encore très mal documenté chez Grafana Labs et semble disponible seulement en tant qu’image Docker. Aucune information quant à une éventuelle disponibilité en tant que standalone (pour un déploiement sur une instance **Logstash** déjà existante)