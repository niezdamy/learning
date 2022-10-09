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
