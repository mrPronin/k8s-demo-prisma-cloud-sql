# k8s-demo-prisma-cloud-sql
Kubernetes demo config for prisma deployment to GCP with Cloud SQL for Postgres

# Parameters
PROJECT_ID: [my_project_id]

PROJECT_NUMBER: [my_project_number]

CLUSTER_NAME: [my_cluster_name]

ZONE: [my_zone], for example: us-central1-f

# Init shell & env
gcloud config set project [my_project_id]

PROJECT_ID=$(gcloud config get-value project)

PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)')"

CLUSTER_NAME=[my_cluster_name]

ZONE=[my_zone]

gcloud config set compute/zone ${ZONE}

DB_INSTANCE_ID=[my_db_instance]

DATABASE_NAME=[my_db_name]

gcloud container clusters get-credentials ${CLUSTER_NAME}

DB_INSTANCE_CONNECTION_NAME="$(gcloud sql instances describe ${DB_INSTANCE_ID} \\\
--format='get(connectionName)')"

DB_PRISMA_USER_NAME=prisma

# Activate GC project
## Create project on GC (if not exist)
gcloud projects create \\\
[PROJECT_ID]

## Initialize Cloud SDK
https://cloud.google.com/sdk/docs/initializing

gcloud init

## Manage configuration
gcloud topic configurations

### Fetch configurations list
gcloud config configurations list

### Create configuration
gcloud config configurations create [CONFIGURATION_NAME]

### Delete configuration
gcloud config configurations delete [CONFIGURATION_NAME]

### Activate existing configuration
gcloud config configurations activate [CONFIGURATION_NAME]

### Check current configuration
https://cloud.google.com/sdk/gcloud/reference/config/list

gcloud config list

### Set region for current configuration
gcloud config set compute/region [REGION] for example 'us-central1'

## Manage account
https://cloud.google.com/sdk/docs/authorizing

### Fetch account list
gcloud auth list

### Authorize gcloud with Google user credentials
gcloud auth login  [ACCOUNT] \\\
--no-launch-browser

### Set active account
gcloud config set account [ACCOUNT]

## Manage projects

### Fetch projects list
gcloud projects list

### Get current PROJECT_ID
gcloud config get-value project

### Get current PROJECT_NUMBER
PROJECT_ID=$(gcloud config get-value project)

gcloud projects describe ${PROJECT_ID} --format='get(projectNumber)'

## Project API

### Services list
gcloud services list

### Enable services
gcloud services enable \\\
	cloudshell.googleapis.com \\\
	sqladmin.googleapis.com \\\
	container.googleapis.com \\\
	containeranalysis.googleapis.com

### Disable services
gcloud services disable \\\
	cloudshell.googleapis.com \\\
	sqladmin.googleapis.com \\\
	container.googleapis.com \\\
	containeranalysis.googleapis.com

# Create Kubernetes Cluster
## Create
gcloud beta container --project ${PROJECT_ID} \\\
clusters create ${CLUSTER_NAME} \\\
--zone ${ZONE} \\\
--machine-type "n1-standard-1" \\\
--num-nodes "3" \\\
--max-nodes=3 \\\
--min-nodes=1 \\\
--image-type "COS" \\\
--enable-stackdriver-kubernetes \\\
--enable-autoupgrade \\\
--enable-autorepair \\\
--no-enable-basic-auth \\\
--disk-type "pd-standard" \\\
--disk-size "100"

## Verify cluster status
gcloud container clusters list

## Delete cluster
gcloud container clusters delete ${CLUSTER_NAME}

## Authenticate kubectl
gcloud container clusters get-credentials ${CLUSTER_NAME}

## 'prisma' namespace
### Describe 'prisma-namespace.yaml' in 'k8s-create' folder

### Apply namespace
kubectl apply -f ./k8s-create/prisma-namespace.yaml

### Check namespaces
kubectl get namespaces

# Create Clous SQL for Postgres
## Create Cloud SQL instance
gcloud sql instances create ${DB_INSTANCE_ID} \\\
--database-version=POSTGRES_11 \\\
--cpu=1 \\\
--memory=3840MiB \\\
--zone=${ZONE} \\\
--storage-auto-increase

## Check Cloud SQL instance
gcloud sql instances list

## Delete Cloud SQL instance (if needed)
gcloud sql instances delete ${DB_INSTANCE_ID}

## Create database
gcloud sql databases create ${DATABASE_NAME} \\\
--instance=${DB_INSTANCE_ID}

## Delete database (if needed)
gcloud sql databases delete ${DATABASE_NAME} \\\
--instance=${DB_INSTANCE_ID}

## Check database
gcloud sql databases list \\\
--instance=${DB_INSTANCE_ID}

## Fetch information about instance
gcloud sql instances describe ${DB_INSTANCE_ID}

### Get instance connection name
DB_INSTANCE_CONNECTION_NAME="$(gcloud sql instances describe ${DB_INSTANCE_ID} \\\
--format='get(connectionName)')"

### Store instance connection name to prod.env
echo "DB_INSTANCE_CONNECTION_NAME=${DB_INSTANCE_CONNECTION_NAME}" >> ./prisma/secrets/prod.env

## Default ('postgres') user
### Generate default user password
DB_DEFAULT_USER_PASSWORD=$(openssl rand -base64 48)

### Store to prod.env
echo "DB_DEFAULT_USER_PASSWORD=${DB_DEFAULT_USER_PASSWORD}" >> ./prisma/secrets/prod.env

