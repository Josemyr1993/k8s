# About Prometheus

Prometheus is a high-scalable open-source monitoring framework. It provides out-of-the-box monitoring capabilities for the Kubernetes container orchestration platform. Also, In the observability space, it is gaining huge popularity as it helps with metrics and alerts.

Explaining Prometheus is out of the scope of this article. If you want to know more about Prometheus, You can watch all the Prometheus-related videos from here. However, there are a few key points I would like to list for your reference.

1. Metric Collection: Prometheus uses the pull model to retrieve metrics over HTTP. There is an option to push metrics to Prometheus using Pushgateway for use cases where Prometheus cannot Scrape the metrics. One such example is collecting custom metrics from short-lived kubernetes jobs & Cronjobs

2. Metric Endpoint: The systems that you want to monitor using Prometheus should expose the metrics on an /metrics endpoint. Prometheus uses this endpoint to pull the metrics in regular intervals.

3. PromQL: Prometheus comes with PromQL, a very flexible query language that can be used to query the metrics in the Prometheus dashboard. Also, the PromQL query will be used by Prometheus UI and Grafana to visualize metrics.

4. Prometheus Exporters: Exporters are libraries that convert existing metrics from third-party apps to Prometheus metrics format. There are many official and community Prometheus exporters. One example is, the Prometheus node exporter. It exposes all Linux system-level metrics in Prometheus format.

5. TSDB (time-series database): Prometheus uses TSDB for storing all the data efficiently. By default, all the data gets stored locally. However, to avoid a single point of failure, there are options to integrate remote storage for Prometheus TSDB.


# Prometheus Architecture

