# Cargill Centralized Logging & Observability Platform (AWS EKS)

This repository contains production-grade Kubernetes deployment manifests and pipeline configurations for **OpenSearch Data Prepper**, integrated with **Apache Kafka (Amazon MSK)** and **Amazon OpenSearch Service**, developed as part of Cargill's Enterprise Observability Platform.

---

## 🏛️ Architecture Overview

```text
[ EKS Microservices / Pods ]
           │
           │ (Fluent Bit DaemonSet / Direct App Streams)
           ▼
[ Apache Kafka (Amazon MSK) ]
   └── Topic: cargill-application-logs (High-throughput buffer & persistence)
           │
           │ (Kafka Consumer Group: data-prepper-consumer-group)
           ▼
[ OpenSearch Data Prepper (3 Replicas on EKS) ]
   ├── 1. Source: Apache Kafka Consumer
   ├── 2. Buffer: Bounded Blocking In-Memory Buffer
   ├── 3. Processors:
   │       ├── JSON Parsing & Normalization
   │       ├── ISO-8601 Timestamp Formatting (@timestamp)
   │       ├── Metadata Enrichment (Env, Cluster, Platform)
   │       └── Noise Reduction (Drop health-check ping events)
   └── 4. Sink: Amazon OpenSearch Service (SigV4 Authentication via IRSA)
           ├── Bulk Ingestion (cargill-app-logs-%{yyyy.MM.dd})
           └── Dead Letter Queue (DLQ -> AWS S3 on indexing errors)
           │
           ▼
[ Amazon OpenSearch Dashboard / Kibana ]
   └── Real-time Dashboards, Log Analytics, Error Rate Monitoring & SRE Alerts
```

---

## 📁 Repository Structure

```text
k8s/
├── 01-namespace-sa.yaml    # Namespace & AWS IRSA ServiceAccount
├── 02-configmaps.yaml      # Data Prepper pipelines.yaml (Kafka -> OpenSearch) & server configs
├── 03-deployment.yaml      # High Availability Deployment (Multi-AZ anti-affinity, JVM flags)
├── 04-service-hpa.yaml     # ClusterIP Service & Horizontal Pod Autoscaler (HPA 3-10 replicas)
└── kustomization.yaml      # Kustomize root manifest
```

---

## ⚙️ Core Components Explained

### 1. Why Apache Kafka before Data Prepper?
* **Burst Absorber:** Prevents OpenSearch cluster from throttling during sudden traffic spikes or batch jobs.
* **Decoupling & Zero Log Loss:** If Data Prepper or OpenSearch undergoes maintenance, Kafka retains logs for up to 7 days.
* **Multi-Consumer Support:** Security team can consume audit logs from the same Kafka topic independently.

### 2. Why OpenSearch Data Prepper?
* **Native OpenSearch Support:** Seamless integration with AWS OpenSearch Service using AWS SigV4 / IAM.
* **High-Throughput Transformation:** Extracts JSON, enriches environment metadata, and drops `/health` status 200 logs to reduce storage costs by ~25%.
* **Dead Letter Queue (DLQ) to S3:** Any malformed or unindexable logs are automatically written to Amazon S3 rather than being dropped.

### 3. Security (AWS IRSA):
* Uses **IAM Roles for Service Accounts (IRSA)** via `eks.amazonaws.com/role-arn`.
* No hardcoded AWS access keys or secrets in Kubernetes manifests.

---

## 🚀 Deployment Instructions

### Deploy all components via Kustomize:
```bash
kubectl apply -k k8s/
```

### Verify Deployment:
```bash
# Check Pods
kubectl get pods -n observability -l app=data-prepper

# Check Logs
kubectl logs -n observability -l app=data-prepper -f

# Check Pipeline Status
kubectl exec -it $(kubectl get pods -n observability -l app=data-prepper -o jsonpath='{.items[0].metadata.name}') -n observability -- curl -s http://localhost:2021/
```

---

## 🎯 Interview Q&A (Cargill Experience Cheatsheet)

**Q: OpenSearch Data Prepper ni ela deploy chesaru?**
> *"Memu Amazon EKS cluster lo `observability` namespace lo Data Prepper ni Deployment object ga deploy chesam. Multi-AZ high availability kosam pod anti-affinity vadam, auto-scaling kosam HPA configure chesam. Kafka broker details and OpenSearch sink details `pipelines.yaml` ConfigMap lo maintain chesam."*

**Q: AWS OpenSearch authentication ela handle chesaru?**
> *"Credentials hardcode cheyakunda, **AWS IRSA (IAM Roles for Service Accounts)** vadam. Data Prepper ServiceAccount ki IAM Role attach chesam with permissions to ingest logs into Amazon OpenSearch and write failed logs to S3 DLQ."*
