---
title: comparatif des solutions d'observabilité
created: 2025-07-16
author:
  - mikamakusa
category:
  - Cloud
tags:
  - Prometheus
  - VictoriaMetrics
  - Splunk
  - Datadog
---
# Comparatif des stacks d’observabilité

Quand on vient du monde de l’OpenSource, l’observabilité se résume bien souvent à Nagios pour les plus anciens d’entre nous ou à Prometheus pour les plus jeunes. Il est vrai que Nagios est présent au coeur de nombreux outils dont Centron, CheckMK, Icinga ou Shinken pour ne citer qu’eux et accuse son âge vénérable (un peu plus de 30 ans) bien qu’il ait reçu de nombreuses mise à jours et qu’ils soit (lui ou ses différents dérivés) toujours utilisé au sein de diverses entreprises.

Du côté des plus jeunes d’entre nous, la puissance de Prometheus n’est plus à démontrer bien qu’il existe encore des poches de résistances avec la stack ELK (ElasticSearch/Logstash/Kibana), InfluxDB ou même Victoria Metrics.

Cependant, d’autres alternatives existent également telles que Datadog, Dynatrace, Splunk sans même évoquer les cas des différentes solutions managées des Cloud Providers AWS, Azure ou Google Cloud (Pour ne citer que les trois plus gros).

Dans cet article, nous allons effectuer un comparatif de certaines solutions de monitoring, non disponible à partir d’un Cloud Provider, les plus récentes :

- Prometheus

- Victoria Metrics

- Datadog

- Splunk

## Le déploiement

On commence rarement par s'intéresser à ce détail, l’aspect “prix” est souvent privilégié lorsque l’on aborde une brique aussi critique que celle de l’observabilité. Mais le fait est que les quatres solutions observées proposent des approches assez opposées en ce qui concerne leur déploiement et leur maintenabilité.

Prometheus ainsi que Victoria Metrics, étant des solutions OpenSource, proposent des approches relativement similaires. S’il est possible de les installer, à l’aide d’un Chart Helm sur un cluster Kubernetes, il est aussi possible de les déployer à la main sur n’importe quel serveur.

Splunk et Datadog, tous deux des solutions propriétaires payantes, proposent de leur côté des solutions SaaS. Cependant Splunk propose également une version déployable on-premise.

Un bon argument pour les entreprises souhaitant garder la maîtrise de leurs données ainsi que de leur infrastructure.

***<span style="text - decoration: underline;">Avantage***</span> : Prometheus, Victoria Metrics…et Splunk dans une certaine mesure.

Le fait que Datadog soit un SaaS peut être un gros frein lorsque la maîtrise des données est primordiale. En revanche, Prometheus, Victoria Metrics et Splunk (dans le cas d’un déploiement on-premise) peuvent être difficilement maintenables (en fonction de contraintes techniques ou humaines).

## La collecte des données

Cet aspect de l’observabilité est crucial, non seulement il permet de déterminer quels composants peuvent être monitorer, le type de métriques que l’on peut observer ainsi que la fréquence des données et, par conséquent, leur fraîcheur et leur précision.

Cette fois-ci, les deux solutions OpenSource se démarquent complètement par leur mode de fonctionnement, Prometheus peut récupérer des données à l’aide de plugins, appelés exporters.

Ceux-ci couvrent de nombreux usages (base de données, stockage, application, système, etc…) et sont développés par la communauté.

En revanche, Victoria Metrics, Datadog et, dans une certaine mesure, Splunk ont fait le choix de la simplicité en ne proposant qu’un seul agent afin de récupérer les métriques des serveurs cibles.

***<span style="text - decoration: underline;">Avantage***</span> : Victoria Metrics, Datadog et Splunk.

La simplicité et la polyvalence des agents proposés par les trois solutions de monitoring permettent d’adresser toutes sortes de problématiques.

## La visualisation des données

