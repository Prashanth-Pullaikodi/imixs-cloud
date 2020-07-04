# Imixs-Cloud - Registry

Docker images are available on docker registries. Most public docker images are available on [Docker Hub](https://hub.docker.com/). In the _Imixs-Cloud_  you can also setup your own private docker registry.
A private registry can be used to push locally build docker images to be used in the cloud infrastructure. Images can be pulled and started as services without the need to build the images from a Docker file.


## Habor

The _Imixs-Cloud_ already includes a configuration to run the registry [Habor](https://goharbor.io/).
_Habor_ is a secure, performant, scalable, and available cloud native repository for Kubernetes. It can be installed useing heml.


## Installation

Habor consists of several services. To make it easy to install Habor the right way you can use `helm`. If you have not yet installed helm, follow the install guide [here](../tools/helm/README.md)

### Add the harbor helm repository

First add the Helm repository for Harbor

	$ helm repo add harbor https://helm.goharbor.io

Now you can install Harbor using the corresponding chart. 


### Install Harbor 

The Harbor Helm chart comes with a lot of parameters which can be applied during installation using the `--set` parameter. See the [Habor Helm Installer](https://github.com/goharbor/harbor-helm) for more information.

The following command installs harbor into the _Imixs-Cloud_. 
	
	$ kubectl create namespace harbor
	$ helm install registry harbor/harbor --set persistence.enabled=false\
	  --namespace harbor\
	  --set expose.type=nodePort --set expose.tls.enabled=true\
	  --set externalURL=https://{MASTER-NODE}:30003\
	  --set expose.tls.commonName={MASTER-NODE}

After a few seconds you can access harbor from your web browser via https:

	https://{MASTER-NODE}:30003

### Using Ingresss / Traefik for public Internet Domain

To make things easier it is recommended to use traefik in combination with Let's Encrypt to  access harbor via a public Internet domain. (see also the [section ingress](./INGRESS.md). To start Harbor with ingress you can use the following install command:

	$ helm install registry harbor/harbor --set persistence.enabled=false\
	    -n harbor --namespace harbor\
	    --set expose.ingress.hosts.core={YOUR-DOMAIN-NAME} \
	    --set externalURL=https://{YOUR-DOMAIN-NAME} \
	    --set expose.tls.enabled=false\
	    --set notary.enabled=false

replace the `{MASTER-NODE}` with the DNS name of your master node. The option ``--set expose.tls.enabled=false`` disables the internal tls support.

### The Web UI

Now you can access the Harbor Web UI form your defined Internet Domain or IP address.

	https://{YOUR-DOMAIN-NAME}
	
<img src="./images/harbor.png" />

	
For the first login the default User/Password is:

	admin/Harbor12345

You can change the admin password and create additonal users. 
	
### Disable Scanners

The harbor scanners are useful to scan docker images for vulnerability. But these services also generates a lot of CPU load. If you want to start Harbor with a minimum of features you can disable the scanners on startup:


	$ helm install registry harbor/harbor --set persistence.enabled=false\
	    -n harbor --namespace harbor\
	    --set expose.ingress.hosts.core={YOUR-DOMAIN-NAME} \
	    --set externalURL=https://{YOUR-DOMAIN-NAME} \
	    --set expose.tls.enabled=false\
	    --set notary.enabled=false \
	    --set trivy.enabled=false\
	    --set clair.enabled=false\
	    --set chartmuseum.enabled=false


	
### Uninstall Harbor	

To uninstall/delete the registry deployment:

	$ helm uninstall registry --namespace harbor

	


## How to grant a Docker Client

After you setup the harbor registry you can upload custom Docker images to be used by services running in the Imixs-Cloud. 

To  be allowed to push/pull images from your new private docker registry you first need to login Docker with the userid and password from the Harbor web ui:

	$ sudo docker login -u admin {YOUR-DOMAIN-NAME}

**Note:** In case you run Harbor with ingress and traefik, there is no need to deal with the TLS certificate because traefik provides you with a Let's Encrypt certificate. See the section below how to deal with a self-signed certificate.


### How to grant a Worker Node

To allow your worker nodes in your Kubernetes Cluster to access the registry, you need to repeat the login procedure for the local Docker Client on each worker node too! Just login to the shell of each worker node and run:	
	
	node-1$ sudo docker login -u admin {YOUR-DOMAIN-NAME}

### Working with a Self-Signed Certificate

In case you run Harbor with a self-signed certificate with the option ``--set expose.tls.enabled=true``  than  you need to download the certificate from Harbor first to be able to login your docker demon. The certificate need to be copied into the docker certs.d directory of your local client and the docker service must be restarted once. You can download the Harbor certificate from the Habor web frontend from your web browser or via command line :

	$ wget -O ca.crt --no-check-certificate https://{MASTER-NODE}:30003/api/v2.0/systeminfo/getcert

replace *{MASTER-NODE}* with your cluster master node name.

Next create a new directly in your local docker/certs.d directory with the name and port number of your registry and copy the certificate:

	$ sudo mkdir -p /etc/docker/certs.d/{MASTER-NODE}:30003
	$ sudo mv ca.crt /etc/docker/certs.d/{MASTER-NODE}:30003/ca.crt
	$ sudo service docker restart
	
Now you can login the docker demon into your registry using the the self-signed certificate from Harbor:

	$ sudo docker login -u admin {MASTER-NODE}:30003

	

## Push a local docker image

To push a local docker image into the registry you first need to tag the image with the repository uri

	$ docker tag SOURCE_IMAGE[:TAG] {YOUR-DOMAIN-NAME}/library/IMAGE[:TAG]

**Note:** '/library/' is the project library name defined in Harbor!

next you can push the image:


	$ docker push {YOUR-DOMAIN-NAME}/library/IMAGE[:TAG]	
	
	
