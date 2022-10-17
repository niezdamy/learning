### Gcloud

#Setup
gcloud auth list
gcloud config set compute/zone us-east1-b
gcloud config set compute/region us-central1

gcloud compute project-info describe --project $(gcloud config get-value project) # detailed project info

export PROJECT_ID=$(gcloud config get-value project)
export ZONE=$(gcloud config get-value compute/zone)
echo -e "PROJECT ID: $PROJECT_ID\nZONE: $ZONE"

gcloud compute instances list --filter="name=('gcelab2')" # filtering

gcloud compute firewall-rules list
gcloud compute firewall-rules list --filter="network='default'"

To allow access for new created machine we nedd to 1. Add a tag to the virtual machine 2. Add firewall rules
gcloud compute instances add-tags gcelab2 --tags http-server,https-server
gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server

gcloud compute firewall-rules list --filter=ALLOW:'80'

curl http://$(gcloud compute instances list --filter=name:gcelab2 --format='value(EXTERNAL_IP)')

# Logging

gcloud logging logs list
gcloud logging read "resource.type=gce_instance" --limit 5
gcloud logging read "resource.type=gce_instance AND labels.instance_name='gcelab2'" --limit 5

# Kubernetes

(create)
gcloud container clusters create --machine-type=e2-medium --zone=us-east1-c lab-cluster
(auth with cluster)
gcloud container clusters get-credentials lab-cluster
(create sample app)
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
(Load balancer expose)
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
(inspect kubernetes, external-ip should be exposed automaticly)
kubectl get service
(delete cluster)
gcloud container clusters delete lab-cluster

# Load Balancers

# Network load balacer

(create compute engine vm and install apache2)

```
  gcloud compute instances create www1 \
    --zone=us-central1-b \
    --tags=network-lb-tag \
    --machine-type=e2-medium \
    --image-family=debian-11 \
    --image-project=debian-cloud \
    --metadata=startup-script='#!/bin/bash
      apt-get update
      apt-get install apache2 -y
      service apache2 restart
      echo "
<h3>Web Server: www1</h3>" | tee /var/www/html/index.html'
```

(firewall rule on tags)
gcloud compute firewall-rules create www-firewall-network-lb \
 --target-tags network-lb-tag --allow tcp:80

gcloud compute instances list

(create static external ip for Load balancer)
gcloud compute addresses create network-lb-ip-1 --region us-central1

(http health check resource)
gcloud compute http-health-checks create basic-check

(create target pool for instances)
gcloud compute target-pools create www-pool --region us-central1 --http-health-check basic-check

(add instances for pool)
gcloud compute target-pools add-instances www-pool \
 --instances www1,www2,www3

(add forwarding rules)
gcloud compute forwarding-rules create www-rule \
 --region us-central1 \
 --ports 80 \
 --address network-lb-ip-1 \
 --target-pool www-pool

(check ip address of forwarding rule)
gcloud compute forwarding-rules describe www-rule --region us-central1

(save ip address of forwarding rule in variable) !!! Handy
IPADDRESS=$(gcloud compute forwarding-rules describe www-rule --region us-central1 --format="json" | jq -r .IPAddress)

(check if instances answer randomly under address)
while true; do curl -m1 $IPADDRESS; done

# HTTP Load balancer

You need to:

Create an instance template.
Create a target pool.
Create a managed instance group.
Create a firewall rule named as allow-tcp-rule-578 to allow traffic (80/tcp).
Create a health check.
Create a backend service, and attach the managed instance group with named port (http:80).
Create a URL map, and target the HTTP proxy to route requests to your URL map.
Create a forwarding rule.

(create load balancer template)
gcloud compute instance-templates create lb-backend-template \
 --region=us-central1 \
 --network=default \
 --subnet=default \
 --tags=allow-health-check \
 --machine-type=e2-medium \
 --image-family=debian-11 \
 --image-project=debian-cloud \
 --metadata=startup-script='#!/bin/bash
apt-get update
apt-get install apache2 -y
a2ensite default-ssl
a2enmod ssl
vm_hostname="$(curl -H "Metadata-Flavor:Google" \
 http://169.254.169.254/computeMetadata/v1/instance/name)"
echo "Page served from: $vm_hostname" | \
 tee /var/www/html/index.html
systemctl restart apache2'

(create managed instance group baased on template)
gcloud compute instance-groups managed create lb-backend-group \
 --template=lb-backend-template --size=2 --zone=us-central1-b

