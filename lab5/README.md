# Lab 5: Set up Redis Data Integration (RDI)

#### 1. Provision a CloudSQL PostgreSQL instance:

#### 2. Deploy Radis Data Integration:
Deploy a Redis Enterprise cluster:
```bash
kubectl create namespace redis
kubectl config set-context --current --namespace=redis

VERSION=`curl --silent https://api.github.com/repos/RedisLabs/redis-enterprise-k8s-docs/releases/latest | grep tag_name | awk -F'"' '{print $4}'`

kubectl apply -n redis -f https://raw.githubusercontent.com/RedisLabs/redis-enterprise-k8s-docs/$VERSION/bundle.yaml
```
```bash
cat <<EOF > rec.yaml
apiVersion: "app.redislabs.com/v1"
kind: "RedisEnterpriseCluster"
metadata:
  name: rec
spec:
  nodes: 3
EOF

kubectl apply -f rec.yaml -n redis 
```
    
Retrieve the password for the Redis Enterprise Cluster's default uesr: demo@redislabs.com:
```bash
export REC_PWD=$(kubectl get secrets -n redis rec -o jsonpath="{.data.password}" | base64 --decode)
```
     
Install Redis Gears:
```bash
kubectl exec -it rec-0 -n redis -- curl -s https://redismodules.s3.amazonaws.com/redisgears/redisgears_python.Linux-ubuntu18.04-x86_64.1.2.6.zip -o /tmp/redis-gears.zip

kubectl exec -it rec-0 -n redis -- curl -k -s -u "demo@redislabs.com:${REC_PWD}" -F "module=@/tmp/redis-gears.zip" https://localhost:9443/v2/modules
```
    
Install RDI CLI container:
```bash
kubectl config set-context --current --namespace=default

cat << EOF > /tmp/redis-di-cli-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: redis-di-cli
  labels:
    app: redis-di-cli
spec:
  containers:
    - name: redis-di-cli
      image: docker.io/redislabs/redis-di-cli
      volumeMounts:
      - name: config-volume
        mountPath: /app
      - name: jobs-volume
        mountPath: /app/jobs
  volumes:
    - name: config-volume
      configMap:
        name: redis-di-config
        optional: true
    - name: jobs-volume
      configMap:
        name: redis-di-jobs
        optional: true
EOF
kubectl apply -f /tmp/redis-di-cli-pod.yml
```
    
Create a new RDI database:
```bash
kubectl exec -it -n default pod/redis-di-cli -- redis-di create --cluster-host localhost 
```
    
Use the following input and answer the rest of the promot:
```bash
rec.redis.svc.cluster.local
demo@redislabs.com
Grab password from $REC_PWD
```
    
Create a RDI config file:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di scaffold --db-type postgresql --preview config.yaml > config.yaml
```
    
Create a ConfigMap for Redis Data Integration:
```bash
kubectl create configmap redis-di-config --from-file=config.yaml -n default
```
    
Deploy the RDI configuration:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di deploy
```
     
Install the Debezium Server:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di scaffold --db-type postgresql --preview debezium/application.properties > application.properties
```

Create a ConfigMap for Debezium Server:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di scaffold --db-type postgresql --preview debezium/application.properties > application.properties
```    
    
Create the Debezium Server Pod:
```bash
cat << EOF > /tmp/debezium-server-pod.yml
apiVersion: v1
kind: Pod
metadata:
  name: debezium-server
  labels:
    app: debezium-server
spec:
  containers:
    - name: debezium-server
      image: docker.io/debezium/server
      livenessProbe:
        httpGet:
            path: /q/health/live
            port: 8088
      readinessProbe:
        httpGet:
            path: /q/health/ready
            port: 8088
      volumeMounts:
      - name: config-volume
        mountPath: /debezium/conf
  volumes:
    - name: config-volume
      configMap:
        name: debezium-config
EOF
kubectl apply -f /tmp/debezium-server-pod.yml
```
    
Create a RDI job:    
Edit emp.yaml:
```bash
cat << 'EOF' > ./emp.yaml
source:
  server_name: example-postgres.default.svc.cluster.local
  db: postgres
  table: emp
transform:
  - uses: rename_field
    with:
      from_field: fname
      to_field: first_name
output:
  - uses: redis.write
    with:
      connection: target
      key:
        expression: concat(['emp:fname:',fname,':lname:',lname])
        language: jmespath
EOF
```
Create a ConfigMap for the job:
```bash
kubectl create configmap redis-di-jobs --from-file=./emp.yaml
```
    
Deploy the RDI job:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di deploy
```
    
Check if the job has been created:
```bash
kubectl exec -it -n default pod/redis-di-cli -- redis-di list-jobs
```
    
Update an existing job:
After editing the ./emp.yaml file, then run:
```bash
kubectl delete configmap redis-di-jobs 
kubectl create configmap redis-di-jobs --from-file=./emp.yaml
```
    
Deploy the job through an updated ConfigMap:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di deploy
```
    
Review the updated job detail in RDI:
```bash
kubectl exec -n default -it pod/redis-di-cli -- redis-di describe-job emp
```


