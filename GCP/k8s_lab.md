gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-monolith-515:1.0.0 .

gcloud container clusters create fancy-production-168 --num-nodes 3

kubectl create deployment fancy-monolith-515 --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-monolith-515:1.0.0

kubectl expose deployment fancy-monolith-515 --type=LoadBalancer --port 80 --target-port 8080

cd ~/monolith-to-microservices/microservices/src/orders
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-orders-663:1.0.0 .

cd ~/monolith-to-microservices/microservices/src/products
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-products-642:1.0.0 .

kubectl create deployment fancy-orders-663 --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-orders-663:1.0.0
kubectl expose deployment fancy-orders-663 --type=LoadBalancer --port 80 --target-port 8081

kubectl create deployment fancy-products-642 --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-products-642:1.0.0
kubectl expose deployment fancy-products-642 --type=LoadBalancer --port 80 --target-port 8082

cd ~/monolith-to-microservices/microservices/src/frontend
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-frontend-209:1.0.0 .

orders: 35.222.105.207
products: 35.224.106.208

kubectl create deployment fancy-frontend-209 --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/fancy-frontend-209:1.0.0

kubectl expose deployment fancy-frontend-209 --type=LoadBalancer --port 80 --target-port 8080

### Implement DevOps in Google Cloud

(create google cloud repository)
gcloud source repos create REPO_DEMO

gcloud config set compute/zone us-east4-c

(sample kubernetes code)
gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
cd orchestrate-with-kubernetes/kubernetes

(sample cluster create)
gcloud container clusters create bootcamp \
 --machine-type e2-small \
 --num-nodes 3 \
 --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"

kubectl explain deployment

kubectl explain deployment --recursive

kubectl explain deployment.metadata.name

kubectl create -f deployments/auth.yaml

kubectl get deployments

kubectl get replicasets

kubectl get pods

kubectl create -f services/auth.yaml
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml

(Create and expose frontend deployment with config map)
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml

kubectl get services frontend

curl -ks https://34.85.248.126

curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"

### Scale a deployment

(info about replicas)
kubectl explain deployment.spec.replicas

kubectl scale deployment hello --replicas=5

(get numbers of replicas)
kubectl get pods | grep hello- | wc -l

### Roling update

kubectl get replicaset

(rollout status and pause and resume)
kubectl rollout history deployment/hello
kubectl rollout pause deployment/hello
kubectl rollout status deployment/hello

kubectl rollout resume deployment/hello

kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

(roll back previous rollout)
kubectl rollout undo deployment/hello

### Canary deployments

(track canary)
kubectl create -f deployments/hello-canary.yaml

(sessionAffinity: ClientIP - same version of app will be served for session with ip)

```
kind: Service
apiVersion: v1
metadata:
  name: "hello"
spec:
  sessionAffinity: ClientIP
  selector:
    app: "hello"
  ports:
    - protocol: "TCP"
      port: 80
      targetPort: 80
```

(Blue-green deployment)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
        track: stable
        version: 2.0.0
    spec:
      containers:
        - name: hello
          image: kelseyhightower/hello:2.0.0
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
          resources:
            limits:
              cpu: 0.2
              memory: 10Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            periodSeconds: 15
            timeoutSeconds: 5
          readinessProbe:
            httpGet:
              path: /readiness
              port: 81
              scheme: HTTP
            initialDelaySeconds: 5
            timeoutSeconds: 1
```

SRE Lab

```
git clone --depth 1 --branch cloudskillsboost_asm https://github.com/GoogleCloudPlatform/cloud-ops-sandbox.git
cd cloud-ops-sandbox/sre-recipes
```

Connect to cluster with steps:

1. Navigate to Navigation Menu > Kubernetes Engine > Clusters. Select the three dots to the right of the cloud-ops-sandbox cluster and select the option to Connect.

2. On the Connect to the cluster modal dialog, click the RUN IN CLOUD SHELL button. Press Enter to run the command once populated in Cloud Shell

3. Restore sre-recipe:

```
./sandboxctl sre-recipes restore "recipe3"
```

Create logs metrics (type counter, Error_Rate_SLI)

```
resource.labels.cluster_name="cloud-ops-sandbox" AND resource.labels.namespace_name="default" AND resource.type="k8s_container" AND labels.k8s-pod/app="recommendationservice" AND severity>=ERROR
```

Request based SLI on
Define SLI details, the Performance Metric must be set to the following value:

```
custom.googleapis.com/opencensus/grpc.io/client/roundtrip_latency
```

Create SLO in
Navigation menu > Monitoring > Services.

### Jenkins in kubernetes

![Diagram jenkins](/GCP/k8s_jenkins_diagram.png)

https://cloud.google.com/architecture/jenkins-on-kubernetes-engine

gcloud config set compute/zone us-central1-b

(Jenkins source code)
gsutil cp gs://spls/gsp051/continuous-deployment-on-kubernetes.zip .
unzip continuous-deployment-on-kubernetes.zip
cd continuous-deployment-on-kubernetes

(provision k8s cluster)
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"

gcloud container clusters list

(get credential for cluster)
gcloud container clusters get-credentials jenkins-cd
kubectl cluster-info

(setup helm)
helm repo add jenkins https://charts.jenkins.io
helm repo update

(install jenkins)
helm install cd jenkins/jenkins -f jenkins/values.yaml --wait

(Configure the Jenkins service account to be able to deploy to the cluster)
kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins

(configure port forwarding to the jenkins ui)
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &

(check services)
kubectl get svc

(connect to jenkins)
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

### Deploying the Application

cd sample-app

(create namespace production)
kubectl create ns production

kubectl apply -f k8s/production -n production

kubectl apply -f k8s/canary -n production

kubectl apply -f k8s/services -n production

(Scale up production frontends)
kubectl scale deployment gceme-frontend-production -n production --replicas 4

(display frontend pods)
kubectl get pods -n production -l app=gceme -l role=frontend

(retrieve the external ip)
kubectl get service gceme-frontend -n production

(store load balncer ip as environment variable)
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

(create jenkins pipeline)
gcloud source repos create default
git init
git config credential.helper gcloud.sh

kubectl proxy &

https://source.cloud.google.com/qwiklabs-gcp-00-9ae61481c1c0/sample-app/

jenkins admin password:
iOjP8L779rCogk8i8a7jg0
