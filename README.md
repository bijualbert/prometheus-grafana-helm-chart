# prometheus-grafana-helm-chart
HELM CHART FOR PROMETHEUS AND GRAFANA

This article will give you a brief idea of how to create a Helm chart to deploy Prometheus and Grafana together using it.

Let us have an overview of the technologies used in this blog:

PROMETHEUS
Prometheus is an open-source tool that is built by SoundCloud.
Prometheus is mostly used for monitoring and alerting purposes.
GRAFANA
Grafana is also a multi-platform opensource tool that is used for the interactive visualization, analysis, and monitoring of the metrics and logs.
Grafana provides a wide range of options when it comes to graphs, charts, alerts, etc.
HELM

Helm is a third-party tool that works as the package manager of Kubernetes.
The packages in Helm are called Charts, we can create charts and then install them to deploy all the resources of an application, that too in just one click.
The main aim of this task is to create a Helm Chart for any of the technology (here, I am going to use Prometheus and Grafana together).

Are there any pre-requisites?
In order to completely understand this practical, you need to have some idea about Kubernetes, Prometheus and Grafana.
Let’s try to carry out the practical.

The very first step is to install the Helm.
You the above command to download the tar file:

wget https://get.helm.sh/helm-v3.5.3-linux-amd64.tar.gz

To extract:
tar -xzf helm-v3.5.3-linux-amd64.tar.gz

Moving the binary file:
mv linux-amd64/helm /usr/bin/
And now, we are all set to use the helm.

Use the above command to confirm the installation:
helm version
Here, Helm will automatically get the cluster details by reading the default kubeconfig file from $HOME/.kube/config.

I am performing all the above steps after configuring the Kubernetes cluster, so this is the most required step to have a Kubernetes cluster pre-setup.

If you want to know how to set up a multi-node Kubernetes cluster using Ansible, follow the above blog:
https://github.com/bijualbert/k8s_ansible_wordpress_mysql/blob/main/README.md

After installing the helm, the next step would be creating a Chart.
You have to create a directory and the name of the folder/directory will be the name of your chart that you are going to create.
To perform the same:
mkdir prom-grafana
cd prom-grafana
According to the perspective of a Chart, every resource in Kubernetes is called a template and this is the reason why we need to put the YAML file of each resource in a fixed directory called ‘templates’.

mkdir templates
cd templates
Inside this directory, we have to keep all the YAML files.

To deploy Prometheus, I am going to use:
Deployment
PVC
Config Map
Service
To deploy Grafana, I am going to use:
Deployment
PVC
Service
Since here I am using the AWS cloud to perform the entire set up, so, while using the PVC, I will be getting the storage from NFS using the EFS service of the AWS cloud.
I have launched the above File System on the top of AWS:

Here are the YAML files for every component:
Service for Grafana:
grafana-service.yml 
```
apiVersion: v1
kind: Service
metadata:
  name: grafana-service
spec:
  selector:
    env: dev
    dc: IN
  type: NodePort
  ports:
  - nodePort: {{ .Values.grafanaPort }}
    port: 3000
```
Here, in the value of NodePort, I have used a variable’s reference. We will get to know how to create this variable further in this blog.

Service for Prometheus:
prometheus-service.yml
```
apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
spec:
  selector:
    env: prod
    dc: IN
  type: NodePort
  ports:
  - nodePort: {{ .Values.prometheusPort }}
    port: 9090
```
PV for Grafana:
grafana-pv.yml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: grafana-pv
spec:
  storageClassName: ""
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    server: "{{ .Values.grafanaNFS }}"
    path: "/"
```
PV for Prometheus:
prometheus-pv.yml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: prometheus-pv
spec:
  storageClassName: ""
  capacity:
    storage: 2Gi
  accessModes:
  - ReadWriteOnce
  nfs:
    server: "{{ .Values.prometheusNFS }}"
    path: "/"
```
PVC for Grafana:
grafana-pvc.yml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  labels:
    env: dev
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

```
PVC for Prometheus:
prometheus-pvc.yml
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-pvc
  labels:
    env: prod
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
Deployment for Grafana:
grafana-deployment.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana-deployment
spec:
  selector:
    matchLabels:
      env: dev
      dc: IN
  template: 
    metadata: 
      labels:
        env: dev
        dc: IN
    spec:
      containers:
      - name: grafana-con
        image: monitoringartist/grafana-xxl:latest
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: mypvc
          mountPath: /var/lib/grafana
      volumes:
      - name: mypvc
        persistentVolumeClaim:
          claimName: grafana-pvc
```
Config Map for Prometheus:
prometheus-cm.yml
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-cm
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    scrape_configs:
      - job_name: 'localhost'
        static_configs:
        - targets: ['localhost:9090']
```
Deployment for Prometheus:
prometheus-deployment.yml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-deployment
spec:
  selector:
    matchLabels:
      env: prod
      dc: IN
  template: 
    metadata: 
      labels:
        env: prod
        dc: IN
    spec:
      containers:
      - name: prometheus-con
        image: prom/prometheus
        ports:
        - containerPort: 9090
        volumeMounts:
        - name: prom-pvc
          mountPath: /prometheus/data
        - name: prom-cm
          mountPath: /etc/prometheus/prometheus.yml
          subPath: prometheus.yml
      volumes:
      - name: prom-pvc
        persistentVolumeClaim:
          claimName: prometheus-pvc
      - name: prom-cm
        configMap:
          name: prometheus-cm
```
After all the YAML files have been created, the next step is to take care of the variables.
You might have seen that I have used a lot of variables at different places in the above YAML files. Where are these variables going to be declared?
There is a fixed file called ‘values.yaml’ in the chart directory and we have to declare all the variables in this file only.
touch values.yaml
Content of the values.yaml file:
```
grafanaPort: 31001
prometheusPort: 31002
grafanaNFS: 172.31.35.144
prometheusNFS: 172.31.24.109
```
In the same location, we have to create a ‘Chart.yaml’ file and inside this file, we will provide the version details of our app. We can say that this is the configuration file of the Helm.
Content of the ‘Chart.yaml’ file:
```
apiVersion: v1
name: prom-grafana
version: 0.1
appversion: 1.0
description: Integration of prometheus and grafana
```
And here comes the final step, to deploy all the above resources, all we need to do is to install this chart. Use the above command to do so:
heml install prom-grafana /helm-ws/prom-grafana
After running the above command, both the Prometheus and Grafana will be deployed. Here is the status:
```
>> kubectl get all
>> kubectl get pv
>> kubectl get pvc
```

Congratulations! All the desired resources have been successfully deployed, just in one click. Here is the result:
```
https://3.6.37.15:31001/login
https://3.6.37.15:31002/targets
```

SUMMARY

So, this was a Helm chart showing the integration of Prometheus and Grafana, and the best part was to launch the entire setup in one click.