(create firewall rule)
gcloud compute firewall-rules create fw-allow-health-check \
 --network=default \
 --action=allow \
 --direction=ingress \
 --source-ranges=130.211.0.0/22,35.191.0.0/16 \
 --target-tags=allow-health-check \
 --rules=tcp:80

(setup global static extternal ip for Load Balancer)
gcloud compute addresses create lb-ipv4-1 \
 --ip-version=IPV4 \
 --global

(check ip address what was reserved)
gcloud compute addresses describe lb-ipv4-1 \
 --format="get(address)" \
 --global

(add health check on load balancer)
gcloud compute health-checks create http http-basic-check \
 --port 80

(create a backend service)
gcloud compute backend-services create web-backend-service \
 --protocol=HTTP \
 --port-name=http \
 --health-checks=http-basic-check \
 --global

(add backend service to instance groups)
gcloud compute backend-services add-backend web-backend-service \
 --instance-group=lb-backend-group \
 --instance-group-zone=us-central1-b \
 --global

(create url map to route incoming requests)
gcloud compute url-maps create web-map-http \
 --default-service web-backend-service

(create a target HTTP proxy to rouste request to you URL Map)
gcloud compute target-http-proxies create http-lb-proxy \
 --url-map web-map-http

(create a global forwarding rule to route incoming requests to the proxy)
gcloud compute forwarding-rules create http-content-rule \
 --address=lb-ipv4-1\
 --global \
 --target-http-proxy=http-lb-proxy \
 --ports=80

### Firebase commands

gcloud projects add-iam-policy-binding $PROJECT_ID \
 --member=user:student-01-bae7fc399609@qwiklabs.net --role=roles/logging.viewer

gcloud projects add-iam-policy-binding $PROJECT_ID \
 --member=user:student-01-bae7fc399609@qwiklabs.net --role roles/source.writer

(token login)
firebase login --no-localhost

firebase init

firebase deploy

(install hugo)
curl -L https://github.com/gohugoio/hugo/releases/download/v${_HUGO_VERSION}/hugo_${_HUGO_VERSION}_Linux-64bit.tar.gz | tar -xz -C /tmp

(create and clone repository with gcloud)
gcloud source repos create my_hugo_site
gcloud source repos clone my_hugo_site

(new site with hugo)
/tmp/hugo new site my_hugo_site --force

(clone template for hugo)
git clone \
 https://github.com/budparr/gohugo-theme-ananke.git \
 themes/ananke
echo 'theme = "ananke"' >> config.toml

(firebase tools for bash)
curl -sL https://firebase.tools | bash

gcloud builds list

gcloud builds log $(gcloud builds list --format='value(ID)' --filter=$(git rev-parse HEAD))

gcloud builds log $(gcloud builds list --format='value(ID)' --filter=$(git rev-parse HEAD)) | grep "Hosting URL"

(tag & run cloud build)
gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1

(cloud run deploy)
gcloud beta run deploy netflix-dataset-service --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 --allow-unauthenticated

### Perform Foundational Infrastructure Tasks in Google Cloud

curl https://upload.wikimedia.org/wikipedia/commons/thumb/a/a4/Ada_Lovelace_portrait.jpg/800px-Ada_Lovelace_portrait.jpg --output ada.jpg

gsutil cp ada.jpg gs://YOUR-BUCKET-NAME

gsutil ls -l gs://qwiklabs-gcp-00-6853b12cd2e7/ada.jpg

gsutil acl ch -u AllUsers:R gs://qwiklabs-gcp-00-6853b12cd2e7/ada.jpg

(apache and php servers)
sudo apt-get update
sudo apt-get install apache2 php7.0
sudo service apache2 restart

(install monitoring agent on vm-s)
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
sudo systemctl status google-cloud-ops-agent"\*"

(Cloud function code)

```
/**
* Background Cloud Function to be triggered by Pub/Sub.
* This function is exported by index.js, and executed when
* the trigger topic receives a message.
*
* @param {object} data The event payload.
* @param {object} context The event metadata.
*/
exports.helloWorld = (data, context) => {
const pubSubMessage = data;
const name = pubSubMessage.data
    ? Buffer.from(pubSubMessage.data, 'base64').toString() : "Hello World";
console.log(`My Cloud Function: ${name}`);
};
```

gsutil mb -p qwiklabs-gcp-01-8be41ff484a9 gs://qwiklabs-gcp-01-8be41ff484a9

gcloud functions deploy helloWorld \
 --stage-bucket qwiklabs-gcp-01-8be41ff484a9 \
 --trigger-topic hello_world \
 --runtime nodejs8

