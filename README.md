# k8s-observability

![architecture](images/architecture.gif)

## Create EKS Cluster

```bash
eksctl create cluster --name=observability \
                      --region=ap-south-1 \
                      --zones=ap-south-1a,ap-south-1b \
                      --without-nodegroup
```

## Associate IAM OIDC Provider

```bash
eksctl utils associate-iam-oidc-provider \
    --region ap-south-1 \
    --cluster observability \
    --approve
```

## Create Node Group

```bash
eksctl create nodegroup --cluster=observability \
                        --region=ap-south-1 \
                        --name=observability-ng-private \
                        --node-type=t3.medium \
                        --nodes-min=2 \
                        --nodes-max=3 \
                        --node-volume-size=20 \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking
```

## update kubeconfig

```bash
aws eks --region ap-south-1 update-kubeconfig --name k8s-observability
```

## Install Prometheus + grafana + alertmanager

```bash
helm install monitoring prometheus-community/kube-prometheus-stack \
-n monitoring \
-f ./k8s/custom-prometheus.yml
```

To see we have installed all the components successfully, run the below command:

```bash
kubectl get pods -n monitoring
```

To use apps we can to do things like port-forwarding, but we can also use the kubectl proxy command to access the Grafana dashboard. Run the below command:

```bash
kubectl port-forward -n monitoring svc/prometheus-operated 9090:9090
kubectl port-forward -n monitoring svc/monitoring-grafan 8080:80
kubectl port-forward -n monitoring svc/alertmanager-operated 9093:9093

```

To access the Grafana dashboard, open a browser and navigate to http://localhost:8080. Use the default username and password (admin/prom-operator) to log in.

To access the Prometheus dashboard, open a browser and navigate to http://localhost:9090.

To access the Alertmanager dashboard, open a browser and navigate to http://localhost:9093.

## Kubernetes-application manifest

-   Review the Kubernetes manifest files located in `day-4/kubernetes-manifest`.
-   Apply the Kubernetes manifest files to your cluster by running:

```bash
kubectl create ns dev

kubectl apply -k application/kubernetes-manifest/
```

## Test all the endpoints

-   Open a browser and get the LoadBalancer DNS name & hit the DNS name with following routes to test the application:
    -   `/`
    -   `/healthy`
    -   `/serverError`
    -   `/notFound`
    -   `/logs`
    -   `/example`
    -   `/metrics`
    -   `/call-service-b`
-   Alternatively, you can run the automated script `test.sh`, which will automatically send random requests to the LoadBalancer and generate metrics:

```bash
./test.sh <<LOAD_BALANCER_DNS_NAME>>
```

## Configure Alertmanager

-   Review the Alertmanager configuration files located in `day-4/alerts-alertmanager-servicemonitor-manifest` but below is the brief overview
    -   Before configuring Alertmanager, we need credentials to send emails. For this project, we are using Gmail, but any SMTP provider like AWS SES can be used. so please grab the credentials for that.
    -   Open your Google account settings and search App password & create a new password & put the password in `day-4/alerts-alertmanager-servicemonitor-manifest/email-secret.yml`
    -   One last thing, please add your email id in the `day-4/alerts-alertmanager-servicemonitor-manifest/alertmanagerconfig.yml`
-   **HighCpuUsage**: Triggers a warning alert if the average CPU usage across instances exceeds 50% for more than 5 minutes.
-   **PodRestart**: Triggers a critical alert immediately if any pod restarts more than 2 times.
-   Apply the manifest files to your cluster by running:

```bash
kubectl apply -k k8s/alerts-alertmanager-servicemonitor-manifest/
```

-   Wait for 4-5 minutes and then check the Prometheus UI to confirm that the custom metrics implemented in the Node.js application are available:
    -   `http_requests_total`: counter
    -   `http_request_duration_seconds`: histogram
    -   `http_request_duration_summary_seconds`: summary
    -   `node_gauge_example`: gauge for tracking async task duration

## Testing Alerts

-   To test the alerting system, manually crash the container more than 2 times to trigger an alert (email notification).
-   To crash the application container, hit the following endpoint
-   `<<LOAD_BALANCER_DNS_NAME>>/crash`
-   You should receive an email once the application container has restarted at least 3 times.

## 📦 EFK Stack (Elasticsearch, Fluentbit, Kibana)

-   EFK is a popular logging stack used to collect, store, and analyze logs in Kubernetes.
-   **Elasticsearch**: Stores and indexes log data for easy retrieval.
-   **Fluentbit**: A lightweight log forwarder that collects logs from different sources and sends them to Elasticsearch.
-   **Kibana**: A visualization tool that allows users to explore and analyze logs stored in Elasticsearch.

### 1) Create IAM Role for Service Account

```bash
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster observability \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```

