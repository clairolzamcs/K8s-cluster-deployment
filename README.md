# Creating K8s Cluster deploy cointainarized stateless applications using the K8s manifest, expose the applications as NodePort Services, and roll out an updated version of the application.


## STEPS AND EXPLANATIONS:

1.	The local K8s cluster is running on your Amazon EC2 instance. Demonstrate that this is a single node cluster and that all the basic K8s components are running successfully.
a.	Make sure to copy all necessary files from working environment going to the ec2 by running `scp -i assignment2 scp -i assignment2 ../manifests/* ../init_kind.sh <public IP of ec2>:/tmp` . Just update files to be copied as necessary. 
b.	Executed the init_kind.sh that reads kind.yaml so our kind cluster will be created.
c.	Once created, to see the created node, i executed `k get nodes` and to get all resources from all namespaces after creation of k8s cluster, i executed `k get all -A` which displayed below as expected. 
 
d.	By the way, this was the output after cluster creation.
 

e.	I tried to get the nodes available in my local cluster and also the cluster IP by running `k get nodes` and `k get all`.
 

 
2.	What is the IP of the K8s API server in your cluster?
a.	In order to know some info about  K8s API server in the cluster:
i.	$ kubectl cluster-info
ii.	Kubernetes control plane is running at https://127.0.0.1:35323
iii.	KubeDNS is running at https://127.0.0.1:35323/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
b.	I also ran:
i.	$ k get nodes -owide
ii.	NAME                 STATUS   ROLES    AGE   VERSION    INTERNAL-IP   EXTERNAL-IP   OS-IMAGE       KERNEL-VERSION                  CONTAINER-RUNTIME
iii.	kind-control-plane   Ready    master   75m   v1.19.11   172.18.0.2    <none>        Ubuntu 21.04   4.14.314-238.539.amzn2.x86_64   containerd://1.5.2
c.	I also ran:
i.	$ k get svc kubernetes
ii.	NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
iii.	service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   16m
d.	Cluster / API server IP: 10.96.0.1
e.	Master Node IP: 172.18.0.2
 

3.	Deploy MySQL and web applications as pods in their respective namespaces.
a.	Needed to create separate namespaces for app and db
i.	k create namespace db
ii.	k create namespace app
 
b.	To pull docker images from EC2: 
i.	Login to docker by running  `aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 009147451403.dkr.ecr.us-east-1.amazonaws.com`
 

c.	Pull 
i.	docker pull 009147451403.dkr.ecr.us-east-1.amazonaws.com/prod-app:<tag>
ii.	docker pull 009147451403.dkr.ecr.us-east-1.amazonaws.com/prod-db:<tag>
 








d.	Apply the manifests to create the pods in their respective namespaces.
i.	Secret.yaml - this will create a k8s secret that contains the DB credentials. Apply in both namespaces.
 

ii.	Db-pod.yaml - pod manifest for db. After applying this, describe the pod and get the IP address. We need this in our app-pod.yaml. `k apply -f db-pod.yaml`
 














1.	Check if we can connect to the database using the correct username and password. Below we can see that we were able to query from the `employee` table under `employees` database.
a.	k exec -it db -n db /bin/bash
b.	mysql -u root -p employees
 

iii.	App-pod.yaml - pod manifest for app. Populate the DBHOST env var first with db-pod IP before applying.
1.	After applying, let’s verify if the pod is running by checking if the pod exists and by looking at its logs `k logs app -n app` 
 

e.	Can both applications listen on the same port inside the container? Explain your answer.
By default, each pod within a Kubernetes cluster has its own IP address, and containers within the pod can communicate with each other using localhost. Therefore, it is possible for both the MySQL and web application containers to listen on the same port inside their respective containers without conflict since they are isolated within separate pods. Also, in this case the DB pod and app pod are in their own respective namespaces. But in a case if both apps are in the same container, it is technically possible for both applications to listen on the same port inside a single container. However, i’ve read that it is not recommended cos of some potential conflicts in managing network connections between the applications. It is generally better to have each application within a pod listen on a different port to allow independent communication.

f.	In order to check if we can get a valid response, let us exec into the pod and try to curl our exposed port by running `curl http://localhost:8080`. And in another terminal, let us tail the server’s logs by running `k logs app -n app -f`. On the left side of the below screenshot, we were able to get a valid response and on the right side, we see that it reflects in the logs.
 

4.	Deploy ReplicaSets of the applications with 3 replicas using ReplicaSet manifest. Use the “app:employees” and “app:mysql” labels respectively to create ReplicaSets for MySQL and web applications.
a.	Let’s apply our db replicaset manifest `k apply -f db-rs.yaml`
b.	Let’s see if the desired count of replicas (3) is generated using `k get rs -n db`. In below screenshot we can see that all 3 replicas desired are ready. We can also see now that there are 4 pods - 1 from the pod manifest we previously created plus 3 from the replicaset we just created. `kgp -n db`
 
c.	Let’s apply our app replicaset manifest `k apply -f app-rs.yaml`
d.	Let’s see if the desired count of replicas (3) is generated using `k get rs -n app`. In below screenshot we can see that all 3 replicas desired are ready. We can also see now that there are 4 pods - 1 from the pod manifest we previously created plus 3 from the replicaset we just created. `kgp -n app`
 

e.	Is the pod created in step 2 governed by the ReplicaSet you created. Explain
The pod that I previously created using pod manifest is not governed by the ReplicaSet I just created because a ReplicaSet is a separate object/resource that specifies the desired number of replicas and the pod template to use. It actively monitors and manages the creation or deletion of pods to maintain the desired state based on the labels that we specifically set (matchLabels). 

