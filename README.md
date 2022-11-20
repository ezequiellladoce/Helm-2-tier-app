# Helm depoy a 2 tier app

In this repo we will deploy the 2 tier app from https://github.com/ezequiellladoce/k8s-2-tier-app in kubernetes with helm. 

Helm is a package manager for Kubernetes. With helm we can simplify the deployment of a complex app by creating charts that contain all the information to run applications on your cluster.

Basically a helm has the following structure:

   Chart.yaml

   values.yaml

   .helmignore

   charts/

   templates/

Where:

 - Chart.yaml: it stores the metadata about the chart and version information
 - values.yaml: it contains the configuration values which are interpolated inside the template
 - charts/: this directory contains the packages of the dependencies we add in the requirements file.
 - templates/: this is the core directory that contains the actual template of our application and contains — application.yaml, deployment.yaml, ingress.yaml, service.yaml and serviceaccount.yaml, etc.

Kubernetes objects will result of the combination of the value.yalm file with the templates/ folder there are Helm templates are.

## Prerequisites 📋

- A k8s cluster v1.9 or later, for testing purposes can be minikube (https://minikube.sigs.k8s.io/docs/). You can check the k8s server with the command kubectl version
- kubectl configured to communicate with the cluster
- Helm V3

## Starting 🚀

Create a a helm chart

```
helm create helm-chart
```
![create](https://user-images.githubusercontent.com/67485607/202907375-4b67f084-cbb2-40ac-92d7-dfd85ead3cdf.png)

We will not need the templates files generated files inside ./templates folder, so we can delete them. We can also clear all the content inside values.yaml.

Now we have to create the chart, first we will create the template of the mysql StatefulSet in the ./templates folder.

```
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{  .Values.mysql.name  }}
  namespace : {{  .Values.mysql.nameSpace  }}
spec:
  replicas: {{ .Values.mysql.replicaCount }}
  serviceName: {{ .Values.mysql.name  }}-headless-svc
  selector:
    matchLabels:
      app: {{  .Values.mysql.name  }}-helm
```

The {{  .Values.mysql.name  }} reference the value located in values.yaml inthe chart folder

```
mysql:
  name: mysql 
```
Another example is:

```
mysql:
  image:
    repository: mysql:5.7
```

In that way we define all that we need in the ./templates and in the value.yaml. If you need to do a more complex templatization you can check the helm official documentation.
 
## Deploy 📦  

Clone the repo https://github.com/ezequiellladoce/Helm-2-tier-app

### Create namespace

kubectl create namespace helm-2-tier-app

### Install the chart

cd the values.yaml folder and install the chart

```
helm upgrade --install myapp --values values.yaml . -n helm-2-tier-app
```
Were my app isn teha name of the release

The outpust is similar to that:
```
Release "myapp" does not exist. Installing it now.
NAME: myapp
LAST DEPLOYED: Sun Nov 20 14:49:15 2022
NAMESPACE: helm-2-tier-app
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

#### Test the MySql SatefulSet

##### Access the database 

We can access directly to the pod with the pod name in a StatefulSet the pod name is

```
kubectl exec -it mysql-0 -n helm-2-tier-app -- mysql -u root -p
```
The output is similar to:
```
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 9
Server version: 5.6.51 MySQL Community Server (GPL)

Copyright (c) 2000, 2021, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```
And you can run a mysql command:
```
mysql> show databases
    -> ;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.02 sec)
```

##### Test the DNS suport

We will connect to the mysql-client pod creted with the release and install the mysql client

```
kubectl exec --stdin --tty mysql-client -n helm-2-tier-app k8s-app -- sh
apk add mysql-client
exit
```
As the StatefulSet  with the Headless svc provides DNS support we can connect throw the pod dns or the service DNS.
The DNS structure of the pod dns is:

(statefulset)-{0..N-1}.$(service).$(namespace).svc.cluster.local

The DNS of the pod 0 is: 

mysql-0.mysql-headless-svc.helm-2-tier-app.svc.cluster.local

we can connect to the database with the pod dns
```

kubectl exec --stdin --tty mysql-client -n helm-2-tier-app -- sh

mysql -h  mysql-0.mysql-headless-svc.helm-2-tier-app.svc.cluster.local -u root -p
MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.002 sec)
```
The DNS structure of the service is:

$(service).$(namespace).svc.cluster.local

The DNS of the svc is:

mysql-headless-svc.helm-2-tier-app.svc.cluster.local


we can connect to the database with the svc dns
```
/ # mysql -h mysql-headless-svc.helm-2-tier-app.svc.cluster.local -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.7.40 MySQL Community Server (GPL)

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MySQL [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| todos              |
+--------------------+
5 rows in set (0.010 sec)
```
### Test the front-end
 
To access the front-end we use the kubectl port-forward   
```
kubectl port-forward service/myadmin-svc 3000:80 -n helm-2-tier-app
```

You can check the PhpMyAdmin in your browser with the url localhost:3000
Login with root /password add check the databases

![front](https://user-images.githubusercontent.com/67485607/202907779-e7a87dcc-99aa-427f-92c2-306e3cdb21fd.PNG)

### Uninstall the chart

helm uninstall myapp -n helm-2-tier-app
