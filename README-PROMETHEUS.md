# 📊 Kubernetes Monitoring with Prometheus & Grafana

This guide explains how to install **Prometheus** and **Grafana** using **Helm charts** and set up a Kubernetes monitoring dashboard.

## 🚀 Prerequisites
- **Helm** installed:  
  ```sh
  curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
  ```
- **Kubernetes Cluster** (Minikube or any K8s environment)
- **kubectl** installed

---

## 📥 1️⃣ Add the Prometheus Helm Repository
First, add the **Prometheus community Helm repository** and update the repo:
```sh
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

---

## 📦 2️⃣ Install Prometheus & Grafana
Now, install the **Kube-Prometheus-Stack**, which includes **Prometheus, Grafana, and Alertmanager**:
```sh
helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring --create-namespace
```

This installs:
✅ **Prometheus** – Monitoring system  
✅ **Grafana** – Dashboard visualization  
✅ **Alertmanager** – Manages alerts  

---

## 🔍 3️⃣ Verify Installation
Check if all the necessary pods are running:
```sh
kubectl get pods -n monitoring
```
You should see:
```
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2     Running   0          5m
prometheus-grafana-79dfbcfbb6-fm8z4                      3/3     Running   0          5m
prometheus-kube-prometheus-operator-76b4dbd65b-4mvpr     1/1     Running   0          5m
prometheus-kube-state-metrics-7c6b78776b-7vvnm           1/1     Running   0          5m
prometheus-prometheus-kube-prometheus-prometheus-0       2/2     Running   0          5m
```

---

## 🌍 4️⃣ Expose Prometheus & Grafana Services
Prometheus and Grafana run inside Kubernetes, so we need to expose them.

### **Expose Prometheus**
Find the Prometheus service:
```sh
kubectl get svc -n monitoring | grep prometheus
```
Expose Prometheus on port **9090**:
```sh
kubectl port-forward svc/prometheus-kube-prometheus-prometheus -n monitoring 9090:9090
```
You can now access Prometheus at:
```
http://localhost:9090
```

### **Expose Grafana**
Find the Grafana service:
```sh
kubectl get svc -n monitoring | grep grafana
```
Expose Grafana on port **3000**:
```sh
kubectl port-forward svc/prometheus-grafana -n monitoring 3000:80
```
Now open Grafana at:
```
http://localhost:3000
```

---

## 🔑 5️⃣ Log into Grafana
### **Default Grafana Credentials**
- **Username**: `admin`
- **Password**: `prom-operator`

If the password is unknown, retrieve it using:
```sh
kubectl get secret -n monitoring prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

---

## 🎛 6️⃣ Add Prometheus as a Data Source in Grafana
1. Open **Grafana** at **http://localhost:3000**
2. Navigate to **Configuration → Data Sources**
3. Click **"Add data source"**
4. Select **Prometheus**
5. Set the **URL** to:
   ```
   http://prometheus-kube-prometheus-prometheus.monitoring.svc.cluster.local:9090
   ```
6. Click **"Save & Test"**  
   ✅ If successful, Grafana confirms the connection.

---

## 📊 7️⃣ Import a Kubernetes Monitoring Dashboard
1. Go to **Grafana → Dashboards**
2. Click **"Import"**
3. Enter **Dashboard ID**: `3119` (or any Kubernetes dashboard)
4. Click **Load**
5. Select the **Prometheus data source** and click **Import**

Now, you will see **real-time Kubernetes metrics** such as:
- Node & Pod CPU usage
- Memory consumption
- Network activity
- Service performance

---

## ✅ 8️⃣ Verify Prometheus Targets
Go to **http://localhost:9090/targets**  
Ensure all targets are **UP**. If some are **DOWN**, check service monitors.

---

## 🎉 Prometheus & Grafana are now set up!
You now have a **fully functional Kubernetes monitoring dashboard** using **Prometheus and Grafana**. 🚀  
Monitor your **Pods, Nodes, Deployments, and Services** in real-time!
