Pour suivre les étapes suivantes, vous pouvez vous appuyer sur le post [Even the smallest side project deserves its CI/CD pipeline](https://medium.com/lucjuggery/even-the-smallest-side-project-deserves-its-ci-cd-pipeline-281f80f39fdf)

1. Créez un compte sur [Gitlab.com](https://gitlab.com)

2. Créez un projet du nom de votre application

3. Installez [Portainer](https://portainer.io) sur votre Swarm

4. En vous basant sur l'exemple de [sophia.events](https://gitlab.com/lucj/sophia.events/), créez un fichier *.gitlab-ci.yaml* dont les stages seront les suivants:

- test: tests basiques de l'application

- build: construction de l'image

- push: envoi de l'image dans le DockerHub ou dans le registry de gitlab

- deploy: mise à jour de l'application tournant sur le Swarm en utilisant un *webhook* de Portainer
