# Kubernetes environment

This environment is a transplant of [the Docker environment](https://github.com/stevenwilkin/docker) onto DigitalOcean Managed Kubernetes.

Please refer to the existing `README`s for details about the symmetric encryption key and dumping the production MongoDB database etc.

The `kubernetes` branch of the `msri.nextgen` and `msri.locomotive` should be checked out.


## Prerequisites

### DigitalOcean

Within the DigitalOcean control panel the following will need to be created:

* MySQL instance
* Redis instance
* Container Registry
* Kubernetes Cluster
* API access token

### DigitalOcean CLI

[Install the `doctl` command](https://github.com/digitalocean/doctl#installing-doctl) and enter the API access token when prompted:

	doctl auth init

### Kubernetes CLI

[Install the `kubectl` command](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and configure it control our Kubernetes cluster:

	doctl kubernetes cluster kubeconfig save <CLUSTER NAME>

### Docker

Docker must be installed and running in order to build and push container images.


## Import MySQL database

The current production MySQL is version *5.7* but the managed instance is *8.0* so a couple of workarounds are required before the database can be imported.

### Enable tables without primary keys

One of the join tables doesn't have a primary key so add the following line to the top of the SQL dump to allow it to be imported:

	SET sql_require_primary_key=0

### Enable native password authentication

The library used by the Rails application doesn't support the newer style of authentication used by the managed instance:

	mysql -u <USER> -p<PASSWORD> -h <HOST> -P <PORT> -D <DATABASE> -e "ALTER USER '<USER>'@'%' IDENTIFIED WITH mysql_native_password BY '<PASSWORD>'"

### Perform the import

	mysql -u <USER> -p<PASSWORD> -h <HOST> -P <PORT> -D <DATABASE> < /path/to/msri_nextgen.sql


## Container images

### Authentication

To allow access to the container registry from the local environment and from Kubernetes run the following:

	doctl registry login
	doctl registry kubernetes-manifest | kubectl apply -f -

### Build the images

	for image in nextgen locomotive proxy solr; do
		pushd $image
			build -t $image:latest .
		popd
	done

### Tag the images

	for image in nextgen locomotive proxy solr; do
		docker tag $image:latest registry.digitalocean.com/nextgen/$image:latest
	done

### Push to the registry

	for image in nextgen locomotive proxy solr; do
		docker push registry.digitalocean.com/nextgen/$image:latest
	done


## Provision the Kubernetes cluster

	kubectl apply -f ./nextgen.yaml -f ./locomotive.yaml -f ./proxy.yaml

Monitor progress with this:

	kubectl get pods --watch

When all the pods are running the external IP of system can be determined with this command:

	kubectl get service proxy


## Reindex Solr

	kubectl apply -f ./solr_reindex.yaml

Progress of the job can be monitored with:

	kubectl get job solr-reindex --watch

When the job has finished it can be removed with this:

	kubectl delete -f ./solr_reindex.yaml


## Import MongoDB database

Find the name of the Mongo pod:

	kubectl get pods

Copy an archive of the production database to the Mongo pod and restore from it:

	kubectl cp /path/to/locomotiveapp_production.tar <MONGO>:locomotiveapp_production.tar
	kubectl exec -it <MONGO> bash
	tar xf locomotiveapp_production.tar
	mongorestore --db locomotiveapp_dev --drop locomotiveapp_production


## Locomotive configuration

Locomotive can't be configured to run from a bare IP address so follow the existing configuration process but append
`.xip.io` to the external IP address of the cluster and use this as the domain name.
