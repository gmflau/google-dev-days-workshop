# Lab 3: Create a Redis Enterprise Cloud subscription on Google Cloud
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