![image](https://user-images.githubusercontent.com/86851766/203935390-3fc54b49-584e-40a3-a56a-37fdc0a36fe9.png)

If you would like to install Prometheus on a Linux VM, please see the Prometheus on Linux guide.

The Kubernetes Prometheus monitoring stack has the following components.

1 . Prometheus Server
2 . Alert Manager
3 . Grafana

In a nutshell, the following image depicts the high-level Prometheus kubernetes architecture that we are going to build. We have separate blogs for each component setup.

![image](https://user-images.githubusercontent.com/86851766/203937769-9849e570-a8b9-4110-90dd-cb4c3079ae6b.png)

# Connect to the Kubernetes Cluster

Connect to your Kubernetes cluster and make sure you have admin privileges to create cluster roles.

```
NOTE: If you are using Google cloud GKE, you need to run the following commands as you need privileges to create cluster roles for this Prometheus setup.
```

```
ACCOUNT=$(gcloud info --format='value(config.account)')
kubectl create clusterrolebinding owner-cluster-admin-binding \
    --clusterrole cluster-admin \
    --user $ACCOUNT
```

# Prometheus Kubernetes Manifest Files

All the configuration files I mentioned in this guide are hosted on Github. You can clone the repo using the following command.

```
git clone https://github.com/techiescamp/kubernetes-prometheus
```

Thanks to James for contributing to this repo. Please don’t hesitate to contribute to the repo for adding features.

You can use the GitHub repo config files or create the files on the go for a better understanding, as mentioned in the steps.

Let’s get started with the setup.

# Create a Namespace & ClusterRole

First, we will create a Kubernetes namespace for all our monitoring components. If you don’t create a dedicated namespace, all the Prometheus kubernetes deployment objects get deployed on the default namespace.

Execute the following command to create a new namespace named monitoring.

```
kubectl create namespace monitoring
```

Prometheus uses Kubernetes APIs to read all the available metrics from Nodes, Pods, Deployments, etc. For this reason, we need to create an RBAC policy with read access to required API groups and bind the policy to the monitoring namespace.

# Step1: Create a file named clusterRole.yaml and copy the following RBAC role.

```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus
rules:
- apiGroups: [""]
  resources:
  - nodes
  - nodes/proxy
  - services
  - endpoints
  - pods
  verbs: ["get", "list", "watch"]
- apiGroups:
  - extensions
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
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: prometheus
subjects:
- kind: ServiceAccount
  name: default
  namespace: monitoring
```

# Step 2: Create the role using the following command

```
kubectl create -f clusterRole.yaml
```

# Create a Config Map To Externalize Prometheus Configurations

All configurations for Prometheus are part of prometheus.yaml file and all the alert rules for Alertmanager are configured in prometheus.rules.

```
NOTE:

#prometheus.yaml#: This is the main Prometheus configuration which holds all the scrape configs, service discovery details, storage locations, data retention configs, etc)
#prometheus.rules#: This file contains all the Prometheus alerting rules
```

By externalizing Prometheus configs to a Kubernetes config map, you don’t have to build the Prometheus image whenever you need to add or remove a configuration. You need to update the config map and restart the Prometheus pods to apply the new configuration.

The config map with all the Prometheus scrape config and alerting rules gets mounted to the Prometheus container in /etc/prometheus location as prometheus.yaml and prometheus.rules files.

#Step 1: Create a file called config-map.yaml and copy the file contents from this link https://raw.githubusercontent.com/bibinwilson/kubernetes-prometheus/master/config-map.yaml

#Step 2#: Execute the following command to create the config map in Kubernetes.

```
kubectl create -f config-map.yaml
```

It creates two files inside the container.

```
Note: In Prometheus terms, the config for collecting metrics from a collection of endpoints is called a job.
```

The prometheus.yaml contains all the configurations to discover pods and services running in the Kubernetes cluster dynamically. We have the following scrape jobs in our Prometheus scrape configuration.

1 . kubernetes-apiservers: It gets all the metrics from the API servers.
2 . kubernetes-nodes: It collects all the kubernetes node metrics.
3 . kubernetes-pods: All the pod metrics get discovered if the pod metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations.
4 . kubernetes-cadvisor: Collects all cAdvisor metrics.
5 . kubernetes-service-endpoints: All the Service endpoints are scrapped if the service metadata is annotated with prometheus.io/scrape and prometheus.io/port annotations. It can be used for black-box monitoring.

# Create a Promethes Deployment

Step 1: Create a file named prometheus-deployment.yaml and copy the following contents onto the file. In this configuration, we are mounting the Prometheus config map as a file inside /etc/prometheus as explained in the previous section.

```
Note: This deployment uses the latest official Prometheus image from the docker hub. Also, we are not using any persistent storage volumes for Prometheus storage as it is a basic setup. When setting up Prometheus for production uses cases, make sure you add persistent storage to the deployment.
```

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
  namespace: monitoring
  labels:
    app: prometheus-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus-server
  template:
    metadata:
      labels:
        app: prometheus-server
    spec:
      containers:
        - name: prometheus
          image: prom/prometheus
          args:
            - "--storage.tsdb.retention.time=12h"
            - "--config.file=/etc/prometheus/prometheus.yml"
            - "--storage.tsdb.path=/prometheus/"
          ports:
            - containerPort: 9090
          resources:
            requests:
              cpu: 500m
              memory: 500M
            limits:
              cpu: 1
              memory: 1Gi
          volumeMounts:
            - name: prometheus-config-volume
              mountPath: /etc/prometheus/
            - name: prometheus-storage-volume
              mountPath: /prometheus/
      volumes:
        - name: prometheus-config-volume
          configMap:
            defaultMode: 420
            name: prometheus-server-conf
  
        - name: prometheus-storage-volume
          emptyDir: {}
```

Step 2: Create a deployment on monitoring namespace using the above file.

```
kubectl create  -f prometheus-deployment.yaml 
```

Step 3: You can check the created deployment using the following command.

```
kubectl get deployments --namespace=monitoring
```

You can also get details from the kubernetes dashboard as shown below.

Connecting To Prometheus Dashboard

You can view the deployed Prometheus dashboard in three different ways.

1. Using Kubectl port forwarding
2. Exposing the Prometheus deployment as a service with NodePort or a Load Balancer.
3. Adding an Ingress object if you have an Ingress controller deployed.

Let’s have a look at all three options.

# Method 1: Using Kubectl port forwarding

Using kubectl port forwarding, you can access a pod from your local workstation using a selected port on your localhost. This method is primarily used for debugging purposes.

Step 1: First, get the Prometheus pod name.

```
kubectl get pods --namespace=monitoring
```

The output will look like the following.

```
➜  kubectl get pods --namespace=monitoring
NAME                                     READY     STATUS    RESTARTS   AGE
prometheus-monitoring-3331088907-hm5n1   1/1       Running   0          5m
```

Step 2: Execute the following command with your pod name to access Prometheus from localhost port 8080.

```
Note: Replace prometheus-monitoring-3331088907-hm5n1 with your pod name.
```

```
kubectl port-forward prometheus-monitoring-3331088907-hm5n1 8080:9090 -n monitoring
```

Step 3: Now, if you access http://localhost:8080 on your browser, you will get the Prometheus home page.

