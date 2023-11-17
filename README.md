# Deploying Spring-boot(demobank-backend) and Angular(bank-app-ui) full-stack project in K8s cluster in AWS EKS.

# Keycloak Setup 
-  Download keycloak 19.0.1 from official website and then go to bin folder.<br>
    $ bin/kc.bat start-dev –http-port 8180
- Hit URL – http://localhost:8180/  and provide default credentials initially as admin/admin to login inside keycloak-server. <br>
Later create keycloak-server-user say, aadi/aadi (better to avoid root admin user).
- Create new REALM. Say, “demobankdev-realm”.  
- Inside that demobankdev-realm, Create new CLIENT. Say “demopublicclient” with “standard flow” only.  Later provide, <br>
          Valid redirect uri - http://localhost:4200/dashboard <br>
          Valid post logout redirect uri - http://localhost:4200/home <br>
          Web origins - * <br>
Advanced -> Advance settings -> set Proof Key for Code Exchange Code Challenge Method as “S256” <br>

 - Now go to USER and create new user for your login in demobank website (enduser).<br>
    Say,                      Username – aadi                               Password - 12345

- Note : You can spin up docker for using keycloak as well, if you want to avoid installing keycloak in your local system. <br>
Or Instantiate EC2 in AWS cloud where you can install keycloak. <br>
Then change your configuration of localhost to AWS provided IP address in springboot.

# MySQL Setup.
- Install MySQL in your system, with username/password as root/root. <br>
Create one database schema with name, “demobank”.
- Note: One can spin up docker OR can use AWS cloud for MySQL too, change your database properties URL accordingly in springboot-properties file.


# Step-1 : Setup K8s Cluster. Say, in AWS EKS.
Pre-Requisites :
- AWS account with admin privileges.
- Instance to manage/access EKS cluster using Kubectl
- AWS CLI access to use Kubectl utility

## Steps to create EKS Cluster in AWS :

### Create VPC (using cloud formation with S3 URL with contains YAML or JSON codes / or do it through Terraform / or Manually via AWS UI Console)
  Say, stack name = EKSVPCCloudFormation

### Create IAM role in AWS
  Entity type – AWS Service <br>
 	Select Usecase as ‘EKS’ => EKS Cluster <br>
 	Role Name – EKSClusterRole (you can give any name here) <br>
  
### Create EKS cluster using Created VPC and IAM role.
  Choose cluster endpoint access as : Public & Private
 
### Choose RedHat EC2 instance. (K8s_Client_Machine)
Connect to K8s_Client_Machine via MobaXTerm.

- Install Kubectl via below commands : <br>
$ curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl   <br>
$ sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl   <br>
$ kubectl version –client   <br>

- Install AWS CLI in K8s_Client_Machine with below commands <br>
$ curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o “awscliv2.zip”  <br>
$ sudo yum install unzip  <br>
$ unzip awscliv2.zip  <br>
$ sudo ./aws/install  <br>

- Configure AWS CLI with credentials <br>
Access key ID : [Your-AWS-Access-key]  <br>
Secret key ID : [Your-AWS-Secret-key]  <br>
$ aws configure  <br>
Note : we can use root user access key & secret key as well.

$ aws eks list-clusters  <br>
$ ls ~/.

- Update Kubeconfig file in remote machine from cluster using below command <br>
$ aws eks update-kubeconfig –name <cluster-name> --region <region-code>  <br>
e.g. aws eks update-kubeconfig –name <aadi_eks> --region <ap-south-1>  <br>

### - Create IAM role for worker nodes (usecase as EC2) with below policies : <br>
a)	AmazonEKSWokerNodePolicy <br>
b)	AmazonEKS_CNI_Policy <br>
c)	AmazonEC2ContainerRegistryReadOnly <br>

### – Create worker node group
-	Go to Cluster => Compute => Node Group
-	Select the role we have created for worker nodes
-	Use t2.large
-	Min 2 and Max 5
  
### – Once node group added then check nodes in K8s_Client_Machine 
$ Kubectl get nodes  <br>
$ kubectl get pods –all-namespaces  <br>

### – Now you can create POD and expose the POD using NodePort service 
NOTE : Enable NodePort in Security Group to access that in our browser.

# Step-2: Environment Setup
2.1	Launch Ubuntu VM in AWS cloud. <br><br>
2.2	Connect to Ubuntu VM using MobaXterm <br><br>
2.3	Install docker in Ubuntu VM using below command <br>
```
$ curl -fsSL get.docker.com | /bin/bash 
$ sudo usermod -aG docker $USER 
$ newgrp docker 
$ docker info 
```
2.4	Install maven in Ubuntu VM using below command <br>
$ sudo apt install git <br><br>

