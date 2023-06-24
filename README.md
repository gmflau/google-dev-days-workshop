# google-dev-days-workshop

    
#### Lab 0: Activate your GCP project
[![Open in Cloud Shell](https://gstatic.com/cloudssh/images/open-btn.svg)](https://ssh.cloud.google.com/cloudshell/editor?cloudshell_git_repo=https%3A%2F%2Fgithub.com%2Fgmflau%2Fgoogle-dev-days-workshop&shellonly=true&cloudshell_image=gcr.io/ds-artifacts-cloudshell/deploysexport PROJECT_ID=$(gcloud info --format='value(config.project)')tack_custom_image)


#### Lab 0: Open a Google Cloud Shell and Enable APIs
![Cloud Shell]()
```bash

```

    
#### Lab 0.5: Download workshop repo
```bash
git clone https://github.com/gmflau/google-dev-days-workshop
cd google-dev-days-workshop
```

       
#### Lab 1: Create Google Cloud Infrastrcuture Components
Create a Cloud Source repo:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
gcloud gcloud source repos create redis-repo
```

Create a VPC:
```bash
export VPC_NETWORK="redis-vpc-network"
exprot SUBNETWORK=$VPC_NETWORK
gcloud compute networks create $VPC_NETWORK \
    --subnet-mode=auto
```
Reserve external static IP addresses:
```bash
gcloud compute addresses create redis-api-gateway-ip --region us-central1
export REDIS-API-GATEWAY-IP="$(gcloud compute addresses describe redis-api-gateway-ip --region=us-central1 --format='value(address)')"
```
```bash
gcloud compute addresses create redis-client-host-ip --region us-central1
export REDIS-CLIENT-HOST-IP="$(gcloud compute addresses describe redisclient-host-ip --region=us-central1 --format='value(address)')"
```
    
Create a GKE cluster:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER_LOCATION=us-central1
export CLUSTER_NAME="redis-gke-cluster-$CLUSTER_LOCATION"

gcloud container clusters create $CLUSTER_NAME \
 --region=$CLUSTER_LOCATION --num-nodes=1 \
 --image-type=COS_CONTAINERD \
 --machine-type=e2-standard-8 \
 --network=$VPC_NETWORK \
 --subnetwork=$SUBNETWORK \
 --enable-ip-alias \
 --workload-pool=${PROJECT_ID}.svc.id.goog
```
    
Provision Anthos Service Mesh:
```bash
gcloud container fleet mesh enable --project $PROJECT_ID

gcloud container fleet memberships register $CLUSTER_NAME-membership \
  --gke-cluster=${CLUSTER_LOCATION}/${CLUSTER_NAME} \
  --enable-workload-identity \
  --project ${PROJECT_ID}

gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_NAME}-membership \
  --project ${PROJECT_ID}
```
    
Wait for about ~10 minutes and run the command below to verify ASM is enabled:
```bash
gcloud container fleet mesh describe --project $PROJECT_ID
```
Enable "default" namespace for sidecar injection
```bash
kubectl label namespace default istio-injection=enabled istio.io/rev-
```

#### Lab 2: Create a Redis Enterprise Cloud subscription on Google Cloud
TBD
Collect db endpoint, password

    
#### Lab 3: Create a Google Cloud Build Trigger and Deploy the Sample App
Import workshop repo:
```bash

```
Create Cloud Build Trigger:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER_LOCATION=us-central1
export CLUSTER_NAME="redis-gke-cluster-$CLUSTER_LOCATION"
export REDIS_API_GATEWAY_IP="$(gcloud compute addresses describe redis-api-gateway-ip --region=us-central1 --format='value(address)')"
export REDIS_CLIENT_IP="$(gcloud compute addresses describe redisclient-host-ip --region=us-central1 --format='value(address)')"
export REDIS_URI=redis://default:xnurcS28JREs9S8HHemx2cKc1jLFi3ua@redis-10996.c279.us-central1-1.gce.cloud.redislabs.com:10996
export REDIS_INSIGHT_PORT=10996

gcloud alpha builds triggers create github \
  --name=glau-cqrs-trigger \
  --repository=projects/$PROJECT_ID/locations/us-central1/connections/github-gmflau/repositories/gmflau-redis-microservices-cqrs-demo \
  --branch-pattern=^main$ \
  --build-config=cloudbuild.yaml \
  --substitutions=_GKE_CLUSTER=$GKE_CLUSTER,_GKE_ZONE=$GKE_ZONE,_API_GATEWAY_IP=$REDIS_API_GATEWAY_IP,_CLIENT_IP=$REDIS_CLIENT_IP,_REDIS_URI=$REDIS_URI,_REDIS_INSIGHT_PORT=$REDIS_INSIGHT_PORT \
  --region=$CLUSTER_LOCATION
```

