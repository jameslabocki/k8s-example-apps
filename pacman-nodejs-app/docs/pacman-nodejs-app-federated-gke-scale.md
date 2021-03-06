# Scale Pac-Man Node.js App On Additional Federated Kubernetes Cluster

This guide will walk you through creating two Kubernetes clusters and use a federation control plane
to deploy the Pac-Man Node.js application onto each cluster. Then, add an additional Kubernetes cluster to
the federation and scale application onto it. This is all using a single cloud provider (GCP).

## High-Level Architecture

Below is a diagram demonstrating the architecture of the game across the federated kubernetes cluster after all the steps are completed.

![Pac-Man Game Architecture](images/Kubernetes-Federation-Game.png)

## Prerequisites

#### Clone the repository

Follow these steps to [clone the repository](../README.md#clone-this-repository).

#### Create the Pac-Man Container Image

Follow these steps to [create the Pac-Man application container image](../README.md#create-application-container-image).

#### Set up Google Cloud SDK and Push Container Image

Follow these steps to
[set up the Google Cloud SDK and push the Pac-Man container image to your Google Cloud Container Registry](../README.md#kubernetes-components).

#### Store the GCP Project Name

```
export GCP_PROJECT=$(gcloud config list --format='value(core.project)')
```

#### Create the Federated Kubernetes Clusters

Follow these steps:

1. [Create 3 GKE Kubernetes clusters in 3 regions i.e. west, central, and east](kubernetes-cluster-gke-federation.md#create-the-kubernetes-clusters)
2. [Verify the clusters](kubernetes-cluster-gke-federation.md#verify-the-clusters)
3. [Store the GCP Project Name](kubernetes-cluster-gke-federation.md#store-the-gcp-project-name)
4. [Create Google DNS managed zone for cluster](kubernetes-cluster-gke-federation.md#cluster-dns-managed-zone)
5. [Download and install kubefed and kubectl](kubernetes-cluster-gke-federation.md#download-and-install-kubefed-and-kubectl)
6. Using [kubefed](https://kubernetes.io/docs/admin/federation/kubefed/) follow these steps:
    1. [Initialize](kubernetes-cluster-gke-federation.md#initialize-the-federated-control-plane) the federated control plane
    2. [Set kubectl to use the federation context](kubernetes-cluster-gke-federation.md#use-federation-context)
    3. Join only the [west](kubernetes-cluster-gke-federation.md#gce-us-west1-1) and [central](kubernetes-cluster-gke-federation.md#gce-us-central1-1) regions
    4. [Verify the clusters in the federation](kubernetes-cluster-gke-federation.md#verify)

#### Export the zones your clusters are in

For example since we're using us-west and us-central:

```
export GCE_ZONES="west central"
```

## Create MongoDB Resources

#### Create MongoDB Persistent Volume Claims

Now using the default storage class in each cluster, we'll create the PVC:

```
for i in ${GCE_ZONES}; do \
    kubectl --context=gke_${GCP_PROJECT}_us-${i}1-b_gce-us-${i}1 \
    create -f persistentvolumeclaim/mongo-pvc.yaml; \
done
```

Verify the PVCs are bound in each cluster:

```
for i in ${GCE_ZONES}; do \
    kubectl --context=gke_${GCP_PROJECT}_us-${i}1-b_gce-us-${i}1 \
    get pvc mongo-storage; \
done
```

#### Create MongoDB Service

This component creates the necessary mongo federation DNS entries for each cluster. The application uses `mongo` as
the host it connects to instead of `localhost`. Using `mongo` in each application will resolve to the local `mongo` instance
that's closest to the application in the cluster.

```
kubectl create -f services/mongo-service.yaml
```

Wait until the mongo service has all the external IP addresses listed:

```
kubectl get svc mongo -o wide --watch
```

#### Create MongoDB Kubernetes Deployment

Now create the MongoDB Deployment that will use the `mongo-storage` persistent volume claim to mount the
directory that is to contain the MongoDB database files. In addition, we will pass the `--replSet rs0` parameter
to `mongod` in order to create a MongoDB replica set.

```
kubectl create -f deployments/mongo-deployment-rs.yaml
```

Scale the mongo deployment

```
kubectl scale deploy/mongo --replicas=2
```

Wait until the mongo deployment shows 2 pods available

```
kubectl get deploy mongo -o wide --watch
```

#### Create the MongoDB Replication Set

We'll have to bootstrap the MongoDB instances to talk to each other in the replication set. For this,
we need to run the following commands on the MongoDB instance you want to designate as the primary (master). For our example,
let's use the us-west1-b instance:

```
MONGO_POD=$(kubectl --context=gke_${GCP_PROJECT}_us-west1-b_gce-us-west1 get pod \
    --selector="name=mongo" \
    --output=jsonpath='{.items..metadata.name}')
kubectl --context=gke_${GCP_PROJECT}_us-west1-b_gce-us-west1 exec -it ${MONGO_POD} -- bash
```

Once inside this pod, feel free to make sure all Mongo DNS entries for each region are resolvable. Otherwise the command to
initialize the Mongo replica set below will fail. You can try to initialize the Mongo replica set first and come back here if you have problems.
You can verify the Mongo DNS entries by installing DNS utilities such as `dig` and `nslookup` using:

```
apt-get update
apt-get -y install dnsutils
```

Then use either `dig` or `nslookup` to perform one of the following lookups for each zone:

```
dig mongo.default.federation.svc.us-west1.<DNS_ZONE_NAME> +noall +answer
nslookup mongo.default.federation.svc.us-west1.<DNS_ZONE_NAME>
```

Also check to make sure the load balancer DNS A record contains an IP address for each zone:

```
dig mongo.default.federation.svc.<DNS_ZONE_NAME> +noall +answer
nslookup mongo.default.federation.svc.<DNS_ZONE_NAME>
```

Once all regions are resolvable, launch the `mongo` CLI:

```
mongo
```

Now we'll create an initial configuration specifying each of the mongos in our replication set. In our example,
we'll use the west, central, and east instances. Make sure to replace `federation.com` with the DNS zone name you created.

```
initcfg = {
        "_id" : "rs0",
        "members" : [
                {
                        "_id" : 0,
                        "host" : "mongo.default.federation.svc.us-west1.federation.com:27017"
                },
                {
                        "_id" : 1,
                        "host" : "mongo.default.federation.svc.us-central1.federation.com:27017"
                },
        ]
}
```

Initiate the MongoDB replication set:

```
rs.initiate(initcfg)
```

Check the status until this instance shows as `PRIMARY`:

```
rs.status()
```

Once you have the other instance showing up as `SECONDARY` and this one as `PRIMARY`, you have a working MongoDB replica set that will replicate data across the cluster.

Go ahead and exit out of the Mongo CLI and out of the Pod.

## Create Pac-Man Resources

#### Create the Pac-Man Service

This component creates the necessary `pacman` federation DNS entries for each cluster. There will be A DNS entries created for each zone, region,
as well as a top level DNS A entry that will resolve to all zones for load balancing.

```
kubectl create -f services/pacman-service.yaml
```

Wait and verify the service has all the external IP addresses listed:

```
kubectl get svc pacman -o wide --watch
```

#### Create the Pac-Man Deployment

We'll need to create the Pac-Man game deployment to access the application on port 80.

```
kubectl create -f deployments/pacman-deployment-rs.yaml
```

Scale the pacman deployment

```
kubectl scale deploy/pacman --replicas=2
```

Wait until the pacman deployment shows 2 pods available

```
kubectl get deploy pacman -o wide --watch
```

Once the `pacman` service has an IP address for each replica, open up your browser and try to access it via its
DNS e.g. [http://pacman.default.federation.svc.federation.com/](http://pacman.default.federation.svc.federation.com/). Make sure to replace `federation.com` with your DNS name.

You can also see all the DNS entries that were created in your [Google DNS Managed Zone](https://console.cloud.google.com/networking/dns/zones).

## Play Pac-Man

Go ahead and play a few rounds of Pac-Man and invite your friends and colleagues by giving them your FQDN to your Pac-Man application
e.g. [http://pacman.default.federation.svc.federation.com/](http://pacman.default.federation.svc.federation.com/) (replace `federation.com` with your DNS name).

The DNS will load balance and resolve to any one of the zones in your federated kubernetes cluster. This is represented by the `Zone:`
field at the top. When you save your score, it will automatically save the zone you were playing in and display it in the High Score list.

See who can get the highest score!

## Scale Pac-Man to Additonal Kubernetes Cluster

Once you've played Pac-Man to verify your application has been properly deployed, we'll scale the application to an additional GKE Kubernetes cluster.

#### Join the east Kubernetes cluster to the federation
With the federation control plane already initialized, you just need to join one more cluster, the us-east region that we previously created and scale
the application onto it.

Using [kubefed](https://kubernetes.io/docs/admin/federation/kubefed/):
1. Join the [east](kubernetes-cluster-gke-federation.md#gce-us-east1-1) region
2. [Verify the clusters in the federation](kubernetes-cluster-gke-federation.md#verify)

#### Export the new zones to include the new cluster

```
export GCE_ZONES="west central east"
```

#### Scale the MongoDB Resources

###### Create MongoDB Persistent Volume Claims

Now using the default storage class in the cluster, we'll create the PVC:

```
kubectl --context=gke_${GCP_PROJECT}_us-east1-b_gce-us-east1 \
    create -f persistentvolumeclaim/mongo-pvc.yaml
```

Verify the PVC is bound in the cluster:

```
kubectl --context=gke_${GCP_PROJECT}_us-east1-b_gce-us-east1 \
    get pvc mongo-storage
```

###### Scale MongoDB Service

As soon as the us-east cluster was joined to the federation, the MongoDB service should scale to the new cluster and you should see an additional
IP address added.

Wait until the mongo service has all the external IP addresses listed:

```
kubectl get svc mongo -o wide --watch
```

###### Scale MongoDB Kubernetes Replica Set

Now scale the MongoDB Replica Set that will use the `mongo-storage` persistent volume claim to mount the
directory that is to contain the MongoDB database files. In addition, we will add the new MongoDB replica to the existing replica set
in order to replicate the MongoDB data set across to the new cluster.

```
kubectl scale deploy/mongo --replicas=3
```

Wait until the mongo deployment shows 3 pods available:

```
kubectl get deploy/mongo -o wide --watch
```

###### Scale the MongoDB Replication Set

We'll have to add the MongoDB instance to the existing MongoDB replica set. For this, we need to run the following command on the MongoDB
instance you designated as the primary (master). For our example, we used the us-west1-b instance so let's re-use it:

```
kubectl --context=gke_${GCP_PROJECT}_us-west1-b_gce-us-west1 exec -it ${MONGO_POD} -- bash
```

Once inside this pod, feel free to make sure the new Mongo DNS entry for the us-east region is resolvable. Otherwise the command to
add to the Mongo replica set below will fail. You can try to add to the Mongo replica set first and come back here if you have problems.
You can verify the Mongo DNS entries by installing DNS utilities such as `dig` and `nslookup` using:

```
apt-get update
apt-get -y install dnsutils
```

Then use either `dig` or `nslookup` to perform the following lookup for the us-east zone:

```
dig mongo.default.federation.svc.us-east1.<DNS_ZONE_NAME> +noall +answer
nslookup mongo.default.federation.svc.us-east1.<DNS_ZONE_NAME>
```

Also check to make sure the load balancer DNS A record contains an IP address for each zone:

```
dig mongo.default.federation.svc.<DNS_ZONE_NAME> +noall +answer
nslookup mongo.default.federation.svc.<DNS_ZONE_NAME>
```

Once all regions are resolvable, launch the `mongo` CLI:

```
mongo
```

Now we'll add the new mongo to our replication set. In our example,
we'll be adding the east instance. Make sure to replace `federation.com` with the DNS zone name you created.

Add to the MongoDB replication set:

```
rs.add('mongo.default.federation.svc.us-east1.federation.com:27017')
```

Check the status until the new instance shows as `SECONDARY`:

```
rs.status()
```

Once you have the instance showing up as `SECONDARY`, you have added a MongoDB instance to the replica set that will replicate data across to the new cluster.

Go ahead and exit out of the Mongo CLI and out of the Pod.

#### Scale Pac-Man Resources

###### Scale the Pac-Man Service

As soon as the us-east cluster was joined to the federation, the `pacman` service should scale to the new cluster and you should see an additional
IP address added.

Wait and verify the service has all the external IP addresses listed:

```
kubectl get svc pacman -o wide --watch
```

#### Scale the Pac-Man Replica Set

We'll need to scale the Pac-Man game to the new cluster.

```
kubectl scale deploy/pacman --replicas=3
```

Wait until the replica set status is ready for the new replica:

```
kubectl get deploy/pacman -o wide --watch
```

Once the new `pacman` replica set is ready, open up your browser and try to access it via its specific
DNS e.g. [http://pacman.default.federation.svc.us-east1.federation.com/](http://pacman.default.federation.svc.us-east1.federation.com/) or
using the top-level DNS [http://pacman.default.federation.svc.federation.com/](http://pacman.default.federation.svc.federation.com/).
Make sure to replace `federation.com` with your DNS name.

You can also see all the DNS entries that were created in your [Google DNS Managed Zone](https://console.cloud.google.com/networking/dns/zones).

#### Play Scale out Pac-Man

Now you should have Pac-Man scaled onto a new federated Kubernetes cluster. You can verify any previously persisted data has been replicated to the
new us-east region and continue playing Pac-Man!

## Cleanup

#### Delete Pac-Man Resources

##### Delete Pac-Man Replica Set and Service

Delete Pac-Man replica set and service. Seeing the replica set removed from the federation context may take up to a couple minutes.

```
kubectl delete deploy/pacman svc/pacman
```

#### Delete MongoDB Resources

##### Delete MongoDB Replica Set and Service

Delete MongoDB replica set and service. Seeing the replica set removed from the federation context may take up to a couple minutes.

```
kubectl delete deploy/mongo svc/mongo
```

##### Delete MongoDB Persistent Volume Claims

```
for i in ${GCE_ZONES}; do \
    kubectl --context=gke_${GCP_PROJECT}_us-${i}1-b_gce-us-${i}1 \
    delete pvc/mongo-storage; \
done
```

#### Delete DNS entries in Google Cloud DNS

Delete the `mongo` and `pacman` DNS entries that were created in your [Google DNS Managed Zone](https://console.cloud.google.com/networking/dns/zones).

#### Cleanup rest of federation cluster

[Steps to clean-up your federation cluster created using kubefed](kubernetes-cluster-gke-federation.md#cleanup).
