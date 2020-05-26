


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



<!-- USAGE EXAMPLES -->
## Usage

Use this space to show useful examples of how a project can be used. Additional screenshots, code examples and demos work well in this space. You may also link to more resources.

_For more examples, please refer to the [Documentation](https://example.com)_



<!-- ROADMAP -->
## Roadmap

See the [open issues](https://github.com/othneildrew/Best-README-Template/issues) for a list of proposed features (and known issues).



<!-- CONTRIBUTING -->
## Contributing

Contributions are what make the open source community such an amazing place to be learn, inspire, and create. Any contributions you make are **greatly appreciated**.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request



<!-- LICENSE -->
## License

Distributed under the MIT License. See `LICENSE` for more information.



<!-- CONTACT -->
## Contact

Your Name - [@your_twitter](https://twitter.com/your_username) - email@example.com

Project Link: [https://github.com/your_username/repo_name](https://github.com/your_username/repo_name)



<!-- ACKNOWLEDGEMENTS -->
## Acknowledgements
* [GitHub Emoji Cheat Sheet](https://www.webpagefx.com/tools/emoji-cheat-sheet)
* [Img Shields](https://shields.io)
* [Choose an Open Source License](https://choosealicense.com)
* [GitHub Pages](https://pages.github.com)
* [Animate.css](https://daneden.github.io/animate.css)
* [Loaders.css](https://connoratherton.com/loaders)
* [Slick Carousel](https://kenwheeler.github.io/slick)
* [Smooth Scroll](https://github.com/cferdinandi/smooth-scroll)
* [Sticky Kit](http://leafo.net/sticky-kit)
* [JVectorMap](http://jvectormap.com)
* [Font Awesome](https://fontawesome.com)





<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/othneildrew/Best-README-Template.svg?style=flat-square
[contributors-url]: https://github.com/othneildrew/Best-README-Template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/othneildrew/Best-README-Template.svg?style=flat-square
[forks-url]: https://github.com/othneildrew/Best-README-Template/network/members
[stars-shield]: https://img.shields.io/github/stars/othneildrew/Best-README-Template.svg?style=flat-square
[stars-url]: https://github.com/othneildrew/Best-README-Template/stargazers
[issues-shield]: https://img.shields.io/github/issues/othneildrew/Best-README-Template.svg?style=flat-square
[issues-url]: https://github.com/othneildrew/Best-README-Template/issues
[license-shield]: https://img.shields.io/github/license/othneildrew/Best-README-Template.svg?style=flat-square
[license-url]: https://github.com/othneildrew/Best-README-Template/blob/master/LICENSE.txt
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/in/othneildrew
[product-screenshot]: images/screenshot.png
