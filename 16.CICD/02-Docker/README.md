# Packaging in a Docker image

## Adding a Dockerfile

The first step is to create a Dockerfile at the root of your repository. This one defines the steps needed to create an image containing your application and all its dependencies.

## Sample Dockerfile for a Java project

```
FROM openjdk:8-jdk
COPY . /
RUN ./mvnw --batch-mode package

FROM openjdk:8-jdk
COPY --from=0 target/*.jar app.jar
ENV SERVER_PORT=8000
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
EXPOSE 8000
```

## Sample Dockerfile for a Python project

```
FROM python:3
COPY . /app
WORKDIR /app
RUN pip install -r requirements.txt
EXPOSE 8000
CMD python /app/app.py
```

## Sample Dockerfile for a Node.js project

```
FROM node:12-alpine
COPY . /app
WORKDIR /app
RUN npm i
EXPOSE 8000
CMD ["npm", "start"]
```

## Sample Dockerfile for a Go project

```
FROM golang:1.12-alpine as build
WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

FROM scratch
COPY --from=build /app/main .
CMD ["./main"]
```


## Build the image

Once the Dockerfile is created, you can build the image with the following command:

```
$ docker build -t api .
```

## Run the application from the image

Let's test the application is running fine when it's launched in a container:

```
$ docker run -ti -p 8000:8000 api
```

You should be able to access it from localhost on port 8000:

```
$ curl http://localhost:8000
```

You can stop the docker container with `ctrl-c`

[Let's create the project in Gitlab](../03-Gitlab)
