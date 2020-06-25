---
layout: post
title:  "Spring boot with elastic search on kubernetes"
date:   2020-06-13 2:32:36 +0530
categories: Java Springboot Kubernetes ElasticSearch
---
I have recently developed a demo **springboot (version 2.2.5)** application which use **java 11** (using **modular system** introduced in java 9), an **elasticsearch (version and 6.6.1)** and **mysql 
(version 8.0.17)**. Then I went ahead and tried to deploy it in a local **kubernetes** cluster (using minikube). Although, there were numerous tutorials about deployment in kubernetes, I could not find a sample application which cover all the aspects which I requires in a single place. In this article, I will briefly go through the some of the issues which I faced during the deployment and the solutions for that ( ofcourse, which I got it from other sources). I am not going through all the details of the kubernetes deployment file which I used because you can find the details in so many other places. 

{% if post.excerpt != post.content %}
    <a href="{{ site.baseurl }}{{ post.url }}">Read more</a>
{% endif %}

I planned to deploy the application in the following way. Mysql database will be running in the system outside of the kubernetes cluster. Springboot app which will run in the kubernetes cluster will write some data to the database. Then elasticsearch index will create using this db data. Then, finally we will search in the created  elastic search index from springboot app. We will have 3 endpoints in the springboot do these actions 1. **addToDb** end point will write  to the db 2. **createIndex** will load data from the db and will create the elastic search index 3. **search** will search in the elastic search index to fetch data which match to our criteria. Source code of the springboot app can see [here][git-hub]. Please see the deployment digaram of the application.

[git-hub]: https://github.com/deleSerna/springbootelasticsearchdemo


![Deployment diagram](/assets/images/2019-03-23-springboot/springbootonkubernetes.png)

I used [minikube][mini-kube] (version 1.11.0) to run kubernetes locally. 
To start the minikube, please  use ```minikube start```.
Since we are using a custom springboot application, first we have to create a docker image of the application and put it in *minikube’s docker image registry*.  We have to pay attention that we have to put the image in minikube’s docker image registry not in our normal local docker image registry [5].
To do that we have to set docker-env environment variables with *eval* command ```eval $(minikube docker-env)```
```
eval $(minikube docker-env)
docker image build --tag=springdemo --rm=true .
minikube ssh
docker images. #springdemo image should present
```

Then deploy the springboot app using  ```kubectl apply -f springboot.yaml```. It deploys the springboot app on a container which runs on *port 8080* (**targetPort**) and a service(**springdemo**) will create which connect to the springboot app. Service *springdemoService* is created as **NodePort type** [3] so that a user can connect to it from externally using the *nodeport*.

[mini-kube]: https://kubernetes.io/docs/setup/learning-environment/minikube/

**springboot helloworld app kubernet file (springboot.yaml)**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: springdemo
  labels:
    app: springdemo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: springdemo
  template:
    metadata:
      labels:
        app: springdemo
    spec:
      containers:
        - name: springdemo
          image: springdemo:latest
          imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Service
metadata:
  name: springdemo
  labels:
    app: springdemo
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  type: NodePort

  selector:
    app: springdemo                   
```
To connect to the service from outside cluster, we have to use **NodeIp:NodePort**.

Get the *NodeIp* by
```sh 
minikube ip
```
Get the *NodePort* by 
``` sh
kubectl describe services serviceName
```
I decided to store the elastic search indices into a local directory outside the kuberenetes cluster so that it will not destroy even after the elasticserach server  *pod* dies. Therfore elasticsearch server deployed as 
*StatefulSet* [2]. 

To mount a local directory into a pod in minikube, you have to mount that local directory into minikube and then use minikube mounted path in [hostpath][host-path].

[host-path]: https://minikube.sigs.k8s.io/docs/handbook/mount/
```
mkdir -p ~/esdemoIndex
minikube mount ~/esdemoIndex:/indexdata
```
**You have to run minikube mount in a separate terminal because it starts a process and stays there until you unmount**.

Another issue which I faced during mounting was elastic search server pod was throwing *java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/nodes* .To solve that, we have to set full permission in /usr/share/elasticsearch/data/nodes. Please see  **initContainers** section in the **elasticStateful.yaml**.

Then deploy the elasticsearch server  using  ```kubectl apply -f elasticStateful.yaml```.

**elasticsearch  kubernet file (elasticStateful.yaml)**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch
spec:
  serviceName: "elasticsearch"
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      initContainers:
      - name: set-permissions
        image: registry.hub.docker.com/library/busybox:latest
        command: ['sh', '-c', 'mkdir -p /usr/share/elasticsearch/data && chown 1000:1000 /usr/share/elasticsearch/data' ]
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:6.6.1
        env:
        - name: discovery.type
          value: single-node
        ports:
        - containerPort: 9200
          name: client
        - containerPort: 9300
          name: nodes
        volumeMounts:
        - name: data
          mountPath: /usr/share/elasticsearch/data
      volumes:
      - name: data
        hostPath:
          path: /indexdata
---
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  labels:
    service: elasticsearch
spec:
  ports:
  - port: 9200
    name: client
  - port: 9300
    name: nodes
  type: NodePort  
  selector:
    app: elasticsearch
```
We need to connect the springbootapp to the **mysqldb** which is already running on the machine(ip **192.168.1.40**) on port no **3308**, outside of the cluster.  To set up a mysql db you can use following **docker-compose.yml**. Please run ``` docker-compose -f docker-compose.yml up -d``` to startup a mysqldb.

