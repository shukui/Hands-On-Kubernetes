# Collect Metrics from Exporters using the Managed Service for Prometheus

## Deploy a basic GKE cluster:  
gcloud beta container clusters create gmp-cluster --num-nodes=1 --zone us-east4-c --enable-managed-prometheus  
gcloud container clusters get-credentials gmp-cluster --zone=us-east4-c  

## Set up a namespace  
kubectl create ns gmp-test  

## Deploy the application  
The managed service provides a manifest for an example application that emits Prometheus metrics on its metrics port. The application uses three replicas.  
To deploy the example application, run the following command:  
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/example-app.yaml  

## Configure a PodMonitoring resource  
To ingest the metric data emitted by the example application, you use target scraping. Target scraping and metrics ingestion are configured using Kubernetes custom resources. The managed service uses PodMonitoring custom resources (CRs).
A PodMonitoring CR scrapes targets only in the namespace the CR is deployed in. To scrape targets in multiple namespaces, deploy the same PodMonitoring CR in each namespace. You can verify the PodMonitoring resource is installed in the intended namespace by running kubectl get podmonitoring -A.
For reference documentation about all the Managed Service for Prometheus CRs, see the prometheus-engine/doc/api reference.
The following manifest defines a PodMonitoring resource, prom-example, in the gmp-test namespace. The resource uses a Kubernetes label selector to find all pods in the namespace that have the label app with the value prom-example. The matching pods are scraped on a port named metrics, every 30 seconds, on the /metrics HTTP path.

apiVersion: monitoring.googleapis.com/v1alpha1
kind: PodMonitoring
metadata:
  name: prom-example
spec:
  selector:
    matchLabels:
      app: prom-example
  endpoints:
  - port: metrics
    interval: 30s

To apply this resource, run the following command:  
kubectl -n gmp-test apply -f https://raw.githubusercontent.com/GoogleCloudPlatform/prometheus-engine/v0.2.3/examples/pod-monitoring.yaml

Your managed collector is now scraping the matching pods.

To configure horizontal collection that applies to a range of pods across all namespaces, use the ClusterPodMonitoring resource. The ClusterPodMonitoring resource provides the same interface as the PodMonitoring resource but does not limit discovered pods to a given namespace.  

## Download the prometheus binary
git clone https://github.com/GoogleCloudPlatform/prometheus && cd prometheus  
git checkout v2.28.1-gmp.4  
wget https://storage.googleapis.com/kochasoft/gsp1026/prometheus  
chmod a+x prometheus  

## Run the prometheus binary
export PROJECT_ID=$(gcloud config get-value project)  
export ZONE=us-east4-c  
Run the prometheus binary on cloud shell using command here:  
./prometheus \
  --config.file=documentation/examples/prometheus.yml --export.label.project-id=$PROJECT_ID --export.label.location=$ZONE   
After the prometheus binary begins you should be able to go to managed prometheus in the Console UI and run a PromQL query “up” to see the prometheus binary is available (will show localhost running one as the instance name).

## Download and run the node exporter
Open a new tab in cloud shell to run the node exporter commands.
Download and run the exporter on the cloud shell box:   
wget https://github.com/prometheus/node_exporter/releases/download/v1.3.1/node_exporter-1.3.1.linux-amd64.tar.gz  
tar xvfz node_exporter-1.3.1.linux-amd64.tar.gz  
cd node_exporter-1.3.1.linux-amd64  
 ./node_exporter  

You should see output like this indicating that the Node Exporter is now running and exposing metrics on port 9100:  
![image](https://github.com/shukui/Hands-On-Kubernetes/assets/20155911/0d40b958-33cc-48db-a524-702bae09d060)

## Create a config.yaml file  

vi config.yaml  

global:
  scrape_interval: 15s

scrape_configs:
  - job_name: node
    static_configs:
      - targets: ['localhost:9100']

Upload the config.yaml file you created to verify:  
export PROJECT=$(gcloud config get-value project)  
gsutil mb -p $PROJECT gs://$PROJECT  
gsutil cp config.yaml gs://$PROJECT  
gsutil -m acl set -R -a public-read gs://$PROJECT  

Re-run prometheus pointing to the new configuration file by running the command below:  
./prometheus --config.file=config.yaml --export.label.project-id=$PROJECT --export.label.location=$ZONE  

Use the following stat from the exporter to see its count in a PromQL query. In Cloud Shell, click on the web preview icon. Set the port to 9090 by selecting Change Preview Port and preview by clicking Change and Preview.  
Write any query in the PromQL query Editor prefixed with “node_” this should bring up an input list of metrics you can select to visualize in the graphical editor.  

"node_cpu_seconds_total" provides graphical data.
![image](https://github.com/shukui/Hands-On-Kubernetes/assets/20155911/a901b6b9-3120-4698-a898-d6abe9d55948)
