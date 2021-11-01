# MySQL Deployment using persistent volumes

In this demo we are setting up one MySQL Server using k8s, we will use a deployment and NFS as storage.
This is just for demo purpose, we will use secrets to store MySQL passwords in a safe manner later on when setting up InnoDB Cluster using our operator.

## Persistent volumes
Setup a NFS Server for your persistent volumes, howto [here](https://github.com/wwwted/Kubernetes/blob/master/nfs.md)
If you are using a public cloud provider you can most likely use dynamic storage options for handling of PV.

In bellow examples I have a NFS Server running on host: 10.0.0.159
The NFS exposes folder:
- /var/nfs/pv0

## Kubernetes configuration
You can look at configuration for kubernetes in [yamls](https://github.com/wwwted/Kubernetes/tree/master/yamls) folder.

First we are creating a persistent volume and a persistant volume clame.
We are specifying that this volume can only be accessed by one node (ReadWriteOnce)
We are also specifying that we will use our NFS server for storage.
More information on PV [here](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

After we have created the persistent volume we will create the MySQL [deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).
First we create a [service](https://kubernetes.io/docs/concepts/services-networking/service/) to expose our application on the network.
Next we create the MySQL deployment using the resourses created earlier.

1) Create a persisten volume (PV):
```
kubectl create -f yamls/01-mysql-pv.yaml
```

2) Start MySQL using a deployments (one MySQL Server using NFS PV)
```
kubectl create -f yamls/01-mysql-deployment.yaml
```

3) Watch deployment being created
```
kubectl get pods --watch
NAME                     READY   STATUS    RESTARTS   AGE
mysql-85b796566c-gw2g6   1/1     Running   0          4s
```

4) Connect to your MySQL instance
```
Fetch Name of pod by running:
kubectl get pods

Then connect to MySQL by running:
kubectl exec -it mysql-85b796566c-gw2g6 -- mysql -uroot -p_MySQL2020_
```

5) Create a MySQL user so you can connect from outside Kubernetes:
```
Login to MySQL:
kubectl exec -it mysql-85b796566c-gw2g6 -- mysql -uroot -p_MySQL2020_

Create new MySQL user:
create user ted@'%' identified by 'Welcome1!';
grant select on *.* to ted@'%';
```

6) Connect to MySQL from the outside using port-forwarding
```
Start port forwading by running (in a separate terminal):
kubectl port-forward service/mysql 13306:3306

Connect using MySQL Client:
mysql -uted -pWelcome1! -hlocalhost -P13306
```

Done!

## If you want to remove everything:
```
kubectl delete -f yamls/01-mysql-deployment.yaml 
kubectl delete -f yamls/01-mysql-pv.yaml

Make sure everything is deleted:
kubectl get pv,pv
kubectl get all -o wide
```

Remember to also empty out the datadir on NFS between tests:
```
sudo rm -fr /var/nfs/pv0/*
ls /var/nfs/pv0/
```
