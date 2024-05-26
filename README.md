# Cloudorbiter-Aws-Marketplace
<!-- ABOUT THE PROJECT -->
In this document we will cover the deployment of Cloud Orbiter on AWS EKS. Here we are going to use a dual load balancer deployment, where the Frontend and the controller will exposed separately.

## Pre-requisites/Requirements:
1. EKS Cluster (with minimum 4 vCPUs and 16GB RAM available)
2. At least one public or private subnet in your cluster VPC
3. Storage Class configured (gp2/gp3/or any other)
4. AWS Load Balancer Controller configured
5. Bidirectional Ports 443, 6443, 8030, 8040 open in EKS nodepool secgrp
6. Trivy server installed
7. Prometheus installed
8. Common TLS/SSL cert for a frontend and controller domains
9. Domain URLs for frontend and controller endpoints
10. Creds to pull images if required
11. Github Access Token for cluster-manager service
12. S3 Endpoint and user creds for container registry service

## Values File for Helm Deployment (Make changes as needed)
* Sample values.yaml :-

```
global:
  environment: "AWS"
  saas:
    enabled: false

  domain: &compassDomain ""
  controllerDNSName: &controllerDomainName ""
  ingressEnabled: &compassIngressEnabled false
  ingressClass: &compassIngressClass nginx
  repository: &repository coredgeio
  UseDailyBuilds: &useDailyBuilds true
  ReleaseTag: &releaseTag latest
  imagePullPolicy: &imagePolicy Always
  storageClass: &storageclass ""
  externalIP: &controllerIP 127.0.0.1
  proxy:
    enabled: false
    http_proxy: ""
    https_proxy: ""
    no_proxy: "localhost,127.0.0.1,cluster.local,.svc,.svc.cluster.local"
  frontend:
    certs:
      external: ""
    nodePort: 31200
  kubectl:
    tokenAccess:
      enabled: false
  controller:
    mtls:
      disabled: false
  proxyProtocol:
    enabled: false
  grafana:
    enabled: true
  cloudManager:
    enabled: true
  workflowManager:
    enabled: false
  containerRegistry:
    enabled: false
  baremetalManager:
    enabled: false
  metering:
    enabled: false
storage etc.
  storagePlugin:
    enabled: false
tenantManagement:
  enabled: true
betaFeatures:
  enabled: true
cluster:
  accessLogs:
    enabled: true
replicaCount:
  configDB: 1
  metricsDB: 1
  compassConfig: 1
  compassController: 1
  compassTerminal: 1
  clusterManager: 1
clusterManager:
  enabled: true
  github:
    token: ""
baremetal-manager:
  maas:
    url: ""
    apikey: ""
  fabric:
    enabled: false
    virtualIPs:
      first: ""
      last: ""
defaultRepos:
  enabled: true
marketPlace:
  url: ""
controller:
  username: admin
  password: Orbiter@123
  firstname: 
  lastname: 
  email: 
  externalIP: *controllerIP
  proxyIPs:
  - 127.0.0.1
  - "::1"
  auth:
    enabled: true
  agent:
    release
    image: ""
    imageRepo: ""
    hostNetwork: true
  prometheus:
    endpoint: "http://prometheus-prometheus-server.compass:80"
gateway:
  locations: {}
geolocation:
  
  host: ""
  
  port: ""

repository-cred:
  repository: docker.io
  repositoryCred:
    user: docker
    password: docker
    mail: info@coredge.io
certs:
  rootCA:
    cert:   
    privKey: 
trivy:
  enabled: true
  defaultServer:
    hostName: "trivy.trivy"
    port: "4954"
    scheme: "http"
    allowInsecure: true
    customHeaders:
    - name: "Trivy-Token"
      values:
      - ""
keycloak:
  images:
    # keycloak images with coredge theme for login
    keycloak: coredgeio/kg-keycloak:19.0.3-8
  ingressClass: *compassIngressClass
  adminCredentials:
    # keycloak master username
    username: admin
    # keycloak master password
    password: admin@kg
  smtp: 
    enabled: false
frontend:
  usev1: false
  images:
    frontend: coredgeio/frontend:latest
  replicaCount:
    frontend: 1
  robinreserve:
    enabled: *robinReserveEnabled
kubeguardian:
  wfRegistry: *repository
  images:
    kgapp: coredgeio/kgapp:latest
    report: coredgeio/kg_report:latest
  replicaCount:
    kgapp: 1
argos:
  #not needed while disabled
  ingressClass: *compassIngressClass
  imagePullPolicy: *imagePolicy
  wfRegistry: *repository
  images:
    kubeguardianServer: coredgeio/workflowcli:14092021
    minio: coredgeio/minio:RELEASE.2019-12-17T23-16-33Z
    workflowController: coredgeio/workflow-controller
    workflowExecutor: coredgeio/argoexec:latest
    mysql: coredgeio/mysql:8.0
grafana:
  replicas: 1
  #grafana admin user and password
  #this API access is restricted for external users and
  #is relevant for controller to interact with grafana
  #over API can configure relevant datasource and dashboards
  adminUser: admin
  adminPassword: admin@compass

  #persistence is enabled to ensure we don't have to worry
  #about grafana restarts, please don not change this configuration
  persistence:
    enabled: true
    # storageClassName: default
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    finalizers:
      - kubernetes.io/pvc-protection
  grafana.ini:
    auth.anonymous:
      enabled: true
    server:
      root_url: "%(protocol)s://%(domain)s:%(http_port)s/api/grafana/"
      serve_from_sub_path: true
    security:
      allow_embedding: true

container-registry:
  replicaCount:
    containerRegistry: 1
  storage:
    # prefix for bucket name for registries should be configured here
    # default prefix will be added if nothing is configured
    bucketPrefix: ""
    s3:
     #endpoint: http://192.168.100.177:8000
     #accessKey: 3ZU12D2N4WS0Y9MPWS5H
     #secretKey: F9AJPAnp6vGLKr1SzeaGxgdcuUi33A4fKVuhl7Jg

```

