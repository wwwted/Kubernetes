# InnoDB Cluster on Kubernetes using the InnoDB Cluster Operator

In this demo we are setting up InnDB Cluster on Kubernetes, we will use our new operator and NFS disks as storage (PV).
This demo is using Kubernetes v1.22 on OL7.9, NFS storage was used as PV so should work for any cloud or on-prem deployment.


## Setup NFS Server to act as your persistent volume.
Setup a NFS Server for your persistent volumes, howto [here](https://github.com/wwwted/Kubernetes/blob/master/nfs.md). 
If you are using a public cloud provider you can most likely use dynamic storage options for handling of PV. Our yaml files on [github](https://github.com/mysql/mysql-operator/tree/trunk/samples) are created for [OKE](https://www.oracle.com/se/cloud-native/container-engine-kubernetes/) on [Oracle Cloud](https://www.oracle.com/se/cloud/).

In bellow example I have a NFS Server on IP: 10.0.0.159 

This NFS server exposes folders:
- /var/nfs/pv0
- /var/nfs/pv1
- /var/nfs/pv2

## Kubernetes configuration
You can look at configuration for kubernetes in [yamls](https://github.com/wwwted/Kubernetes/tree/master/yamls) folder. The two yaml files for the operator demo are 03-innodb-cluster-operator-pv.yaml and 03-innodb-cluster-operator.yaml

## Documentation
- MySQL [docs](https://dev.mysql.com/doc/mysql-operator/en/)
- Project on [Github](https://github.com/mysql/mysql-operator)

## Deploy the InnoDB Cluster Operator
```
kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-crds.yaml
kubectl apply -f https://raw.githubusercontent.com/mysql/mysql-operator/trunk/deploy/deploy-operator.yaml
```
Find more information:
```
kubectl get pods -n mysql-operator
kubectl describe deployment -n mysql-operator mysql-operator
kubectl get deployment -n mysql-operator mysql-operator
kubectl logs -n mysql-operator mysql-operator-6fd4df855f-794m4
```

## Create a InnoDB Cluster 
Before we start, lets create a dedicated namespace for our InnoDB Cluster.

##### Create unique namespace for the cluster:
```
kubectl create namespace innodb-cluster
kubectl get namespaces
```
Set default namespace to innodb-cluster for the rest of the session:
```
kubectl config set-context --current --namespace=innodb-cluster
```

##### Create secret for storing MySQL user password
Next we need to store the MySQL password in a safe mannaer, let's create a secret. This code should of cource not be shared on git for production setup but for this demo it's not important.
```
kubectl create secret generic  mypwds \
        --from-literal=rootUser=root \
        --from-literal=rootHost=% \
        --from-literal=rootPassword="root"
```
View secrets by running (remember that secrects are only available withn the current namespace):
```
kubectl get secrets
```

##### Create PV:
Now it's time  to create three persistent volumes (pv0-pv2) for our InnoDB Cluster nodes.
In the 03-innodb-cluster-operator-pv.yaml file we are specifying that these PV can only be accessed by one node (ReadWriteOnce), We are also specifying that we will use our NFS server for storage.
We also need to have matching storageClassName in the both yaml files.
Name of PV's are not that important, but if you want you can allways create names that match the PVC created by the operator, these are named datadir-mycluster-n (where 'n' starts at 0).
We will set the storageClassName to "slow" so these PV can only be claimed by PVC with matching storageClass.
More information on PV [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

```
kubectl create -f yamls/03-innodb-cluster-operator-pv.yaml
kubectl get pv (short for kubectl get persistentvolumes)
               (should be in STATUS Available)
```

##### Deploy InnoDB Cluster
After we have created the persistent volumes we can create the cluster. Lets start three MySQL instances and one MySQL Router instance. Also remember to set storageClassName to "slow" to claim PV created above.

```
kubectl create  -f  yamls/03-innodb-cluster-operator.yaml
```

##### Look at:
```
kubectl get pv,pvc
kubectl get innodbcluster
kubectl get innodbcluster --watch
```

If there are problems look at logs: 
```
kubectl get pods
kubectl logs mycluster-0 mysql
kubectl get pods -n mysql-operator
kubectl logs -n mysql-operator mysql-operator-6fd4df855f-794m4
```
Look at configuration:
```
kubectl describe pod mycluster-0
```

##### Login to MySQL host on pod mycluster-0:
```
kubectl exec -it  mycluster-0 -c mysql -- bash
```

##### Login to MySQL on pod mycluster-0:
```
kubectl exec -it  mycluster-0 -c mysql -- mysql -uroot -proot
+
status;
show databases;
```

##### View status of the cluster using MySQL Shell
```
kubectl exec -it mycluster-0 -c mysql -- mysqlsh root:root@localhost:3306
+
cluster=dba.getCluster()
cluster.status()
\quit
```

##### Connect to InnoDB Cluster via MySQL Router
Fist, look at the services:
```
kubectl get services
kubectl describe service mycluster
```

The services for our InnoDB Cluster does not expose any external IP so we need some way to connect to the internal IP. For this demo we will use a simple port-forward command. Start a new termainal and run below command:
```
kubectl port-forward service/mycluster 6446:6446
```
This will make the service "mycluster" runnin on port 6446 available on localhost.

Now we can connect to the cluster by running:
```
mysql -uroot -proot -h127.0.0.1 -P6446
+
select @@port,@@hostname;
status;
show databases;
exit
```
MySQL Client can be installed by running ```sudo yum install mysql```

##### Status output of a healthy cluster
```
[opc@k8s-node1 ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
mycluster-0              2/2     Running   0          3m13s
mycluster-1              2/2     Running   0          2m26s
mycluster-2              2/2     Running   0          48s
mycluster-router-z7q4x   1/1     Running   0          2m26s
```

## Remove Cluster and PV/PVS + empty NFS
```
kubectl delete -f 03-innodb-cluster-operator.yaml
kubectl delete pvc datadir-mycluster-0
kubectl delete pvc datadir-mycluster-1
kubectl delete pvc datadir-mycluster-2
kubectl delete -f 03-innodb-cluster-operator-pv.yaml
kubectl get pv,pvc
(+ remeber to delete all data under NFS mounts)
(optional)
kubectl delete secrets mypwds
```
Make sure all is deleted:
```
kubectl get pv,pv
kubectl get all -o wide
```
Remember to also empty out the datadir on NFS between tests:
```
sudo rm -fr /var/nfs/pv[0,1,2]/*
ls /var/nfs/pv[0,1,2]/
```

## Extras
- Install MySQL Client: ```sudo yum install mysql```
- Install MySQL Shell: ```sudo yum install mysql-shell```
- More information around InnoDB Cluster [here](https://github.com/wwwted/MySQL-InnoDB-Cluster-3VM-Setup)
- Whenever deploying new stuff look at: watch kubectl get all -o wide
- Good training on Kubernetes: https://www.youtube.com/user/wenkatn/playlists
- Good training on Kubernetes: https://github.com/justmeandopensource