Aspect un peu secondaire, comparé à la collecte de données, mais tout aussi important. Ici nous avons deux approches diamétralement opposées entre les solutions opensources ne proposant que de l’exploration de données mais de nombreuses possibilités d’intégration avec des outils tiers (tels que Grafana, par exemple)...ou des fonctionnalités de graphes très avancées intégrant également du calcul pré-création et post-création de graphes.

***<span style="text - decoration: underline;">Avantage***</span> : Datadog et Splunk

La puissance et l’aspect pratique des différents types de graphiques totalement personnalisables associés au fait que tout est déjà intégré au sein d’une même interface.

## Le monitoring en temps réel et rétention de données

Le cœur du sujet, tout simplement. Chacune des solutions au coeur de ce comparatif propose du monitoring en temps réel ainsi que des possibilités de retour dans le temps (Dans le cas d’une investigation suite à un incident rencontré).

Dans le cas de Prometheus, la rétention de données est limitée à 30 jours et peut être rallongée si on l’associe à Thanos.

Victoria Metrics fonctionne différemment du précédent dans le sens où il dispose d’un moteur de stockage optimisé regroupant et optimisant les données à stocker.

Datadog et Splunk proposent tous deux deux fonctionnalités similaires en termes de monitoring : Gestion de logs et des métriques, performances applicatives, surveillance utilisateurs, analyse réseau et cybersécurité.

***<span style="text - decoration: underline;">Avantage***</span> : Tout dépend des usages.

Par conséquent match nul.

## Le Machine Learning

Le machine learning n’est présent qu’au sein de Datadog et de Splunk à fins d'analyse automatique des performances d’infrastructure ou d’applications.

Le principe de cette intégration est de détecter des anomalies, de la latence un peu trop élevée…un comportement considéré comme exotique par Datadog ou Splunk.

Cela peut également être utile pour calculer des SLO ou des valeurs étranges au sein de graphes.

***<span style="text - decoration: underline;">Avantage***</span> : Datadog et Splunk…mais encore une fois, tout dépend des usages.

## La recherche

Pour requêter une métrique, il faut une syntaxe particulière et c’est à ce moment qu’entre en scène les langages de requêtes. Contrairement aux autres, Datadog a choisi de complexifier son fonctionnement en développant plusieurs langages de requêtes en fonction du contexte.

Par exemple :

- Log Management utilise du full-text (exemple : *@http.url_details.path:"/api/v1/test"*)

- Trace Explorer utilise des mots clé et des expressions (exemple : *(hostname:web-server OR env:prod)*)

***<span style="text - decoration: underline;">Avantage***</span> : Prometheus, Victoria Metrics et Splunk

Devoir utiliser plusieurs langages de requêtes au sein d’un même outil, bien que les usages soient différents, peut être un frein à l’adoption de l'outil en question.

## Tarification et support

Ici nous pouvons observer 3 points de vue différents.

En ce qui concerne la documentation, chacune propose une documentation officielle extrêmement complète et, dans certains cas, une communauté également très active.

D’un côté Prometheus est OpenSource est entièrement gratuit mais pouvant entraîner des coûts annexes (hébergement, maintenance, stockage des données)...constat pouvant être similaire avec Victoria Metrics.

De leur côté, Datadog et Splunk proposent plusieurs niveaux de tarifications incluant des fonctionnalités plus avancées ainsi que du support et une version Cloud.

ont également suivi le chemin.

De plus, la tarification de Splunk manque de lisibilité et il semble obligatoire de les contacter pour obtenir les informations nécessaires.

Victoria Metrics a suivi le même chemin que les deux précédents : Plusieurs niveaux de tarification (en complément de l’offre gratuite) associés à des fonctionnalités plus avancées en termes de rétention de données, de sécurité et de support.

***<span style="text - decoration: underline;">Avantage***</span> : Prometheus et, dans une certaine mesure, Victoria Metrics.

## Conclusion
Chaque solution à ses avantages (gratuité, documentation complète, communauté active) et ses inconvénients (tarifs, coûts annexes).
