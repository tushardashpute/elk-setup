# ELK Stack on Kubernetes

This repository contains configurations and instructions to set up an ELK (Elasticsearch, Logstash, Kibana) stack on Kubernetes along with a sample Nginx application to generate logs. This setup is designed to help you aggregate and search through logs effectively.

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
  - [Deploy ELK Stack](#deploy-elk-stack)
  - [Deploy Sample Application](#deploy-sample-application)
  - [Configure and Deploy Filebeat](#configure-and-deploy-filebeat)
- [Accessing Kibana](#accessing-kibana)

## Introduction

The ELK stack consists of Elasticsearch, Kibana, Logstash, and Filebeat. It is used to aggregate logs and explore them, which is crucial in a microservices architecture for debugging purposes.

- **Elasticsearch**: Stores all the logs.
- **Kibana**: Visualization platform to query Elasticsearch.
- **Logstash**: Data ingestion tool that processes and sends logs to Elasticsearch.
- **Filebeat**: Log exporter that forwards logs to Logstash.

## Prerequisites

- Kubernetes Cluster
- Helm v3 installed

## Setup Instructions

### Deploy ELK Stack

1. **Install Helm (if not already installed)**

    ```bash
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    ```

2. **Add Elastic Helm Repository**

    ```bash
    helm repo add elastic https://helm.elastic.co
    helm repo update
    ```

3. **Deploy Elasticsearch**

    ```bash
    helm install elasticsearch elastic/elasticsearch -f elsticsearch.yaml  --namespace logging --create-namespace
    ```

4. **Deploy Logstash**

    ```bash
    helm install logstash elastic/logstash -f logstash.yaml --namespace logging
    ```

5. **Deploy Kibana**

    ```bash
    helm install kibana elastic/kibana --set service.type=LoadBalancer -f kibana.yaml --namespace logging
    ```

### Deploy Sample Application

1. **Create the Nginx Deployment**

    Create a `nginx-deployment.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
          - name: nginx
            image: nginx
            ports:
            - containerPort: 80
            volumeMounts:
            - name: varlog
              mountPath: /var/log/nginx
          volumes:
          - name: varlog
            emptyDir: {}
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: nginx-service
    spec:
      selector:
        app: nginx
      ports:
      - protocol: TCP
        port: 80
        targetPort: 80
    ```

    Deploy the Nginx application:

    ```bash
    kubectl apply -f nginx-deployment.yaml
    ```

### Configure and Deploy Filebeat

1. **Deploy Filebeat DaemonSet**

    ```bash
    helm install elk-filebeat elastic/filebeat -f filebeat.yaml --namespace logging
    ```

## Accessing Kibana

To access Kibana, get the LoadBalancer IP:

```bash
kubectl get svc kibana-kibana
```

Open Kibana in your browser and configure the index pattern to visualize the logs.

