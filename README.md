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
- [Contributing](#contributing)
- [License](#license)

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
    helm install elasticsearch elastic/elasticsearch --set replicas=1
    ```

4. **Deploy Logstash**

    Create a `logstash-configmap.yaml`:

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: logstash-config
    data:
      logstash.conf: |
        input {
          beats {
            port => 5044
          }
        }
        output {
          elasticsearch {
            hosts => ["http://elasticsearch-master:9200"]
            index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
          }
        }
    ```

    Apply the ConfigMap and deploy Logstash:

    ```bash
    kubectl apply -f logstash-configmap.yaml
    helm install logstash elastic/logstash --set logstashConfig.configMap=logstash-config
    ```

5. **Deploy Kibana**

    ```bash
    helm install kibana elastic/kibana --set service.type=LoadBalancer
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

1. **Create Filebeat ConfigMap**

    Create a `filebeat-configmap.yaml`:

    ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: filebeat-config
      namespace: default
    data:
      filebeat.yml: |
        filebeat.inputs:
        - type: container
          paths:
            - /var/log/containers/*.log

        output.logstash:
          hosts: ["logstash:5044"]
    ```

    Apply the ConfigMap:

    ```bash
    kubectl apply -f filebeat-configmap.yaml
    ```

2. **Deploy Filebeat DaemonSet**

    Create a `filebeat-daemonset.yaml`:

    ```yaml
    apiVersion: apps/v1
    kind: DaemonSet
    metadata:
      name: filebeat
      namespace: default
      labels:
        k8s-app: filebeat
    spec:
      selector:
        matchLabels:
          k8s-app: filebeat
      template:
        metadata:
          labels:
            k8s-app: filebeat
        spec:
          containers:
          - name: filebeat
            image: docker.elastic.co/beats/filebeat:7.12.1
            args: [
              "-c", "/etc/filebeat.yml",
              "-e",
            ]
            env:
            - name: ELASTICSEARCH_HOST
              value: "elasticsearch-master"
            - name: ELASTICSEARCH_PORT
              value: "9200"
            volumeMounts:
            - name: config
              mountPath: /etc/filebeat.yml
              subPath: filebeat.yml
            - name: varlog
              mountPath: /var/log
          volumes:
          - name: config
            configMap:
              name: filebeat-config
          - name: varlog
            hostPath:
              path: /var/log
    ```

    Deploy the Filebeat DaemonSet:

    ```bash
    kubectl apply -f filebeat-daemonset.yaml
    ```

## Accessing Kibana

To access Kibana, get the LoadBalancer IP:

```bash
kubectl get svc kibana-kibana
```

Open Kibana in your browser and configure the index pattern to visualize the logs.

## Contributing

Contributions are welcome! Please fork this repository and submit a pull request.

## License

This project is licensed under the MIT License.

---

This README provides a comprehensive guide to setting up and testing the ELK stack on Kubernetes with a sample application to generate logs. It covers all necessary steps, from prerequisites to accessing Kibana for log visualization.
