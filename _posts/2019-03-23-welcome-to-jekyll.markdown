---
layout: post
title:  "Spring boot with elastic search on kubernetes"
date:   2020-06-13 2:32:36 +0530
categories: Java Springboot Kubernetes ElasticSearch
---
Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse

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
