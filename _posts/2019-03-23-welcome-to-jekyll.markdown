---
layout: post
title:  "Spring boot with elastic search on kubernetes"
date:   2020-06-13 2:32:36 +0530
categories: Java Springboot Kubernetes ElasticSearch
---
I have recently developed a demo springboot ( version 2.2.5) application which use java 11 ( using modular system introduced in java 9), an elasticsearch( version and 6.6.1) and mysql 
(Version 8.0.17). Then I went ahead and tried to deploy it in a local kubernetes cluster. Although there was numerous tutorials about deployment in Kubernetes, I could not find a single sample application which cover all the aspects in a single place. In this article, I will briefly go through the some of the issues which I faced during the deployment and the solution for that. I am not going through all the details  of  Kubernetes deployment file I used because you can find the details in so many other places. Source code can be found [here][git-hub].

[git-hub]: https://github.com/deleSerna/springbootelasticsearchdemo

I planned to deploy the application in the following way. Mysql database will be running in the system outside of the Kubernetes cluster. Springboot app which will run in the kubernetes cluster will write some data to the database. Then elasticsearch index will create using the db data. Then finally we will search in the created  elastic search index. We will have 3 endpoints in the springboot to these action 1. CreateDbdata end point will write  to the db 2. CreateEsindex will load data from the db and will create the elastic search index 3.Search will search in the elastic search index to fetch data which matches to our criteria. Source code of the springboot app can see here (TODO, need to provide a GitHub repo link).

I used [minikube][mini-kube] (version 1.11.0) to run Kubernetes locally. 
To start the minikube please  use ```minikube start```
Since we are using   a custom springboot application, first we have to create a docker image of the application and put it in minikube’s docker image registry.  We have to pay attention that we have to put the image in minikube’s docker image registry not in our normal local docker image registry [1].
We need to set the environment variable with eval command ```eval $(minikube docker-env)```
```
eval $(minikube docker-env)
docker image build --tag=springdemo --rm=true .
minikube ssh
docker images. #springdemo image should present
```

Then deploy the springboot app using  ```kubectl apply -f springboot.yaml```.  It deploys the springboot app on a container which runs on port 8080 (**targetPort**). Then a service will create which expose the springboot app on port 8080. 
To connect to the springboot externally, we made it as **NodePort type**[3].

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
To connect to the service from outside cluster, we have to use NodeIp:NodePort.

Get the NodeIp by
```sh 
minikube ip
```
Get the NodePort by 
``` sh
kubectl describe services serviceName
```
I decided to store the elastic search indices into a local directory outside the machine. Therfore deployed elasticsearch as 
*StatefulSet*[2]. To mount a local directory into a pod in minikube (version - v1.9.2), you have to mount that local directory into minikube then use minikube mounted path in [hostpath][host-path].
[host-path]: https://minikube.sigs.k8s.io/docs/handbook/mount/
```
mkdir -p ~/esdemoIndex
minikube mount ~/esdemoIndex:/indexdata
```
*You have to run minikube mount in a separate terminal because it starts a process and stays there until you unmount*.

Another issue which I faced during mounting was elastic search server pod was throwing *java.nio.file.AccessDeniedException: /usr/share/elasticsearch/data/nodes* .To solve that, we have to use **initContainers** to set full permission in /usr/share/elasticsearch/data/nodes.

Then deploy the elasticsearch server  using  ```kubectl apply -f elasticStateful.yaml```
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
We want to connect the Springboot to the **mysqldb** which is already running on the machine(ip *192.168.1.40*) on port no *3308*, outside of the cluster.  To run. amysql db you can use following docker-compose.yml. Please run ``` docker-compose -f docker-compose.yml up -d```
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
Please create the *person_details* table on the mysql db
```sql
CREATE TABLE person_details (
    persId INT  PRIMARY KEY,
    name VARCHAR(50),
    profession VARCHAR(50))
ENGINE=Innodb;
```
Now, we will create a serivice in the kubernetes to conneect to the above mysql db. Normally when we create a service to connect to a pod on the same name space, service internally find the ip:port of the  the pod which matches to the selector. But here we wants to connect to mysql db which is running outiside. Therefore we have to explicilty create an **Endpoints** with the ip and port on which mysql is running[4].
Then deploy the mysql services  using  ```kubectl apply -f mysqlService.yaml```.
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
Please check the status of pods, deployments, statefulsets, services to ensure that everythings works fine
```sh
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get StatefulSets
kubectl get  endpoints
```
Add data to the msyqldb ( if data does not exist in the table already)
```sh
curl --request POST "http://192.168.64.3:30302/pers/addToDb"

output - true
```
Create the elastic search index in the  ~/esdemoIndex folder 
```sh
 curl --request POST "http://192.168.64.3:30302/pers/createIndex?indexName=index2&indexType=demo"
 
output -  5
```
Search for person name 'Tom' who is 'Doctor' by profession
```sh
curl --request GET "http://192.168.64.3:30302/pers/search?name=Tom&profession=Doctor"

output -  id:1,Name:Tom, Profession:Doctor%
```
References
1. https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968
2. https://www.magalix.com/blog/kubernetes-statefulsets-101-state-of-the-pods#:~:text=A%20StatefulSet%20is%20another%20Kubernetes,more%20suited%20for%20stateful%20apps.&text=By%20nature%2C%20a%20StatefulSet%20needs,state%20and%20data%20across%20restarts
3. https://www.bmc.com/blogs/kubernetes-port-targetport-nodeport/
4. https://theithollow.com/2019/02/04/kubernetes-endpoints/
