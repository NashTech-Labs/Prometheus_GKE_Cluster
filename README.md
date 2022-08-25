# Prometheus monitoring

* Prometheus is a free software application used for event monitoring and alerting.It records real-time metrics in a time series database (allowing   for high dimensionality) built using a HTTP pull model, with flexible queries and real-time alerting .

**Scenario**

* A GKE cluster where an application is running and that application need to be monitored.


## create a K8s cluster .

1. After creating your cluster, you need to get authentication credentials to interact with the cluster
```
gcloud container clusters get-credentials argocd
```
* This command configures kubectl to use the cluster you created.


## Application deployed on k8s engine which is already exposing prometheus metrics .

**link of application**

```
link: https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.4.3-gke.0/examples/example-app.yaml

```
1. create a menifest file for deployment 

**Manifest file (app.yml)**
```apiVersion: apps/v1
kind: Deployment
metadata:
  name: prom-example
  labels:
    app: prom-example
spec:
  selector:
    matchLabels:
      app: prom-example
  replicas: 3
  template:
    metadata:
      labels:
        app: prom-example
    spec:
      nodeSelector:
        kubernetes.io/os: linux
        kubernetes.io/arch: amd64
      containers:
      - image: nilebox/prometheus-example-app@sha256:dab60d038c5d6915af5bcbe5f0279a22b95a8c8be254153e22d7cd81b21b84c5
        name: prom-example
        ports:
        - name: metrics
          containerPort: 1234
        command:
        - "/main"
        - "--process-metrics"
        - "--go-metrics"
```

2. Apply the manifest file using command
  
**command**
```
kubectl apply -f apps.yaml
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
* You can then use kubectl describe crd <crd_name> to get a description of the CRD. And of course kubectl get crd <crd_name> -o yaml to get the complete definition of the CRD. 

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
apiVersion: v1
kind: Service
metadata:
  name: applicationservice
  labels:
    name: applicationservice
    
spec:
  selector:
    app: prom-example
  ports:
  - port: 5000
    targetPort: 1234
    name: port-name

```
```
kubectl apply -f applicationservice.yaml

```


## Create cluster role, role binding, and service account

1. To scrape targets and get to the Alertmanager clusters, the Prometheus server needs access to the Kubernetes API.
As a result, a ServiceAccount must be created and associated with a ClusterRole in order to grant access to those resources: 
```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: prometheus
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
  namespace: default
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/metrics
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources:
  - configmaps
  verbs: ["get"]
- apiGroups:
  - networking.k8s.io
  resources:
  - ingresses
  verbs: ["get", "list", "watch"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: prometheus
  namespace: default

```
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
```
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: prometheus
  labels:
    name: prometheus
spec:
  selector:
    matchLabels: 
      name: applicationservice  
  endpoints:
    - port: port-name

    
```
* apply this yaml file
```
kubectl apply -f service-monitor.yaml

```
## Create prometheus custom resource



1. After creating the Prometheus ServiceAccount and giving it access to the Kubernetes API, we can deploy the Prometheus instance.

Create a file prometheus.yaml 

**manifest file(prometheus.yaml)**

```
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
spec:
  serviceAccountName: prometheus
  
  serviceMonitorSelector: 
    matchLabels:
      name: prometheus
  resources:
    requests:
      memory: 400Mi

```
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




 










