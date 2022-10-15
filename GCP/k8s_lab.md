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
