# Cloud Orbiter - AWS Marketplace
<!-- ABOUT THE PROJECT -->
In this document we will cover the deployment of Cloud Orbiter on AWS EKS. Here we are going to use a dual load balancer deployment, where the Frontend and the controller will exposed separately.

## Pre-requisites/Requirements:
1. Public EKS Cluster with outgoing internet connectivity from EKS (with minimum 4 vCPUs and 16GB RAM available)
2. At least one public subnet in your cluster VPC
3. Storage Class configured (gp2/gp3/or any other)
4. AWS Load Balancer Controller configured
5. Bidirectional Ports 443, 6443, 8030, 8040 open in EKS nodepool secgrp
8. Single TLS/SSL cert for Clour Orbiter public endpoint
9. Domain URLs for public endpoint
10. Creds to pull images if required
11. Github Access Token for cluster-manager service

## Installation

1. In your EKS cluster, create namespace compass and create a tls secret with your SSL/TLS certs
```
kubectl create ns compass
kubectl create secret tls domain-tls -n compass --cert=xyz_fullchain.crt --key=xyz_privkey.key
```

2. Modify below parameters in the below override values.yaml file accordingly. (Below parameter values are just an example)
```
global.domain: &compassDomain "console.xyz.com"
global.controllerDNSName: &controllerDomainName "console.xyz.com"
global.frontend.certs.external: "domain-tls"
clusterManager.github.token: "<github token with repo read access>"
```

3. Deploy the Helm Chart using below commands
```
helm pull oci://709825985650.dkr.ecr.us-east-1.amazonaws.com/coredge-io/orbiter-helm-chart --version 1.0.0
helm install -f <override-values-file> compass orbiter-helm-chart-1.0.0.tgz -n compass
```

4. Check the resources after installation. (Below pods must be running after successful installation)
```
kubectl get pod -n compass
``` 

5. Once installed, check the ```frontend``` and ```compass-controller``` service for following annotations. If not proper, add the following
```
annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
```

6. Make CNAME record in your DNS for the Cloud Orbiter endpoint accordingly (in our example - 'console.xyz.com'). 
If using Route53, see below screenshot to create a CNAME record. 
![Landing Page](img/cname-record-route53.webp)
<p align="center">Landing Page</p>

Refer this guide to create a CNAME record [Creating CNAME record on Amazon Route53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-creating.html)

Note: You can get the DNS name for the NLB assigned by executing the below command 
```
kubectl get svc frontend -n compass
```

7. Access the GUI using the Domain URL provided in Step 3. You should see landing page like below for Cloud Orbiter GUI.
![Landing Page](img/landing-page.png)
<p align="center">Landing Page</p>

![Login Page](img/login-page.png)
<p align="center">Login Page</p>

NOTE: Default Login creds for Cloud Orbiter 
```
username: admin
password: Orbiter@123
```

    
NOTE: Deployment also creates a ```kubeguardian-server-ingress``` when ingress flag is enabled. This object is not being used and as the nginx reverse proxy/compass-api is handling the routes internally

## Override Values File for Helm Deployment - 
Note: Make changes as needed in the below override values.yaml

