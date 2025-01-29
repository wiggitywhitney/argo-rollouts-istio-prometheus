# Demo an Automated Canary Deployment on Kubernetes with Argo Rollouts, Istio, and Prometheus

### Prerequisites
* [Google Cloud Project with Kubernetes API enabled](https://console.cloud.google.com/marketplace/product/google/container.googleapis.com)
* [Helm](https://helm.sh/docs/intro/install/)
* [hey](https://github.com/rakyll/hey) (HTTP load generator)
* [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl)
* [yq](https://github.com/mikefarah/yq)
* [Argo Rollouts Kubectl plugin](https://argoproj.github.io/argo-rollouts/installation/#kubectl-plugin-installation)


### Create Cluster

```
export KUBECONFIG=$PWD/kubeconfig.yaml

export USE_GKE_GCLOUD_AUTH_PLUGIN=True

# Replace `[...]` with your Google Cloud Project ID
export PROJECT_ID=[...]

export CLUSTER_NAME=rollouts-$(date +%Y%m%d%H%M)

gcloud container clusters create $CLUSTER_NAME --project $PROJECT_ID \
          --zone us-east1-b --machine-type e2-standard-4 \
          --enable-autoscaling --num-nodes 1 --min-nodes 1 \
          --max-nodes 3 --enable-network-policy \
          --no-enable-autoupgrade

```

### Install Istio

```
helm upgrade --install istio-base base \
    --repo https://istio-release.storage.googleapis.com/charts \
    --namespace istio-system --create-namespace --wait

helm upgrade --install istiod istiod \
    --repo https://istio-release.storage.googleapis.com/charts \
    --namespace istio-system --wait

helm upgrade --install istio-ingress gateway \
    --repo https://istio-release.storage.googleapis.com/charts \
    --namespace istio-system
```

### Verify & Get external IP

```
kubectl -n istio-system get svc

export ISTIO_IP=$(kubectl --namespace istio-system \
    get service istio-ingress \
    --output jsonpath="{.status.loadBalancer.ingress[0].ip}")

# This is the IP address by which you can access applications running in your cluster!

echo $ISTIO_IP
```

### Install Prometheus and connect to the Prometheus dashboard

```
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.24/samples/addons/prometheus.yaml


kubectl port-forward svc/prometheus -n istio-system 9090:9090 &
# View Prometheus at http://localhost:9090/ and verify that it is receiving Istio metrics by typing `istio` into the expression field and seeing whether any available metrics populate into the drop-down menu.

```
Below are the PromQL queries we'll use later to help verify the success of our rollout. You don't need to do anything with them now.

This query computes the total rate of successful requests (non-5xx responses) to the istio-rollout-canary service:
```
sum(
  irate(
    istio_requests_total{destination_service=~"istio-rollout-canary.rollouts-demo-istio.svc.cluster.local",reporter="source",response_code!~"5.*"}[40s]
  )
)
```

This query computes the total rate of all requests (regardless of response code) to the istio-rollout-canary service:
```
sum(
  irate(
    istio_requests_total{destination_service=~"istio-rollout-canary.rollouts-demo-istio.svc.cluster.local",reporter="source"}[40s]
  )
)

```
Later for our success condition we'll divide the rate of successful requests by the rate of all requests, which will give us our success rate. 


### Install Argo Rollouts

```
kubectl create ns argo-rollouts

kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml
```

### Create the Networking Resources


Store hostname in a variable
```
export HOST=wiggitywhitney.$ISTIO_IP.nip.io
```
Verify. This is the address where we'll be able to access our running application later!
```
echo $HOST
```

Modify the VirtualService yaml file to use $HOST as the host

```
yq --inplace ".spec.hosts[0] = \"$HOST\"" \
    virtualservice.yaml
```

Apply the Namespace, Gateway, Services, and VirtualService resources. 
```
kubectl apply -f namespace.yaml -f gateway.yaml -f services.yaml -f virtualservice.yaml
```

<!--- As a side note, I spent forever troubleshooting why Istio wouldn't work and it ended up being the `istio: ingressgateway` selector on the Gateway resource. I needed to change it to `istio: ingress` since I installed Istio using Helm. --->

Make sure Istio is working correctly
```
curl -i $HOST
```
This returns a 503 error but also lets you know that the request is hitting an istio-envoy server

### Run the Argo Rollouts demo application

Start running the load tester
```
hey -z 60m "http://$HOST/" &
```
Cat the AnalysisTemplate resource, understand it, apply it!

```
cat analysis.yaml

kubectl apply -f analysis.yaml
```

Cat the Rollout resource, understand it, apply it!
```
cat rollout.yaml

kubectl apply -f rollout.yaml
```

### There is so much to see!

The running application!
```
echo "http://$HOST/"

# open the url from the output!
```
Argo Rollouts Dashboard!
```
kubectl argo rollouts dashboard -n rollouts-demo-istio &

# Argo Rollouts Dashboard is now available at http://localhost:3100/rollouts 
```
Watch the rollout status!
```
# open a new tab in your terminal
kubectl argo rollouts get rollout istio-rollout -n rollouts-demo-istio --watch

# you might need to run `export KUBECONFIG=$PWD/kubeconfig.yaml` again to connect to your cluster
```

### Update your rollout! Watch all the things!
```
kubectl argo rollouts -n rollouts-demo-istio set image istio-rollout "*=argoproj/rollouts-demo:yellow"
```

### Manually promote the rollout

Based on our Rollout definition, 10% of traffic is going to the new release the rollout pauses and waits for manual promotion. 

```
kubectl argo rollouts -n rollouts-demo-istio promote istio-rollout

# or press the 'promote' button in the UI!
```

And now we're off! Wheeeeee! As long as the traffic is flowing successfully, the new app will get promoted eventually receive 100% of traffic, and the old app will get scaled down.

See the AnalysisRun progress in the UI, or by running:
```
kubectl -n rollouts-demo-istio get analysisrun

# Replace `[...]` with then name of the analysisrun you want to see

kubectl -n rollouts-demo-istio describe analysisrun [...]
```

### What if the new application sucks?

Simulate a failing application:
```
kubectl argo rollouts -n rollouts-demo-istio set image istio-rollout "*=argoproj/rollouts-demo:bad-purple"
```

The UI seems to have a bug where I can't click into the AnalysisRun and see what failed exactly. But I can see it with kubectl by getting and describing the related analysisrun.
```
kubectl -n rollouts-demo-istio get analysisrun

# Replace `[...]` with then name of the analysisrun you want to see

kubectl -n rollouts-demo-istio describe analysisrun [...]
```

After a failure, our Rollout is `Degraded`. To be able to do anything else, we need to make our Rollout `Healthy` again by changing the desired state back to the previous, stable version.

```
kubectl argo rollouts -n rollouts-demo-istio set image istio-rollout "*=argoproj/rollouts-demo:yellow"
```

### We did it! Huzzah!!!

To learn more and play on your own, check out these resources!

[Argo Rollouts Getting Started Guide](https://argoproj.github.io/argo-rollouts/getting-started/) - Approachable!

[Argo Rollouts Demo Application](https://github.com/argoproj/rollouts-demo/tree/master) (This repo is based on the [istio example](https://github.com/argoproj/rollouts-demo/tree/master/examples/istio)) - Not for the faint of heart! 

