### Gcloud

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
