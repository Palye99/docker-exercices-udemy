En utilisant le langage de votre choix, développez un serveur web simple ayant les caractéristiques suivantes:
- écoute sur le port 8000
- expose le endpoint */* en GET
- retourne la chaine 'Hi!' pour chaque requète reçue.

Note: vous pouvez utiliser l'un des templates ci-dessous, réalisé dans différents langages:

- NodeJs
- Python
- Ruby
- Go
- Java
- .net core

## NodeJs

### Code source

- index.js

```
var express = require('express');
var app = express();
app.get('/', function(req, res) {
    res.setHeader('Content-Type', 'text/plain');
    res.end("Hi!");
});
app.listen(8000);
```

- package.json

```
{
  "name": "www",
  "version": "0.0.1",
  "main": "index.js",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "express": "^4.14.0"
  }
}
```

### Installation de expressjs

```
$ npm install
```

### Lancement du serveur

```
$ npm start
```

### Test

```
$ curl localhost:8000
Hi!
```

---


## Python

### Code source

- app.py

```
from flask import Flask
app = Flask(__name__)

@app.route("/")
def hello():
    return "Hi!"

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=8000)
```

- requirements.txt

```
Flask==1.0.2
```

### Installation des dépendances

```
$ pip install -r requirements.txt
```

### Lancement du serveur

```
$ python app.py
```

### Test

```
$ curl localhost:8000
Hi!
```

---

## Ruby

### Code source

- app.rb

```
require 'sinatra'
set :port, 8000
get '/' do
  'Hi!'
end
```

- Gemfile

```
source :rubygems
gem "sinatra"
```

### Installation des dépendances

```
$ bundle install
```

### Lancement du serveur

```
$ ruby app.rb -s Puma
```

### Test

```
$ curl localhost:8000
Hi!
```

--

## Go

### Code source

- main.go

```
package main

import (
        "io"
        "net/http"
)

func handler(w http.ResponseWriter, req *http.Request) {
    io.WriteString(w, "Hi!")
}

func main() {
        http.HandleFunc("/", handler)
        http.ListenAndServe(":8000", nil)
}
```

### Lancement du serveur

```
$ go run main.go
```

### Test

```
$ curl localhost:8000
Hi!
```

---

## Java / Spring

Généré depuis [https://start.spring.io/](https://start.spring.io/) en utilisant les options suivantes

![Spring generator](./images/spring-generator.png)

### Packaging de l'application

```
$ ./mvnw package
```

### Lancement du serveur

```
$ java -jar target/demo-0.1.jar
```

### Test

```
$ curl http://localhost:8080/
hello World
```


## DotNetCore

### Génération du projet

Généré avec la commande suivante:

```
$ dotnet new webapi -o webapi
```

### Compilation

```
$ cd webapi
$ dotnet restore
$ dotnet publish -c Release -o out
```

### Lancement du serveur

```
$ dotnet out/webapi.dll
```

### Test

```
$ curl https://localhost:5000/api/values
["value1","value2"]
```

Dans la partie suivante, nous allons ajouter un Dockerfile à notre application.