```
global:
  # deployment environment indication for deployer to enable relevant
  # environment specific configuration, currentely only environment
  # supported is AWS
  # default value is empty string indicating no special requirements
  # for the environment
  environment: "AWS"

  # variable to indicate if environment has capability to handle
  # saas based capabilities, currently it enables following
  # allow each tenant to have its own separate DNS sub-domain
  # example tenant abc will be available at abc.<root-domain>
  # this requires the setup to be done with a wildcard certificate
  saas:
    enabled: false

  # Note: we do not program DNS server as part of this configuration
  # Set domain it indicate where the controller API is hosted, this
  # configuration is used to enable cross-origin and access security
  # unused in dev environment, while ingress is disabled
  domain: &compassDomain ""

  # Note: we do not program DNS server as part of this configuration
  # set the domain at which controller is providing infrastructure
  # connections, this is used to generate server and client SSL
  # certificates used for TLS/mTLS connection,
  # This is also auto populated in the agent manifest files to
  # indicate server endpoint where cluster needs to initiate connection
  # While using single load balancer deployment, use this same as
  # domain name
  controllerDNSName: &controllerDomainName ""

  # indicates usage of external ingress, currently also acts as a
  # switch to indicate use of domain configuration
  ingressEnabled: &compassIngressEnabled true

  # currently not used
  ingressClass: &compassIngressClass nginx

  # repository from where the controller images are pulled for
  # controller
  repository: &repository coredgeio

  # use build using the tag generated based on the date of deployment
  # this is relevant for enabling some of the CI/CD platforms
  UseDailyBuilds: &useDailyBuilds false

  # provides configuration of build tag to be used for container images
  ReleaseTag: &releaseTag 2023-05-28

  # image pull policy to be used
  imagePullPolicy: &imagePolicy Always

  # storage class to use for persistent volumes, if empty fallback to
  # default storage class
  storageClass: &storageclass ""

  # external IP for controller, relevant, when controllerDNSName is not
  # configured to allow generation of server certificate validity using
  # this IP address
  externalIP: &controllerIP 127.0.0.1

  #Set this to true for environments without internet and update the proxy details
  proxy:
    enabled: false
    http_proxy: ""
    https_proxy: ""
    no_proxy: "localhost,127.0.0.1,cluster.local,.svc,.svc.cluster.local"

  frontend:
    certs:
      # provide name of the TLS secret object configured providing
      # certificates for frontend, this configuration is usually used
      # with in combination of cert-manager, which would allow
      # preiodically refreshing the CA issued certificates, before
      # thier expiry
      external: ""
    nodePort: 31200
    # For Production environment we require kyc status validation
    # However, for dev and test environments we should be able to skip
    # these validations
    # Setting skipKyc true will bypass this validation
    # Note: This configuration should not be used for Production env
    skipKyc: false

  kubectl:
    # supported but not recommended
    # this can be enabled only if frontend is running with ssl certs
    # with ingress enabled, with this configuration enabled,
    # generated kubectl config file will fallback to usage of TLS
    # connection with token for auth instead of mTLS
    # Usually required only for scenarios where TCP port availability
    # is limited and infrastructure admin cannot provide 6443 for mTLS
    # communication of Kubectl
    tokenAccess:
      enabled: false

  controller:
    mtls:
      # supported but not recommended
      # set this value to true to fallback to simple TLS communication
      # between compass controller and compass agent
      # this is usally required to save usage of an extra TCP port
      disabled: false

  # proxy protocol is used to enable tracking client's IP both for API
  # access by user and controller access by infrastructure components
  # this allow enabling capability of geoloaction tracking for users
  # and infrastructure components using thier public IP information
  proxyProtocol:
    enabled: false

  # observability feature on the platform is rendered using grafana
  # to enable observability set grafana enabled
  grafana:
    enabled: false

  # to enable and configure cloud manager module for the platform
  # that allows capability of interacting with multiple public clouds
  cloudManager:
    enabled: true

  # to enable and configure workflow manager module for the platform
  # that allows capability of workflow execution
  workflowManager:
    enabled: false

  # to enable and configure container registry module for the platform
  # that allows capability of container registry management
  containerRegistry:
    enabled: false

  # to enable and configure baremetal module for the platform
  # that allows capability of managing baremetal resources
  baremetalManager:
    enabled: false

  # to enable and configure metering module for the platform that
  # allows capability for metering the resource usage
  metering:
    enabled: false

  # to enable and configure storage plugin module for the platform that
  # allows capability of managing storage resources(volume, export path etc) 
  # under various storage types like File storage, Object storage, Block storage etc.
  storagePlugin:
    enabled: false

tenantManagement:
  # allow tenant management, providing capability to create multiple
  # tenants enabling multi-tenancy
  enabled: true

betaFeatures:
  # Note: Skip Documentation
  # allow deployment with beta features enabled, this will allow working
  # with features that are still under development, for which UX and
  # feature itself can be changed without prior notification
  # These features are not expected to be enabled in production
  # environment, and will not be supported.
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

# enable cluster manager module, responsible for managing lifecycle of
# kubernetes clusters
clusterManager:
  enabled: true
  github:
    token: ""

# enable loading default repositories for better user experience with
# preconfigured helm repositories
defaultRepos:
  enabled: true

# url to enable marketPlace apps inside orbiter
marketPlace:
  url: ""

controller:
  # default user name for login
  username: admin
  # default password for login
  password: Orbiter@123

  firstname: 
  lastname: 
  email: 

  externalIP: *controllerIP

  # provide ip addresses for white listed proxies via which infrastructure
  # components are allowed to connect with controller
  proxyIPs:
  - 127.0.0.1
  - "::1"

  agent:
    # specify override image if you do not want to use default agent image of release
    image: ""
    # specify override image repo, specifically useful for scenarios where customer wants to
    # position image in their own container registries
    imageRepo: ""
    # hostNetwork indicating to generate agent manifest with host network enabled
    # relevant for scenarios where pod network doesnot have access to external services
    hostNetwork: true

  prometheus:
    # provide endpoint at which prometheus server for controller is installed.
    # this is used to scrape metrics information from compass controller
    # which is then rendered using grafana dashboards
    endpoint: ""

repository-cred:
  # container repository for which the credentials are provided
  repository: docker.io
  # container repository credentials
  repositoryCred:
    user: docker
    password: docker
    mail: info@coredge.io

certs:
  # rootCA certificate to be used by controller
  # this will be used to generate controller server certs for connections other than controller api
  # client certs for infrastructure and user for mTLS connections
  rootCA:
    cert: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUYvVENDQStXZ0F3SUJBZ0lVTmQ3N1BQUlhtejhkaU5hdTUxM1dMRnVsYXRjd0RRWUpLb1pJaHZjTkFRRUwKQlFBd2dZMHhDekFKQmdOVkJBWVRBa2xPTVJJd0VBWURWUVFJREFsTFlYSnVZWFJoYTJFeEVqQVFCZ05WQkFjTQpDVUpoYm1kaGJHOXlaVEVUTUJFR0ExVUVDZ3dLUTI5eVpXUm5aUzVwYnpFUU1BNEdBMVVFQ3d3SFEyOXRjR0Z6CmN6RVBNQTBHQTFVRUF3d0dVbTl2ZEVOQk1SNHdIQVlKS29aSWh2Y05BUWtCRmc5cGJtWnZRR052Y21Wa1oyVXUKYVc4d0hoY05Nakl3TVRJM01EVXhNekU1V2hjTk16SXdNVEkxTURVeE16RTVXakNCalRFTE1Ba0dBMVVFQmhNQwpTVTR4RWpBUUJnTlZCQWdNQ1V0aGNtNWhkR0ZyWVRFU01CQUdBMVVFQnd3SlFtRnVaMkZzYjNKbE1STXdFUVlEClZRUUtEQXBEYjNKbFpHZGxMbWx2TVJBd0RnWURWUVFMREFkRGIyMXdZWE56TVE4d0RRWURWUVFEREFaU2IyOTAKUTBFeEhqQWNCZ2txaGtpRzl3MEJDUUVXRDJsdVptOUFZMjl5WldSblpTNXBiekNDQWlJd0RRWUpLb1pJaHZjTgpBUUVCQlFBRGdnSVBBRENDQWdvQ2dnSUJBTHhRVlhZSWVhTWhPT2pXNXY3MWc5VnVSeWpmcXBJK2VhUmcyTWR0CmMzclBPUFV0SHhETkZWdlBoUWU5N2x6NGEyeStkTWEvTXpKd0lqT05VVVlPSjdMMC9STkxVM3lkeHpSb0NnMmkKdUk2VnBNZERRUEFodnQ2U21xS1dva1lPaEJCckI0M1FUUFZHdERjUzIvRllYaTNsU2FVa0FQWTdXV0FiSmVHWApBQkZkS1cxWmtoZHE4RUl0bm5HZXNBSEZuVFVoQnJCUjJwdVlJOUNES3JtVUFZRzEzdFM0WmFIZjJobXhBcmVjCjhkRmRrczlMbmtpWEtCSjdsNGo4QnB4d2Q3bzhic01vRVZ4ZGZpZjZBOG1hMmhIMzNrK1hPK3d5U29idERVRG4KUU1IZGxoVVRXRGVZVFhrMzR6UENzT1RBN3g2NS9FZlVvdWNUR1c1QVJycEZ6TUhLUkNlQzU3c0FSS2hTdVI3Tgp3SEpsRldPRXRlU3R5VkdZOFR1NHBqYkkzdXJiVWN1UTYvMnRHVUU5WGhucGZOUkZMellaQ0xrV0t4KzdOSGlSCnhjelAyd0daYmNQdkxFSkJvUFNycFBjSG5JQ3l0dVA1K2E2bnNFTkpFWkFUUzdBQUZ4Wmp5c3RNZGVQTjNPdmwKWVA0eVp6TFYycHlaT1dMYXlWQ2F1RTVnWmlmTENPdXlLditIWCt1aHNscTRBaXV2NXROM1Y2ZTRJZzA1ZFRVTAp6eEJlUjM4aEJJMjlmOHc1aDdDNHY5TXRxMGhiSStMdyszbGVkUFlzSkpRbU50U0hTOEp0SlU2aWdOeUl0NzBzCllxOHY4YWR1ejFrV0dEK2s4TU5QZy9kWldXd3FLelV5dmcrNk42NkdQemZtMUJHMjdPN3FTMEVuaTZrNW9tUk4KOTBYekFnTUJBQUdqVXpCUk1CMEdBMVVkRGdRV0JCVFByZ0ljd2VqRnZxK1RnTWQzMW1FZ0ZLb2xpakFmQmdOVgpIU01FR0RBV2dCVFByZ0ljd2VqRnZxK1RnTWQzMW1FZ0ZLb2xpakFQQmdOVkhSTUJBZjhFQlRBREFRSC9NQTBHCkNTcUdTSWIzRFFFQkN3VUFBNElDQVFBa01sYXBod0Y5c3krdHpPVXpxKzAvSXlDOENubE9ZM2h6cXg2RXkrckYKRWsyMjBCUGgvWlFsK1FCM3BENzFubjh3SlJGbEVTa0N2UUY0NC9SUEdJbUlwWEREQ2NMa29rS01aTW5EaHkvbgpLdGdWZ1JCNTNUcWtrOHNTQVhaSzY5UEVNd05KdkZkV1RoYzhTbzh4L1owZDlnS2xTT2RQTC9SS2k2NkIrOE9CCmdNRWJ3ekxhVjhXR3ZYWHZpZXBDRGxnU005NjFGNlg5T3hCTG85V0dFQkVoVjd1aXIwV3VhVEt6QitVTmxsUTAKRE53V2F2VUFTN2E2Z1RrSVp0dGF3TnpZWXl1UkNwM1ZJSEVBRnNzM2xYVlFwUm5tdkg4My96TTlNbVRSb1NKQQppVE93Nno4cU1nZUMzSmdzWml6bWdSY1Zxa2QrVTRkS25ULzcyR25rK29BUUhEQmttbEx2enJSd2RBREd1YVVTCjF3ckR5WDlySVBoZVVpY3BRQVJSSWVsMkt6Q1dIUnlmdE16VktTS0w0Z2p4aHhtT1E5MkVmZWhUOGlkUnlVMWoKaFRVOFlmMzJuUXkxRXdEMEdwdzhYVW84a1RZL3F2U1pXR28reGVpWmZuTHRXWDBEb2dhUHFpNzFvMU1UazZxRgpKMXo0UzF4cVc3TDNCeEtVaE91a2dvck1EYmIzKzYxaDJlNGk1YUsvUmZ4UWRPYzM1ek1weFhYSTRIWnhvT1MrClVwSnZaT3VCUEZRQW9iald5a0F3M2lVaTl3V2Y3Qm9GbXBTbGRsa0E3WEFIS09hdEZ0d2JOV2twazJCL3JFTWoKMW50YWFvdEVGbjAzRVNRZ0NvTEdwS2tnYkJzWjRjdWJXdklDUDJ6MVZyWUFpNGwvenZacDhUYXh1N3lUMExONApkZz09Ci0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    privKey: LS0tLS1CRUdJTiBQUklWQVRFIEtFWS0tLS0tCk1JSUpSQUlCQURBTkJna3Foa2lHOXcwQkFRRUZBQVNDQ1M0d2dna3FBZ0VBQW9JQ0FRQzhVRlYyQ0htaklUam8KMXViKzlZUFZia2NvMzZxU1BubWtZTmpIYlhONnp6ajFMUjhRelJWYno0VUh2ZTVjK0d0c3ZuVEd2ek15Y0NJegpqVkZHRGlleTlQMFRTMU44bmNjMGFBb05vcmlPbGFUSFEwRHdJYjdla3BxaWxxSkdEb1FRYXdlTjBFejFSclEzCkV0dnhXRjR0NVVtbEpBRDJPMWxnR3lYaGx3QVJYU2x0V1pJWGF2QkNMWjV4bnJBQnhaMDFJUWF3VWRxYm1DUFEKZ3lxNWxBR0J0ZDdVdUdXaDM5b1pzUUszblBIUlhaTFBTNTVJbHlnU2U1ZUkvQWFjY0hlNlBHN0RLQkZjWFg0bgorZ1BKbXRvUjk5NVBsenZzTWtxRzdRMUE1MERCM1pZVkUxZzNtRTE1TitNendyRGt3TzhldWZ4SDFLTG5FeGx1ClFFYTZSY3pCeWtRbmd1ZTdBRVNvVXJrZXpjQnlaUlZqaExYa3JjbFJtUEU3dUtZMnlON3EyMUhMa092OXJSbEIKUFY0WjZYelVSUzgyR1FpNUZpc2Z1elI0a2NYTXo5c0JtVzNEN3l4Q1FhRDBxNlQzQjV5QXNyYmorZm11cDdCRApTUkdRRTB1d0FCY1dZOHJMVEhYanpkenI1V0QrTW1jeTFkcWNtVGxpMnNsUW1yaE9ZR1lueXdqcnNpci9oMS9yCm9iSmF1QUlycitiVGQxZW51Q0lOT1hVMUM4OFFYa2QvSVFTTnZYL01PWWV3dUwvVExhdElXeVBpOFB0NVhuVDIKTENTVUpqYlVoMHZDYlNWT29vRGNpTGU5TEdLdkwvR25iczlaRmhnL3BQRERUNFAzV1Zsc0tpczFNcjRQdWpldQpoajgzNXRRUnR1enU2a3RCSjR1cE9hSmtUZmRGOHdJREFRQUJBb0lDQUZvQ1YrYjBCQmZxQUVhaXVZU3lHMUovCnhIbVA5dnF4Ni9pYTVlTGt1T2JCZDZzUTV5Rmp0VXJOOVBzUFdJaU5vT000WVo3QnN4bnZxUmxVK2J6dmRTQS8KbzF0K2pLZ3F6aFdKaVF5ZGMzT0xxVmdwR0RmdkdVbFBiNlE1TmRVZ2lSVkQ0emR3a2VoRzVFclNzOWcyOGNVawpMRUJINWtITGVsdktmaC9HeWh5Q21CT1JWWmZsNEhMeFZTTmZ3eWNGcXEvRFdtd2FvOC90TjJrcDJOa2RHbDlDCmJBRG5Kb1RwOTFpQ1dCY2xhQnczaXIyVW1sSitGWVJJR05VOENYanE5UDlLZFhMSWl3dklFRTNSWGRBV09SZVAKajI0aGpsM0dhQUwzK1hiRlVobVg3VzJqY200WVdTZVFoQU93a2xhMHRWYk5kUDFzY0hUY2x6SXdmTjM2RVBUWQo1Q0wzdDVURTdnaWlRR1JPbEhsdVNQNlJkVjl5VmpqYUZvRHdnNVdpNXNLY1JyaDAxZVM2QzRST1NXRzQ3ekM0CjhKeDcvZS8wYW8rbGdvMTFDRUZMN0ZjR1NxQVZGVXkxQTNwNHhLc3hBc2NldTF4ZE8rcU9PTHNXZnlCcUY3R20KYXAwS0xVS0JtYWg3c2pyUVpqVW1EK1Z5Nk5BTjdHcnNRbHB2ZVhkQU10OURRTHROSUVFbEVRYkFoNGNwT0pqbAplUGswanFaNGdxWEFib0NmbDFWVXY3eS9LblNQQmZLYzZoV1RpZ0NlU2JQaS9UVHlleXpvaHF4ckdTSmoxNkE3CnZaUHJHaVNWSWhFbHdOV25BaExZYmgvY05FY2FMVThDczV2aHhxUVVzcktYc09ORTEzMjNPM2pCbENmUDNDSzcKZVRGQ3E5ZXJjaXV4REtiVmVZYTVBb0lCQVFEeXAwNTN2Wm9oU1dvbHZLS242VEx0VXI2L0RwbG5lVG9EL2JwRQpCbFJqOGxCWFNMZW5RRm53VmtkTUZnRE9JNjFJT1ZSYi9Pb0UyWHVoMVlGeTV0SytBZUo0dG9qTFozQ01LZVp6ClJRZk93dnE2SnMweVhiNzlua2dHckRlR3QzclhBUVk5T003eDRPNDJJWGorS0hveTQ4WThkMUNwWHdJUlRPNmoKYWt3V1pzYnNNOUhIYzBGQ2FVaDBJL0ZNbCtZeGx3OUg5QjFaR2ZFeWE2bk9ZTkFsdkVveHVxc0ZISXE2TTJ6Ygpvc2doQi9tdGZsNVFLUVM5MHhpWUg0bUpYS3Z0RUxQN0wxd2F0YysyZkp6T0RtbXhRNmFWN0dHejhCZy8yeXk1CmJ3U2U4ODZCb20wUEFYWUZDVktIVmxxblZjZ1RyUDAwRkY1bEVqNlVRR1duTVVGM0FvSUJBUURHcStUenRKQ00KK1I4aWVmTjNBci9ibzhjWnYzRzRUZE9xc2V3TVFDODZIMGpYTkFCWDhoOHducHM1NVZXdnB6dnN1alkwRElsZworcWRUblNwbHhOQjRoMm9jeWFZRlJEQUI2SFcwZ3h2V2RteUVtZFo5cWIvT1EyNFlpeGhKc2Z0VTcrTGFVOTY2CjVuaFU1cHBnL1RoeE12YUFhVEtIK0ZDVmQyUlFHMDlJcWhxY2haQWp4KzQyMHVTVHQ0M2FxTnpnekU3WGRwdEIKQ2IzbnpPTlVVRTdRNnRnYStmNmdBM1pNNEJVYXVuL09UZlFaSE5mMjd2dHhERU1sZi8vSzYzZVQwU0QxVG92NwpLRTJLTVFlOCtkQUJKdStYdGU4WjRIMGJWT2N5RGtPcUp1eFFleWlrTHgrcDRJelhlR3prMkhzL1RWZEZhUHptCm1QZ1RWdVhJSjU1bEFvSUJBUURLU2RYdGx0L21QaGpDcXZhQ3VyTWRDKzArdzhINWRDTjhia3FaS1JtelZLL0wKaERDdXVzUC95ejJXM1lVQVZOZkJyU0Y1cW1mQmNUbFRHZlhYdnp3UzhPbEhMd3p0WFNlRGdlNi9TOTROYlppdQpGV2pkUXkvVXFONXN5YWRrcEpOQXFIYjJGT1RZMm1aY05CMTA3SE9xOXg0dERGN3ZRK2dxV2hOYm9tTWFEY1pwCjVXMU9NL1JFbEJhMTA3ejIyRzhzQ2ozUUExOXdCMk4vWUNmb2grY2VmbER3RWlrK2txUElSTlRNYVhFanNFMWkKYUVYaDE4QS9LN3VHSGt4L2VnVk9GYTJsaXI3aStZelhHaDF5M3FzWC8wamlGWFVDRi9kdlRKMkZYVnJoMUdqawp3Mjdkb3A4cjViQ3FhTUFjWGpQMHl6TXU5b2dYQkZXdEs5NVN1b3BUQW9JQkFRQ2FZUXlDUzdkZnRGMzdQbVJ3CkFGVHg2ZXhYRjZyWW1yRjJITmZlRlNvZHNoMjZESkNQeG5keUltMWdxZExSc2RRZytmb1FyQVU4dE1tOWNZMTIKazEranFTWk54R3diLzRhR2xRcVNBS2RyR1k3dDQxVUhSUmJrd3dVVVVWSElRbU9ZOXVlQzNGVmhTcUlLNXo3agpTeUhHNU9Fam01dEdpVENsVktkQWtGZ2xrUGtvMDZqVUJSSVl5L3dPeFQyWHdrd1E2dklCQUF0WW1LZFhMcUVmCjdWK3hmQ3Y5bW5IQUNiQ3R3QnJtTURJTU1Bc1VVSk9KTU45MlV4OENUdHFINWoxL1FheW9zaWFZUGhNeTVUS3MKS0RyNENqZDMza28wQTN0ejk1L0lCOG1RdUxvOU45YTI3bDllZEQwOVdqalVBMXlTTGhrNHVJSjg5alVmWWhFZwpYWmo1QW9JQkFRQ3ZRdjBNUXc4dUs5UEVHbnA3Yk5MTkViYkdlWWVPR3NEVms3WjI3SGlEOVhOUXd3WEppZWcvCkROOEZjZkdpS2k2VVlyTytRZysxV25IYkhCVVRQYjhHN0p3Yjc5Q0NnaXNjaU1SV2dlVThwYzJObDBBWndGNisKZDFZZnJJUmxIWkgvemZoeXY2MFN4TGMyQVBUUUxJczZ2RVNuV3IzMm1VazJKNkhtek5pMTNla2dxL05iODNESgp5T0RXKzJCVnRPUGRvVC80RHMwcXR2Qm5NQkQwVHA2cEdpRGNOdStXeTFvaTJ5Y2JmaFBFcjRSR1l4OVFYYnY3ClEyNjdueDFpeGtLaWRFVklkQ3pOZ2dWVkNUc1dHWEFCWnN6Qi9ONGROVFl1eHhibTdOM0hOYmcrK0JlYXR3L2oKd3BuQVpzcXQzWUlxOGxCS3Juei9KbTRsMzIvRDFST0QKLS0tLS1FTkQgUFJJVkFURSBLRVktLS0tLQo=

keycloak:
  images:
    # keycloak images with coredge theme for login
    keycloak: coredgeio/kg-keycloak:19.0.3-8
  ingressClass: *compassIngressClass
  # note endpoint to login on this master keycloak instance is disabled by
  # default for security concerns. However it is being used internally by
  # controller to interact with keycloak API
  adminCredentials:
    # keycloak master username
    username: admin
    # keycloak master password
    password: admin@kg

```