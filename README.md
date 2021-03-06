# Kubernetes and MySQL

Below are my demos for running MySQL on Kubernetes. From a simple setup of one single MySQL instance using a deployment and NFS disk to running our two cluster databases using our new k8s operators.

All work below assumes you already have a Kubernetes cluster up and running, if you need some guidance on installing Kubernetes v1.22 on OL7.9 then you can find my notes in this [file](https://github.com/wwwted/Kubernetes/blob/master/misc/k8s-install.txt)

### Running MySQL on Kubernetes
These demos have been tested using Oracle Cloud (OCI) but there are no spcific OCI features used. Standard Kubernetes and NFS have been used in all examples so everything should work also on other cloud providers or on-prem.

##### Storage for PV
In below demos below I have a NFS Server running on IP 10.0.0.159 that shares 3 folders.

The NFS Server exposes folders:
- /var/nfs/pv0
- /var/nfs/pv1
- /var/nfs/pv2 

NFS Server setup [here](https://github.com/wwwted/Kubernetes/blob/master/nfs.md)

##### Demos: 
- [MySQL using deployment with persistent volumes](https://github.com/wwwted/Kubernetes/blob/master/k8s_mysql.md)
- [InnoDB Cluster using StatefulSets](https://github.com/wwwted/Kubernetes/blob/master/k8s_innodb_cluster.md)
- [InnoDB Cluster using our Operator](https://github.com/wwwted/Kubernetes/blob/master/k8s_innodb_cluster_operator.md)
