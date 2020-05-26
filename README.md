


<!-- TABLE OF CONTENTS -->
## Table of Contents

* [About the Project](#about-the-project)
  * [Built With](#built-with)
* [Getting Started](#getting-started)
  * [Prerequisites](#prerequisites)
  * [Installation](#installation)
* [Usage](#usage)
* [Roadmap](#roadmap)
* [Contributing](#contributing)
* [License](#license)
* [Contact](#contact)
* [Acknowledgements](#acknowledgements)



<!-- ABOUT THE PROJECT -->
## About The Project

How to setup pgadmin4 end to end ssl in azure kubernetes cluster behind azure application ingress controller AGIC.

### Built With

* [pgadmin4](https://www.pgadmin.org/download/pgadmin-4-container/)
* [aks](https://docs.microsoft.com/en-us/azure/aks/intro-kubernetes)
* [AGIC](https://github.com/Azure/application-gateway-kubernetes-ingress)



<!-- GETTING STARTED -->
## Getting Started

1.Setup PV and PVC
2.Setup secrets
3.Apply root certificate to AGIC
4.Setup deployment
5.Setup service
6.Setup ingress for AGIC
7.Verify

### Prerequisites

Assuming you have below up and running
AKS
AGIC

### Installation

1. Setup pv and PVC

a. create secret
```sh
STORAGE_KEY=$(az storage account keys list --resource-group RGname --account-name storageaccountname --query "[0].value" -o tsv)
```  
b.  Create NS
```sh
kubectl create ns pgadmin
```
c.  Create azure storage secret
```sh
kubectl create secret generic pgadmin-azure-secret --from-literal=azurestorageaccountname=storageaccountname --from-literal=azurestorageaccountkey=$STORAGE_KEY -n pgadmin
```   
d.  Create PersistentVolume

pv-azurefile-mountoptions-pgadmin-var.yaml

```sh
Note you need ReadWriteMany
```

```sh
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-azurefile-pgadmin-var
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: pgadmin-azure-secret
    shareName: pgadmin-var-data
    readOnly: false
  mountOptions:
    - dir_mode=0777
    - file_mode=0777
    - uid=5050
    - mfsymlinks
    - nobrl
 ```
====================================

Note share should exist inside your storage,if not create it

```sh
kubectl create -f pv-azurefile-mountoptions-pgadmin-var.yaml
```
```sh
kubectl get pv
```

NAME                    CAPACITY ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM       STORAGECLASS     REASON             AGE

pv-azurefile-pgadmin-var 20Gi    RWX            Retain           Bound    pgadmin/pvc-azurefile-pgadmin-var               17h

e. Create PersistentVolumeClaim

pvc-azurefile-static-pgadmin-var.yaml

```sh
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: pvc-azurefile-pgadmin-var
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 20Gi
  storageClassName: ""
  volumeName: pv-azurefile-pgadmin-var
```

```sh
kubectl create -f pvc-azurefile-static-pgadmin-var.yaml -n pgadmin
```
```sh
kubectl get pvc -n pgadmin
```
NAME                        STATUS   VOLUME                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE

pvc-azurefile-pgadmin-var   Bound    pv-azurefile-pgadmin-var   20Gi       RWX                           17h




2. setup secrets

Note for safe side we use private registry and we dont want pgadmin to pull latest image if container is restared.
We pull pgadmin4 and push in private registry.

a.Image pull secret
```sh
kubectl --namespace pgadmin create secret docker-registry pgadmin-pull-secret --docker-server=xx.xx.xx --docker-username=abcd --docker-password=Hxxxxxxbinxxx8O --docker-email=niteen_kole@xxxxxx.ca
```
b. pgadmin ssl certificate secret.
```sh
kubectl create secret generic pgadmin-ssl-key --from-file=/certs/server.key  --from-file=/certs/server.cert -n pgadmin
```
Note your server.cert should include all server,root and intermidiate 

example.
```sh
cat server.cert

-----BEGIN CERTIFICATE-----
MIIHTTCCBjWgAwIBAgIRAMHZwo2wytiNAAAAAFDzaT0wDQYJKoZIhvcNAQELBQAw
gboxCzAJBgNVBAYTAlVTMRYwFAYDVQQKEw1FbnRydXN0LCBJbmMuMSgwJgYDVQQL
....
....
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIEPjCCAyagAwIBAgIESlOMKDANBgkqhkiG9w0BAQsFADCBvjELMAkGA1UEBhMC
VVMxFjAUBgNVBAoTDUVudHJ1c3QsIEluYy4xKDAmBgNVBAsTH1NlZSB3d3cuZW50
....
....
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIFDjCCA/agAwIBAgIMDulMwwAAAABR03eFMA0GCSqGSIb3DQEBCwUAMIG+MQsw
CQYDVQQGEwJVUzEWMBQGA1UEChMNRW50cnVzdCwgSW5jLjEoMCYGA1UECxMfU2Vl
....
....
-----END CERTIFICATE-----
```

c. TLS secret for ingress.
```sh
kubectl create secret tls pgadmin-ingress-portal-tls -n pgadmin --key="server.key" --cert="server.cert"
```
3. Apply root certificate to AGIC

a. Point to subscription where your AGW is running

b. Apply root certificate

```sh
az network application-gateway root-cert create --cert-file root.cer --gateway-name APPgwname --name root-cert1 --resource-group AGWRG-Name
```
4. Setup deployment

pgadmin-deployment-tls-file.yaml

```sh
kind: Deployment
apiVersion: apps/v1
metadata:
  name: pgadmin-prod
  namespace: pgadmin
  labels:
    k8s-app: pgadmin-prod
    application-name: pgadmin-prod
    version-no: "01"
    owner: niteen_kole
    env: production
    release-no: "01"
    tier: "01"
    customer-facing: "yes"
    app-role: web
    project-id: design
  annotations:
    deployment.kubernetes.io/revision: '1'
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: pgadmin-prod
      application-name: pgadmin-prod
      version-no: "01"
      owner: niteen_kole
      env: production
      release-no: "01"
      tier: "01"
      customer-facing: "yes"
      app-role: web
      project-id: design
  template:
    metadata:
      name: pgadmin-prod
      creationTimestamp:
      labels:
        k8s-app: pgadmin-prod
        application-name: pgadmin-prod
        version-no: "01"
        owner: niteen_kole
        env: production
        release-no: "01"
        tier: "01"
        customer-facing: "yes"
        app-role: web
        project-id: design
    spec:
      containers:
      - name: pgadmin-prod
        image: your-registry/pgadmin4:latest
        volumeMounts:
        - name: pgadmin-var
          mountPath: /var/lib/pgadmin
        - name: pgadmin-cert
          mountPath: /certs
        env:
        - name: PGADMIN_DEFAULT_EMAIL
          value: "niteen_kole@xxxxx.ca"
        - name: PGADMIN_DEFAULT_PASSWORD
          value: "XXXXXXX"
        - name: PGADMIN_ENABLE_TLS
          value: "True"
        - name: PGADMIN_CONFIG_ENHANCED_COOKIE_PROTECTION
          value: "False"
        resources: {}
        imagePullPolicy: Always
      imagePullSecrets:
      - name: pgadmin-pull-secret
      schedulerName: default-scheduler
      volumes:
      - name: pgadmin-var
        persistentVolumeClaim:
          claimName: pvc-azurefile-pgadmin-var
      - name: pgadmin-cert
        secret:
          secretName: pgadmin-ssl-key

```
```sh
kubectl create -f pgadmin-deployment-tls-file.yaml -n pgadmin
```

5.Setup service

pgadmin-prod-svc.yaml

```sh
apiVersion: v1
kind: Service
metadata:
  name: pgadmin-svc
  namespace: pgadmin
spec:
  ports:
  - port: 443
    targetPort: 443
    protocol: TCP
  type: ClusterIP
  selector:
    k8s-app: pgadmin-prod

```
```sh
kubectl create -f pgadmin-prod-svc.yaml -n pgadmin
```

6.Setup ingress for AGIC

pgadmin-ingress.yaml

```sh
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: pgadmin-ingress
  namespace: pgadmin
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
    appgw.ingress.kubernetes.io/backend-protocol: "https"
    appgw.ingress.kubernetes.io/backend-hostname: "pg.xxxxx.com"
    appgw.ingress.kubernetes.io/appgw-trusted-root-certificate: "root-cert1"
spec:
  tls:
  - hosts:
    - pg.xxxxx.com
    secretName: pgadmin-ingress-portal-tls
  rules:
  - host: pg.xxxxx.com
    http:
      paths:
      - path:
        backend:
          serviceName: pgadmin-svc
          servicePort: 443

```
```sh
kubectl create -f pgadmin-ingress.yaml -n pgadmin
```
7.Verify
```sh
kubectl get all -n pgadmin

NAME                                READY   STATUS    RESTARTS   AGE
pod/pgadmin-prod-7fd8944c75-z5vjk   1/1     Running   0          3h2m

NAME                     TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)         AGE
service/pgadmin-lb-pvt   LoadBalancer   xxx.xx.xxx.xx   xxx.xx.xxx.xx   443:32450/TCP   3h1m
service/pgadmin-svc      ClusterIP      xxx.xx.xxx.xx    <none>           443/TCP         19h

NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pgadmin-prod   1/1     1            1           3h2m

```

```sh
kubectl get ingress -n pgadmin

NAME              HOSTS                 ADDRESS         PORTS     AGE
pgadmin-ingress   pg.xxxxx.com   xxx.xx.xxx.xx   80, 443   165m

```

<!-- CONTACT -->
## Contact

Project Link: [https://github.com/niteenkole/pgadmin4](https://github.com/niteenkole/pgadmin4)

<!-- ACKNOWLEDGEMENTS -->
## Acknowledgements
* [application-gateway-kubernetes-ingress](https://github.com/Azure/application-gateway-kubernetes-ingress)
* [pgadmin4](https://www.pgadmin.org/download/pgadmin-4-container/)
