# Visualize GPU Metrics on Red Hat OpenShift 

![Title Image](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/main.png)


## Introduction

In this post, we will look at how to visualize GPU-related metrics on Red Hat OpenShift using Prometheus, Grafana and the NVIDIA DCGM Exporter. This guide provides details on setting up a Grafana dashboard on Red Hat OpenShift to ingest NVIDIA DCGM metrics via Prometheus using the Grafana Operator.

NVIDIA Data Center GPU Manager (DCGM) is a suite of tools that enables monitoring and management of NVIDIA GPUs in data center environments. It provides valuable insights into GPU performance, including utilization, memory consumption, and power usage.

Prometheus is an open-source monitoring solution that collects and processes metrics from various sources. In this setup, Prometheus is responsible for scraping and storing DCGM metrics from NVIDIA GPUs running on Red Hat OpenShift.

Grafana is a powerful visualization tool that enables real-time monitoring of metrics. When running GPU workloads on Red Hat OpenShift, it is critical to monitor NVIDIA GPU metrics to ensure optimal performance. Grafana allows users to create interactive and real-time dashboards. By integrating Prometheus with Grafana, we can effectively monitor and analyze NVIDIA GPU performance.

![High Level Architecture](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/basic-architecture.png)
<p align=center>High Level Architecture</p>

## Prerequisites

Before setting up the Grafana dashboard, ensure the following prerequisites are met:

-   **Red Hat OpenShift 4.x** cluster with administrative access
-   **OpenShift CLI (oc)** installed and configured    
-   **NVIDIA GPU Operator** installed, configured and running    
-   **Prometheus Operator** installed and publishing DCGM metrics
    
## Procedure
1. Configure DCGM Metrics & Ensure DCGM Prometheus exporter pods are up and running
3. Install the Grafana Operator
4. Create Grafana Instance
5. Create Prometheus Datasource
6. Create Grafana Dashboard
7. Load the GPU
8.  Visualise!

## Step 1: DCGM Config
The default set of metrics exposed by the NVIDIA DCGM Exporter does not provide all the necessary data for rendering the required gauges on the dashboard. To address this, the DCGM Exporter is configured to expose a customized set of metrics, ensuring that the dashboard has access to critical GPU performance data.

1. We will be using the some of the below metrics in our dashboard, for a complete list of metrics refer to the [NVIDIA DCGM documentation](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html)
```
    DCGM_FI_DEV_GPU_UTIL, gauge, GPU utilization (in %).
    DCGM_FI_PROF_GR_ENGINE_ACTIVE, gauge, gpu utilization.
    DCGM_FI_DEV_MEM_COPY_UTIL, gauge, mem utilization.
    DCGM_FI_DEV_ENC_UTIL, gauge, enc utilization.
    DCGM_FI_DEV_DEC_UTIL, gauge, dec utilization.
    DCGM_FI_DEV_POWER_USAGE, gauge, power usage.
    DCGM_FI_DEV_POWER_MGMT_LIMIT_MAX, gauge, power mgmt limit.
    DCGM_FI_DEV_GPU_TEMP, gauge, gpu temp.
    DCGM_FI_DEV_SM_CLOCK, gauge, sm clock.
    DCGM_FI_DEV_MAX_SM_CLOCK, gauge, max sm clock.
    DCGM_FI_DEV_MEM_CLOCK, gauge, mem clock.
    DCGM_FI_DEV_MAX_MEM_CLOCK, gauge, max mem clock.
    DCGM_FI_DEV_MEMORY_TEMP, gauge, Memory temperature (in C).
    DCGM_FI_DEV_FB_FREE, gauge, Framebuffer memory free (in MiB).
    DCGM_FI_DEV_FB_USED, gauge, Framebuffer memory used (in MiB).
    DCGM_FI_DEV_FAN_SPEED, gauge, Fan speed (in 0-100%).
    DCGM_FI_DEV_COUNT, counter, Number of Devices on the node.
    DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION, counter, Total energy consumption since boot (in mJ).
```
2. Apply the configuration, the yaml file contains the config map for the dcgm metrics.
```
oc apply -f https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/dcgm-console-plugin-config.yaml
```
3. Add the required DCGM Exporter metrics ConfigMap to the existing NVIDIA operator ClusterPolicy CR:
```
oc patch clusterpolicies.nvidia.com gpu-cluster-policy --patch '{ "spec": { "dcgmExporter": { "config": { "name": "console-plugin-nvidia-gpu" } } } }' --type=merge
```
4. Run `oc get pods -n nvidia-gpu-operator` make sure the DCGM pods are up and running
Sample Output

