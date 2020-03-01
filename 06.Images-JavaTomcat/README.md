# Un projet Java / Tomcat avec docker

Dans cet exercice, nous allons illustrer l'une des manière de faire tourner une application Java sous Tomcat. Nous verrons comment configurer cette application.

Pour la réalisation de cet exercice, on peut utiliser un simple daemon docker, une instance D.O, play-with-docker, etc.


## Récupérer le code source du projet example.

```
git clone https://gitlab.com/lucg/docker-exercices.git
cd docker-exercices/20.Java-Tomcat/sample-war-project

# si on dispose de maven
mvn package 
```

Regardez les sources du projet

```
.
├── pom.xml
└── src
    └── main
        └── webapp
            ├── WEB-INF
            │   ├── tomcat-web.xml
            │   └── web.xml
            └── index.jsp
```

Notez le paramètre de configuration dans `tomcat-web.xml` et son affichage dans `index.jsp`

## Le Dockerfile

Nous allons utiliser un multi-stage build qui nous permet de construire une image docker avec maven, mais qui ne contiendra au final que le runtime tomact et le war file de l'application.

A la racine du projet créer un fichier `Dockerfile` avec le contenu suivant :

```Dockerfile
FROM maven:3
COPY . /build
WORKDIR /build
RUN mvn package -DinteractiveMode=false

FROM tomcat:9
COPY --from=0 /build/target/sample-war-project $CATALINA_HOME/webapps/ROOT
```

> Note : sources de cette image : https://github.com/docker-library/tomcat

## Construction de l'image docker

```
docker build -t sample-war:1.0 .
```

Cela peut être un peu long, les artifacts sont systématiquement récupérés du repository.

Il sera peut-être possible un jour d'utiliser un cache disque lors du build ...

## Démarrer le container

```
docker run -ti -p 8080:8080 sample-war:1.0
```

L'exécution du container reste ici en avant-plan, on peut ainsi visualiser les logs qui sont envoyés en sortie standard.

Ouvrir le browser (ou `curl`) sur http://localhost:8080 (remplacer localhost par le nom DNS externe si besoin)

Stopper le container avec `ctrl-c`

## Injecter la configuration dans le container avec un fichier externe

Ouvrir un autre shell, créer un fichier `tomcat-web-production.xml` avec le contenu suivant : 

```xml
<!DOCTYPE web-app PUBLIC
 "-//Sun Microsystems, Inc.//DTD Web Application 2.3//EN"
 "http://java.sun.com/dtd/web-app_2_3.dtd" >

<web-app>
  <context-param>
    <param-name>myparam</param-name>
    <param-value>Production value</param-value>
  </context-param>
</web-app>
```

Cela sera notre fichier de configuration.

## Démarrer le container en injectant cette configuration

```bash
docker run -ti -p 8080:8080 \
  -v $(pwd)/tomcat-web.xml:/usr/local/tomcat/webapps/ROOT/WEB-INF/tomcat-web.xml \ 
  sample-war:1.0
```

Vérifier le changement dans l'application web.

## Configurer le container avec une variable d'environement Docker

L'utilisation de fichiers de configuration à travers des volumes est une manière simple et permet de rapidement **containeriser* une application. 

Pour encore plus de simplicité et de souplesse, il est possible de se passer d'un volume et d'injecter des variables d'environement lors du démarrage du container.

Nous allons utiliser un **entrypoint**.

Modifier le fichier `WEB-INF/tomcat-web.xml`

```xml
...
  <context-param>
    <param-name>myparam</param-name>
    <param-value>MYPARAM_VALUE</param-value>
  </context-param>
...
```

Ajouter un fichier `docker-entrypoint.sh` à la racine

```bash
#!/bin/bash
set -e

sed -i "s/MYPARAM_VALUE/$MYPARAM/g" /usr/local/tomcat/webapps/ROOT/WEB-INF/tomcat-web.xml

exec "$@"
```

Ajouter dans le `Dockerfile`

```Dockerfile
COPY docker-entrypoint.sh /usr/local/bin
RUN chmod a+x /usr/local/bin/docker-entrypoint.sh 
ENTRYPOINT ["docker-entrypoint.sh"]

# Note: If CMD is defined from the base image, setting ENTRYPOINT will reset CMD to an empty value. 
CMD ["catalina.sh", "run"]
```

Builder à nouveau l'image, mais démarrer le container comme suit : 

```bash
docker run -ti --rm -p 8080:8080 -e MYPARAM="My production value"  sample-war:1.0
```

Cet exemple est minimaliste, l'utilisation de `sed` (quand le nombre de valeur augmente et que l'on doit **escaper** les valeurs) peut devenir rapidement compliquée.

Il sera alors mieux d'utiliser un outil de templating un peu plus puissant comme **envtpl/jinja2**
