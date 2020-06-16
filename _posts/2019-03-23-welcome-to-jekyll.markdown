---
layout: post
title:  "Spring boot with elastic search on kubernetes"
date:   2020-06-13 2:32:36 +0530
categories: Java Springboot Kubernetes ElasticSearch
---
I have recently developed a demo springboot ( version 2.2.5) application which use java 11 ( using modular system introduced in java 9), an elasticsearch( version and 6.6.1) and mysql 
(Version 8.0.17). Then I went ahead and tried to deploy it in a local kubernetes cluster. Although there was numerous tutorials about deployment in Kubernetes, I could not find a single sample application which cover all the aspects in a single place. In this article, I will briefly go through the some of the issues which I faced during the deployment and the solution for that. I am not going through all the details  of  Kubernetes deployment file I used because you can find the details in so many other places.

I planned to deploy the application in the following way. Mysql database will be running in the system outside of the Kubernetes cluster. Springboot app which will run in the kubernetes cluster will write some data to the database. Then elasticsearch index will create using the db data. Then finally we will search in the created  elastic search index. We will have 3 endpoints in the springboot to these action 1. CreateDbdata end point will write  to the db 2. CreateEsindex will load data from the db and will create the elastic search index 3.Search will search in the elastic search index to fetch data which matches to our criteria. Source code of the springboot app can see here (TODO, need to provide a GitHub repo link).

I used [minikube][mini-kube] (version 1.11.0) to run Kubernetes locally. 
To start the minikube please  use ```minikube start```
Since we are using   a custom springboot application, first we have to create a docker image of the application and put it in minikube’s docker image registry.  We have to pay attention that we have to put the image in minikube’s docker image registry not in our normal local docker image registry [1].
We need to set the environment variable with eval command ```eval $(minikube docker-env)```
```
minikube ssh
eval $(minikube docker-env)
docker image build --tag=dymmyapp --rm=true .
docker images
```

Then deploy the springboot app using  ```kubectl apply -f springboot.yaml```.  It deploys the springboot app on a container which runs on port 8080 (**targetPort**). Then a service will create which expose the springboot app on port 8080. 
To connect to the springboot externally, we made it as **NodePort type**.
[—- To be updated]

[mini-kube]: https://kubernetes.io/docs/setup/learning-environment/minikube/

**springboot helloworld app kubernet file**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
  template:
    metadata:
      labels:
        app: helloworld
    spec:
      containers:
        - name: helloworld
          image: helloworld:latest
          imagePullPolicy: IfNotPresent
---
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
spec:
  ports:
    - port: 8080
      protocol: TCP
      targetPort: 8080
  type: NodePort

  selector:
    app: helloworld          

```
To connect to the service from outside cluster, we have to use NodeIp:NodePort
Get the NodeIp by
```sh 
minikube ip
```
Get the NodePort from 
``` sh
kubectl describe services serviceName
```
Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/

**elasticsearch  kubernet file**
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
Springboot app is also connecting to the **mysqldb** which is running on the machine(ip *192.168.1.40*) on port no *3308*, outside of the cluster.

**Mysql service's(connect to local db) kubernet file**
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
References
1. https://medium.com/bb-tutorials-and-thoughts/how-to-use-own-local-doker-images-with-minikube-2c1ed0b0968