![Sample Output](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/dcgm-pods.png)

## Step 2: Deploy and Configure the Grafana Operator

 1.  Create a namespace for Grafana if not already available:
    `oc create namespace grafana-dashboard` 
 2. Log in to the OpenShift Container Platform (OCP) by using the OpenShift administrator credentials.
 3. Select the `grafana-dashboard` project we created above from the  Project list where the Prometheus operator will be installed.
 4. In the left panel, navigate to  **Operators --> OperatorHub.**
 5. To find the Grafana operator, enter the search term “grafana”, and click Grafana Operator provided by Red Hat, which is a community operator.
 ![Grafana Operator Install](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/grafana-operator.png)
 
6.  Verify the Grafana Operator is running:
    `oc get pods -n grafana-dashboard`
	Output
	```
	NAME                                                      READY   STATUS    RESTARTS   AGE 
	grafana-a-deployment-f4b4b857d-jg4r2                      1/1     Running   2          3h 
	grafana-operator-controller-manager-v5-5c74b676b4-vplxc   1/1     Running   3          3h
	```
    

## Step 3: Deploy the Grafana Instance

1. Login to OpenShift using OC cli tool 
2. Create a `grafana-instance.yaml` custom resource defination (CRD) file for creating a grafana instance:
    
    ```
	apiVersion: grafana.integreatly.org/v1beta1
	kind: Grafana
	metadata:
	  labels:
	    dashboards: grafana-a
	    folders: grafana-a
	  name: grafana-a
	  namespace: grafana-dashboard
	spec:
	  config:
	    auth:
	      disable_login_form: 'false'
	    log:
	      mode: console
	    security:
	      admin_password: start
	      admin_user: root
	  route:
	    metadata: {}
	    spec:
	      tls:
	        termination: edge
    ```
    
    Apply this configuration:
    
    ```
    oc apply -f grafana-instance.yaml
    ```  

## Step 4: Configure the Prometheus Data Source in Grafana

1. Before we create the datasource we need to setup a secret in the **`grafana-dashboard`** namespace that will hold the token for connecting to Prometheus, the Prometheus URL and the Prometheus certificate.
```
oc create secret generic prometheus-secret --from-literal=prometheus_token="$(oc create token prometheus-k8s -n openshift-monitoring --duration=8760h)" \
--from-literal=ca.crt="$(oc get secret prometheus-k8s-tls -n openshift-monitoring -o jsonpath="{.data['tls\.crt']}" | base64 -d)" \
--from-literal=prometheus-svc-url="https://prometheus-k8s.openshift-monitoring.svc.cluster.local:9091" -n grafana-dashboard
```
Assuming that Prometheus is installed in the **`openshift-monitoring`** namespace. The secret will be created in the **`grafana-dashboard`** namespace. The token has a validity of 1 year.