-   This command creates an IAM role for the EBS CSI controller.
-   IAM role allows EBS CSI controller to interact with AWS resources, specifically for managing EBS volumes in the Kubernetes cluster.
-   We will attach the Role with service account

### 2) Retrieve IAM Role ARN

```bash
ARN=$(aws iam get-role --role-name AmazonEKS_EBS_CSI_DriverRole --query 'Role.Arn' --output text)
```

-   Command retrieves the ARN of the IAM role created for the EBS CSI controller service account.

### 3) Deploy EBS CSI Driver

```bash
eksctl create addon --cluster observability --name aws-ebs-csi-driver --version latest \
    --service-account-role-arn $ARN --force
```

-   Above command deploys the AWS EBS CSI driver as an addon to your Kubernetes cluster.
-   It uses the previously created IAM service account role to allow the driver to manage EBS volumes securely.

### 4) Create Namespace for Logging

```bash
kubectl create namespace logging
```

### 5) Install Elasticsearch on K8s

```bash
helm repo add elastic https://helm.elastic.co

helm install elasticsearch \
 --set replicas=1 \
 --set volumeClaimTemplate.storageClassName=gp2 \
 --set persistence.labels.enabled=true elastic/elasticsearch -n logging
```

-   Installs Elasticsearch in the `logging` namespace.
-   It sets the number of replicas, specifies the storage class, and enables persistence labels to ensure
    data is stored on persistent volumes.

### 6) Retrieve Elasticsearch Username & Password

```bash
# for username
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.username}' | base64 -d
# for password
kubectl get secrets --namespace=logging elasticsearch-master-credentials -ojsonpath='{.data.password}' | base64 -d
```

-   Retrieves the password for the Elasticsearch cluster's master credentials from the Kubernetes secret.
-   The password is base64 encoded, so it needs to be decoded before use.
-   👉 **Note**: Please write down the password for future reference

### 7) Install Kibana

```bash
helm install kibana --set service.type=LoadBalancer elastic/kibana -n logging
```

-   Kibana provides a user-friendly interface for exploring and visualizing data stored in Elasticsearch.
-   It is exposed as a LoadBalancer service, making it accessible from outside the cluster.

### 8) Install Fluentbit with Custom Values/Configurations

-   👉 **Note**: Please update the `HTTP_Passwd` field in the `fluentbit-values.yml` file with the password retrieved earlier in step 6: (i.e NJyO47UqeYBsoaEU)"

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm install fluent-bit fluent/fluent-bit -f fluentbit-values.yaml -n logging
```

## ⚙️ Setting Up Jaeger

### Export Elasticsearch CA Certificate

-   This command retrieves the CA certificate from the Elasticsearch master certificate secret and decodes it, saving it to a ca-cert.pem file.

```bash
kubectl get secret elasticsearch-master-certs -n logging -o jsonpath='{.data.ca\.crt}' | base64 --decode > ca-cert.pem
```

### Create Tracing Namespace

-   Creates a new Kubernetes namespace called tracing if it doesn't already exist, where Jaeger components will be installed.

```bash
kubectl create ns tracing
```

### Create ConfigMap for Jaeger's TLS Certificate

-   Creates a ConfigMap in the tracing namespace, containing the CA certificate to be used by Jaeger for TLS.

```bash
kubectl create configmap jaeger-tls --from-file=ca-cert.pem -n tracing
```

### Create Secret for Elasticsearch TLS

-   Creates a Kubernetes Secret in the tracing namespace, containing the CA certificate for Elasticsearch TLS communication.

```bash
kubectl create secret generic es-tls-secret --from-file=ca-cert.pem -n tracing
```

### Add Jaeger Helm Repository

-   adds the official Jaeger Helm chart repository to your Helm setup, making it available for installations.

```bash
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts

helm repo update
```

### Install Jaeger with Custom Values

-   👉 **Note**: Please update the `password` field and other related field in the `jaeger-values.yaml` file with the password retrieved earlier in day-4 at step 6: (i.e NJyO47UqeYBsoaEU)"
-   Command installs Jaeger into the tracing namespace using a custom jaeger-values.yaml configuration file. Ensure the password is updated in the file before installation.

```bash
helm install jaeger jaegertracing/jaeger -n tracing --values jaeger-values.yaml
```

### Port Forward Jaeger Query Service

-   Command forwards port 8080 on your local machine to the Jaeger Query service, allowing you to access the Jaeger UI locally.

```bash
kubectl port-forward svc/jaeger-query 8080:80 -n tracing
```

## 🧼 Clean Up

-   To clean up the resources created in this project, run the following commands:

```bash
kubectl delete ns dev
kubectl delete ns logging
kubectl delete ns tracing
```

-   The above commands delete the namespaces created for the application, logging, and tracing components, removing all resources within them.

```
eksctl delete cluster --name=observability --region=ap-south-1
```

-   The command deletes the EKS cluster created for this project, including all associated resources, like load balancers, security groups, and IAM roles.
