---
title: Understanding YAML Structure
date:  2026-09-15 22:00:00 +0800
categories: [Knowledge, YAML]
tags: [yaml, kubernetes, docker, documentation, knowledge]
description: A guide on YAML structure for Docker and Kubernetes.
---
# Understanding YAML Files in Containerized Infrastructure

Moving from standard Python scripting to containerized cloud architectures requires shifting from running imperative code to defining declarative environments. This document explains what **YAML files** are, why they are essential for **Containers**, and how they orchestrate applications inside Red Hat **OpenShift**.

# 1. What is a YAML File?

A **YAML** (Yet Another Markup Language) file is a human-readable text file used primarily for configuration. 

In Python scripting, you write *imperative* code—you tell the computer a sequential list of steps to execute. In cloud infrastructure, you write *declarative* YAML files—you describe the **final desired state** of your system (e.g., "I want three copies of this Python application running on port 8080 with 2Gi of memory"). The underlying platform reads this blueprint and automatically configures the infrastructure to match it.

# 2. Why YAML Files are Used in Containers

Containers (like those built via **Docker**) package an application and its exact dependencies into a single isolated unit. While a `Dockerfile` defines how to *build* that image, **YAML files** define how that container *behaves and interacts* with the outside world once it runs.

Using YAML configuration provides three primary engineering benefits:
* **Infrastructure as Code (IaC):** Your server setups, network routes, and environment variables are documented as text files. They can be tracked, versioned, and rolled back using Git just like application code.
* **Separation of Concerns:** The core application logic stays inside your Python files, while operational configurations (like database URLs or container scaling limits) live externally in YAML files.
* **Consistency Across Environments:** The exact same container image can be deployed to local testing, staging, or production environments simply by changing a few values in the accompanying YAML file.

# 3. How YAML Connects to OpenShift

Red Hat **OpenShift** is an enterprise container orchestration platform powered by **Kubernetes**. OpenShift acts as the factory controller, and your **YAML files** are the manufacturing blueprints you hand to it.

Every infrastructure resource in OpenShift requires a YAML file. OpenShift determines how to process the file by looking at the **`kind:`** field defined at the top of the document.

### OpenShift Blueprint Categories

Depending on what your containerized application needs to function, you will author different types of YAML resources:

* **Application Lifecycle Managers:**
  * **`kind: Deployment`**: Manages the running state of your containers. It defines how many identical copies (**replicas**) must run at all times, dictates how to roll out updates without downtime, and automatically restarts containers if they crash.
  * **`kind: CronJob`**: Serves as a scheduled trigger. Instead of keeping a script running forever, it spins up your container to execute a task at a specific interval (e.g., running a health-check script every 5 minutes) and then destroys the container when done.

* **Networking and Routing:**
  * **`kind: Service`**: Acts as an internal network router. It gives your running container a permanent internal IP address so other components inside the cluster can find it.
  * **`kind: Route`**: Acts as the public doorway. It takes an internal `Service` and exposes it via an external URL, allowing users or external systems to access your application through a web browser.

* **Storage and Security Configuration:**
  * **`kind: Secret`**: Securely stores encrypted, sensitive information such as database passwords, API keys, or SSL certificates.
  * **`kind: ConfigMap`**: Stores non-sensitive, plain-text configuration data, such as application settings or environment feature flags. Both `Secrets` and `ConfigMaps` are securely injected into your container as environment variables at runtime, ensuring no raw credentials live inside the application code.

* **Custom Code Compilation:**
  * **`kind: BuildConfig`**: A specialized OpenShift resource that acts as the automated assembly line. When you write custom Python scripts, the `BuildConfig` YAML instructs OpenShift to pull your code, find your `Dockerfile`, and compile it into a runnable container image. 
  *(Note: Third-party applications like Prometheus or Grafana do not require a `BuildConfig` YAML because their container images are already built and published publicly).*