5.	Create deployments of the MySQL and web applications using deployment manifests.
a.	Let’s apply our db deployment manifest `k apply -f db-deployment.yaml`. Note that we are using the same labels as we used in replicaset as selectors.
b.	Let’s see if the desired count of replicas (3) is generated using `k get deployment -n db`. In below screenshot we can see that all 3 replicas desired are ready. And upon executing `kgp -n db`, we can see that the previously created pods from replicaset are starting to terminate. And later on, we landed with a final pod count of 4 - 1 from the pod manifest and 3 from the deployment manifest.
 

c.	Let’s apply our app deployment manifest `k apply -f app-deployment.yaml`. Note that we are using the same labels as we used in replicaset as selectors.
d.	Let’s see if the desired count of replicas (3) is generated using `k get deployment -n app`. In below screenshot we can see that all 3 replicas desired are ready. And upon executing `kgp -n app`, we can see that the previously created pods from replicaset are starting to terminate. And later on, we landed with a final pod count of 4 - 1 from the pod manifest and 3 from the deployment manifest.
 

e.	Is the replicaset created in step 3 part of this deployment? Explain.
When I applied the deployment manifest, I have observed that the pods from replicaset were terminated one by one. This is expected because the pods are not directly managed by a deployment, but a deployment manages a replicaset therefore since they used the same selector labels, the deployment took over the management of the pods created from the replicaset. We can observe this if we run `k get rs -n db` and `k get rs -n app` where we will see 2 replicasets - 1 from the replicaset manifest and 1 from the deployment manifest.
 

6.	Expose web application on NodePort 30000 using service manifest. 
a.	Let’s apply our app service manifest `k apply -f app-service.yaml`.
b.	Let’s see if it was indeed created
 

7.	Demonstrate that you can reach the application from your Amazon EC2 instance using curl and from the browser.
a.	Running `curl 3.86.107.248:30000`
 

b.	Trying to access in a browser
 

8.	Expose MySQL using Service of type ClusterIP
a.	Let’s apply our db service manifest `k apply -f db-service.yaml`.
b.	Let’s see if it was indeed created. And yes, our db service was created as we see now it has its clusterIP.
 

9.	Testing the application
a.	Let’s now test the application by filling out the form. 
 

b.	Let’s check if it was reflected in the database by hopping into the db. As you can see in the screenshot below, it was able to write it in our database meaning our application server in the app namespace was able to communicate properly to the database in db namespace.
 


CHALLENGES ENCOUNTERED:

1.	After applying my first version of pod manifests, both app and db pods were CrashLoopBackOff status. I checked the app pod first by running “k logs app -n app” and it says `File "app.py", line 15, in <module> DBPORT = int(os.environ.get("DBPORT"))` so i thought it was an issue in environment variables - I have to add them in my container.
2.	Same issue with the db pod where it says
a.	You need to specify one of the following as an environment variable:
- MYSQL_ROOT_PASSWORD
- MYSQL_ALLOW_EMPTY_PASSWORD
- MYSQL_RANDOM_ROOT_PASSWORD
b.	I remember in the first assignment where we started a container using the command `docker run -d -e MYSQL_ROOT_PASSWORD=pw  my_db`, I realized I have to set the MYSQL_ROOT_PASSWORD in my db container’s env vars
3.	After I recreated the db pod with the correct env var, its status became “running”. And so I can now get its IP address by running the command `k describe pod db -n mysql` and look for `IP Address`
4.	We have to make the database credentials private so we cannot commit them to the git repository. I had to look for ways on how I can implement a solution that makes me able to create pods/deployments, etc. using db credentials without having to hardcode them.This is when I decided to use kubernetes secrets.
a.	USER=root
b.	$ PASSWORD=pw
c.	$ echo -n ${USER} | base64
d.	$ echo -n ${PASSWORD} | base64
e.	The output of above will be used in our secret manifest to be applied in both app and db namespaces.
5.	When I was about to test if I can connect to the db pod, i was executing `mysql -u root -p -h 10.244.0.5 employees` however I was prompted with an error `-bash: mysql: command not found`. Solution was to run yum install mysql. I then added that in the ec2’s user data so I won’t have to run it again once I decide to continue with the activity.
6.	When I was trying to get a valid response from my app pod using `curl http://localhost:80`, I got the error `bash: curl: command not found`. I then installed curl first using `apt-get update` then `apt-get install curl`. 
7.	Once curl was successfully installed, i tried to run the `curl http://localhost:80` command again and I got `curl: (7) Failed to connect to localhost port 80: Connection refused`. I then looked back at the app Docker file and saw that we are using 8080. After trying `curl http://localhost:8080`, i got a valid response.
8.	From my cloud9, I was about to check if I can access the running app server through the created service in port 30000 so i ran `curl 3.86.107.248:30000` where 3.86.107.248 is my ec2’s public IP. However, it was just running indefinitely and no output so I thought, there must be something about the connection. I then headed to the ec2’s security group and there I added a new rule that will allow all traffic (just to test if my curl will get through) and indeed that was it. After doing that, my curl returned a valid response.
 

9.	Trying to add item, it was not added in DB
 
	 
I thought maybe the app was not able to communicate to the DB. So I changed the DBHOST to point to the db service’s fully qualified domain name db-service.db.svc.cluster.local
