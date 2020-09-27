# La stack ELK (Elastic)

Cette stack est très souvent utilisée notamment pour ingérer et indexer des logs. Elle est composée de 3 logiciels:
* Logstash qui permet de filtrer / formatter les données entrantes et de les envoyer à Elasticsearch (et à d'autres applications)
* Elasticsearch, le moteur responsable de l'indexation des données
* Kibana, l'application web permettant la visualisation des données

## Le but de cet exemple

Dans cet exemple, nous allons utiliser Docker Compose pour lancer une stack ELK en configurant Logstash de façon à :
- ce qu'il puisse recevoir des entrées de log sur un endpoint HTTP
- extraire le champs présent dans chaque entrée et ajouter des informations de reverse geocoding
- envoyer chaque ligne dans Elasticsearch

L'interface de Kibana nous permettra de visualiser les logs et de créer des dashboards.

Note: nous considérerons que les fichiers de log sont générés par un serveur web comme apache / nginx, cela nous sera utile pour spécifier la façon dont Logstash doit réaliser le parsing.


## Définition de l'application dans un fichier au format Compose

Afin de définir notre stack ELK, créez un répertoire *elk* et, à l'intérieur de celui-ci, le fichier docker-compose.yml avec le contenu suivant:

```
version: '3.8'
services:
  logstash:
    image: logstash:7.8.0
    environment:
      discovery.seed_hosts: logstash
      LS_JAVA_OPTS: "-Xms512m -Xmx512m"
    volumes:
      - ./logstash.conf:/config/logstash.conf
    command: ["logstash", "-f", "/config/logstash.conf"]
    ports:
      - 8080:8080
  elasticsearch:
    image: elasticsearch:7.8.0
    environment:
      discovery.type: single-node
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
  kibana:
    image: kibana:7.8.0
    ports:
      - 5601:5601
```

Note:

- Le service Logstash est basé sur l'image officielle logstash:7.8.0.
Nous précisons, sous la clé volumes, que le fichier de configuration logstash.conf présent dans le répertoire est monté sur /config/logstash.conf dans le container. Il sera pris en compte par Logstash au démarrage

- Le service Kibana est basé sur l'image officielle kibana:7.8.0. Le mapping de port permettra à l'interface web d'être disponible sur le port 5601 de la machine hôte.

## Fichier de configuration de Logstash

Afin de pouvoir indexer des fichiers de logs existant, nous allons configurer Logstash. Dans le répertoire *elk* (ou se trouve le fichier docker-compose.yml), créez le fichier logstash.conf avec le contenu suivant

```
input {
 http {}
}

filter {
 grok {
   match => [ "message" , "%{COMBINEDAPACHELOG}+%{GREEDYDATA:extra_fields}"]
   overwrite => [ "message" ]
 }
 mutate {
   convert => ["response", "integer"]
   convert => ["bytes", "integer"]
   convert => ["responsetime", "float"]
 }
 geoip {
   source => "clientip"
   target => "geoip"
   add_tag => [ "nginx-geoip" ]
 }
 date {
   match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
   remove_field => [ "timestamp" ]
 }
 useragent {
   source => "agent"
 }
}

output {
 elasticsearch {
   hosts => ["elasticsearch:9200"]
 }
 stdout { codec => rubydebug }
}
```

Ce fichier peu sembler un peu compliqué. Il peut être découpé en 3 parties:
* input: permet de spécifier les données d'entrée. Nous spécifions ici que Logstash peut recevoir des données (entrées de logs)  sur du http

* filter: permet de spécifier comment les données d'entrée doivent être traitées avant de passer à l'étape suivante. Plusieurs instructions sont utilisées ici:
  * grok permet de spécifier comment chaque entrée doit être parsée. De nombreux parseurs sont disponibles par défaut et nous spécifions ici (avec COMBINEDAPACHELOG) que chaque ligne doit être parsée suivant un format de log apache, cela permettra une extraction automatique des champs comme l'heure de création, l'url de la requête, l'ip d'origine, le code retour, ...
  * mutate permet de convertir les types de certains champs
  * geoip permet d'obtenir des informations géographique à partir de l'adresse IP d'origine
  * date est utilisée ici pour reformatter le timestamp

* output: permet de spécifier la destination d'envoi des données une fois que celles-ci sont passées par l'étape filter

## Lancement de la stack ELK

Pre-réquis: avant de lancer la stack ELK, il est nécessaire de modifier un paramètre du noyau Linux pour assurer le bon fonctionnement de Elasticsearch, pour cela lancer la commande suivante:

```
$ sudo sysctl -w vm.max_map_count=262144
```

Vous pouvez ensuite lancer la stack avec la commande suivante

```
$ docker-compose up -d
```

Une fois les images téléchargées (depuis le Docker Hub), le lancement de l'application peut prendre quelques secondes.

L'interface web de Kibana est alors accessible sur le port 5601 de la machine hôte.

![ELK](./images/elk1.png)

Cliquez sur l'option permettant de manipuler vos propres données

![ELK](./images/elk2.png)

Sur la page suivante, vous pourrez avoir un aperçu de toutes les fonctionnalités disponibles.
Cliquez sur *Discover*.

![ELK](./images/elk3.png)

Cette page montre qu'il n'y a pas encore de données dans Elasticsearch, Kibana n'est pas en mesure de détecter un index.

## Utilisation d'un fichier de logs de test

Nous allons maintenant utiliser un fichier de log de test et envoyer son contenu dans Logstash, contenu qui sera filtré et envoyé à Elasticsearch.

Nous utilisons pour cela l'image *mingrammer/flog* afin de générer des entrées de log au format Nginx. Le fichier nginx.log généré contient 1000 entrées de logs.

```
$ docker run mingrammer/flog -f apache_combined > nginx.log
```

La commande suivante permet d'envoyer chaque ligne à Logstash (assurez-vous d'avoir remplacé HOST par l'adresse IP de la machine sur laquelle la stack Elastic a été lancée)

```
while read -r line; do curl -s -XPUT -d "$line" http://HOST:8080; done < ./nginx.log
```

:fire: vous devriez voir une succession de *ok* s'afficher, cela permet simplement de s'assurer que l'envoi des entrées de log s'est déroulée correctement

Une fois le script terminé, cliquez sur le bouton permettant de rafraichir les données. Vous pourrez alors créer un index.

![ELK](./images/elk4.png)
![ELK](./images/elk5.png)

Depuis le menu *Discover* vous pourrez alors voir les logs que vous avez envoyés précédemment.

![ELK](./images/elk6.png)
![ELK](./images/elk7.png)


## En résumé

Nous avons vu ici la facilité de déploiement de la stack Elastic à l'aide de Docker Compose. Je vous invite à naviguer dans l'interface de Kibana afin d'avoir une vue globale des nombreuses fonctionnalités qui sont disponibles.
