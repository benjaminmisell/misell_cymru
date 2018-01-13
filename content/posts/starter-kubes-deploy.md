---
date: "2018-01-13T18:37:38Z"
title: "A useful starting deployment for Kubernetes"
authors: []
tags:
  - kubernetes
draft: false
---

I often find myself in need of a starter file for deploying a web-based docker container to Kubernets (starting a new project etc). You need a deployment then a service and an ingress definition, all a lot to type out. I felt the need to share one with the world so here's the one I typically start with:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: PRJOJECT_NAME
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: server
  namespace: PRJOJECT_NAME
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: server
    spec:
      containers:
      - name: sever
        image: DOCKER_IMAGE
        imagePullPolicy: Always
        ports:
          - containerPort: 80
            protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  namespace: PROJECT_NAME
  name: server
  labels:
    app: server
spec:
  selector:
    app: server
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: server
  namespace: PROJECT_NAME
  labels:
    app: server
  annotations:
    kubernetes.io/tls-acme: "true" # This won't work without kube-lego instaled
spec:
  tls: # Not needed if you dont wan't https, but I strongly advise you use https
  - hosts:
    - "DOMAIN_NAME"
    secretName: server-tls # This must be created manually if you are not using kube-lego
  rules:
  - host: "DOMAIN_NAME"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: server
          servicePort: 80
```

All that's needed to use this is to replace all the things in CAPITALS with the necessary values. Note: this deployment assumes that you have kube-lego setup to get SSL certificates from Let's Encrypt automagically. If you don't have it don't worry; you can remove the annotation on the Ingress and set up the secret manually or you can run without SSL (I don't recommend it). If you want kube-lego the good news is that it's pretty easy to install. There are many tutorials out there on how to do so, I might even write one in future.

I hope you find this useful in your adventures in Kubernetes! I'm releasing this under the MIT License so you are free to use this anywhere!