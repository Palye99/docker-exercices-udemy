## Skelette d'une application DotNetCore

Généré avec la commande suivante:

```
$ dotnet new webapi -o webapi
```

## Compilation du projet

```
$ cd webapi
$ dotnet restore
$ dotnet publish -c Release -o out
```

## Lancement du serveur

```
$ dotnet out/webapi.dll
```

## Test

```
$ curl https://localhost:5000/api/values
["value1","value2"]
```