(function status check)
gcloud functions describe helloWorld

DATA=$(printf 'Hello World!'|base64) && gcloud functions call helloWorld --data '{"data":"'$DATA'"}'

(execution logs check)
gcloud functions logs read helloWorld

(create pubsub topic)
gcloud pubsub topics create myTopic

gcloud pubsub topics list

(create pubsub subscription)
gcloud pubsub subscriptions create --topic myTopic mySubscription

gcloud pubsub topics publish myTopic --message "Hello"

(pull by default get only one message, message will be displayed only once)
gcloud pubsub subscriptions pull mySubscription --auto-ack

(limit of displayed messages)
gcloud pubsub subscriptions pull mySubscription --auto-ack --limit=3

SERVICE_URL=https://netflix-dataset-service-611-4iagnqcipa-uc.a.run.app
curl -X GET $SERVICE_URL

### Build a Website on Google Cloud

(auth docker to registry)
gcloud auth configure-docker us-central1-docker.pkg.dev

(enable apis for cloud build, artifactory, cloud run)
gcloud services enable artifactregistry.googleapis.com \
 cloudbuild.googleapis.com \
 run.googleapis.com

(start build process)
gcloud builds submit --tag us-central1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/monolith-demo/monolith:1.0.0

(cloud run deployment)
gcloud run deploy monolith --image us-central1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/monolith-demo/monolith:1.0.0 --region us-central1

gcloud run services list
(more detailed )
gcloud beta run services list

(concurency - FaaS - function as a service handle only one request at a time)
gcloud run deploy monolith --image us-central1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/monolith-demo/monolith:1.0.0 --region us-central1 --concurrency 1

(enable compute engine api)
gcloud services enable compute.googleapis.com

(devshell_project id = unique name)
gsutil mb gs://fancy-store-$DEVSHELL_PROJECT_ID

(nvm - node version manager, long term support )
nvm install --lts

(startup script provided by google)

- Installs the Logging agent. The agent automatically collects logs from syslog.

- Installs Node.js and Supervisor. Supervisor runs the app as a daemon.

- Clones the app's source code from Cloud Storage Bucket and installs dependencies.

- Configures Supervisor to run the app. Supervisor makes sure the app is restarted if it exits unexpectedly or is stopped by an admin or process. It also sends the app's stdout and stderr to syslog for the Logging agent to collect.

```
#!/bin/bash
# Install logging monitor. The monitor will automatically pick up logs sent to
# syslog.
curl -s "https://storage.googleapis.com/signals-agents/logging/google-fluentd-install.sh" | bash
service google-fluentd restart &
# Install dependencies from apt
apt-get update
apt-get install -yq ca-certificates git build-essential supervisor psmisc
# Install nodejs
mkdir /opt/nodejs
curl https://nodejs.org/dist/v16.14.0/node-v16.14.0-linux-x64.tar.gz | tar xvzf - -C /opt/nodejs --strip-components=1
ln -s /opt/nodejs/bin/node /usr/bin/node
ln -s /opt/nodejs/bin/npm /usr/bin/npm
# Get the application source code from the Google Cloud Storage bucket.
mkdir /fancy-store
gsutil -m cp -r gs://fancy-store-[DEVSHELL_PROJECT_ID]/monolith-to-microservices/microservices/* /fancy-store/
# Install app dependencies.
cd /fancy-store/
npm install
# Create a nodeapp user. The application will run as this user.
useradd -m -d /home/nodeapp nodeapp
chown -R nodeapp:nodeapp /opt/app
# Configure supervisor to run the node app.
cat >/etc/supervisor/conf.d/node-app.conf << EOF
[program:nodeapp]
directory=/fancy-store
command=npm start
autostart=true
autorestart=true
user=nodeapp
environment=HOME="/home/nodeapp",USER="nodeapp",NODE_ENV="production"
stdout_logfile=syslog
stderr_logfile=syslog
EOF
supervisorctl reread
supervisorctl update
```

gcloud compute instances create frontend \
 --machine-type=n1-standard-1 \
 --tags=frontend \
 --metadata=startup-script-url=https://storage.googleapis.com/fancy-store-$DEVSHELL_PROJECT_ID/startup-script.sh

(frontend and backend rules on firewall)
gcloud compute firewall-rules create fw-fe \
 --allow tcp:8080 \
 --target-tags=frontend

gcloud compute firewall-rules create fw-be \
 --allow tcp:8081-8082 \
 --target-tags=backend

