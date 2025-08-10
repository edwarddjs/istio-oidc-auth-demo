# Demo Setup
## Step 1: Create k3d cluster with traefik disabled (default ingress - we are going to configure istio)
k3d cluster create istio-demo \
  --agents 1 \
  --port 80:80@loadbalancer \
  --port 443:443@loadbalancer \
  --k3s-arg '--disable=traefik@server:0'

## Step 2: Install istio in the cluster
istioctl install --set profile=default

## Step 3: Enable istio injection in default namespace
kubectl label namespace default istio-injection=enabled
Verify with: kubectl get namespace default --show-labels - should see the label

## Step 4: Deploy Gateway & httpbin
kubectl apply -f gateway.yaml  
kubectl apply -f httpbin.yaml 

## Step 5: Verify httpbin connectivity
curl http://localhost/httpbin/get

## Step 6: Deploy DEX service entry 
kubectl apply -f dex-service-entry.yaml 

## Step 7: Start DEX & configure host resolution
docker compose up
Add '127.0.0.1       host.k3d.internal' to etc/hosts (note sudo should be used e.g. sudo nano /etc/hosts)

## Step 8: Verify DEX connectivity from within the cluster
kubectl run curl-test --image=radial/busyboxplus:curl -it --rm --restart=Never -- sh
once at terminal run: curl http://host.docker.internal:5556/dex/.well-known/openid-configuration

## Step 9: Create authservice image and copy to k3d cluster
ensure go is installed (to install: brew install go)
run: make build
run: make docker
k3d image import authservice:latest-arm64 -c istio-demo

## Step 10: Deploy authservice & authorization policies & request authorisation policies
kubectl apply -f authservice.yaml
kubectl apply -f istio-security.yaml

## Step 11: Deploy mesh wide destination rule to force tls to be used


# Debuging
The below commands can help increase the amount of logging:

kubectl exec -it <gateway-pod-name> -c istio-proxy -n istio-system -- curl -X POST "localhost:15000/logging?level=debug"

istioctl proxy-config routes <gateway-pod-name> -n istio-system

kubectl exec -it <gateway-pod-name> -c istio-proxy -n istio-system -- curl -X POST "localhost:15000/logging?level=debug"

istioctl proxy-config log <ingress-gateway-name> --level grpc:debug -n istio-system (note: grpc can be replaced by the item you are trying to debug)# istio-oidc-auth-demo
