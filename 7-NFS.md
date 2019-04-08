![icp000](images/icp000.png)



![image-20190329160530064](images/image-20190329160530064-3871930.png)



# NFS persistent Storage with IBM Cloud Private



Network File System (NFS), due to its simplicity and the well-established techniques, are commonly available in the Enterprise IT infrastructure.

Kubernetes support the NFS based Persistent Volume (PV). 

This document explains the steps to enable the NFS dynamic volume provisioning in IBM Cloud Private (ICP) 3.1.2, and some simple validation test.

#### 0. Prerequisites

Its assumed that an NFS server is already available where the IP/hostname and the exported path are known.

In the meantime, in **all the nodes** of the cluster, make sure the **NFS client** is installed. For an example, in Ubuntu run the following command :

```
sudo apt-get -y update
sudo apt-get -y install nfs-common
```

or for RHEL:

```console
yum -y install nfs-util 
```

>  Important : See in **annex A** how to setup a **NFS server** on a separated VM before you continue this lab.

#### **1. Enable Image Security in ICP**

Be default ICP 3.1.2 enforce the image security allowing only those images that are defined in a white list to run in the cluster.

Execute the following object,

```
kubectl create -f - <<ZZZ
apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
kind: ClusterImagePolicy
metadata:
  name: my-cluster-images-nfs-client
spec:
  repositories:
    - name: quay.io/external_storage/*
ZZZ
```

#### 2. Deploy the NFS Client Provisioner Helm Chart

Run the helm command to install the helm chart with the following command

```
helm repo update

helm install --name nfs-provisioner --set podSecurityPolicy.enabled=true --set nfs.server=<ipaddressNFSserver> --set nfs.path=/data stable/nfs-client-provisioner --tls
```

Supply the NFS server’s IP address/hostname, and the exported NFS path (/data in that case). We also enable the flag *podSecurityPolicy.enabled* which is required for ICP 3.1.2. *(For production usage, you may want to define a name instead of a randomly generated release name)*. Storage class is : nfs-client.

Watch the pod is running, and the storage class is created.

```
# kubectl get pods | grep nfs-client
vigilant-grizzly-nfs-client-provisioner-59887f6dd-86m7p   1/1     Running   0          2d14h

# kubectl get sc | grep nfs-client
nfs-client                 cluster.local/vigilant-grizzly-nfs-client-provisioner   2d14h
```

#### 3. Testing

Let's request a PVC. Create and apply the following K8s object,

```
kubectl create -f - <<ZZZ
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs-claim
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-client
  resources:
    requests:
      storage: 2Gi
ZZZ
```

Watch the PVC is bound,

```
# kubectl get pvc | grep pvc-nfs-claim
pvc-nfs-claim   Bound    pvc-ed1a01ee-51a0-11e9-9d79-06bbfc5cbe29   2Gi        RWO            nfs-client     28s
```

Now let's create a test deployment using that PVC.

```
kubectl create -f - <<ZZZ
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-testing
  labels:
    app: nfs-testing
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-testing
  template:
    metadata:
      labels:
        app: nfs-testing
    spec:
      containers:
      - name: nfs-testing
        image: busybox
        imagePullPolicy: IfNotPresent
        command: ["sh"]
        args: ["-c", "while true; do sleep 60; done"]
        volumeMounts:
          - name: pv-data
            mountPath: /data
      volumes:
        - name: pv-data
          persistentVolumeClaim:
            claimName: pvc-nfs-claim
ZZZ
```

In the deployment, we define a volume using the PVC created in the previous step, then mount it into the container to the path of /data.

Wait for the pod is running, then

```
# kubectl get pods | grep nfs-test
nfs-testing-5f9889c98-v5dx6                               2/2     Running   0          38s
```

Exec into it, check out the /data file system

```
# kubectl exec -it nfs-testing-5f9889c98-v5dx6 -- sh
Defaulting container name to nfs-testing.
Use 'kubectl describe pod/nfs-testing-5f9889c98-v5dx6 -n default' to see all of the containers in this pod.
/ # cd /data
/data # df -h .
Filesystem                Size      Used Available Use% Mounted on
169.50.201.243:/data/default-pvc-nfs-claim-pvc-ed1a01ee-51a0-11e9-9d79-06bbfc5cbe29
                         98.1G     14.2G     78.8G  15% /data
```

Create a file and validate the content,

```
# echo $(hostname) > abc
/data # cat abc
nfs-testing-5f9889c98-v5dx6
```

Exit, and delete the current pod, a new pod should be created. Exec into it again, the file should be persisted with the last content.

```
# kubectl delete pods nfs-testing-5f9889c98-v5dx6
pod "nfs-testing-5f9889c98-v5dx6" deleted

# kubectl get pods | grep nfs-test
nfs-testing-5f9889c98-6g9tq                               2/2     Running   0          55s

# kubectl exec -it nfs-testing-5f9889c98-6g9tq -- sh
Defaulting container name to nfs-testing.
Use 'kubectl describe pod/nfs-testing-5f9889c98-6g9tq -n default' to see all of the containers in this pod.
/ # 

/ # cd /data
/data # cat abc
nfs-testing-5f9889c98-v5dx6
/data # 

```

This is the end.

### Annex A

Find below some basic instructions to setup and configure a NFS server:

1. Install a specific server with RHEL version 7 and some storage

2. Install nfs-util (below an example on RHEL)

`yum -y install nfs-util`

3. Create a data directory

`mkdir /data`

4. Change permissions (important)

`chmod 777 /data`

5. Configure the following NFS server file :

`nano /etc/exports`

Content of the exports file

```console
# After modif
#systemctl restart nfs-config
#systemctl restart nfs-server
#repetoire  host(option)
/data remoteip(rw,async,no_root_squash) 
```

Where /data is your shared directory and remoteip is the ip address of your kubernetes nodes. You need to add all the ip addresses (and options) for **all members (nodes)** of the cluster in a **row**.

For example :

```console
/data 158.176.83.201(rw,async,no_root_squash) 158.176.83.202(rw,async,no_root_squash) 158.176.83.203(rw,async,no_root_squash) 158.176.83.204(rw,async,no_root_squash) 158.176.83.205(rw,async,no_root_squash)
```

Then don't forget also to **restart** both the nfs-config and nfs-server

```console
# systemctl restart nfs-config
# systemctl restart nfs-server
```

![icp000](images/icp000.png)







