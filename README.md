__<h1>Deploying Content Management Site with Database over Amazon EKS</h1>__

![EFS_EKS_Joomla_MySQL](https://miro.medium.com/max/875/1*An1TBJ9wFG6yScr4_5oagw.jpeg)<br>

<h3>What is Amazon EKS ?</h3>
<p><b>Amazon EKS</b> is a Kubernetes service provided by AWS that makes it easier to setup Kubernetes on AWS without the need to maintain our own Kubernetes control planes.</p><br>
 
<h3>What is Joomla ?</h3>
<p><b>Joomla</b> is a free and open-source content management system for publishing web content, developed by Open Source Matters, Inc. It is built on a model–view–controller web application framework that can be used independently of the CMS.</p><br>

<h3>What is MySQL ?</h3>
<p><b>MySQL</b> is a relational database management system based on SQL — Structured Query Language. The application is used for a wide range of purposes, including data warehousing, e-commerce, and logging applications. The most common use for MySQL however, is for the purpose of a web database.</p><br>

<h3>What is Amazon EFS ?</h3>
<p><b>Amazon Elastic File System</b> is a cloud storage service provided by Amazon Web Services designed to provide scalable, elastic, concurrent with some restrictions, and encrypted file storage for use with both AWS cloud services and on-premises resources.</p><br><br>


<p>This blog tells how to launch CMS like Joomla with MySQL database as a backend with AWS EFS as a Persistent Volume and on a worker node launched using Amazon EKS.</p><br>

<h2>Part 1 : Setting up EKS Cluster</h2>

<p>First, we need to create a YAML file for setting up EKS including specification of <b>Node Groups</b> (group of Worker Nodes in a Cluster).</p><br>

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig 

metadata:
  name: task-cluster
  region: ap-south-1

nodeGroups:
  - name: nodeGroup1
    desiredCapacity: 2
    instanceType: t2.micro
    ssh:
      publicKeyName: task
  - name: nodeGroup2
    desiredCapacity: 1
    instanceType: t2.small
    ssh:
      publicKeyName: task
  - name: nodeGroup-mixed
    minSize: 2
    maxSize: 5
    instancesDistribution:
      maxPrice: 0.017
      instanceTypes: ["t3.small", "t3.medium"]
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 50
      spotInstancePools: 2
    ssh:
      publicKeyName: task 
```

<br>
<p>But, before creating our cluster , let’s setup CLI tool required for the same i.e., <b>eksctl</b>, basically eksctl is a CLI tool for creating clusters in EKS and it is developed by <b>weaveworks</b>.</p><br>
<p>For setting up eksctl in respective OS , check out the below link.</p>
https://github.com/weaveworks/eksctl <br><br>

<p>After setting up our eksctl CLI tool, let’s set up the cluster.</p><br>

```
eksctl create cluster -f <filename>.yml
```

<br>
<p>After the cluster has been setup , we need to configure kubectl so as to setup connection with Amazon EKS , the command for the same are as follows:</p><br>

```
aws eks update-kubeconfig  --name <name of the cluster specified in YAML file>
```

<br>
<p>It is usually a good practice to create a namespace as it avoids the interference into setup of other works, also it makes it easier to manage multiple work. It can be created using the below command.</p><br>

```
kubectl create namespace <namespace>
```

<br>
<p>After creating a new namespace , we need to set up context in <b>kubeconfig</b> (it provides access to cluster) to the new namespace created and it could be done using the below command.</p><br>

```
kubectl config set-context  --current  --namespace=<namespace>
```

<br>
<p>In AWS console, the EKS cluster setup would look like as follows:</p><br>

![EKS_Cluster_Setup](https://miro.medium.com/max/875/1*ClaZfdal-BGdoBxTSMjzIg.png)<br>

<br>
<p>After executing eksctl command,</p><br>

![EKS_Cluster_Setup](https://miro.medium.com/max/875/1*7pYZFJM7ZXMg75jfiQGa_w.png)<br>

<p align="center"><b>Cluster Launch</b></p><br><br>


<h2>Part 2 : Setting up EFS</h2>
<p>First, access the EFS console in AWS, create a File System using <b>VPC</b> and <b>Security Group ID</b> specified in the EKS cluster created before.</p><br>
<p>First, access the EKS console and then access the cluster created , click on Networking, it would specify the VPC ID and the Security Group ID , place the same while creating File System in EFS.</p><br>
<p>After creating the same , it would look like this :</p><br>

![EKS_Setup](https://miro.medium.com/max/875/1*Wsxx1r_0-HG1jdcyBPlhPw.png)<br><br>

<br>
<h2>Part 3 : Setting up EFS Provisioner and Role-Based Access Control (RBAC)</h2>
<p><b>EFS Provisioner</b> could be used to setup EFS as a Persistent Volume in Kubernetes by using efs-provisioner as image. The YAML code for the same is as follows:</p><br>

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-provisioner
spec:
  selector:
    matchLabels:
      app: efs-provisioner
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:v0.1.0
          env:
            - name: FILE_SYSTEM_ID
              value: fs-9247d243
            - name: AWS_REGION
              value: ap-south-1
            - name: PROVISIONER_NAME
              value: satyam/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-9247d243.efs.ap-south-1.amazonaws.com
            path: /    
```

<br>
<p><b>Note :</b> Use the same File System ID as the one that gets generated after creating EFS and use the DNS Name specified in the EFS in the server field in the YAML file above.</p><br>
<p><b>Role-based access control</b>, is an authorization mechanism for managing permissions around <b>Kubernetes</b> resources.It could be setup using YAML file shown as below:</p><br>

```yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: nfs-provisioner-role-binding
subjects:
  - kind: ServiceAccount
    name: default
    namespace: task
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

<br><br>
<h2>Part 4 : Setting up Storage Class and Deployment for Joomla and MySQL</h2>
<p>A <b>Storage Classes</b> provides a way for administrators to describe the “<b>classes</b>” of <b>storage</b> they offer and use provisioners that are specific to the storage platform or cloud provider to give <b>Kubernetes</b> access to the physical media being used. In our case, the provisioner is Amazon EFS and the YAML file for Storage Class is as shown below:</p><br>

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-efs
provisioner: satyam/aws-efs
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-joomla
  annotations:
    volume.beta.kubernetes.io/storage-class: "aws-efs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-mysql
  annotations:
    volume.beta.kubernetes.io/storage-class: "aws-efs"
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

<br>
<p>Here, along with specifying Storage Classes , PVC has also been specified for both Joomla as well as MySQL, alternatively,each PVC could also be specified in their respective deployment as well.</p><br>
<p><b>Note :</b>Here, the accessModes used is <b>“ReadWriteMany”</b> for both as this setup includes multiple nodes.</p><br>
<p>Let’s now take a look on Deployment file for Joomla and MySQL.</p><br>

<h3>Joomla</h3>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: joomla
  labels:
    app: joomla
spec:
  ports:
    - port: 80
  selector:
    app: joomla
    tier: frontend
  type: LoadBalancer
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: joomla
  labels:
    app: joomla
spec:
  selector:
    matchLabels:
      app: joomla
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: joomla
        tier: frontend
    spec:
      containers:
      - image: joomla:3.9.18-apache
        name: joomla
        env:
        - name: JOOMLA_DB_HOST
          value: joomla-mysql
        - name: JOOMLA_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 80
          name: joomla
        volumeMounts:
        - name: joomla-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: joomla-persistent-storage
        persistentVolumeClaim:
          claimName: efs-joomla
```

<br>
<h3>MySQL</h3>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: joomla-mysql
  labels:
    app: joomla
spec:
  ports:
    - port: 3306
  selector:
    app: joomla
    tier: mysql
  clusterIP: None
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: joomla-mysql
  labels:
    app: joomla
spec:
  selector:
    matchLabels:
      app: joomla
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: joomla
        tier: mysql
    spec:
      containers:
      - image: mysql:5.6
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD 
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: efs-mysql
```

<br>
<p><b>Note:</b> In case of Joomla, Service type has been setup to <b>LoadBalancer</b> for provisioning Load Balancer for the Service.</p><br><br>

<h2>Part 5 : Setting up Secret and Kustomization File</h2>
<p><b>Secrets</b> in Kubernetes is used for storing and managing sensitive information, such as passwords, OAuth tokens, and ssh keys.</p><br>
<p><b>Kustomize</b> is used for setting up management for Kubernetes Objects like Volume, Pods and many more, also secret could be generated as well by using secretGenerator field in the file, in this case, MySQL password is generated. Under resources, the order in which file should be executed has been specified .The file is as follows:</p><br>

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
secretGenerator:
- name: mysql-pass
  literals:
  - password=*******
resources:
  - efs-provisioner.yaml
  - rbac.yaml
  - storage-class.yaml
  - mysql-deployment.yaml
  - joomla-deployment.yaml
```

<br>
<p><b>Note:</b> The file name should be <b>kustomization.yaml</b> only and the command to execute the same is given below.</p><br>

```
kubectl apply -k <kustomization_directory>  -n <namespace>
```

<br>

![kustomization_Setup](https://miro.medium.com/max/875/1*UT-0PyEOpE14vG3RmvR3IQ.png)<br>

<p align="center"><b>Setup using kustomization.yaml file</b></p><br><br>

![pod_launched](https://miro.medium.com/max/790/1*rGets0EaCtXv6oBdMJF23w.png)<br>

<p align="center"><b>Pods launched</b></p><br><br>

![Joomla_external_IP](https://miro.medium.com/max/875/1*un-ruZ6fwIcxpaueiCJUUQ.png)<br>

<p align="center"><b>Address for accessing Joomla could be obtained from External-IP for service/joomla</b></p><br><br>

<h2>Execution</h2>

![First](https://miro.medium.com/max/875/1*1ubZG5do4caIGnOKAfsOVQ.png)<br>

<p align="center"><b>After accessing that address and after doing some initial setup , the screen would look like this</b></p><br><br>

![Second](https://miro.medium.com/max/875/1*jXK3XhcKF2h31bIsOzXJyQ.png)<br>

<p align="center"><b>Article created in Joomla</b></p><br><br>

![Third](https://miro.medium.com/max/875/1*o9uRLQLy67p68NM6d2kHpA.png)<br>

<p align="center"><b>After accessing MySQL in joomla-mysql deployment, the database created while setting up Joomla(i.e., mysql) is visible inside the database indicating that MySQL as a backend has been properly set up</b></p><br><br>


<h2>Thank You :smiley:<h2>
<h3>LinkedIn Profile</h3>
https://www.linkedin.com/in/satyam-singh-95a266182
