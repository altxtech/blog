---
title: "Deploy to Cloudrun with Terraform and Github Actions"
date: 2024-01-17T21:27:10-03:00
publishDate: 2024-01-19T12:00:00-3:00
draft: false
author: André Luiz Tiago Soares
tags: ['Guide', 'Devops', 'Cloud', 'Terraform', 'Github Actions', 'GCP']
---

![cover](cover.png)

## Intro

This post is going to be about how to setup a basic CI/CD pipeline for Google Cloud Run, by using Terraform (or OpenTofu) for Infrastructure as Code (IaC), and Github Actions for CI/CD.

Here are some advantages of deploying Cloud Run apps like this:
- Deploy any supporting cloud resources along with you app
- Single deployment step (`terraform plan` & `terraform apply`)
- Easy environment replication
- Keep your Cloud Infrastructure organized
- Works great for applications composed by multiple Cloud Run services
- Simplify you CI/CD pipelines

First I'll focus just on the Terraform configurations and just deploying the enviroment locally from the command line. And then on how to automate that with Github Actions.

If you don't want to implement everything from scratch, I have also made this pipeline available as public repository template in my [Github](https://github.com/altxtech/cloudrun-project-template). 
It takes about 20 minutes to configure and also deploys a Firestore database and Secret Manager secret with preconfigured permissions.

