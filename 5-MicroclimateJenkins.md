

---
# Microclimate and Jenkins Lab
---

![microclimate](images/mcmicroclimate.png)
This lab is compatible with IBM Cloud Private version 3.1.2

---


In this tutorial, you create, install, deploy and run a **cloud-native microservice application** on an IBM Cloud Private platform on Kubernetes.

Microclimate will guide you thru the creation of complete project including all the directories, the manifest files, the monitoring option that you need for a perfect application.

[Link to Microclimate documentation here](https://microclimate-dev2ops.github.io/)

---




# Task 1: Access the ICP console 


From a machine that is hosting your environment, open a web browser and go to one of the following URLs to access the IBM Cloud Private management console:
  - Open a browser
  - go to https://<masterip>:8443
  - Where **masterip** is the ip address of the ICP cluster given by the **instructor** with the password.

![Login to ICP console](./images/login2icp.png)



# Task 2: Install Microclimate

Application workloads can be deployed to run on an IBM Cloud Private cluster. The deployment of an application workload must be made available as a Helm package. Such packages must either be made available for deployment on a Helm repository, or loaded into an IBM Cloud Private internal Helm repository.

Be sure you are connected to your environment : ./connect2icp.sh

Also check helm.

## 1. Prepare the environment

There are several prerequisites:
- create new namespaces

- add persistent volumes

- Create a Docker registry secret for Microclimate

- Patch this secret to a service account.

- Ensure that the target namespace for deployments can access the docker image registry.

- Create a secret so Microclimate can securely use Helm.

- Set the Microclimate and Jenkins hostname values


### Create a microclimate namespace

We are going to install Microclimate into a new namespace called "microclimate". For Jenkins, it will be installed in an isolated namespace.

`kubectl create namespace microclimate`

Results:

```
# kubectl create namespace microclimate
namespace/microclimate created
```

Login with Cloudctl (replace the ipaddress with the one for the cluster):

`cloudctl login -a https://<masterip>:8443 -n microclimate --skip-ssl-validation`

Results:

```
# cloudctl login -a https://159.8.182.80:8443 -n microclimate --skip-ssl-validation

Username> admin

Password> 
Authenticating...
OK

Select an account:
1. mycluster Account (id-mycluster-account)
Enter a number> 1
Targeted account mycluster Account (id-mycluster-account)

Targeted namespace microclimate

Configuring kubectl ...
Property "clusters.mycluster" unset.
Property "users.mycluster-user" unset.
Property "contexts.mycluster-context" unset.
Cluster "mycluster" set.
User "mycluster-user" set.
Context "mycluster-context" created.
Switched to context "mycluster-context".
OK

Configuring helm: /root/.helm
OK
```

### Ensure target namespace for deployments

The chart parameter jenkins.Pipeline.TargetNamespace defines the namespace that the pipeline deploys to. Its default value is "microclimate-pipeline-deployments". This namespace must be created before using the pipeline. Ensure that the default service account in this namespace has an associated image pull secret that permits pods in this namespace to pull images from the ICP image registry. For example, you might create another docker-registry secret and patch the service account:

`kubectl create namespace microclimate-pipeline-deployments`

Results:

```
# kubectl create namespace microclimate-pipeline-deployments
namespace/microclimate-pipeline-deployments created
```

## Create a new ClusterImagePolicy

Microclimate pipelines pull images from repositories other than `docker.io/ibmcom`. To use Microclimate pipelines, you must ensure you have a cluster image policy that permits images to be pulled from these repositories.

A new cluster image policy can be created with the necessary image repositories by the following command:

**Change mycluster.icp with your own cluster name** if necessary in the example below.

Then execute that command

```
cat <<EOF | kubectl create -f -
apiVersion: securityenforcement.admission.cloud.ibm.com/v1beta1
kind: ClusterImagePolicy
metadata:
  name: microclimate-cluster-image-policy
spec:
  repositories:
  - name: mycluster.icp:8500/*
  - name: docker.io/maven:*
  - name: docker.io/jenkins/*
  - name: docker.io/docker:*
EOF
```

Result:

```
clusterimagepolicy.securityenforcement.admission.cloud.ibm.com/microclimate-cluster-image-policy created
```

### Create a Docker registry secret for Microclimate

Change **mycluster.icp with your own cluster name** if necessary in the example below.

```
kubectl create secret docker-registry microclimate-registry-secret \
  --docker-server=mycluster.icp:8500 \
  --docker-username=admin \
  --docker-password=admin \
  --docker-email=null
```

Results:

```console
# kubectl create secret docker-registry microclimate-registry-secret \
>   --docker-server=mycluster.icp:8500 \
>   --docker-username=admin \
>   --docker-password=admin \
>   --docker-email=null
secret "microclimate-registry-secret" created
```

### Create a secret to use Tiller over TLS

Microclimate's pipeline deploys applications by using the Tiller at kube-system. Secure communication with this Tiller is required and must be configured by creating a Kubernetes secret that contains the required certificate files as detailed below.

To create the secret, use the following commands:

```
export HELM_HOME=/root/.helm

kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=$HELM_HOME/cert.pem --from-file=ca.pem=$HELM_HOME/ca.pem --from-file=key.pem=$HELM_HOME/key.pem
```

Results:

```console 
# kubectl create secret generic microclimate-helm-secret --from-file=cert.pem=.helm/cert.pem --from-file=ca.pem=.helm/ca.pem --from-file=key.pem=.helm/key.pem
secret "microclimate-helm-secret" created
```

### Create a new secret and Patch this secret to a service account

And then use these command (**change my cluster.icp with your own cluster name** if necessary):

```
kubectl create secret docker-registry microclimate-pipeline-secret --docker-server=mycluster.icp:8500 --docker-username=admin --docker-password=admin1! --docker-email=null --namespace=microclimate-pipeline-deployments
```

And then patch the serviceaccount :

```
kubectl patch serviceaccount default --namespace microclimate-pipeline-deployments -p "{\"imagePullSecrets\": [{\"name\": \"microclimate-pipeline-secret\"}]}"
```

Results:

```console
# kubectl patch serviceaccount default -p '{"imagePullSecrets": [{"name": "microclimate-registry-secret"}]}'
serviceaccount "default" patched
```



### Add persistent volumes

We need to create some persistent volumes before we start.

First, ensure your persistent storage is using the correct permissions:

`chmod -R 777 /tmp/data01`

Then use that command (cut and paste the complete block and press enter):

```console
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwo-mc1
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwo-mc2
spec:
  accessModes:
  - ReadWriteOnce
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwm-mc1
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle  
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: hostpath-pv-rwm-mc2
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 10Gi
  hostPath:
    path: /tmp/data01
  persistentVolumeReclaimPolicy: Recycle  
EOF
```
Results:
```console
# kubectl create  -f ./pv-mc.yaml
persistentvolume "hostpath-pv-rwo-mc1" created
persistentvolume "hostpath-pv-rwo-mc2" created
persistentvolume "hostpath-pv-rwm-mc1" created
persistentvolume "hostpath-pv-rwm-mc2" created
```

Once created, these 4 volumes (hostpath) can be listed with the following command:

`kubectl get pv` 


Results:
```console
# kubectl get pv
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                       STORAGECLASS               REASON    AGE
helm-repo-pv                  5Gi        RWO            Delete           Bound       kube-system/helm-repo-pvc                   helm-repo-storage                    3h
hostpath-pv-many-test1        50Gi       RWX            Recycle          Available                                                                                    2h
hostpath-pv-once-test1        30Gi       RWO            Recycle          Available                                                                                    2h
hostpath-pv-rwm-mc1           10Gi       RWX            Recycle          Available                                                                                    1m
hostpath-pv-rwm-mc2           10Gi       RWX            Recycle          Available                                                                                    1m
hostpath-pv-rwo-mc1           10Gi       RWO            Recycle          Available                                                                                    1m
hostpath-pv-rwo-mc2           10Gi       RWO            Recycle          Available                                                                                    1m
image-manager-5.10.96.73      20Gi       RWO            Retain           Bound       kube-system/image-manager-image-manager-0   image-manager-storage                3h
logging-datanode-5.10.96.73   20Gi       RWO            Retain           Bound       kube-system/data-logging-elk-data-0         logging-storage-datanode             3h
mongodb-5.10.96.73            20Gi       RWO            Retain           Bound       kube-system/mongodbdir-icp-mongodb-0        mongodb-storage                      3h
```



## 2. Install from the command line 

First define the **ibm-charts** helm repo (if not done in a previous exercise):
```
helm repo add ibm-charts https://raw.githubusercontent.com/IBM/charts/master/repo/stable/
```

Then install Microclimate (change ipaddress with your cluster address)

> **VERY IMPORTANT** : Change the **<masterip>** with the one of your master node.
> It can take a few minutes before you can see the following results:

```
helm install --name microclimate --namespace microclimate --set global.rbac.serviceAccountName=micro-sa,jenkins.rbac.serviceAccountName=pipeline-sa,global.ingressDomain=<masterip>.nip.io ibm-charts/ibm-microclimate --tls
```
Results:

```console
NAME:   microclimate
LAST DEPLOYED: Sat Dec  1 15:18:55 2018
NAMESPACE: microclimate
STATUS: DEPLOYED

RESOURCES:
==> v1/ConfigMap
NAME                                                 DATA  AGE
microclimate-jenkins                                 7     50s
microclimate-jenkins-tests                           1     50s
microclimate-ibm-microclimate-create-mc-secret       1     50s
microclimate-ibm-microclimate-create-target-ns       1     50s
microclimate-helmtest-devops                         1     50s
microclimate-ibm-microclimate-jenkinstest            1     50s
microclimate-ibm-microclimate-fixup-jenkins-ingress  1     50s

==> v1/ServiceAccount
NAME         SECRETS  AGE
pipeline-sa  1        50s
micro-sa     1        50s

==> v1beta1/ClusterRole
NAME                                     AGE
microclimate-ibm-microclimate-cr-devops  50s
microclimate-ibm-microclimate-cr-micro   50s

==> v1beta1/Deployment
NAME                                  DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
microclimate-jenkins                  1        1        1           0          50s
microclimate-ibm-microclimate-atrium  1        1        1           1          50s
microclimate-ibm-microclimate-devops  1        1        1           0          50s
microclimate-ibm-microclimate         1        1        1           1          49s

==> v1beta1/Ingress
NAME                           HOSTS                             ADDRESS  PORTS  AGE
microclimate-jenkins           jenkins.159.8.182.80.nip.io       80, 443  49s
microclimate-ibm-microclimate  microclimate.159.8.182.80.nip.io  80, 443  49s

==> v1/Pod(related)
NAME                                                   READY  STATUS   RESTARTS  AGE
microclimate-jenkins-86f48bc947-bjbbx                  0/1    Running  0         50s
microclimate-ibm-microclimate-atrium-75d746f75c-6x6bc  1/1    Running  0         50s
microclimate-ibm-microclimate-devops-675b78d749-w42lx  0/1    Running  0         50s
microclimate-ibm-microclimate-65cb4576bc-w6xh9         1/1    Running  0         49s

==> v1/Secret
NAME                           TYPE    DATA  AGE
microclimate-jenkins           Opaque  2     50s
microclimate-ibm-microclimate  Opaque  3     47s

==> v1/PersistentVolumeClaim
NAME                           STATUS  VOLUME                  CAPACITY  ACCESS MODES  STORAGECLASS  AGE
microclimate-jenkins           Bound   hostpath-pv-once-test1  200Gi     RWO           50s
microclimate-ibm-microclimate  Bound   hostpath-pv-many-test1  200Gi     RWX           50s

==> v1beta1/ClusterRoleBinding
NAME                                      AGE
microclimate-ibm-microclimate-crb-devops  50s
microclimate-ibm-microclimate-crb-micro   50s

==> v1/RoleBinding
NAME                  AGE
microclimate-binding  50s

==> v1/Service
NAME                                  TYPE       CLUSTER-IP  EXTERNAL-IP  PORT(S)                     AGE
microclimate-jenkins-agent            ClusterIP  10.0.0.30   <none>       50000/TCP                   50s
microclimate-jenkins                  ClusterIP  10.0.0.127  <none>       8080/TCP                    50s
microclimate-ibm-microclimate-devops  ClusterIP  10.0.0.63   <none>       9191/TCP                    50s
microclimate-ibm-microclimate         ClusterIP  10.0.0.175  <none>       4191/TCP,9191/TCP,9091/TCP  50s


NOTES:
ibm-microclimate-1.8.0

1. Access the Microclimate portal at the following URL: https://microclimate.159.8.182.80.nip.io
```



Check that all deployments are ready (the available column should have a 1 on every row) :
`kubectl get deployment`

Results:
```console 
# kubectl get deployment
NAME                                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
microclimate-ibm-microclimate          1         1         1            1           5m
microclimate-ibm-microclimate-atrium   1         1         1            1           5m
microclimate-ibm-microclimate-devops   1         1         1            1           5m
microclimate-jenkins                   1         1         1            1           5m
```

Application workloads can be deployed to run on an IBM Cloud Private cluster. The deployment of an application workload must be made available as a Helm package. Such packages must either be made available for deployment on a Helm repository, or loaded into an IBM Cloud Private internal Helm repository. 

Microclimate has been installed for you on a specific IBM Cloud Private cluster instance. To check that Microclimate and Jenkins are running, go to the ICP console, then **open the hamburger menu on the top left part of the page, then click on Workloads and Helm Releases **



![image-20190308141646744](images/image-20190308141646744-2051006.png)



Then on the search field type **microclimate** (this will search all helm releases containing microclimate string)

![image-20190308141818349](images/image-20190308141818349-2051098.png)

Microclimate should be already deployed and the light should be green. The latest version is 1.11.0.

Click on the blue link (i.e **microclimate**) to see more information about the application:

![image-20190308142252004](images/image-20190308142252004-2051372.png)

Browse the page to the bottom to look at the **Deployment section** :

![image-20190308151545678](images/image-20190308151545678-2054545.png)

Check that you have all 1 in all columns for the 4 deployments. 

Then go to the **bottom** of that page :

![image-20190308151710072](images/image-20190308151710072-2054630.png)



Copy the URL : https://microclimate.ipaddress.nip.io

Finally get access to the **Microclimate portal** with the following link in your browser:
(replace <ipaddress> with your icp ip address)

Accept the license agreement:

![Accept the license](./images/mcaccept.png)

You should click on **No, not this time** and  **Done** and then you are on the main menu:
![Main](./images/mcaccess.png)

This concludes the Microclimate login. 

> Microclimate has been installed on a kubernetes cluster (on IBM Cloud Private) but it can also be installed locally along with your favorite editor.


# Task 3: Install a simple application

You’re now ready to deploy your Kubernetes application to the IBM Cloud Private environment.  In this case, the deploy command will :


Click on **New project** button:

![project](./images/mcproject.png)

Choose Node.JS (because we want to create a Node.js application) and type the name: **nodeone** and click **Next**

![image-20190308152249201](images/image-20190308152249201-2054969.png)

On the next menu, don't choose any service (but you can notice that we can bind a service to your new application), then click on **Create**

![image-20190308152416608](images/image-20190308152416608-2055056.png)

Be patient (it could take a **few minutes**). Your application should appear and the building process could still be running: 

![image-20190311101347322](images/image-20190311101347322-2295627.png)

Notice a few different information on the screen :

- Auto Build : when you change something in your code then you will rebuild automaticaly
- You can see the Application POD ID
- The Application URL is also mentionned
- The Run Load button will send some requests to your application so that you can see how it performs

![image-20190311101816995](images/image-20190311101816995-2295897.png)

After a few minutes, the application should be running (check the green light):

![image-20190308152803014](images/image-20190308152803014-2055283.png)

To access the application, click on the **Open App** button on the left pane:

![image-20190308152831529](images/image-20190308152831529-2055311.png)

The application should appear (this is a very simple page):

![image-20190308152908524](images/image-20190308152908524-2055348.png)

Navigate on the left pane to the **Edit** button:

![image-20190308153005837](images/image-20190308153005837-2055405.png)



At this point, the editor can be used to edit the Node.JS application.

![image-20190308153615235](images/image-20190308153615235-2055775.png)

Expand **nodeone** and you can see all predefined files (like Dockerfile, Jenkinsfile, manifest files) and you can look around the code and files necessary for your application. 

![image-20190308153925679](images/image-20190308153925679-2055965.png)

Expand **nodeone>public>index.html**

![index](./images/mcindex.png)

Go to the end of the index.html file and modify the **congratulation** line by adding your name :

![congratulation](./images/mcmodify.png)

Save (**File>Save**) and then re-build the application (because Auto Build is on, building will start automatically after saving) .

Wait until the green light(Running) to see the modification your simple application:

![build](./images/mcpht.png)

From the main menu of the application, click on **Run Load** and then on **Monitor** to see some metrics:

![build](./images/mcmon.png)



From here, Click on Projects at the top of the page:

![image-20190308154744243](images/image-20190308154744243-2056464.png)



# Task 4: Import and deploy an application

To import a new application from Github, click on the **Project** button. Then choose **Import project**:

![build](./images/mcimport.png)

On the Import page, type `https://github.com/microclimate-demo/node` as the Git location :

![image-20190308154844264](images/image-20190308154844264-2056524.png)

Enter a name : `nodetwo` and click **Finish import and validate**

![image-20190308154938734](images/image-20190308154938734-2056578.png)

The validation will detect automatically detect the type of project and then click on **Finish Validation**:

![image-20190308155104877](images/image-20190308155104877-2056664.png)



Wait until the application is running (it could take 1 minute) : 

![build](./images/mcbuild2.png)

Click on **Open App** icon to go to the application:

![image-20190311103946371](images/image-20190311103946371-2297186.png)


# Task 5: Using a pipeline

Microclimate has been linked with **Jenkins** and so you don't have to install a separated Jenkins instance in IBM Cloud Private. 

This part focuses on Microclimate integration with Jenkins pipeline to build and deploy your microservice application to IBM Cloud Private Kubernetes cluster. You will create a new DevOps pipeline in Microclimate, and then run the CI/CD pipeline to deploy your microservice to IBM Cloud Private.

Goto the **pipeline** icon and **create a pipeline** button:

![image-20190308155730684](images/image-20190308155730684-2057050.png)

Type a pipeline name **pipe2** and a repository to get the application (optional)

![image-20190308155928017](images/image-20190308155928017-2057168.png)  



 Click on **Create pipeline**:

![image-20190308160044319](images/image-20190308160044319-2057244.png)

![image-20190308160140683](images/image-20190308160140683-2057300.png)

Click on **Open Pipeline** to get access to Jenkins  (enter your credentials at some point if needed):

![image-20190308160247361](images/image-20190308160247361-2057367.png)



Wait until the progression ends:

You can look at the bottom left side to see there are some slave executor available :

![image-20190311105944709](images/image-20190311105944709-2298384.png)

Click on the **master** branch to see detailed information on the pipeline:

![build](./images/mcstage.png)



The pipeline is composed of 3 stages (Extract, Build and Notification).

After 2 or 3 minutes, you should see that the deployment is fine :

![image-20190311110504261](images/image-20190311110504261-2298704.png)



If you want to see some details about each stage : move your cursor on the green area :

![image-20190311110658061](images/image-20190311110658061-2298818.png)

And then click on **Logs**:

![image-20190311110735752](images/image-20190311110735752-2298855.png)

From this log, you can see all the steps done during the Extract stage.

Another possibility to get access to the logs is to click on the **#1** :

![image-20190311111025389](images/image-20190311111025389-2299025.png)

Then Click on the Console Output on the left side of the screen:

![image-20190311111114785](images/image-20190311111114785-2299074.png)

And you can scrowll the log till the end to look at the success:

![image-20190311111218408](images/image-20190311111218408-2299138.png)

If you look thru the log, you will understand that the the Node application has been containerized using docker and then the image has pushed to the IBM Cloud Private registry (using port 8500). But the application has not been deployed yet on the cluster. 



# Task 6: Deploying with the pipeline

Microclimate provides a deployment solution. 

After you set up a pipeline to build the application and Docker image, you can now deploy applications.

Click the **Add deployment** button.

![image-20190311113251922](images/image-20190311113251922-2300371.png)



Select **Last successful build for branch** to automatically deploy the latest successful build from a specific branch. When a new build for that branch completes successfully, the deployed build is automatically upgraded with the new build.

![image-20190311113547783](images/image-20190311113547783-2300547.png)

Be sure that you checked the last successful  for the **master branch**. 

Click on **Save** and after a while, you can see the blinking harmer changed to a green mark. 

![image-20190311120857047](images/image-20190311120857047-2302537.png)

You can click on Open Pipeline to look at Jenkins if you want.

You can check on the IBM Cloud Private console : 

https://ipaddress:8443 

From the hamburger menu, **Worksloads** > **Helm Releases**

![image-20190311121857597](images/image-20190311121857597-2303137.png)

Search your deployment (pipeline2...) and click on the link.

At the end of the page, find the service section and identify the node port : 

![image-20190311122151534](images/image-20190311122151534-2303311.png)

Then use the following link to check your application :

http://ipaddress:30251

![image-20190311122731830](images/image-20190311122731830-2303651.png)





# Congratulations

You have successfully created and installed a microservice application with Microclimate and Jenkins.