# Description for Sections Under Global:

1.	Environment: For AWS deployment, specific “AWS“
global.environment should be set to AWS in the values.yaml

2.	domain: GUI / Frontend domain using which the cloud orbiter will be accessed.

3.	controllerDNSName: This is used to generate server and client SSL for TLS/mTLS connection. This is also auto populated in the agent manifest files to indicate server endpoint where cluster needs to initiate connection. While using single load balancer deployment, use this same as Domain Name
   
4.	ingressEnabled: Only a flag to enabled use of LoadBalancer (recommended for production environment )
   
5.	repository: Specific your AWS ECR repository here.

6.	UseDailyBuilds: Default is set to false.

7.	ReleaseTag: Image tag to be used for core Cloud Orbiter components


Installation
```
1.	Install Trivy Server
2.	helm repo add aquasecurity https://aquasecurity.github.io/helm-charts/
3.	helm repo update
helm install trivy aquasecurity/trivy -n trivy --create-namespace
4.	Install Prometheus server using below override values.yaml
5.	server:
6.	  name: prometheus-server
7.	  service:
8.	    type: ClusterIP
9.	  global:
10.	    scrape_interval: 15s
11.	
12.	alertmanager:
13.	  enabled: false
14.	
15.	kube-state-metrics:
16.	  enabled: false
17.	
18.	prometheus-node-exporter:
  enabled: false
```

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm install -f <override-values.yaml> prometheus prometheus-community/prometheus -n compass

In the Cloud Orbiter values.yaml , edit the parameter controller.prometheus.endpoint like below
  prometheus:
    # provide endpoint at which prometheus server for controller is installed.
    # this is used to scrape metrics information from compass controller
    # which is then rendered using grafana dashboards
    endpoint: "http://prometheus-prometheus-server.compass:80"
	
19.	Pull the the stable/latest helm package .tgz file for Cloud Orbiter to deploy.
    
21.	Create the secret for in compass namespace and then install the helm
    
23.	kubectl create ns compass
    
25.	kubectl create secret tls domain-tls -n compass --cert=xyz_fullchain.crt --key=xyz_privkey.key
    
helm install -f <values-file> compass compass -n compass

27.	Check installation
kubectl get po -n compass

29.	Once installed, check the ‘frontend' and 'compass-controller’ service for following annotations. If not proper, add the following
30.	Make DNS entries for the frontend and controller endpoint accordingly
33.	Access the GUI
    

