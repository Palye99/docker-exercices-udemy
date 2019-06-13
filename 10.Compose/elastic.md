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
version: '3.6'
services:
  logstash:
    image: logstash:6.7.1
    volumes:
      - ./logstash.conf:/config/logstash.conf
    command: ["logstash", "-f", "/config/logstash.conf"]
    ports:
      - 8080:8080
  elasticsearch:
    image: elasticsearch:6.7.1
  kibana:
    image: kibana:6.7.1
    ports:
      - 5601:5601
```

Note:

- Le service Logstash est basé sur l'image officielle logstash:6.7.1.
Nous précisons sous la clé volumes le fichier de configuration logstash.conf présent dans le répertoire est monté sur /config/logstash.conf dans le container afin d'être pris en compte au démarrage

- Le service Kibana est basé sur l'image officielle kibana:6.7.1. Le mapping de port permettra à l'interface web d'être disponible sur le port 5601 de la machine hôte.

## Fichier de configuration de Logstash

Afin de pouvoir indexer des fichiers de logs existant, nous allons configurer Logstash. Dans le réperoire *elk* (ou se trouve le fichier docker-compose.yml), créez le fichier logstash.conf avec le contenu suivant

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
* input: permet de spécifier les données d'entrée. Nous spécifions ici que logstash peut recevoir des données (entrées de logs)  sur du http

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

Il n'y a pas encore de données dans Elasticsearch, Kibana n'est pas en mesure de détecter un index.

## Utilisation d'un fichier de logs de test

Nous allons maintenant utiliser un fichier de log de test et envoyer son contenu dans Logstash, contenu qui sera donc filtrer et envoyé à Elasticsearch.

Récupérez en local le fichier nginx.log avec la commande suivante :

```
curl -s -o nginx.log https://gist.githubusercontent.com/lucj/83769b6a74dd29c918498d022442f2a0/raw
```

Ce fichier contient 500 entrées de logs au format Apache. Par exemple, la ligne suivante correspond à une requête :
- reçue le 28 novembre 2018
- de type GET
- appelant l'URL https://mydomain.net/api/object/5996fc0f4c06fb000f83b7
- depuis l'adresse IP 46.218.112.178
- et depuis un navigateur Firefox

```
46.218.112.178 - - [28/Nov/2018:15:40:04 +0000] "GET /api/object/5996fc0f4c06fb000f83b7 HTTP/1.1" 200 501 "https://mydomain.net/map" "Mozilla/5.0 (Windows NT 10.0; WOW64; rv:55.0) Gecko/20100101 Firefox/55.0" "-"
```

On utilise alors la commande suivante pour envoyer chaque ligne à Logstash:

```
while read -r line; do curl -s -XPUT -d "$line" http://localhost:8080; done < ./nginx.log
```

Une fois le script terminé, allez dans le menu *discover* il vous sera demandé de créer un index. Ceci est maintenant possible car des entrées de logs ont été indéxées par Elasticsearch.

![ELK](./images/elk2.png)
![ELK](./images/elk3.png)

Assurez vous d'avoir une période de recherche assez large afin de couvrir la date de l'entrée que nous avons spécifiée, vous pouvez configurer cette période en haut à droite de l'interface.

![ELK](./images/elk4.png)

A partir de ces données, nous pouvons par exemple créer une visualisation permettant de lister les pays d'ou proviennent ces requêtes.

![ELK](./images/elk5.png)

En allant un peu plus loin, nous pouvons, pour chaque pays, faire un découpage supplémentaire sur le code retour de la requête.

![ELK](./images/elk6.png)

Nous pourrions ensuite, grace aux filtres, voir de quels pays proviennent les requêtes dont le code de retour est 401 (Unauthorized).

## En résumé

Nous avons vu ici une nouvelle fois la puissance et la facilité d'utilisation de Docker Compose. En peu de temps nous avons déployé et utilisé une stack ELK. Bien sur, ce que l'on a vu ici n'est qu'un aperçu. Je vous invite à utiliser d'autres fichiers de log, à modifier la configuration de logstash, à créer d'autres visualisation et des dashboard regroupant plusieurs visualisations. Cela vous permettra de découvrir d'autres fonctionnalités parmi les nombreuses qui sont disponibles.
