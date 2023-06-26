# Lab 2: Create Google Cloud Infrastrcuture Components

Create a VPC:
```bash
export VPC_NETWORK="redis-vpc-network"
export SUBNETWORK=$VPC_NETWORK
gcloud compute networks create $VPC_NETWORK \
    --subnet-mode=auto
```
     
On success, you should view your VPC network as follows:
![VPC Network](./img/Redis_VPC_Network.png)
    

Reserve external static IP addresses:
```bash
gcloud compute addresses create redis-api-gateway-ip --region us-central1
export REDIS_API_GATEWAY_IP="$(gcloud compute addresses describe redis-api-gateway-ip --region=us-central1 --format='value(address)')"
```
```bash
gcloud compute addresses create redis-client-host-ip --region us-central1
export REDIS_CLIENT_HOST_IP="$(gcloud compute addresses describe redis-client-host-ip --region=us-central1 --format='value(address)')"
```    