### Set password for default user
gcloud sql users set-password postgres \\\
--instance=${DB_INSTANCE_ID} \\\
--password=${DB_DEFAULT_USER_PASSWORD}

## 'prisma' user
### Set prisma user name
DB_PRISMA_USER_NAME=prisma

### Store prisma user name to prod.env
echo "DB_PRISMA_USER_NAME=${DB_PRISMA_USER_NAME}" >> ./prisma/secrets/prod.env

### Generate password for 'prisma' user
DB_PRISMA_USER_PASSWORD=$(openssl rand -base64 48)

### Store to prod.env
echo "DB_PRISMA_USER_PASSWORD=${DB_PRISMA_USER_PASSWORD}" >> ./prisma/secrets/prod.env

### Create 'prisma' user
gcloud sql users create ${DB_PRISMA_USER_NAME} \\\
--instance=${DB_INSTANCE_ID} \\\
--password=${DB_PRISMA_USER_PASSWORD}

## Check users
gcloud sql users list \\\
--instance=${DB_INSTANCE_ID}

## Delete user (if needed)
gcloud sql users delete [USER_NAME] \\\
--instance=${DB_INSTANCE_ID}

# Create Credentials for Cloud SQL Proxy
https://cloud.google.com/iam/docs/service-accounts

## Creating Service Account 'proxy-user'
gcloud iam service-accounts create proxy-user \\\
--display-name "proxy-user"

## Check service account
gcloud iam service-accounts list

## Get service account email
DB_PROXY_USER_SERVICE_ACCOUNT_EMAIL="$(gcloud iam service-accounts list \\\
--format='get(email)' \\\
--filter="proxy-user")"

## Grant service account 'CloudSQL Client' role
gcloud projects add-iam-policy-binding ${PROJECT_ID} \\\
--member serviceAccount:${DB_PROXY_USER_SERVICE_ACCOUNT_EMAIL} \\\
--role roles/cloudsql.client

## Create 'json' file with credentials
gcloud iam service-accounts keys create ./prisma/secrets/keyProxy.json \\\
--iam-account=${DB_PROXY_USER_SERVICE_ACCOUNT_EMAIL}

## Create secret 'cloudsql-credentials' from 'keyProxy.json'
kubectl create secret generic cloudsql-credentials \\\
--namespace=prisma \\\
--from-file=credentials.json=./prisma/secrets/keyProxy.json

# Deploy prisma to GKE

## Create prisma management secret key

### Generate prisma management secret key
PRISMA_MANAGEMENT_API_SECRET=$(openssl rand -base64 48)

### Store to prod.env
echo "PRISMA_MANAGEMENT_API_SECRET=${PRISMA_MANAGEMENT_API_SECRET}" >> ./prisma/secrets/prod.env

## Create prisma API secret key

### Generate prisma API secret key
PRISMA_API_SECRET=$(openssl rand -base64 48)

### Store to prod.env
echo "PRISMA_API_SECRET=${PRISMA_API_SECRET}" >> ./prisma/secrets/prod.env

## PRISMA_CONFIG secret
### Describe 'prisma-config-secret.yaml' in './prisma/secrets' folder
apiVersion: v1 \
kind: Secret \
metadata: \
  name: "prisma-config-secret" \
  namespace: prisma \
type: Opaque \
stringData: \
  PRISMA_CONFIG: |- \
    port: 4466 \
    managementApiSecret: [PRISMA_MANAGEMENT_API_SECRET] \
    enableManagementApi: true \
    databases: \
      default: \
        connector: postgres \
        host: localhost \
        database: [DATABASE_NAME] \
        user: [DB_PRISMA_USER_NAME] \
        ssl: false \
        password: [DB_PRISMA_USER_PASSWORD] \
        rawAccess: true \
        port: '5432' \
        migrations: true \

### Apply 'prisma-config-secret.yaml'
kubectl apply -f ./prisma/secrets/prisma-config-secret.yaml

### Check secrets
kubectl get secrets \\\
--namespace=prisma

## prisma deployment
### Describe 'prisma-deployment.yaml' in './k8s' folder

### Apply prisma deployment
kubectl apply -f ./k8s/prisma-deployment.yaml

### Check prisma deployment
kubectl get pods -n prisma

kubectl get deployment -n prisma

## prisma service
### Describe 'prisma-service.yaml' in './k8s' folder

### Apply prisma service
kubectl apply -f ./k8s/prisma-service.yaml

### Check prisma service
kubectl get services -n prisma

# Client-side prisma
## Configure Prisma CLI
kubectl port-forward svc/prisma 5000:4466 -n prisma

## Create / update 'prisma.yml' inside 'prisma' folder
endpoint: ${env:PRISMA_API_URL}

datamodel: datamodel.graphql

secret: ${env:PRISMA_API_SECRET}

## Check prisma status from 'prisma' folder
prisma info -e secrets/prod.env

# Cleanup
## Cleanup prisma
kubectl delete -f ./prisma/secrets/prisma-config-secret.yaml

kubectl delete -f k8s/prisma-deployment.yaml

kubectl delete -f k8s/prisma-service.yaml

## Cleanup DB
kubectl delete secret cloudsql-credentials

gcloud iam service-accounts delete proxy-user

gcloud sql instances delete ${DB_INSTANCE_ID}

## Delete cluster
gcloud container clusters delete ${CLUSTER_NAME}

## Disable services
gcloud services disable \\\
	cloudshell.googleapis.com \\\
	sqladmin.googleapis.com \\\
	container.googleapis.com \\\
	containeranalysis.googleapis.com