watch -n 2 curl http://34.172.26.28:8080

(instance-templates create and list )
gcloud compute instance-templates create fancy-fe \
 --source-instance=frontend

gcloud compute instance-templates list

(managed instance groups MIGs)
gcloud compute instance-groups managed create fancy-fe-mig \
 --base-instance-name fancy-fe \
 --size 2 \
 --template fancy-fe

(health check create)

gcloud compute firewall-rules create allow-health-check \
 --allow tcp:8080-8081 \
 --source-ranges 130.211.0.0/22,35.191.0.0/16 \
 --network default

gcloud compute instance-groups managed update fancy-fe-mig \
 --health-check fancy-fe-hc \
 --initial-delay 300

(http load balancer health checks)
gcloud compute http-health-checks create fancy-fe-frontend-hc \
 --request-path / \
 --port 8080

(backend services that are the target for load balancing traffic)
gcloud compute backend-services create fancy-fe-frontend \
 --http-health-checks fancy-fe-frontend-hc \
 --port-name frontend \
 --global

(add load balancer's backend services)
gcloud compute backend-services add-backend fancy-fe-frontend \
 --instance-group fancy-fe-mig \
 --instance-group-zone us-central1-f \
 --global

(url map for backend redirection)
gcloud compute url-maps create fancy-map \
 --default-service fancy-fe-frontend

(path matcher for url map)

gcloud compute url-maps add-path-matcher fancy-map \
 --default-service fancy-fe-frontend \
 --path-matcher-name orders \
 --path-rules "/api/orders=fancy-be-orders,/api/products=fancy-be-products"

(proxy ties to url map)
gcloud compute target-http-proxies create fancy-proxy \
 --url-map fancy-map

(global forwardin rule that ties a public IP address and port to the proxy)
gcloud compute forwarding-rules create fancy-http-rule \
 --global \
 --target-http-proxy fancy-proxy \
 --ports 80

gcloud compute forwarding-rules list --global

(force switch instsance with --max-unavailable parameter)
gcloud compute instance-groups managed rolling-action replace fancy-fe-mig \
 --max-unavailable 100%

watch -n 2 gcloud compute instance-groups list-instances fancy-fe-mig

(autoscaling policy)
gcloud compute instance-groups managed set-autoscaling \
 fancy-fe-mig \
 --max-num-replicas 2 \
 --target-load-balancing-utilization 0.60

(cdn enable)
gcloud compute backend-services update fancy-fe-frontend \
 --enable-cdn --global

gcloud compute instances set-machine-type frontend --machine-type custom-4-3840

gcloud compute instances describe fancy-fe-0dmn | grep machineType

watch -n 2 gcloud compute instance-groups list-instances fancy-fe-mig

watch -n 2 gcloud compute backend-services get-health fancy-fe-frontend --global

(break the instance)
gcloud compute ssh fancy-fe-1228
sudo supervisorctl stop nodeapp; sudo killall node
exit

(monitor repair operations)
watch -n 2 gcloud compute operations list \
--filter='operationType~compute.instances.repair.\*'

### Kubernetes commands

(k8s uses default compute/zone)
gcloud config set compute/zone us-central1-f

gcloud services enable container.googleapis.com

(cluster with 3 nodes)
gcloud container clusters create fancy-cluster --num-nodes 3

gcloud compute instances list

gcloud services enable cloudbuild.googleapis.com

cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 .

(deployment should be stored in yaml)
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

kubectl create deployment monolith --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0

kubectl get all

(debugging of kubernetes)
kubectl describe pod monolith
kubectl describe pod/monolith-7d8bc7bf68-2bxts
kubectl describe deployment monolith
kubectl describe deployment.apps/monolith

# Show pods

kubectl get pods

# Show deployments

kubectl get deployments

# Show replica sets

kubectl get rs
#You can also combine them
kubectl get pods,deployments

(delete pod to check if replicaSet is creating new instance)
kubectl delete pod/<POD_NAME>
kubectl get all

(create load balancer with external ip)
kubectl expose deployment monolith --type=LoadBalancer --port 80 --target-port 8080

(to check external ip)
kubectl get service

kubectl scale deployment monolith --replicas=3

kubectl get all

(deployment with zero downtime)
kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0

(clenaup)
rm -rf monolith-to-microservices

# Delete the container image for version 1.0.0 of the monolith

gcloud container images delete gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:1.0.0 --quiet

# Delete the container image for version 2.0.0 of the monolith

gcloud container images delete gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:2.0.0 --quiet

# The following command will take all source archives from all builds and delete them from cloud storage

# Run this command to print all sources:

# gcloud builds list | awk 'NR > 1 {print $4}'

gcloud builds list | grep 'SOURCE' | cut -d ' ' -f2 | while read line; do gsutil rm $line; done

kubectl delete service monolith
kubectl delete deployment monolith

### Migrating a Monolithic Website to Microservices on Google Kubernetes Engine

gcloud container clusters create fancy-cluster --num-nodes 3 --machine-type=e2-standard-4

(migrate products from monolith to microservice)

1. Create docker container
   cd ~/monolith-to-microservices/microservices/src/products
   gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0 .

2. Deploy container to gke
   kubectl create deployment products --image=gcr.io/${GOOGLE_CLOUD_PROJECT}/products:1.0.0

3. Expose the GKE Container
   kubectl expose deployment products --type=LoadBalancer --port 80 --target-port 8082

4. Check ip:
   kubectl get service products

orders: 34.148.30.10
product: 34.73.97.38

5. Change envs and rebuild frontend config files
   npm run build:monolith

6. Create docker container with cloud build
   cd ~/monolith-to-microservices/monolith
   gcloud builds submit --tag gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:3.0.0 .

7. Deploy container to GKE
   kubectl set image deployment/monolith monolith=gcr.io/${GOOGLE_CLOUD_PROJECT}/monolith:3.0.0

### Serverless Cloud Run Development

#### PDF Convert solution

![PDF_diagram](/GCP/diagrams/pdf_diagram.png)

sample code

```
git clone https://github.com/rosera/pet-theory.git
cd pet-theory/lab03
```

package.json add script

```
    "start": "node index.js",
```

instal aditional dependencies

```
npm install express
npm install body-parser
npm install child_process
npm install @google-cloud/storage
```

cloud build

```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

cloud run instance create

```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --no-allow-unauthenticated \
  --max-instances=1
```

service url as local variable

```
SERVICE_URL=$(gcloud beta run services describe pdf-converter --platform managed --region us-east1 --format="value(status.url)")
echo $SERVICE_URL
```

!!! request with auth token

```
curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL
```

create buckets

```
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload
gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed

```

create pub/sub service

```
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload
```

create invoker service account

```
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

```add permissions to service account
gcloud beta run services add-iam-policy-binding pdf-converter --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --platform managed --region us-east1
```

get project number

```
gcloud projects list
```

set local var

```
PROJECT_NUMBER=29539156516
```

enable your project to create Cloud Pub/Sub authentication tokens:

```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```

Create Pub/Sub subscription

```
gcloud beta pubsub subscriptions create pdf-conv-sub --topic new-doc --push-endpoint=$SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

Test files in upload bucket

```
gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload
```

Cleanup bucket

```gsutil -m rm gs://$GOOGLE_CLOUD_PROJECT-upload/*

```

pdf convert

```
const {promisify} = require('util');
const exec        = promisify(require('child_process').exec);
const cmd = 'libreoffice --headless --convert-to pdf --outdir ' +
            `/tmp "/tmp/${fileName}"`;
const { stdout, stderr } = await exec(cmd);
if (stderr) {
  throw stderr;
}
```

install libre office in dockerfile

```
FROM node:12
RUN apt-get update -y \
    && apt-get install -y libreoffice \
    && apt-get clean
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

whole index.js

```(javascript)
const {promisify} = require('util');
const {Storage}   = require('@google-cloud/storage');
const exec        = promisify(require('child_process').exec);
const storage     = new Storage();
const express     = require('express');
const bodyParser  = require('body-parser');
const app         = express();
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  try {
    const file = decodeBase64Json(req.body.message.data);
    await downloadFile(file.bucket, file.name);
    const pdfFileName = await convertFile(file.name);
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);
    await deleteFile(file.bucket, file.name);
  }
  catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
async function downloadFile(bucketName, fileName) {
  const options = {destination: `/tmp/${fileName}`};
  await storage.bucket(bucketName).file(fileName).download(options);
}
async function convertFile(fileName) {
  const cmd = 'libreoffice --headless --convert-to pdf --outdir /tmp ' +
              `"/tmp/${fileName}"`;
  console.log(cmd);
  const { stdout, stderr } = await exec(cmd);
  if (stderr) {
    throw stderr;
  }
  console.log(stdout);
  pdfFileName = fileName.replace(/\.\w+$/, '.pdf');
  return pdfFileName;
}
async function deleteFile(bucketName, fileName) {
  await storage.bucket(bucketName).file(fileName).delete();
}
async function uploadFile(bucketName, fileName) {
  await storage.bucket(bucketName).upload(`/tmp/${fileName}`);
}
```

cloud run instance with env-vars

```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --max-instances=1 \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed
```

#### Asynchronous System with Cloud RUn

Enable cloudrun api

```
gcloud services enable run.googleapis.com
```

Create pubsub

```

gcloud pubsub topics create new-lab-report

```

lab 05

```

git clone https://github.com/rosera/pet-theory.git
cd pet-theory/lab05/lab-service
npm install express
npm install body-parser
npm install @google-cloud/pubsub

```

index.js

```(javascript)
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  try {
    const labReport = req.body;
    await publishPubSubMessage(labReport);
    res.status(204).send();
  }
  catch (ex) {
    console.log(ex);
    res.status(500).send(ex);
  }
})
async function publishPubSubMessage(labReport) {
  const buffer = Buffer.from(JSON.stringify(labReport));
  await pubsub.topic('new-lab-report').publish(buffer);
}
```

dockerfile

```
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

deploy the lab-report-service

create deploy.sh

```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service
gcloud run deploy lab-report-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
  --platform managed \
  --region us-east1 \
  --allow-unauthenticated \
  --max-instances=1
```

make file executable

```
chmod u+x deploy.sh
./deploy.sh
```

Lab Report Service as local var

```
export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service --platform managed --region us-east1 --format="value(status.address.url)")
```

post-reports.sh

```
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 12}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 34}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 56}" \
  $LAB_REPORT_SERVICE_URL &
