# Prometheus monitoring


## create a K8s cluster .

1. After creating your cluster, you need to get authentication credentials to interact with the cluster
```
gcloud container clusters get-credentials argocd
```
* This command configures kubectl to use the cluster.


## Application deployed on k8s engine which is already exposing prometheus metrics .


1. create a menifest file for deployment 


2. Apply the manifest file using command
  
**command**
```
kubectl apply -f nameofthefile.yaml
```
* check the deployment using command 
```
kubectl get deployment
```

## Deploy helm chart(prometheus-community/kube-prometheus-stack) in a different namespace 

1. Get Helm Repository Info
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

```
helm repo update
```

2. Install Helm Chart

```
helm install prometheus prometheus-community/kube-prometheus-stack -n=prometheus-operator --create-namespace
```
* check the status of the namespace
```
kubectl get namespace
```

3. port forwarding to 5000 to access the metrics 
**command**

```
 kubectl port-forward deployment/prom-example 5000:1234
```
4. open a new tab on your browser
and type 
```
http://localhost:5000/metrics


```
![alt text](https://i.postimg.cc/FRTxT4Fy/Screenshot-from-2022-07-29-13-26-53.png)
## Create service file for application 

**manifest file (applicationservice.yaml)**

```
kubectl apply -f applicationservice.yaml

```


## Create cluster role, role binding, and service account


* apply this yaml file 
```
kubectl apply -f rbac.yaml

```
* Check that the role was created and bound to the ServiceAccount
```
kubectl describe clusterrolebinding prometheus

```

## Create service monitor resource/pod monitor resource

1. Create a file service-monitor.yaml with the following content to add a ServiceMonitor so that the Prometheus server scrapes only its own metrics endpoints.

**menifest file**
* apply this yaml file
```
kubectl apply -f service-monitor.yaml

```
## Create prometheus custom resource



1. After creating the Prometheus ServiceAccount and giving it access to the Kubernetes API, we can deploy the Prometheus instance.

Create a file prometheus.yaml 

**manifest file(prometheus.yaml)**

```
kubectl apply -f prometheus.yaml

```
## Grafana Dashboard

* Forwarding the port to access the grafana dashboard using command

```
kubectl port-forward svc/prometheus-grafana 9000:80 --namespace=prometheus-operator

```
* Go to your browser and type 
 **127.0.0.1:9000**

* for username and password type these commands on your terminal
 ```
 kubectl get secrets -n prometheus-operator
 ```
 ```
 kubectl get secrets prometheus-grafana --namespace=prometheus-operator -o yaml
 ```
 * now decode the username and password using command
 ```
 echo cHJvbS1vcGVyYXRvcg== | base64 --decode
 ```
 * open your dashboard of grafana 
  1. settings
  2. Click on "Data Sources
  3. Click on "Add data source"
  4. name : prometheus
  5. URL : Set the appropriate Prometheus server URL (ex : http://prometheus-operated.default.svc.cluster.local:9090 )
  6. Adjust other data source settings as desired (for example, choosing the right Access method).
     Click "Save & Test" to save the new data source. 

  ![alt text](https://i.postimg.cc/kXg5rdJV/Screenshot-from-2022-07-29-13-27-22.png)




 










