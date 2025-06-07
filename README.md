# prometheus + alertmanager

![prometheus architecture 3](https://github.com/user-attachments/assets/eb286651-0822-4071-9de8-bff5fe46dbbe)

## General Setup Overview
The goal is to monitor the health and performance of your Kubernetes cluster and the applications running within it. Prometheus is used for collecting and storing metrics, while Alertmanager handles alerts based on those metrics. They are typically deployed inside the Kubernetes cluster they are monitoring.

### Core Components and Their Focus:
### 1. Prometheus Server:
What it is: The central component that scrapes (pulls) metrics from various sources, stores them in a time-series database, and allows querying using the PromQL language. It also evaluates alerting rules.
How it's deployed: Usually as a Kubernetes StatefulSet (or sometimes Deployment) within the cluster. A StatefulSet is often preferred to provide stable network identifiers and persistent storage (using PersistentVolumeClaims) for the metrics data.
Focus: Pulling metrics from configured targets, storing metrics, evaluating rules, sending alerts to Alertmanager.
### 2. Targets (What Prometheus Scrapes): 
Prometheus needs data sources. In a typical Kubernetes setup, these include:
Node Exporter: An agent deployed as a DaemonSet (one pod per Kubernetes node). It collects node-level metrics like CPU usage, memory, disk I/O, network statistics for the underlying host machine. Prometheus discovers and scrapes each Node Exporter pod.
Kube-State-Metrics (KSM): A service deployed usually as a Deployment. It listens to the Kubernetes API Server and converts information about the state of Kubernetes objects (Deployments, Pods, Services, Nodes, DaemonSets, StatefulSets, etc. - their count, status, resource requests/limits) into metrics that Prometheus can scrape. It tells you about the orchestration layer.
Kubernetes API Server: The API server itself often exposes internal metrics about its performance.
Kubelet: Each node's Kubelet exposes metrics about container lifecycle, volumes, etc.
Application Pods: Your own application pods can be instrumented to expose custom metrics (e.g., request latency, error counts, business KPIs) on a specific HTTP endpoint (like /metrics). Prometheus needs to find and scrape these.
(Optional) cAdvisor: Often integrated within the Kubelet, provides container-level resource usage metrics.
### 3. Service Discovery:
Problem: How does Prometheus automatically find all the Node Exporters, KSM, Kubelets, and potentially hundreds of application pods to scrape, especially when pods are constantly being created, deleted, and rescheduled?
Solution: Prometheus uses Kubernetes Service Discovery (kubernetes_sd_config in its configuration). Prometheus queries the Kubernetes API Server to get lists of pods, services, endpoints, nodes, or ingresses based on specified labels, annotations, or namespaces. It then automatically configures scrape targets based on this discovered information. This is crucial for dynamic environments.
### 4. Alertmanager:
What it is: Handles alerts fired by the Prometheus server. It receives alerts, then de-duplicates, groups, and routes them to the correct notification channels (like Slack, PagerDuty, email, etc.). It also manages silencing and inhibitions.
How it's deployed: Usually as a StatefulSet (to maintain state about silences and alert grouping) within the cluster. It needs its own configuration file. Prometheus is configured with the address of the Alertmanager service.
Focus: Receiving alerts, processing alerts (grouping, silencing), and sending notifications.

## Where Do ConfigMap and Alertmanager Fit In?
### ConfigMap:
Role: Kubernetes ConfigMaps are used to store configuration files separately from the application container images. This allows you to update configurations without rebuilding images.
Usage for Prometheus: The main Prometheus configuration file (prometheus.yml), which defines global settings, scrape configurations (including service discovery rules), and alerting rule file paths, is stored inside a ConfigMap. This ConfigMap is then mounted as a volume into the Prometheus Server pod, making the prometheus.yml file accessible to the Prometheus process.
Usage for Alerting Rules: Prometheus alerting rules (usually defined in separate .rules.yml files) are also typically stored in one or more ConfigMaps and mounted into the Prometheus Server pod.
Usage for Alertmanager: Similarly, the Alertmanager configuration file (alertmanager.yml), which defines routing trees, receivers (Slack, PagerDuty, etc.), and templates, is stored in a ConfigMap and mounted into the Alertmanager pod(s).
In short: ConfigMaps hold the configuration data that Prometheus and Alertmanager use to function.
### Alertmanager:
Role: As described above, it's the dedicated alert processing and notification component.
Fit: It sits between Prometheus and your notification systems. Prometheus detects a problem based on metrics and rules, fires an alert, and sends that alert to Alertmanager. Alertmanager then applies its logic (grouping, silencing) and sends the final notification out to Slack, PagerDuty, etc. It decouples alert detection (Prometheus) from alert notification (Alertmanager).

## In general, components involved in this setup and their role:

Kubernetes Cluster: The entire managed environment containing all components.
Control Plane: The brain of the cluster, managing state and scheduling.
API Server: The entry point for all cluster management commands and interactions.
etcd: A distributed key-value store holding the cluster's configuration and state.
Scheduler: Decides which Worker Node a new Pod should run on.
Controller Manager: Runs controllers to ensure the cluster matches the desired state.
Worker Nodes: Machines (VMs/physical) that run the application containers (Pods).
Kubelet: An agent on each Worker Node ensuring containers in Pods are running and healthy.
Kube-proxy: Manages network rules on Worker Nodes for Service routing.
Container Runtime: Software that actually runs the containers (e.g., CRI-O, containerd).
Node Exporter Pod: Collects hardware and OS metrics from the Worker Node it runs on.
App Pod (A, B, C): Runs the container(s) for a specific application instance.
DB Pod (A, B, C): Runs the container(s) for a specific database instance.
Prometheus Server Pod: Scrapes metrics, stores them, evaluates alert rules based on metrics.
Kube-State-Metrics Pod: Provides metrics about the state of Kubernetes objects (Deployments, Pods, etc.).
Alertmanager Pod: Receives alerts from Prometheus, de-duplicates/groups them, and sends notifications. (Highlighted Role)
Service (App/DB/Prom/AlertMgr): Provides a stable internal IP address and DNS name to access a set of Pods.
ConfigMap: Stores non-sensitive configuration data (like .yml or .env files) externally from Pods. (Highlighted Role)
External Notifications: External systems like Slack, PagerDuty, Email where Alertmanager sends alerts.
User/Admin: Interacts with the cluster (management or application access).