```

chmod u+x post-reports.sh

#### email service

```
cd ~/pet-theory/lab05/email-service
```

index.js (pubsub handle)

```
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`Email Service: Report ${labReport.id} trying...`);
    sendEmail();
    console.log(`Email Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendEmail() {
  console.log('Sending email');
}
```

#### Configure Pub/Sub to trigger the Email Service

create service account

```
gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"
```

Give the new service account permission to invoke the Email Service:

````
gcloud run services add-iam-policy-binding email-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-east1 --platform managed```
````

Project number as variable

```
PROJECT_NUMBER=$(gcloud projects list --filter="qwiklabs-gcp" --format='value(PROJECT_NUMBER)')
```

Pub sub authentication tokens create

```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator
```

email service url in env

```
EMAIL_SERVICE_URL=$(gcloud run services describe email-service --platform managed --region us-east1 --format="value(status.address.url)")
```

pub sub subscription on email service

```
gcloud pubsub subscriptions create email-service-sub --topic new-lab-report --push-endpoint=$EMAIL_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

#### sms service

```
cd ~/pet-theory/lab05/sms-service
```

index.html with sms

```
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);
  try {
    console.log(`SMS Service: Report ${labReport.id} trying...`);
    sendSms();
    console.log(`SMS Service: Report ${labReport.id} success :-)`);
    res.status(204).send();
  }
  catch (ex) {
    console.log(`SMS Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendSms() {
  console.log('Sending SMS');
}
```

deploy.sh

```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service
gcloud run deploy sms-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service \
  --platform managed \
  --region us-east1 \
  --no-allow-unauthenticated \
  --max-instances=1