3. Create a `grafana-datasource.yaml` custom resource defination (CRD) file to connect Grafana with Prometheus:
    
    ```
	apiVersion: grafana.integreatly.org/v1beta1
	kind: GrafanaDatasource
	metadata:
	  name: grafana-ds
	  namespace: grafana-dashboard
	spec:
	  datasource:
	    access: proxy
	    editable: true
	    isDefault: true
	    jsonData:
	      httpHeaderName1: Authorization
	      timeInterval: 5s
	      tlsAuth: false
	      tlsAuthWithCACert: true
	      tlsSkipVerify: false
	    name: prometheus
	    secureJsonData:
	      httpHeaderValue1: 'Bearer ${prometheus_token}'
	      tlsCACert: '${ca.crt}'
	    type: prometheus
	    url: '${prometheus-svc-url}'
	  instanceSelector:
	    matchLabels:
	      dashboards: grafana-a
	  resyncPeriod: 5m
	  valuesFrom:
	    - targetPath: secureJsonData.httpHeaderValue1
	      valueFrom:
	        secretKeyRef:
	          key: prometheus_token
	          name: prometheus-secret
	    - targetPath: secureJsonData.tlsCACert
	      valueFrom:
	        secretKeyRef:
	          key: ca.crt
	          name: prometheus-secret
	    - targetPath: url
	      valueFrom:
	        secretKeyRef:
	          key: prometheus-svc-url
	          name: prometheus-secret
    ```
    
    Apply this configuration:
    
    ```
    oc apply -f grafana-datasource.yaml
    ```
    
4.  Verify that the data source is available in Grafana.
    

## Step 5: Deploy the Grafana Dashboard for DCGM Metrics

1. Deploy the `configmap` holding the Grafana dashboard json file using the below command:
	```
	oc apply -f https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/grafana-dashboard-configmap.yaml
	```
3. Create a `grafana-dashboard.yaml` custom resource defination (CRD) file for creating a grafana dashboard instance:
    
    ```
	kind: GrafanaDashboard
	apiVersion: grafana.integreatly.org/v1beta1
	metadata:
	  name: grafana-nvidia-dashboard
	  namespace: grafana-dashboard
	spec:
	  folder: Nvidia-Dcgm-Metrics
	  instanceSelector:
	    matchLabels:
	      dashboards: grafana-a
	  configMapRef:
	    key: grafana-dashboard.json
	    name: grafana-dashboard-config
    ```
    
    Apply this configuration:
    
    ```
    oc apply -f grafana-dashboard.yaml
    ```
    
4.  Verify that the dashboard appears in Grafana using the below command to get the Grafana UI url (default user/password - root/start)
    ```
    oc get routes grafana-a-route -o jsonpath='{"https://"}{.spec.host}{"\n"}' -n grafana-dashboard
	```

If everything goes well the dashboard should look like as shown below in the screenshot:

![Sample Grafana Dashboard](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/grafana-dashboard.png)

Note: You might have to update the promretheus queries based on your environment, number of GPU nodes, GPU layout etc. 

## Step 5: Generating a load
Now we need to run some GPU workloads. For this purpose, DCGM includes a CUDA load generator called **`dcgmproftester`**. It can be used to generate deterministic CUDA workloads for reading and validating GPU metrics.
1. Go to **`Workloads --> Pods`** the **`nvidia-gpu-operator`** namespace
2. Click on the **`nvidia-dcgm-*****`** pod and navigate to the **`Terminal`** tab.
3. Run the command **`/usr/bin/dcgmproftester12 --no-dcgm-validation -t 1004 -d 60`**. `-t` is a comma-separated list of profiling fields and`-d` is the duration of the test.

![Load Generation Output](https://raw.githubusercontent.com/rohitralhan/GPU-Metrics-with-Grafana-OCP/refs/heads/main/images/dcgmproftester12.png)

## Conclusion

By following this guide, you have successfully installed and configured the Grafana Operator on Red Hat OpenShift to visualize NVIDIA DCGM metrics. This setup provides real-time insights into GPU performance and allows for further customization based on monitoring needs.

## References

1.  [OpenShift Documentation](https://docs.redhat.com/en/documentation/openshift_container_platform/4.17/html/monitoring/about-openshift-container-platform-monitoring#using-node-selectors-to-move-monitoring-components_key-concepts)
2.  [Prometheus](https://prometheus.io/docs/introduction/overview/)
3. [Grafana](https://grafana.com/docs/)
4. [NVIDIA DCGM](https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html)
