### Gcloud

#Setup
gcloud auth list
gcloud config set compute/zone us-east1-b
gcloud config set compute/region us-east1

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