**docker-compose.yml**
```yaml
version: '3'

services:

  mysql-development:
    image: mysql:8.0.17
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: dummydb
    ports:
      - "3308:3306"
```
Please create the *person_details* (this table's data will be used by elastic search server to create the indices) table on the mysql db.
```sql
CREATE TABLE person_details (
    persId INT  PRIMARY KEY,
    name VARCHAR(50),
    profession VARCHAR(50))
ENGINE=Innodb;
```
Now, we will create a service(**mysql-service**)  to conneect to the above mysql db. Normally, when we create a service to connect to a pod on the same namespace, service will internally find the **ip:port** of the the pod which matches to the selector criteria. But here, we wants to connect to mysql db which is running outiside. Therefore we have to explicilty create an **Endpoints** with the ip and port on which mysql is running[4].
Please deploy the mysql services  using  ```kubectl apply -f mysqlService.yaml```.

**Mysql service's(connect to local db) kubernet file (mysqlService.yaml)**
```yaml
apiVersion: v1
kind: Service
metadata:
    name: mysql-service
spec:
    ports:
        - protocol: TCP
          port: 1443
          targetPort: 3308
    type: NodePort      
---
apiVersion: v1
kind: Endpoints
metadata:
    name: mysql-service
subsets:
    - addresses:
        - ip: 192.168.1.40
      ports:
        - port: 3308  
```  
Please check the status of *pods, deployments, statefulsets, services* to ensure that everythings works fine.
```sh
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get StatefulSets
kubectl get  endpoints
```
If status of every kubernet entity is fine then we can connect to the endpoints of the springboot app. 

1) To **add data** to the msyqldb (if the data does not exist in the table already)
```sh
curl --request POST "http://192.168.64.3:30302/pers/addToDb"

output - true
```
2) To **create the elastic search index** in the  ~/esdemoIndex folder 
```sh
 curl --request POST "http://192.168.64.3:30302/pers/createIndex?indexName=index2&indexType=demo"
 
output -  5
```

3) To **search** for person name 'Tom' who is 'Doctor' by profession
```sh
curl --request GET "http://192.168.64.3:30302/pers/search?name=Tom&profession=Doctor"

output -  id:1,Name:Tom, Profession:Doctor%
```
**References**
1. [https://github.com/deleSerna/springbootelasticsearchdemo][ref-1]
2. [https://www.magalix.com/blog/kubernetes-statefulsets-101-state-of-the-pods#:~:text=A%20StatefulSet%20is%20another%20Kubernetes,more%20suited%20for%20stateful%20apps.&text=By%20nature%2C%20a%20StatefulSet%20needs,state%20and%20data%20across%20restarts][ref-2]
3. [https://www.bmc.com/blogs/kubernetes-port-targetport-nodeport/][ref-3]
4. [https://theithollow.com/2019/02/04/kubernetes-endpoints/][ref-4]
5. [https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968][ref-5]

[ref-1]: https://github.com/deleSerna/springbootelasticsearchdemo
[ref-2]: https://www.magalix.com/blog/kubernetes-statefulsets-101-state-of-the-pods#:~:text=A%20StatefulSet%20is%20another%20Kubernetes,more%20suited%20for%20stateful%20apps.&text=By%20nature%2C%20a%20StatefulSet%20needs,state%20and%20data%20across%20restarts
[ref-3]: https://www.bmc.com/blogs/kubernetes-port-targetport-nodeport/
[ref-4]: https://theithollow.com/2019/02/04/kubernetes-endpoints/
[ref-5]: https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968
