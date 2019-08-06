# Deploying a React Web App with Tekton Pipelines

This repository contains a simple, generated React.js application,
and a bunch of YAML files to set up test and production deployment
pipelines, along with a [Knative Service](https://knative.dev/docs/serving/).

## Web Application

The web application in `src` is just a generated React.js application
with a single added `script`.

```json
"deploy:test": "nodeshift build --dockerImage=nodeshift/centos7-s2i-web-app --imageTag=10.x"
```

This script uses the [`nodeshift`](https://github.com/nodeshift/nodeshift) module
to create an [`s2i`](https://github.com/openshift/source-to-image) binary build,
resulting in an OpenShift `ImageStream`.

To run the application locally, from the `src` directory run `npm install && npm start`.
In test and production, the application is served via nginx.

## Deployment Pipelines

We use a single parameterized Tekton `Pipeline` to deploy the application images to
both test and production. A Knative `Service` runs and scales the image and creates
a `Route` to it.

## Deploying the `Pipeline`, `Task` and `Resource` Objects 

The deployment pipelines require a privileged service account. To set this up, be sure
you are logged into an openshift instance with admin privileges and execute the
`setup-service-account.sh` script. Now you are ready to install and configure the
Kubernetes objects that comprise the `Pipeline` and `Service`.

* Create the service account and grant it privileges
    * `./setup-service-account.sh`
* Install the pipeline
    * `oc apply -f deploy/webapp-pipeline.yaml`
* Install the tasks
    * `oc apply -f deploy/task/s2i.yaml`
    * `oc apply -f deploy/task/openshift-client-task.yaml`
    * `oc apply -f deploy/task/webapp-build-runtime-task.yaml`
* Install the test deployment `Resource`s
    * `oc apply -f deploy/test/webapp-test-build-resource.yaml`
    * `oc apply -f deploy/test/webapp-test-runtime-resource.yaml`
* Install the production deployment `Resource`s
    * `oc apply -f deploy/production/webapp-prod-build-resource.yaml`
    * `oc apply -f deploy/production/webapp-prod-runtime-resource.yaml`

## Running the Pipeline Build

The pipeline for this application assumes that there is an existing image that
contains the built HTML, CSS and JavaScript files. This image is started and
the files are copied to a new image with nginx configured and run at startup. 

### Test Deployment

For the test workflow, the build image comes from the `deploy:test` script in
the web application's package.json file. To deploy this image run the script.

```sh
npm run deploy:test
```

Once the build image has been created, you can run the pipeline to generate
the nginx runtime image.

```sh
oc apply -f deploy/test/webapp-test-pipeline-run.yaml
```

### Production Deployment

For the production workflow, we assume that we don't want an image that has been
created by running nodeshift on a developer's computer. Instead, we want to use
the committed source in the GitHub repository, and run an `s2i` build on it use
the `s2i` `Task` we installed earlier. To do this, apply the `TaskRun` script.

```sh
oc apply -f deploy/production/webapp-prod-build-taskrun.yaml
```

This will create an image containing the static files that will be copied to a
new nginx image for runtime. To create the runtime image, execute the production
pipeline by applying the `PipelineRun` object.

```sh
oc apply -f deploy/production/webapp-prod-pipeline-run.yaml
```

## Starting the Knative Service

The Knative `Service` sets up networking (routes/ingress) for the application
and ensures there are sufficient replicas of the running container. To deploy
the `Service` run the following.

```sh
oc apply -f deploy/production/production-service.yaml
```
