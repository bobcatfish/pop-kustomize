# Demo: Devops Best Practices

This repo is a fork of https://github.com/nateaveryg/pop-kustomize which is meant to demonstrate
setting up a project in GCP that follows devops best practices.

The demo will be used to display:
- [x] Cloud workstations
- [x] GCB triggering
- [x] Build and push to AR
- [] Cloud Deploy promotion across environments
- [] Image scanning (AR can do this on push)
- [] Provenance generation
- [] Cloud Deploy canary deployment
- [] Cloud Deploy label: allows link back to git sha
- [] DORA stats (Cloud Deploy?)
- [] Security insights in Cloud Deploy (can show cloud build panel at least, ideally both)
- [] Binauthz gating of deployment

## Setup tutorial

After forking the repo, follow along here.

## Setup: enable APIs

Set the PROJECT_ID environment variable. This variable will be used in forthcoming steps.

```bash
export PROJECT_ID=<walkthrough-project-id/>
# sets the current project for gcloud
gcloud config set project $PROJECT_ID
# Enables various APIs you'll need
gcloud services enable \
  container.googleapis.com \
  cloudbuild.googleapis.com \
  artifactregistry.googleapis.com \
  clouddeploy.googleapis.com \
  cloudresourcemanager.googleapis.com \
  secretmanager.googleapis.com \
  containeranalysis.googleapis.com
```

## Add CI

### Setup a Cloud Build trigger on PRs

Configure Cloud Build to run each time a change is pushed to the main branch. To do this, add a Trigger in Cloud Build:
  1. Follow https://cloud.google.com/build/docs/automating-builds/github/connect-repo-github to connect
     your GitHub repo
  2. Follow https://cloud.google.com/build/docs/automating-builds/github/build-repos-from-github?generation=2nd-gen to setup triggering:
    * Setup PR triggering to run cloudbuild-test.yaml
    * Setup triggering on the `main` branch to run cloudbuild-deploy.yaml

## Add Deployment

### Setup AR repo to push images to

```bash
gcloud artifacts repositories create pop-stats
  --location=us-central1 \
  --repository-format=docker \
   --project=$PROJECT_ID
```

### Create Google Cloud Deploy pipeline

Create the cloud deploy pipeline:
```bash
# customize the clouddeploy.yaml 
sed -i "s/project-id-here/${PROJECT_ID}/" clouddeploy.yaml
# creates the Google Cloud Deploy pipeline
gcloud deploy apply --file clouddeploy.yaml \
  --region=us-central1 --project=$PROJECT_ID
```

Verify that the Google Cloud Deploy pipeline was created in the 
[Google Cloud Deploy UI](https://console.cloud.google.com/deploy/delivery-pipelines)

### Create Google Cloud Deploy pipeline

Create the GKE clusters:
* testcluster
* stagingcluster
* productcluster

```bash
./bootstrap/gke-cluster-init.sh
```

Verify that they were created in the [GKE UI](https://pantheon.corp.google.com/kubernetes/list/overview)

### Setup a Cloud Build trigger to deploy on merge to main

#### IAM and service account setup
You must give Cloud Build explicit permission to trigger a Google Cloud Deploy release.
1. Read the [docs](https://cloud.google.com/deploy/docs/integrating-ci)
2. Navigate to [IAM](https://console.cloud.google.com/iam-admin/iam)
  * Check "Include Google-provided role grants"
  * Locate the service account named "Cloud Build service account"
3. Add these two roles
  * Cloud Deploy Releaser
  * Service Account User

#### Set up the trigger

Follow https://cloud.google.com/build/docs/automating-builds/github/build-repos-from-github?generation=2nd-gen to setup triggering:
  * Setup triggering on the `main` branch to run cloudbuild-deploy.yaml

Do a push to the `main` branch and observe that the Cloud Deploy rollout is successful
at https://pantheon.corp.google.com/deploy/delivery-pipelines.

## (Optional) Turn on automated container vulnerability analysis
Google Cloud Container Analysis can be set to automatically scan for vulnerabilities on push (see [pricing](https://cloud.google.com/container-analysis/pricing)). 

Enable Container Analysis API for automated scanning:

<walkthrough-enable-apis apis="containerscanning.googleapis.com"></walkthrough-enable-apis>

```bash
gcloud services enable \
  containerscanning.googleapis.com
```


## Demo Overview

[![Demo flow](https://user-images.githubusercontent.com/76225123/145627874-86971a34-768b-4fc0-9e96-d7a769961321.png)](https://user-images.githubusercontent.com/76225123/145627874-86971a34-768b-4fc0-9e96-d7a769961321.png)

The demo flow outlines a typical developer pathway, submitting a change to a Git repo which then triggers a CI/CD process:
1. Push a change the main branch of your forked repo. You can make any change such as a trivial change to the README.md file.
2. A Cloud Build job is automatically triggered, using the <walkthrough-editor-open-file filePath="cloudbuild.yaml">
cloudbuild.yaml</walkthrough-editor-open-file>  configuration file, which:
  * builds and pushes impages to Artifact Registry
  * creates a Google Cloud Deploy release in the pipeline
3. You can then navigate to Google Cloud Deploy UI and shows promotion events:
  * test cluster to staging clusters
  * staging cluster to product cluster, with approval gate


## Tear down

To remove the three running GKE clusters, run:
```bash
. ./bootstrap/gke-cluster-delete.sh
```

## Local dev (optional)

To run this app locally, start minikube:
```bash 
minikube start
```

From the pop-kustomize directory, run:
```bash
skaffold dev
```

The default skaffold settings use the "dev" Kustomize overlay. Once running, you can make file changes and observe the rebuilding of the container and redeployment. Use Ctrl-C to stop the Skaffold process.

To test the staging overlays/profile run:
```bash
skaffold dev --profile staging
```

To test the staging overlays/profile locally, run:
```bash
skaffold dev --profile prod
```
## About the Sample app - Population stats

Simple web app that pulls population and flag data based on country query.

Population data from restcountries.com API.

Feedback and contributions welcomed!
