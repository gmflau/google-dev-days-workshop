# DevDay Workshop: Build Better, Faster Applications – Faster (google-dev-days-workshop)

#### Workshop Agenda: [~4 hours duration]
Check-in: 30 mts
Age of real time: 20 mts
Modern Application Architectures: 30 mts
Development and Deployment Labs - Part I: 45 mts
Break: 15 mts
Development and Deployment Labs - Part II: 45 mts
Next Gen AI powered applications - How Gen AI + Redis VSS can work together: 30 mts
Event closing: 15 mts
    
Top three key takeaways:    
* How to build cloud native applications using the latest capabilities of Google Cloud and Redis Enterprise
* The latest and greatest features of real-time applications
* How the combination of Gen AI and Redis as a Vector database can help you build Gen AI powered applications
         

#### Lab Agenda:
* Lab 0
* Lab 1
* Lab 2
* Lab 3
* Break [15 minutes]
* Lab 4
* Lab 5
* Lab 6
     
          
#### Lab 0: Open a Google Cloud Shell and Enable APIs
[![Cloud Shell](./img/GCP_Cloud_Shell.png)](https://shell.cloud.google.com/)
    
    
#### Lab 1: Create a Cloud Source repo for the workshop sample app
Set your GCP Project:
```bash
gcloud config set project <PROJECT_ID>
```
Create a new Cloud Source repo:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export REDIS_REPO="redis-repo-$(date +'%Y%m%d%H%M%S')"
gcloud source repos create $REDIS_REPO
git clone https://source.developers.google.com/p/$PROJECT_ID/r/$REDIS_REPO
```
Deposit workshop sample app into the repo:
```bash
cd $REDIS_REPO
gsutil cp gs://redis-dev-days-workshop/google-dev-days-workshop-main.zip .
unzip google-dev-days-workshop-main.zip
mv google-dev-days-workshop-main/{*,.[^.]*,..?*} .
rm -fr google-dev-days-workshop-main
rm google-dev-days-workshop-main.zip
git add .
git commit -m "Initial commit"
git push -u origin master
```

       
#### Lab 2: Create Google Cloud Infrastrcuture Components
Create a VPC:
```bash
export VPC_NETWORK="redis-vpc-network"
export SUBNETWORK=$VPC_NETWORK
gcloud compute networks create $VPC_NETWORK \
    --subnet-mode=auto
```
Reserve external static IP addresses:
```bash
gcloud compute addresses create redis-api-gateway-ip --region us-central1
export REDIS_API_GATEWAY_IP="$(gcloud compute addresses describe redis-api-gateway-ip --region=us-central1 --format='value(address)')"
```
```bash
gcloud compute addresses create redis-client-host-ip --region us-central1
export REDIS_CLIENT_HOST_IP="$(gcloud compute addresses describe redis-client-host-ip --region=us-central1 --format='value(address)')"
```
    
Create a GKE cluster:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export PROJECT_NUMBER=$(gcloud projects list --filter="$PROJECT_ID" --format="value(PROJECT_NUMBER)")
export CLUSTER_LOCATION=us-central1
export CLUSTER_NAME="redis-gke-cluster-$CLUSTER_LOCATION"

gcloud container clusters create $CLUSTER_NAME \
    --project=$PROJECT_ID \
    --region=$CLUSTER_LOCATION \
    --machine-type=e2-standard-8 \
    --network=$VPC_NETWORK \
    --subnetwork=$SUBNETWORK \
    --num-nodes=1 \
    --workload-pool=${PROJECT_ID}.svc.id.goog \
    --labels="mesh_id=proj-${PROJECT_NUMBER}"
```
    
Provision Anthos Service Mesh:
Enable Anthos Service Mesh on your project's Fleet:
```bash
gcloud container fleet mesh enable --project $PROJECT_ID
```
Register the cluster to the project's Fleet:
```bash
gcloud container fleet memberships register $CLUSTER_NAME-membership \
  --gke-cluster=${CLUSTER_LOCATION}/${CLUSTER_NAME} \
  --enable-workload-identity \
  --project ${PROJECT_ID}
```
Provision managed Anthos Service Mesh on the cluster using the Fleet API:
```bash
gcloud container fleet mesh update \
  --management automatic \
  --memberships ${CLUSTER_NAME}-membership \
  --project ${PROJECT_ID}
```
Wait for about ~10 minutes and run the command below to verify ASM is enabled:
```bash
gcloud container fleet mesh describe --project $PROJECT_ID
```
You should see the following states equal to `ACTIVE`  when the installation of Anthos Service Mesh has been completed:
```bash
membershipStates:
  projects/319143195410/locations/us-central1/memberships/redis-gke-cluster-us-central1-membership:
    servicemesh:
      controlPlaneManagement:
        details:
        - code: REVISION_READY
          details: 'Ready: asm-managed'
        state: ACTIVE
      dataPlaneManagement:
        details:
        - code: OK
          details: Service is running.
        state: ACTIVE
```
Enable "default" namespace for sidecar injection
```bash
kubectl label namespace default istio-injection=enabled istio.io/rev-
```
You should see the following output on success:
```bash
label "istio.io/rev" not found.
namespace/default labeled
```
    
       
#### Lab 3: Create a Redis Enterprise Cloud subscription on Google Cloud
Create a Redis Cloud subscription:    
* Follow this [link](https://docs.redis.com/latest/rc/rc-quickstart/#create-an-account) through step 6. 
* In step 4, choose Google Cloud. Then come back here to continue on section 3 below to initialize two environments for this workshop.
* Collect the Redis Enterprise database connection string:
![SM Connect 01](./img/SM_Connect_01.png)    
![SM Connect 02](./img/SM_Connect_02.png)    
Back in your Google Cloud shell and set the following environment variable:
```bash
export REDIS_URI=<Redis connection string from the previous step>
export REDIS_INSIGHT_PORT=<Redis Enterprise databaase port number>

For example,
export REDIS_URI=redis://default:xnurcS28JREs9S8HHemx2cKc1jLFi3ua@redis-10996.c279.us-central1-1.gce.cloud.redislabs.com:10996
export REDIS_INSIGHT_PORT=10996 
```
    
       
#### Lab 4: Create a Google Cloud Build Trigger and Deploy the Sample App
Create Cloud Build Trigger:
```bash
export PROJECT_ID=$(gcloud info --format='value(config.project)')
export CLUSTER_LOCATION=us-central1
export CLUSTER_NAME="redis-gke-cluster-$CLUSTER_LOCATION"
export REDIS_API_GATEWAY_IP="$(gcloud compute addresses describe redis-api-gateway-ip --region=us-central1 --format='value(address)')"
export REDIS_CLIENT_HOST_IP="$(gcloud compute addresses describe redis-client-host-ip --region=us-central1 --format='value(address)')"
export REDIS_CLOUD_BUILD_TRIGGER="redis-cb-trigger"

gcloud alpha builds triggers create cloud-source-repositories \
  --name=$REDIS_CLOUD_BUILD_TRIGGER \
  --repo=$REDIS_REPO \
  --branch-pattern=^master$ \
  --build-config=cloudbuild.yaml \
  --substitutions=_GKE_CLUSTER=$CLUSTER_NAME,_GKE_ZONE=$CLUSTER_LOCATION,_API_GATEWAY_IP=$REDIS_API_GATEWAY_IP,_CLIENT_IP=$REDIS_CLIENT_HOST_IP,_REDIS_URI=$REDIS_URI,_REDIS_INSIGHT_PORT=$REDIS_INSIGHT_PORT \
  --region=$CLUSTER_LOCATION
```
Run the trigger to deploy the sample app:
```bash
gcloud alpha builds triggers run $REDIS_CLOUD_BUILD_TRIGGER --branch=master --region=$CLUSTER_LOCATION
```

#### Lab 5: Set up Redis Data Integration (RDI)
Provision a CloudSQL PostgreSQL instance:
Deploy Radis Data Integration:
Verify Redis Data Integration setup:
       
     
#### Lab 6: Update the Google Cloud Trigger with new CloudSQL PostgreSQL connection


    
#### Lab 7: Re-deploy the sample app with new CloudSQL PostgreSQL backend along with RDI 
Run the trigger to deploy the sample app:


#### Lab 8: Microservices Observability Walkthrough
Explore the microservices inside Google Cloud Console:
![ASM SM Topology](./img/ASM_SM_Topology.png)


