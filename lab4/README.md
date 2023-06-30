# Lab 4: Create a Google Cloud Build Trigger and Deploy the Sample App
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
On success, you should see the newly created trigger inside Google Cloud Console:
![CB Trigger](./img/CB_Trigger.png)
    
Run the trigger to deploy the sample app:
```bash
gcloud alpha builds triggers run $REDIS_CLOUD_BUILD_TRIGGER --branch=master --region=$CLUSTER_LOCATION
```
    
On success, you should see a similar build-in-process like below:
![CB Trigger History](./img/CB_Trigger_Build.png))
    
A successful build will look like the following:
![CB Trigger Build Success](./img/CB_Trigger_Build_Success.png)
     
You can now access the sample app and make a few purchases by pointing your browser at:
```bash
http://<$REDIS_CLIENT_HOST_IP>:4200
```
![Redis Shopping](./img/Redis_Shopping.png)
    
After making a couple purchases, here is a sample screen shot of the order history page:
![Order History](./img/Order_History.png)
The order history data is retrieved from Redis Enterprise (in-memory).