## Prerequisites
- A [GCP Project](https://cloud.google.com/resource-manager/docs/creating-managing-projects)
- A [Cloud Storage bucket](https://cloud.google.com/storage/docs/creating-buckets) for the terraform state
- [gcloud CLI](https://cloud.google.com/sdk/docs/install#linux) installed and configured to access you project
- [Terraform](https://developer.hashicorp.com/terraform/install) or [OpenTofu](https://opentofu.org/docs/intro/install/)

## Terraform backend configuration and providers

For providers, we're going to need:
- `docker`, to build and push images 
- `google` to deploy Artifact Registry repositories and Cloud Run services

For the variables, I recommed setting things up the same way I usually do. This will make 
it easier to replicate configs between environments later.

I always like to include, in all my GCP configs: 
- `project_id` and `region` for replication between environments
- `service_name` and `env` for consistent naming conventions and namespacing of resources

We are also going to need:
- `gcloud_access_token` for docker authentication

I also don't set the backend configuration specification, and instead prefer to pass those through `--backend-config`. This is also make replication between environments easier.

When we get to the Github Actions part this all of this is going to make sense.

Here is what it looks like:

```hcl
# 1.0 BACKEND
terraform {
 backend "gcs" {}
 required_providers {
	 docker = {
		 source  = "kreuzwerker/docker"
		 version = "3.0.2"
	 }
 }
}

# 1.1 VARIABLES

variable "project_id" {
  description = "Google Cloud Project ID"
}

variable "region" {
  description = "Google Cloud region"
  default     = "us-west1"
}

variable "service_name" {
  description = "Name of the service. Defines the resource names of both the AR repository and Cloud Run service."
}

variable "env" {
  description = "Name of the environment (e.g dev or prod)"
  default     = "dev"
}

variable "gcloud_access_token" {
	description = "Access token of the gcloud CLI. Needed for docker auth"
	sensitive = true
}

# 1.3 PROVIDERS
provider "google" {
  project = var.project_id
  region  = var.region
}

provider "docker" {
	host = "unix:///var/run/docker.sock"
	registry_auth {
	  address = "${var.region}-docker.pkg.dev"
	  username = "oauth2accesstoken"
	  password = var.gcloud_access_token
	}
}
```

Take notice on how the the `gcloud_access_token` was used to configure the authentication for docker.

## Build and push a docker image to Artifact Registry with Terraform

To build and push the image from within Terraform, we are going to first create our GAR repository with [google_artifact_registry_repository](https://registry.terraform.io/providers/hashicorp/google/latest/docs/resources/artifact_registry_repository), then build the image with [docker_image](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/image) and the push it with [docker_registry_image](https://registry.terraform.io/providers/kreuzwerker/docker/latest/docs/resources/registry_image)

```hcl
# 2 RESOURCES

# 2.1 ARTIFACT REGISTRY REPO
resource "google_artifact_registry_repository" "repo" {
  location      = var.region
  repository_id = "${var.service_name}-${var.env}"
  format        = "DOCKER"
}

# 2.2 BUILD AND PUSH IMAGE
resource "docker_image" "build_image" {
  name = "${var.region}-docker.pkg.dev/${var.project_id}/${var.service_name}-${var.env}/${var.service_name}-${var.env}:1.0"

  build {
    context = "${path.cwd}"
  }
}

resource "docker_registry_image" "image" {
  name = docker_image.build_image.name
}
```

Now, to deploy this configuration, first make sure you have gcloud CLI configured, and then make sure it is properly authenticated. 

```bash
gcloud auth login
```

And:
{{< highlight bash >}}
gcloud auth application-default login
{{< / highlight >}}

Both commands will start an oauth flow to authenticate the gcloud CLI.  

Next create a .tvars file like this example:
```hcl
project_id = "your-gcp-project-id"
region = "us-west1"
env = "dev"
service_name = "dockertest"
```

Then, on the directory where you main.tf is located, initialize terraform.

```bash
tofu init \ 
--backend-config "bucket=your-tf-state-bucket" \
--backend-config "prefix=your-tf-state-prefix"
```

Make sure docker is running. In WSL2 this can be done with:
```bash
sudo service docker start
```

Finally, deploy your infrastructure. 
```bash
terraform apply \
--var-file .tfvars \
--var "gcloud_access_token=$(gcloud auth print-access-token)"
	
```

So far, this will deploy an Artifact Registry repository, build a Docker image and push it to the previously deployed
repository. The next step is to create the Cloud Run service and have it use the image we just pushed.

## Deploying the Cloud Run service 

Next, just include the following in your main.tf.
```hcl
# 2.3 SERVICE ACCOUNT

resource "google_service_account" "sa" {
  account_id   = "${var.service_name}-${var.env}-svc"
  display_name = "Service account for cloud run"
}

# 2.4 SERVICE

# Define a Google Cloud Run service
resource "google_cloud_run_service" "app" {
  name     = "${var.service_name}-${var.env}"
  location = var.region 

  template {
    spec {
      containers {

        image = docker_registry_image.image.name
        env {
          name  = "ENV"
          value = var.env
        }
      }
      service_account_name = google_service_account.sa.email
    }
  }
}
```

Now, redeploy the configuration with:
```bash
terraform apply \
--var-file .tfvars \
--var "gcloud_access_token=$(gcloud auth print-access-token)"
	
```

Terraform should also create a Cloud Run service that uses the image that was pushed to the repository. 
Right now, we are already at the point where we can deploy everything we need from the local terminal. Now, let's move
on to automating these steps with Github Actions.

## Github Actions CI/CD pipeline

Since Terraform is doing most of the heavy lifting, specially by building and pushing our docker
image, our Github Workflow file is ends up actually being pretty simple. It is barely an "wrapper" script that just call
our terraform commands.  

The workflow file below will also support deployng to the prod and dev environments by pushing to the `main` and `dev` branches
respectively.

Add this to .github/workflows/deploy.yaml

```yaml
name: Build and Deploy to Cloud Run through OpenTofu

on:
  push:
    branches: [ "main", "dev" ]

jobs:
  deploy:
    permissions:
      contents: 'read'
      id-token: 'write'

    runs-on: ubuntu-latest
    environment:  ${{ fromJSON('["dev", "prod"]')[github.ref_name == 'main'] }}
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Google Auth
        id: auth
        uses: 'google-github-actions/auth@v2'
        with:
          token_format: 'access_token'
          workload_identity_provider: '${{ vars.WIF_PROVIDER }}' 
          service_account: '${{ vars.WIF_SERVICE_ACCOUNT }}' 

      - name: Tofu Setup
        uses: opentofu/setup-opentofu@v1

      - name: Tofu Init
        run: tofu init --backend-config "bucket=${{ vars.TF_BUCKET }}" --backend-config "prefix=${{ vars.TF_PREFIX }}"

      - name: Tofu Select Workspace
        if: ${{ env.tf_workspace != '' }}
        run: tofu workspace select --or-create ${{ vars.tf_workspace }}

      - name: Tofu Plan
        run: |
          tofu plan \
          --var "project_id=${{ vars.PROJECT_ID }}" \
          --var "region=${{ vars.REGION }}" \
          --var "service_name=${{ vars.SERVICE }}" \
          --var "env=${{ vars.ENV }}" \
          --var "gcloud_access_token=${{ steps.auth.outputs.access_token }}" \
          --out tofu-plan

      - name: Tofu Apply
        run: tofu apply tofu-plan

```

Here is a quick overview on what's happening:
1. Checkout
2. Authenticate the gcloud cli using Workload Identity Federation. 
3. Setup, initialize and select workspace with OpenTofu (for pratical purposes, it is the same thing as Terraform) 
4. Plan and deploy our application (which will include the docker build and push steps

When you commit and push it to your repository, at this point, the deployment will fail. There are two reasons for that.  
The first error we are going to get is in the 'Google Auth' step, because we still didn't gave permission for Github Actions
to manage infrastructure in our GCP account. The second reason is because we have some repository environment variables to set.  

We are going over these 2 things next.


## Configuring Workload Identity Federation (WIF) for Github Actions

To configure WIF, we have to basically do 2 things:
1. Create Service Account for Github Actions to use and deploy resources
2. Use a WIF pool to tell GCP that our repository workflows are authorized to use that Service Account

I'm not going into much depth on the inner workings of WIF. You can read more about it [here](https://cloud.google.com/iam/docs/workload-identity-federation). Instead,
here is how you set it up as quickly as possible using the gcloud CLI. You can use the same service account and WIF provider
if you intend to deploy both `prod` and `dev` environments in the same GCP project. But if you intend to isolate your enviroments in
different projects (which I recommend) you are going to have to run these steps twice, once for each enviroment / project.

Create a Service Account
```bash
$WIF_SERVICE_ACCOUNT_NAME=devops-svc
gcloud iam service-accounts create $WIF_SERVICE_ACCOUNT_NAME \
    --description="Devops account for Cloud Run projects" \
    --display-name="Devops Service Account for Cloud Run"
```

Create a WIF pool
```bash
gcloud iam workload-identity-pools create "ci-cd" \
  --location="global" \
  --display-name="CI/CD"
```

Create a WIF provider
```bash
gcloud iam workload-identity-pools providers create-oidc "github-actions" \
  --location="global" \
  --workload-identity-pool="ci-cd" \
  --display-name="Github Actions" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

You can then, print the workload identity pool name with:
```bash
gcloud iam workload-identity-pools describe ci-cd --location global
```

Finally, give Github Actions access to impersonate this account.
```bash
$PROJECT_ID=your-gcp-project-id
$GH_REPO=your-org/your-repo
gcloud iam service-accounts add-iam-policy-binding "${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --project="${PROJECT_ID}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${WIF_POOL}/attribute.repository/${GH_REPO}"
```

So, at this point, Github Actions can impersonate that account, but the account itself has no permissions within the project. So we have to give it some. The command below will give the account admin access to Artifact Registry, Cloud Run, IAM Service Accounts and Project IAM.
```bash
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/artifactregistry.admin" \
	--condition=None

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountUser" \
	--condition=None

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountAdmin" \
	--condition=None

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/resourcemanager.projectIamAdmin" \
	--condition=None

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/run.admin" \
	--condition=None
```

The account also needs access to manage state in the TF bucket
```bash	
TF_BUCKET=your-terraform-bucket
gcloud storage buckets add-iam-policy-binding gs://${TF_BUCKET} \
        --member="serviceAccount:${WIF_SERVICE_ACCOUNT_NAME}@${PROJECT_ID}.iam.gserviceaccount.com" \
        --role=roles/storage.objectUser
```

And that's all we need for the authorization setup. The only thing remaining to do is is configuring some enviroment variables.


## Configuring repository Environment Variables

There are three sets of environment variables we need to configure. First we need to set the WIF environment variables, which will be used in the Google Auth step.
- **WIF_PROVIDER**
  - **Name:** `WIF_PROVIDER`
  - **Description:** Workload Identity Federation configuration. Should be in the format `projects/736537866288/locations/global/workloadIdentityPools/ci-cd/providers/github-actions`.
  - **Example:** `projects/0123456789/locations/global/workloadIdentityPools/ci-cd/providers/github-actions`

- **WIF_SERVICE_ACCOUNT**
  - **Name:** `WIF_SERVICE_ACCOUNT`
  - **Description:** Workload Identity Federation configuration. Recommended to have separate configuration for each environment.
  - **Example:** `devops-svc@my-project-dev.iam.gserviceaccount.com`


Then, whe have the backend configuration definition, which are mostly passed as `--backend-config` into `tofu init`
- **TF_BUCKET**
  - **Name:** `TF_BUCKET`
  - **Description:** Backend configuration for OpenTofu. At least one of these needs to be environment-specific. Recommended to have separate buckets for each environment.
  - **Example:** `my-tf-bucket-dev-213496087`

- **TF_PREFIX**
  - **Name:** `TF_PREFIX`
  - **Description:** Backend configuration for OpenTofu. If storing state for both environments in a single bucket, specify a different prefix or workspace for each environment.
  - **Example:** `my-service-state`

- **TF_WORKSPACE**
  - **Name:** `TF_WORKSPACE`
  - **Description:** Backend configuration for OpenTofu (Optional). If storing state for both environments in a single bucket, specify a different prefix or workspace for each environment.
  - **Example:** `dev-workspace`

And finally we have the variables that are passed into as variables to `tofu plan`. You'll maybe recognize these from the section on this guide about the terraform variables. 
- **SERVICE**
  - **Name:** `SERVICE`
  - **Description:** Name of the service. Used for naming deployed resources. Recommended to set as a Repository Variable for consistent naming of resources across environments.
  - **Example:** `my-service`

- **ENV**
  - **Name:** `ENV`
  - **Description:** Used for namespacing resources to their specific environment. Also passed as an environment variable to the Cloud Run service.
  - **Example:** `dev`

- **PROJECT_ID**
  - **Name:** `PROJECT_ID`
  - **Description:** Project where the application stack will be deployed. Recommend using a separate project per environment.
  - **Example:** `my-project-dev`

- **REGION**
  - **Name:** `REGION`
  - **Description:** Region where the application stack will be deployed.
  - **Example:** `us-central1`

## Conclusion
Now if you try to rerun the deployment job, you should have a function CI/CD pipeline that deploys resources with IaC and manages a `prod` and `dev` environments. 

There are many things that I enjoy about this setup. One of them is that the power Terraform will allow your CI/CD pipeline to remain simple as your application
infrastructure grows. If you get to the point that your application is composes of multiple microservices on Cloud Run,
you can just copy paste  the configuration 
(or create a [module](https://developer.hashicorp.com/terraform/language/modules/develop)) 
with different build contexts that point to different deployment packages in you repository.  
