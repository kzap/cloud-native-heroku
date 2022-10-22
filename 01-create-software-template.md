# Create Software Templates

Backstage allows you to have your own software template with scaffolding and all
the actions you'd like to take when a new service is created.

## Hello World

Backstage allows you to define software templates together with the code
scaffolding that will be used in the initial commit of the Git repo it creates.
We'll create a hello world template to get a taste of what it does and how.

We can have the templates as part of the Backstage app and when we run `yarn
build` they would be included. But in order to create and add them step by step,
we will create a Github repository for Backstage to pull the templates from.

Create a new repository in Github called and have a `templates` folder in it
with a new folder called `00-only-github`.
```bash
# We are in https://github.com/muvaf/cloud-native-heroku
mkdir -p templates/00-only-github
```

We'll create the following template object which just creates a repo and
bootstraps it with the content in `skeleton` folder.
```yaml
# Content of templates/00-only-github/skeleton
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: bootstrap-nodejs-repo
  title: Bootstrap Nodejs Repo
spec:
  owner: muvaf/kubecon-na-2022
  type: service

  parameters:
    - title: Provide metadata
      required:
        - serviceName
        - owner
      properties:
        serviceName:
          title: Service Name
          type: string
          description: Unique name of the component
          ui:field: EntityNamePicker
        owner:
          title: Owner
          type: string
          description: Owner of the component
          ui:field: OwnerPicker
          ui:options:
            allowedKinds:
              - Group
    - title: Choose a location
      required:
        - repoUrl
      properties:
        repoUrl:
          title: Repository Location
          type: string
          ui:field: RepoUrlPicker
          ui:options:
            allowedHosts:
              - github.com

  steps:
    - id: fetch-base
      name: Fetch Base
      action: fetch:template
      input:
        url: ./skeleton
        values:
          serviceName: ${{ parameters.serviceName }}
          owner: ${{ parameters.owner }}

    - id: publish
      name: Publish
      action: publish:github
      input:
        allowedHosts: ['github.com']
        description: This is ${{ parameters.name }}
        repoUrl: ${{ parameters.repoUrl }}
        repoVisibility: public

    - id: register
      name: Register
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: '/catalog-info.yaml'
```

In `skeleton` folder, we'll have our very simple hello world application.
```bash
mkdir -p templates/00-only-github/skeleton
```

A `server.js` and `package.json` is all we need for NodeJS to work. A
`Dockerfile` to build an image and a `catalog-info.yaml` for Backstage to
identify the application will be there.
```yaml
# Content of templates/00-only-github/skeleton/catalog-info.yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: ${{values.serviceName | dump}}
spec:
  type: service
  lifecycle: experimental
  owner: ${{values.owner | dump}}
```
```dockerfile
# Content of templates/00-only-github/skeleton/Dockerfile
FROM node:16

WORKDIR /usr/src/app

COPY package*.json ./
RUN npm install

COPY server.js .
CMD [ "node", "server.js" ]
```
Content of `templates/00-only-github/skeleton/package.json`
```json
{
    "name": "hello-world",
    "version": "1.0.0",
    "description": "Kubecon NA demo",
    "author": "First Last <first.last@example.com>",
    "main": "server.js",
    "scripts": {
      "start": "node server.js"
    },
    "dependencies": {
      "express": "^4.16.1"
    }
  }
```
Content of `server.js`
```javascript
'use strict';

const express = require('express');

// Constants
const PORT = 8080;
const HOST = '0.0.0.0';

// App
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World! My name is ${{ values.serviceName }} and my owner is ${{ values.owner }}');
});

app.listen(PORT, HOST, () => {
  console.log(`Running on http://${HOST}:${PORT}`);
});
```

Now let's create a commit and push it to our Git repo.

Visit `http://127.0.0.1:7007/catalog-import` and supply the path of
`template.yaml` in your Git repo. For example:
```
https://github.com/muvaf/cloud-native-heroku/blob/main/templates/00-only-github/template.yaml
```

When you click `Create...` on the side bar now, you'll see that there is a new
template called `Bootstrap Nodejs Repo`. Go ahead and choose it to bootstrap a
new repo.

![Hello world template for Backstage](assets/only-github-instance-created.png)

## Add Image and Helm Chart

In this template we will add image building capabilities and a Helm chart that
can be deployed to a cluster to deploy our application.

Copy the earlier template to make changes.
```bash
cp -a templates/00-only-github templates/01-image-chart
```

Change the `metadata` of `template.yaml`
```yaml
metadata:
  name: hello-world-on-kubernetes
  title: Hello World on Kubernetes
```

Let's add a basic Helm chart.
```bash
mkdir -p templates/01-image-chart/skeleton/chart
```
It will have chart metadata and basic `Deployment` and `Service` definitions.
```bash
mkdir -p templates/01-image-chart/skeleton/chart/templates
```
```yaml
# Content of templates/01-image-chart/skeleton/chart/Chart.yaml
apiVersion: v2
name: ${{ values.serviceName }}
description: A Helm chart for ${{ values.serviceName }} owned by ${{ values.owner }}
type: application
version: 0.1.0
appVersion: "1.16.0"
```
```yaml
# Content of templates/01-image-chart/skeleton/chart/values.yaml
registry: ghcr.io
image: ${{ values.githubRepositoryOrg }}/${{ values.githubRepositoryName }}
tag: v0.1.0
```
```yaml
# Content of templates/01-image-chart/skeleton/chart/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  selector:
    app: hello-world
  ports:
    - name: http
      port: 80
      targetPort: http
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
          image: ${{ .Values.image.registry }}/${{ .Values.image.registry }}:{{ .Chart.Version }}
          ports:
            - name: http
```

Now we will add a `.github` folder that will contain Github Actions workflow
to build the image and Helm chart as OCI image, and then push both to Github
Container Registry (GHCR). 

Create the following file in `.github/workflows/ci.yaml`
```bash
mkdir -p templates/01-image-chart/.github/workflows
```
```yaml
# Content of templates/01-image-chart/skeleton/.github/workflows/ci.yaml
name: Create and publish a Docker image

on:
  push:
    branches: ['main']

jobs:
  build-and-push-image:
    runs-on: ubuntu:22.04
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to the Container registry
        uses: docker/login-action@f054a8b539a109f9f41c372932f1ae047eff08c9
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@98669ae865ea3cffbcbaa878cf57c20bbf1c6c38
        with:
          images: ghcr.io/${{ github.repository }}

      - name: Build and push Docker image
        uses: docker/build-push-action@ad44023a93711e3deb337508980b4b5e9bcdc5dc
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```