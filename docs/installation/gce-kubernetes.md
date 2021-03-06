---
title: GCE Kubernetes
---

## Setup

- If you don’t already have a Google Account (Gmail or Google Apps), you must [create one](https://accounts.google.com/SignUp). Then, sign-in to Google Cloud Platform console ([console.cloud.google.com](http://console.cloud.google.com/)) and create a new project:


- Store your project ID into a variable as many commands below use it:

    ```
    export PROJECT_ID="your-project-id"
    ```

- Next, [enable billing](https://console.cloud.google.com/billing) in the Cloud Console in order to use Google Cloud resources and [enable the Container Engine API](https://console.cloud.google.com/project/_/kubernetes/list).

- Install [Docker](https://docs.docker.com/engine/installation/), and [Google Cloud SDK](https://cloud.google.com/sdk/).

- Finally, after Google Cloud SDK installs, run the following command to install `kubectl`:

    ```
    gcloud components install kubectl
    ```

- Choose a [Google Cloud Project zone](https://cloud.google.com/compute/docs/regions-zones/regions-zones) to run your service. We will be using `us-west1-a`. This is configured on the command line via:

    ```
    gcloud config set compute/zone us-west1-a
    ```

## Create and format a persistent data disk for postgres

- Create a persistent disk. (min. 1 GB) with a name `pg-data-disk`.

    ```
    gcloud compute disks create pg-data-disk --size 1GB
    ```

- The disk created is un formatted and needs to be formatted. To do that, we need to create a temporarily compute instance.

    ```
    gcloud compute instances create pg-disk-formatter
    ```

- Wait for the instance to get created. Once done, attach the disk to that instance.

    ```
    gcloud compute instances attach-disk pg-disk-formatter --disk pg-data-disk
    ```

- SSH into the instance.

    ```
    gcloud compute ssh "pg-disk-formatter"
    ```

- In the terminal, use the `ls` command to list the disks that are attached to your instance and find the disk that you want to format and mount

    ```
    ls /dev/disk/by-id
    ```
    
    ```
    google-example-instance       scsi-0Google_PersistentDisk_example-instance
    google-example-instance-part1 scsi-0Google_PersistentDisk_example-instance-part1
    google-[DISK_NAME]            scsi-0Google_PersistentDisk_[DISK_NAME]
    ```

    where `[DISK_NAME]` is the name of the persistent disk that you attached to the instance.
    
    The disk ID usually includes the name of your persistent disk with a `google-` prefix or a `scsi-0Google_PersistentDisk_` prefix. You can use either ID to specify your disk, but this example uses the ID with the `google-` prefix


- Format the disk with a single `ext4` filesystem using the `mkfs` tool. This command deletes all data from the specified disk.

    ```
    sudo mkfs.ext4 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/disk/by-id/google-[DISK_NAME]
    ```

- Now, the disk is formatted and ready. Detach the disk from the instance.

    ```
    gcloud compute instances detach-disk pg-disk-formatter --disk pg-data-disk
    ```

_You can delete the instance if your not planning to use it for anything else. But make sure the disk `pg-data-disk` is not deleted._

Repeat the same procedure and create another disk named `nfs-data-disk`.

## Create your Kubernetes Cluster

- Create a cluster via the `gcloud` command line tool:

    ```
    gcloud container clusters create opev-cluster --image-type=container_vm
    ```

- Get the credentials for `kubectl` to use.

    ```
    gcloud container clusters get-credentials opev-cluster
    ```

## Pre deployment steps 
- A domain name (Eg. eventyay.com, google.com, hello.io). - A free domain can be registered at http://www.freenom.com .
- Reserve a static external IP address 
	
	```bash
	gcloud compute addresses create testip --region us-west1
	```
	
	The response would be similar to 
	
	```
	address: 123.123.123.123
	creationTimestamp: '2017-05-16T05:26:24.894-07:00'
	description: ''
	id: '1234556789'
	kind: compute#address
	name: test
	selfLink: https://www.googleapis.com/compute/v1/projects/eventyay/global/addresses/test
	status: RESERVED
	```	
	
	Note down the address. (In this case `123.123.123.123`). We'll call this **External IP Address One**.
- Add the **External IP Address One** as an `A` record to your domain's DNS Zone.
- Add the **External IP Address One** to `kubernetes/yamls/nginx/service.yml` for the parameter `loadBalancerIP`.
- Add your domain name to `kubernetes/yamls/web/ingress-notls.yml` & `kubernetes/yamls/web/ingress-tls.yml`. (replace `eventyay.com`)
- Add your email ID to `kubernetes/yamls/lego/configmap.yml` for the parameter `lego.email`.
- Get your cluster internal IP range by running

	```
	kubectl get services kubernetes
	```
	
	The response would be similar to 
	
	```
	NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
	kubernetes   10.3.240.1   <none>        443/TCP   115d
	```
	
	Note down the cluster IP. (In this case `10.3.240.1`). Pick an IP in the same range. (For this case `10.3.1.1` to `10.3.255.255`). We'll call the IP address you picked as **Internal IP Address One**.
- Add the **Internal IP Address One** to `kubernetes/yamls/persistent-store/nfs-pv.yml` for the property `server`
- Add the **Internal IP Address One** to `kubernetes/yamls/persistent-store/nfs-server-service.yml` for the property `clusterIP`

## Deploy our pods, services and deployments

- From the project directory, use the provided deploy script to deploy our application from the defined configuration files that are in the `kubernetes` directory.

    ```
    ./kubernetes/deploy.sh
    ```

- The Kubernetes master creates the load balancer and related Compute Engine forwarding rules, target pools, and firewall rules to make the service fully accessible from outside of Google Cloud Platform. 
- Wait for a few minutes for all the containers to be created and the SSL Certificates to be generated and loaded. 
- You can track the progress using the Web GUI as mentioned below.
- Once deployed, your instance will be accessible at your domain name.
    

## Other handy commands

- Delete all created pods, services and deployments

    ```
    kubectl delete -R -f kubernetes/yamls/
    ```
    
-  Access The Kubernetes dashboard Web GUI

    Run the following command to start a proxy.
    
    ```
    kubectl proxy
    ```
    
    and Goto [http://localhost:8001/ui](http://localhost:8001/ui)

- Deleting the cluster
    ```
    gcloud container clusters delete opev-cluster
    ```
