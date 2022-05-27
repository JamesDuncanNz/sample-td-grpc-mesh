# Set up a proxyless gRPC service mesh

### Enable the required Google Cloud API services
```
gcloud services enable --project=$PROJECT_ID \
  container.googleapis.com \
  gkehub.googleapis.com \
  trafficdirector.googleapis.com \
  networkservices.googleapis.com
```

### Deploy a GKE cluster
```
gcloud container clusters create gke-1 \
  --zone=us-west1-a \
  --enable-ip-alias \
  --workload-pool=$PROJECT_ID.svc.id.goog \
  --scopes=https://www.googleapis.com/auth/cloud-platform \
  --enable-mesh-certificates \
  --release-channel=regular \
  --project=$PROJECT_ID
  ```

### Configure the IAM permissions for the data plane
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member "group:$PROJECT_ID.svc.id.goog:/allAuthenticatedUsers/" \
  --role "roles/trafficdirector.client"
```

### Register the cluster to a fleet

```
gcloud container hub memberships register gke-1 \
  --gke-cluster us-west1-a/gke-1 \
  --enable-workload-identity \
  --project=$PROJECT_ID
```
### Configure the  Mesh  resource

```
gcloud alpha network-services meshes import grpc-mesh \
  --source=mesh.yaml \
  --location=global
```
### Deploy sample gRPC service
```
kubectl apply -f grpc-td-helloworld.yaml
```

### Configure Network Resources
#### Create gRPC Health Check
```
gcloud compute health-checks create grpc grpc-helloworld-health-check \
  --use-serving-port
```
#### Create Firewall rule to allow Health Checking
[More Info can be found here](https://cloud.google.com/load-balancing/docs/health-checks#fw-rule)
```
gcloud compute firewall-rules create grpc-vm-allow-health-checks \
  --network=default \
  --action=ALLOW \
  --direction=INGRESS \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags allow-health-checks \
  --rules=tcp,udp
```
### Create Backend Service
```
  gcloud compute backend-services create grpc-helloworld-service \
  --global \
  --load-balancing-scheme=INTERNAL_SELF_MANAGED \
  --protocol=GRPC \
  --port-name=grpc-helloworld-port \
  --health-checks grpc-helloworld-health-check
```
### Add the NEG to the backend Service
```
  gcloud compute backend-services add-backend \
  grpc-helloworld-service \
  --network-endpoint-group=sampleneg \
  --network-endpoint-group-zone=us-west1-a \
  --balancing-mode=RATE \
  --max-rate-per-endpoint=100 \
  --global
```
### Set up routing with  GRPCRoute
You will need to change the `Project Number` in the `grpc-route.yaml` file to the project id of your GCP project. 
You can get the number by running the following command:
```
gcloud projects list \
--filter="$(gcloud config get-value project)" \
--format="value(PROJECT_NUMBER)"
```
```
name: helloworld-grpc-route
hostnames:
- helloworld
meshes:
- projects/$PROJECT_NUMBER/locations/global/meshes/grpc-mesh
rules:
- action:
destinations:
- serviceName: projects/$PROJECT_NUMBER/locations/global/backendServices/grpc-helloworld-service
```
```
gcloud alpha network-services grpc-routes import helloworld-grpc-route \
  --source=grpc-route.yaml \
  --location=global
```
### Create gRPC Client for testing
```
kubectl apply -f grpc-client.yaml
```
### Exec into client
```
kubectl exec -it static-sleeper -- /bin/sh
```
### Download and install the `grpcurl` tool:
```
cd /home/curl_user
curl -L https://github.com/fullstorydev/grpcurl/releases/download/v1.8.1/grpcurl_1.8.1_linux_x86_64.tar.gz | tar -xz
```
### Curl the gRPC Server
```
./grpcurl --plaintext -d '{"name": "world"}' xds:///helloworld helloworld.Greeter/SayHello
```
The output is similar to the following, where `INSTANCE_HOST_NAME` is the hostname of the Pod:

```
Greetings: Hello world, from INSTANCE_HOST_NAME
```

### Notes
This guide is a merge of:
- [Set up a proxyless gRPC service mesh](https://cloud.google.com/traffic-director/docs/set-up-proxyless-gke-mesh)
- [Set up proxyless gRPC services](https://cloud.google.com/traffic-director/docs/set-up-proxyless-mesh#configure-grpc-server)