```

#### Configure Cloud Pub/Sub to trigger the SMS Service

Set the permissions to allow Pub/Sub to trigger the SMS Service:

```
gcloud run services add-iam-policy-binding sms-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-east1 --platform managed
```

The first step is to put the URL address of the SMS Service in an environment variable:

```
SMS_SERVICE_URL=$(gcloud run services describe sms-service --platform managed --region us-east1 --format="value(status.address.url)")
```

create the Pub/Sub subscription:

```
gcloud pubsub subscriptions create sms-service-sub --topic new-lab-report --push-endpoint=$SMS_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

### Developing a REST API with Go and Cloud Run

```
gcloud config set project $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')
```

```
git clone https://github.com/rosera/pet-theory.git && cd pet-theory/lab08
```

main.go

```
package main
import (
  "fmt"
  "log"
  "net/http"
  "os"
)
func main() {
  port := os.Getenv("PORT")
  if port == "" {
      port = "8080"
  }
  http.HandleFunc("/v1/", func(w http.ResponseWriter, r *http.Request) {
      fmt.Fprintf(w, "{status: 'running'}")
  })
  log.Println("Pets REST API listening on port", port)
  if err := http.ListenAndServe(":"+port, nil); err != nil {
      log.Fatalf("Error launching Pets REST API server: %v", err)
  }
}
```