2.5	Install git client in Ubuntu VM using below command <br>
$ sudo apt install git <br><br>

# Step-3: Backend Application Deployment
3.1 Clone backend application using git clone  <br>
	$ git clone [repo-url]  <br>
       e.g. $ git clone https://github.com/AadityaUoHyd/demobank-backend.git <br><br>
       
3.2 Perform maven build for backend application  <br>
	$ cd [project-directory] <br>
	$ mvn clean package <br><br>
 
3.3 Write Dockerfile for backend application  <br>
```
	FROM  openjdk:17  
	COPY  target/demobank-backend.jar  /usr/app/
	WORKDIR  /usr/app/
	ENTRYPOINT  [“java”, “-jar”, “demobank-backend.jar”]
	EXPOSE  8080
```
3.4 Create docker image for backend application using below command <br>
```
	$ docker  build  -t  demobankbackendimage  .
	$ docker tag demobankbackendimage  aadiraj48dockerhub/demobankbackendimage
	$ docker login
	$ docker push aadiraj48dockerhub/demobankbackendimage
```
3.5 Connect to K8s Cluster Control Plane. <br><br>

3.6 Create Deployment Manifest file(say, demobank-backend-deployment.yml) for backend application like below <br>
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
 	name: demobankbackendappdeployment
spec:
 	replicas: 2
 	selector:
 		matchLabels:
 			app: demobankbackend
spec:
 	containers:
 		- name: demobankbackendcontainer
 		  image: aadiraj48dockerhub/demobankbackendimage
 		  ports:
 		  - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
 	name: demobankbackendsvc
spec:
 	type: Nodeport
 	selector:
 		app: demobankbackend
 	ports:
 	- port: 80
 	  targetPort: 8080
 	  nodePort: 30001
---
```
3.7 Deploy backend application on K8s cluster <br>
```
	$ kubectl apply -f demobank-backend-deployment.yml
	$ kubectl get pods
	$ kubectl get pods -o wide
	$ kubectl get svc
```
3.8 Access backend applicatiob using URL <br>
	URL: http://node-port:nodeip/
# Step-4: Frontend Application Deployment

4.1 Install Node and Angular CLI in Ubuntu VM using below command <br>
```
$ curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash
$ source ~/.bashrc
$ nvm install node
$ node version
$ npm install -g @angular/cli
$ ng v
```
4.2 Clone Frontend application using git clone <br>
 	$ git clone [repo-url]  <br>
       e.g. $ git clone https://github.com/AadityaUoHyd/demo-bank-ui.git <br><br>
       
4.3 Configure Backend Application URL in Frontend Application <br>
```
	$ cd bank-app-ui
	$ cd src/app/environments/
	$ vi environment.ts
```
Note : configure backend url in frontend application for integration. <br>
Say, rooturl=”http://13.215.12.143:30001”   <br><br>

4.4 Build frontend Application <br>
	$ ng build  <br>
Note: If you get a problem saying, “could-not-find-the-implementation-for-builder-angular-devkit-build-angulardev”, then execute below command. <br>
 	 $ npm install - -save-dev @angular-devkit/build-angular  <br><br>
   
4.5 Create Dockerfile for Angular application <br>
```
#use official nginx image as the base image
FROM nginx:latest
#Copy the build output to replace the default nginx contents
COPY  /dist/bank-app-ui  /usr/share/nginx/html
#Expose port 80
EXPOSE 80
```

4.6 Create docker image for Frontend Application <br>
```
	$ docker build -t bank_app_ui_ng_app .
	$ docker tag bank_app_ui_ng_app  aadiraj48dockerhub/bank_app_ui_ng_app
	$ docker login
	$ docker push aadiraj48dockerhub/bank_app_ui_ng_app
```

4.7 Create deployment manifest for Frontend Application(bankappui-deployment.yml). <br>
```
---
apiVersion: apps/v1
kind: deployment
metadata:
	name: bankappuideployment
spec:
	replicas: 2
	selector:
		matchLabels:
			app: bankappui
	template:
		metadata:
			name: bankappui
			labels:
				app: bankappui
		spec:
			containers:
			- name: bankappuicontainer
			  image: aadiraj48dockerhub/bank_app_ui_ng_app
			  ports:
			  - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
	name: bankappuisvc
spec:
	type: Nodeport
	selector:
		app: bankappui
	ports:
	- port: 80
 	  targetport: 80
 	  nodePort: 30002
---
```

4.8 Deploy frontend application on K8s and expose as nodeport <br>
```
 	$ kubectl apply -f bankappui-deployment.yml
 	$ kubectl get pods
 	$ kubectl get pods -o wide
 	$ kubectl get svc
```

4.9 Access frontend application using URL <br>
 	URL : http://node-ip:nodeport/ 
