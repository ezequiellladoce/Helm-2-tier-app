# Helm-2-tier-app

kubectl create namespace helm-2-tier-app

helm install -f values.yaml mysql . -n helm-2-tier-app  

helm uninstall mysql -n helm-2-tier-app
 
helm upgrade --install mysql --values values.yaml . -n helm-2-tier-app

PS C:\00_Devops\Helm-2-tier-app\MySql> helm upgrade --install mysql --values values.yaml . -n helm-2-tier-app
Release "mysql" has been upgraded. Happy Helming!
LAST DEPLOYED: Fri Nov 18 13:44:03 2022
NAMESPACE: helm-2-tier-app
REVISION: 2
TEST SUITE: None

PS C:\00_Devops\Helm-2-tier-app\MySql> kubectl get pods -n helm-2-tier-app
NAME           READY   STATUS    RESTARTS   AGE
mysql-1        1/1     Running   0          9s
mysql-2        1/1     Running   0          5s
mysql-client   1/1     Running   0          102s

PS C:\00_Devops\Helm-2-tier-app\MySql> kubectl get svc -n helm-2-tier-app 
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
mysql-headless-svc   ClusterIP   None         <none>        3306/TCP   2m17s

PS C:\00_Devops\Helm-2-tier-app\MySql> kubectl get pv  -n helm-2-tier-app
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                STORAGECLASS   REASON   AGE
pvc-0fb60844-82ed-432c-86c8-414bdd89d6ea   1Gi        RWO            Delete           Bound    helm-2-tier-app/mysql-disk-mysql-2   hostpath                79s
pvc-c41c0044-f841-4c10-91dc-7711db4e5da0   1Gi        RWO            Delete           Bound    helm-2-tier-app/mysql-disk-mysql-0   hostpath                84m
pvc-d48c14dc-307c-4a5f-8222-07874dea4a81   1Gi        RWO            Delete           Bound    helm-2-tier-app/mysql-disk-mysql-1   hostpath                82s
PS C:\00_Devops\Helm-2-tier-app\MySql> kubectl get pvc  -n helm-2-tier-app
NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
mysql-disk-mysql-0   Bound    pvc-c41c0044-f841-4c10-91dc-7711db4e5da0   1Gi        RWO            hostpath       85m
mysql-disk-mysql-1   Bound    pvc-d48c14dc-307c-4a5f-8222-07874dea4a81   1Gi        RWO            hostpath       94s
mysql-disk-mysql-2   Bound    pvc-0fb60844-82ed-432c-86c8-414bdd89d6ea   1Gi        RWO            hostpath       90s
PS C:\00_Devops\Helm-2-tier-app\MySql>

#### Test the MySql SatefulSet

##### Access the database 

We can access directly to the pod with the pod name in a StatefulSet the pod name is

```

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
As you can see the todos database was created throw the mysql image env in the stateful set introduced by the ConfigMap
 
In the StatefulSet
```
configMapKeyRef:
  name: mysql-configmap
  key: MYSQL_DATABASE     
```
In the ConfigMap
```
data:
  MYSQL_DATABASE: todos
```

##### Test the DNS suport

First we will create a Msql client pod
```
kubectl apply -f mysql-client.yaml -n k8s-app

```
we connect to the pod and install the mysql client

```
kubectl exec --stdin --tty mysql-client -n k8s-app -- sh
apk add mysql-client
exit
```
As the StatefulSet  with the Headless svc provides DNS support we can connect throw the pod dns or the service DNS.
The DNS structure of the pod dns is:

(statefulset)-{0..N-1}.$(service).$(namespace).svc.cluster.local

The DNS of the pod 0 is: 

mysql-0.headless-mysql-svc.k8s-app.svc.cluster.local

we can connect to the database with the pod dns
```
kubectl exec --stdin --tty mysql-client -n k8s-app -- sh
mysql -h mysql-0.headless-mysql-svc.k8s-app.svc.cluster.local -p
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

The NDS of the svc is:

headless-mysql-svc.k8s-app.svc.cluster.local


we can connect to the database with the svc dns
```
/ # mysql -h headless-mysql-svc.k8s-app.svc.cluster.local -p
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