Dockerfile

```
FROM gcr.io/distroless/base-debian10
WORKDIR /usr/src/app
COPY server .
CMD [ "/usr/src/app/server" ]
```

build go app

```
go build -o server
```

Cloud Build

```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1
```

deploy rest-api

```
gcloud run deploy rest-api \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/rest-api:0.1 \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --max-instances=2
```

Test data import

```
gsutil cp -r gs://spls/gsp645/2019-10-06T20:10:37_43617 gs://$GOOGLE_CLOUD_PROJECT-customer

gcloud beta firestore import gs://$GOOGLE_CLOUD_PROJECT-customer/2019-10-06T20:10:37_43617/
```

main.go file

```
package main

import (
	"cloud.google.com/go/firestore"
	"context"
	"encoding/json"
	"fmt"
	"github.com/gorilla/handlers"
	"github.com/gorilla/mux"
	"google.golang.org/api/iterator"
	"log"
	"net/http"
	"os"
)

var client *firestore.Client

func main() {
	var err error
	ctx := context.Background()
	client, err = firestore.NewClient(ctx, "qwiklabs-gcp-04-ceeccfc6813f")
	if err != nil {
		log.Fatalf("Error initializing Cloud Firestore client: %v", err)
	}
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}
	r := mux.NewRouter()
	r.HandleFunc("/v1/", rootHandler)
	r.HandleFunc("/v1/customer/{id}", customerHandler)
	log.Println("Pets REST API listening on port", port)
	cors := handlers.CORS(
		handlers.AllowedHeaders([]string{"X-Requested-With", "Authorization", "Origin"}),
		handlers.AllowedOrigins([]string{"https://storage.googleapis.com"}),
		handlers.AllowedMethods([]string{"GET", "HEAD", "POST", "OPTIONS", "PATCH", "CONNECT"}),
	)
	if err := http.ListenAndServe(":"+port, cors(r)); err != nil {
		log.Fatalf("Error launching Pets REST API server: %v", err)
	}
}

func rootHandler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "{status: 'running'}")
}
func customerHandler(w http.ResponseWriter, r *http.Request) {
	id := mux.Vars(r)["id"]
	ctx := context.Background()
	customer, err := getCustomer(ctx, id)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": '%s'}`, err)
		return
	}
	if customer == nil {
		w.WriteHeader(http.StatusNotFound)
		msg := fmt.Sprintf("`Customer \"%s\" not found`", id)
		fmt.Fprintf(w, fmt.Sprintf(`{"status": "fail", "data": {"title": %s}}`, msg))
		return
	}
	amount, err := getAmounts(ctx, customer)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
		return
	}
	data, err := json.Marshal(amount)
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
		fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
		return
	}
	fmt.Fprintf(w, fmt.Sprintf(`{"status": "success", "data": %s}`, data))
}

type Customer struct {
	Email string `firestore:"email"`
	ID    string `firestore:"id"`
	Name  string `firestore:"name"`
	Phone string `firestore:"phone"`
}

