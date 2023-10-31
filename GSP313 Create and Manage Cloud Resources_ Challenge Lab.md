Lab Name: Create and Manage Cloud Resources: Challenge Lab (GSP313)

```sh
# Activate Cloud Shell and set default variables
VM_NAME=nucleus-vm
CLUSTER_NAME=nucleus-webserver
REGION=us-east1
ZONE=us-east1-b

gcloud config set compute/region $REGION
gcloud config set compute/zone $ZONE
```

Task 1: Create a project jumphost instance

```sh
gcloud compute instances create $VM_NAME \
          --machine-type e2-micro  \
          --image-family debian-11  \
          --image-project debian-cloud
```

Task 2: Create a Kubernetes service cluster

```sh
gcloud container clusters create $CLUSTER_NAME

gcloud container clusters get-credentials $CLUSTER_NAME

kubectl create deployment hello-app \
          --image=gcr.io/google-samples/hello-app:2.0

kubectl expose deployment hello-app \
          --type=LoadBalancer \
          --port 8080

kubectl get service
```

Task 3: Set up an HTTP load balancer

```sh
# Run code to configure the web server
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

# Creating an instance template
gcloud compute instance-templates create web-server-template \
          --metadata-from-file startup-script=startup.sh

# Creating a target pool
gcloud compute target-pools create web-server-pool

# Creating a managed instance group
gcloud compute instance-groups managed create web-server-group \
          --base-instance-name web-server \
          --size 2 \
          --template web-server-template \
          --target-pool web-server-pool \
          --region $REGION

gcloud compute instances list

# Creating a firewall rule to allow traffic (80/tcp)
gcloud compute firewall-rules create web-server-firewall \
          --allow tcp:80

gcloud compute forwarding-rules create nginx-lb \
          --ports=80 \
          --target-pool web-server-pool

gcloud compute forwarding-rules list

# Creating a health check
gcloud compute http-health-checks create http-basic-check

gcloud compute instance-groups managed \
          set-named-ports web-server-group \
          --named-ports http:80 \
          --region $REGION

# Creating a backend service and attach the managed instance group
gcloud compute backend-services create web-server-backend \
          --protocol HTTP \
          --http-health-checks http-basic-check \
          --global

gcloud compute backend-services add-backend web-server-backend \
          --instance-group web-server-group \
          --instance-group-region $REGION \
          --global


# Creating a URL map and target HTTP proxy to route requests to your URL map
gcloud compute url-maps create web-server-map \
          --default-service web-server-backend

gcloud compute target-http-proxies create http-lb-proxy \
          --url-map web-server-map

# Creating a forwarding rule
gcloud compute forwarding-rules create http-content-rule \
        --global \
        --target-http-proxy http-lb-proxy \
        --ports 80

gcloud compute forwarding-rules list
```