func getCustomer(ctx context.Context, id string) (*Customer, error) {
	query := client.Collection("customers").Where("id", "==", id)
	iter := query.Documents(ctx)
	var c Customer
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		err = doc.DataTo(&c)
		if err != nil {
			return nil, err
		}
	}
	return &c, nil
}
func getAmounts(ctx context.Context, c *Customer) (map[string]int64, error) {
	if c == nil {
		return map[string]int64{}, fmt.Errorf("Customer should be non-nil: %v", c)
	}
	result := map[string]int64{
		"proposed": 0,
		"approved": 0,
		"rejected": 0,
	}
	query := client.Collection(fmt.Sprintf("customers/%s/treatments", c.Email))
	if query == nil {
		return map[string]int64{}, fmt.Errorf("Query is nil: %v", c)
	}
	iter := query.Documents(ctx)
	for {
		doc, err := iter.Next()
		if err == iterator.Done {
			break
		}
		if err != nil {
			return nil, err
		}
		treatment := doc.Data()
		result[treatment["status"].(string)] += treatment["cost"].(int64)
	}
	return result, nil
}
```

#### Creating PDFs with Go and Cloud Run

activate lab account

```
gcloud auth list --filter=status:ACTIVE --format="value(account)"
```

Clone git

```
git clone https://github.com/Deleplace/pet-theory.git
cd pet-theory/lab03
```

Server.go - invoice microservice

```
package main
import (
      "fmt"
      "io/ioutil"
      "log"
      "net/http"
      "os"
      "os/exec"
      "regexp"
      "strings"
)
func main() {
      http.HandleFunc("/", process)
      port := os.Getenv("PORT")
      if port == "" {
              port = "8080"
              log.Printf("Defaulting to port %s", port)
      }
      log.Printf("Listening on port %s", port)
      err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
      log.Fatal(err)
}
func process(w http.ResponseWriter, r *http.Request) {
      log.Println("Serving request")
      if r.Method == "GET" {
              fmt.Fprintln(w, "Ready to process POST requests from Cloud Storage trigger")
              return
      }
      //
      // Read request body containing Cloud Storage object metadata
      //
      gcsInputFile, err1 := readBody(r)
      if err1 != nil {
              log.Printf("Error reading POST data: %v", err1)
              w.WriteHeader(http.StatusBadRequest)
              fmt.Fprintf(w, "Problem with POST data: %v \n", err1)
              return
      }
      //
      // Working directory (concurrency-safe)
      //
      localDir, errDir := ioutil.TempDir("", "")
      if errDir != nil {
              log.Printf("Error creating local temp dir: %v", errDir)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Could not create a temp directory on server. \n")
              return
      }
      defer os.RemoveAll(localDir)
      //
      // Download input file from Cloud Storage
      //
      localInputFile, err2 := download(gcsInputFile, localDir)
      if err2 != nil {
              log.Printf("Error downloading Cloud Storage file [%s] from bucket [%s]: %v",
gcsInputFile.Name, gcsInputFile.Bucket, err2)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }
      //
      // Use LibreOffice to convert local input file to local PDF file.
      //
      localPDFFilePath, err3 := convertToPDF(localInputFile.Name(), localDir)
      if err3 != nil {
              log.Printf("Error converting to PDF: %v", err3)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error converting to PDF.")
              return
      }
      //
      // Upload the freshly generated PDF to Cloud Storage
      //
      targetBucket := os.Getenv("PDF_BUCKET")
      err4 := upload(localPDFFilePath, targetBucket)
      if err4 != nil {
              log.Printf("Error uploading PDF file to bucket [%s]: %v", targetBucket, err4)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }
      //
      // Delete the original input file from Cloud Storage.
      //
      err5 := deleteGCSFile(gcsInputFile.Bucket, gcsInputFile.Name)
      if err5 != nil {
              log.Printf("Error deleting file [%s] from bucket [%s]: %v", gcsInputFile.Name,
gcsInputFile.Bucket, err5)
         // This is not a blocking error.
         // The PDF was successfully generated and uploaded.
      }
      log.Println("Successfully produced PDF")
      fmt.Fprintln(w, "Successfully produced PDF")
}
func convertToPDF(localFilePath string, localDir string) (resultFilePath string, err error) {
      log.Printf("Converting [%s] to PDF", localFilePath)
      cmd := exec.Command("libreoffice", "--headless", "--convert-to", "pdf",
              "--outdir", localDir,
              localFilePath)
      cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
      log.Println(cmd)
      err = cmd.Run()
      if err != nil {
              return "", err
      }
      pdfFilePath := regexp.MustCompile(`\.\w+$`).ReplaceAllString(localFilePath, ".pdf")
      if !strings.HasSuffix(pdfFilePath, ".pdf") {
              pdfFilePath += ".pdf"
      }
      log.Printf("Converted %s to %s", localFilePath, pdfFilePath)
      return pdfFilePath, nil
}
```

Dockerfile

```
FROM debian:buster
RUN apt-get update -y \
  && apt-get install -y libreoffice \
  && apt-get clean
WORKDIR /usr/src/app
COPY server .
CMD [ "./server" ]
```

Build image

```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter
```

Cloud run with 2GB of RAM

```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed \
  --max-instances=3
```

pub sub with service account

```
gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload

gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"

gcloud run services add-iam-policy-binding pdf-converter \
  --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
  --role=roles/run.invoker \
  --region us-east1 \
  --platform managed

PROJECT_NUMBER=$(gcloud projects list \
 --format="value(PROJECT_NUMBER)" \
 --filter="$GOOGLE_CLOUD_PROJECT")

 gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
  --role=roles/iam.serviceAccountTokenCreator

```

pubsub subscription created

```
gcloud pubsub subscriptions create pdf-conv-sub \
  --topic new-doc \
  --push-endpoint=$SERVICE_URL \
  --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```
