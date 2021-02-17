---
layout: post
title: Elastic ECK arm64 raspberry-pi cluster for fun and non-profit
date: 2021-02-16
categories: linux elasticsearch k8s rpi
---

**Disclaimer**: This POC was build for learning and experimenting purposes, don't put sensible or important data to work on it, might not be stable. 

Only Elasticsearch images are available for **ECK arm64**, this means, no other service/application from **ECK** will work like Kibana or apm-server. 

### Idea: 

Create a kubernetes [(k3s)](https://k3s.io) cluster on raspberry-pi that can manage the new Elastic **ECK** (Elasticsearch Cloud on Kubernetes) for learning purposes.
We want to be able to deploy 3 elasticsarch nodes on that cluster, expose it with LoadBalancer, and use an external **StorageClass** service where the data will be stored.

### Stuff:

* 3 raspberry-pi model 4B with 4Gb of RAM
* 1 raspberry-pi model 3B 
* 1 USB external disk
* 4 32Gb SD cards
* 1 5 port switch

You'll need at least 3 raspberry-pi version 4b with 4Gb of RAM, otherwise you wont have enough memory to run it correctly. 

### Blocks

* Prerequisites
* Install k3s and related apps
* Build and deploy ECK
* Run ECK on k3s 
* ECK advanced mode

## Prerequisites

#### Install arm64 system

You need an arm64 linux version, I've choosed the raspbian beta version [from here](https://www.raspberrypi.org/forums/viewtopic.php?f=117&t=275370).
Write the image to your sd cards, enable ssh and boot them.
Update the system and do all the typical things you do on a pi.

#### Install k3sup and k3s

Once installed and updated, you must install K8s or K3s to it.

I've used [Arkade](https://github.com/alexellis/arkade) and [k3sup](https://github.com/alexellis/k3sup) on one of the nodes to do this process.

```
$ curl -SLfs https://dl.get-arkade.dev | sudo sh

$ arkade get 

```
## Install k3s and related apps

#### Deploy k3s cluster with 3 master nodes
**ECK** needs to run with 3 kubernetes master nodes, you'll need to install your cluster based on this configuration.

On your first master node where k3sup is installed we'll do the following to initialize the cluster

```
k3sup install --ip 192.168.21.179 --user pi --k3s-extra-args "--cluster-init --no-deploy traefik" --cluster --k3s-version v1.19.1+k3s1

```
Note the `--cluster-init` option to initialize the cluster and `--cluster` to make it master node.
Once the first k3s node is running we'll install the 2 other ones.

```
k3sup join --ip 192.168.21.180 --user pi --k3s-extra-args "--no-deploy traefik" --server-user pi --server-ip 192.168.21.179 --server --k3s-version v1.19.1+k3s1

k3sup join --ip 192.168.21.181 --user pi --k3s-extra-args "--no-deploy-traefik" --server-user pi --server-ip 192.168.21.179 --server --k3s-version v1.19.1+k3s1
```

Check the cluster availability

```
pi@pi4:~ $ kubectl get nodes
NAME     STATUS   ROLES         AGE   VERSION
node01   Ready    etcd,master   1h    v1.19.1+k3s1
node02   Ready    etcd,master   1h    v1.19.1+k3s1
pi4      Ready    etcd,master   1h    v1.19.1+k3s1
```


## Build and deploy ECK

You need docker and a registry to be able to build the needed files.
Follow the instructions to install docker from the official [page](https://docs.docker.com/engine/install/debian/). Choose **arm64** option.

I've installed docker on the first master node and created a registry on github. Keep in mind that k3s is working apart from docker using it's own container service, I've just installed docker to not cross-compile on other host.

Building **ECK** needs docker `buildx` option, wich is only available on experimental version.

Enable experimental on your docker.

Add the following to your `.docker/config.json` file and restart docker

```
{
 "experimental": "enabled"
}
```

Once you've docker installed, download the **ECK** code from github [elastic/cloud-on-k8s](https://github.com/elastic/cloud-on-k8s) and follow the [dev-setup](https://github.com/elastic/cloud-on-k8s/blob/master/dev-setup.md) to install all the required packages.

Create a registry `.env` file inside cloud-on-k8s directory

```
pi@pi4:~/github/cloud-on-k8s $ cat .registry.env
REGISTRY = ghcr.io/netmanito
REGISTRY_NAMESPACE = eck-dev
E2E_REGISTRY_NAMESPACE = eck-dev
```

Keep in mind that you'll need to login on your registry to be able to push and pull the packages, follow this [guide](https://docs.github.com/en/packages/guides/pushing-and-pulling-docker-images) to configure your registry on github if you don't have access to a registry repository.

Once everything is up we need to build and deploy the code.
You can follow the github issue we're the **arm64** is beeing worked [here](https://github.com/elastic/cloud-on-k8s/issues/3504).

You don't need to modify any file at all. Should be as simple as putting your `.registry.env` file and then run

```
$ make docker-build
$ make deploy
```

Once done, you should have a packet created in your github regsitry and your custom `all-in-one.yaml` file created in `cloud-on-k8s/config` directory.
Also the build image should be in your docker images on localhost.

```
pi@pi4:~ $ docker images
REPOSITORY                                  TAG                       IMAGE ID       CREATED       SIZE
ghcr.io/netmanito/eck-dev/eck-operator-pi   1.5.0-SNAPSHOT-2b2bce84   766d4e44bbd6   2 hours ago   185MB
```

The operator will also be deployed in your kubernetes cluster.
Check if the operator is running.

```
pi@pi4:~ $ kc get pods -n elastic-system
NAME                 READY   STATUS    RESTARTS   AGE
elastic-operator-0   1/1     Running   2          137m
```
Yes!! we're half done!! we now have the elastic-operator runing on our rpi-cluster. This will let us deploy some elasticsearch nodes on our k3s-rpi-cluster.

## Run ECK on k3s

Check if it works.

Follow the elastic guide [https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-deploy-elasticsearch.html](https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-deploy-elasticsearch.html) and deploy your first node.

##### NOTE: Don't assign too much RAM to the elastic node or the k3s cluster will crash.

Create a file with the 1 node config.

```
pi@pi4:~ $ cat quickstart-es.yml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.10.2
  nodeSets:
  - name: default
    count: 1
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms512m -Xmx512m
```

Deploy it.

```
$ kubectl apply -f quickstart-es.yml
```
Check if it runs. It will take a few minutes to be deployed, so be patient.

```
$ kc get elasticsearch
NAME          HEALTH    NODES     VERSION   PHASE         AGE
quickstart    green     1         7.10.2     Ready         1m
```

Try to make a request to the elastic node following the **Log in the cluster** section below.

Perfect! we've our elastic node in k3s arm64, amazing job !!

If everything is working, we can go ahead with the project.

## ECK advanced mode

Destroy the node with `kubectl delete -f quicstart-es.yml` and continue to create our infrastructure.
The operator will still be running.

```
pi@pi4:~ $ kc get pods -n elastic-system
NAME                 READY   STATUS    RESTARTS   AGE
elastic-operator-0   1/1     Running   2          154m
```

If you remove the operator and want to install it later again, just run

```
$ kubectl apply -f /path_to/cloud-on-k8s/config/all-in-one.yaml
```

### Add StorageClass based on NFS

Lets go a bit further now. ECK works with [VolumeClaimTemplates](https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-volume-claim-templates.html) and StorageClass services. I wanted to learn how all this works so I tried to create a service storage that could be used by ECK.

NFS storage is the easiest way to create, but we have to configure it as a StorageClass, and that's a bit more tricky. 
I've followed this [guide](https://levelup.gitconnected.com/how-to-use-nfs-in-kubernetes-cluster-storage-class-ed1179a83817) to create it on my cluster and have to say thanks for the info.

Elasticsearch does not performe well with NFS volumes, this aproach is only to learn create a **storageClass** on Kubernetes that **ECK** could use. We can live with poor performance for the moment.

Clone the repo [https://github.com/fabiofernandesx/k8s-volumes](https://github.com/fabiofernandesx/k8s-volumes) on your master pi node and cd to `k8s-volumes/yml/StorageClass`.

You'll need to make some changes in order to match the names ECK expects to find.

These are the files used on our deployment.

```
 service-account.yml
 cluster-role.yml
 cluster-role-binding.yml
 role.yml
 role-binding.yml
 storage-class.yml
 provisioner-deploy.yml
 volume-claim.yml
```

On those files, replace any `pv-demo` namespace with `default`. 

```
sed -i 's/pv-demo/default/g' *yml
```

On `provisioner-deploy.yml` replace the hostname and NFS address acordingly to your **nfs-server**.

On `storage-class.yml`, change name to `standard` and provisioner with your nfs node name like `node03.lan/nfs`. Add also the following 

```
volumeBindingMode: WaitForFirstConsumer
```

On `volume-claim.yml`, change the file like this diff

```
diff --git a/yml/StorageClass/volume-claim.yml b/yml/StorageClass/volume-claim.yml
index 28f03e4..975fd34 100644
--- a/yml/StorageClass/volume-claim.yml
+++ b/yml/StorageClass/volume-claim.yml
@@ -1,12 +1,12 @@
 apiVersion: v1
 kind: PersistentVolumeClaim
 metadata:
-  name: ssd-nfs-pvc-1
-  namespace: pv-demo
+  name: elasticsearch-data
+  namespace: default
 spec:
-  storageClassName: ssd-nfs-storage
+  storageClassName: standard
   accessModes:
     - ReadWriteMany
   resources:
     requests:
-      storage: 10Mi
+      storage: 1Gi

```

When you're ready, deploy this files starting from `service-account.yml` and last one `volume-claim.yml`

Once deployed, check if it's running

```
pi@pi4:~/github/k8s-volumes/yml/StorageClass $ kc get pvc -n default
NAME                                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data                           Pending                                                                        standard       105m
```

Hurray!! we now have a StorageClass that will be auto discovered by ECK service.

Lets try now to create a 3 node cluster wich will use the external storage we've configured.

```
pi@pi4:~ $ cat quickstart-es-nfs.yml
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.10.2
  nodeSets:
  - name: default
    count: 3
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
    config:
      node.master: true
      node.data: true
      node.ingest: true
      node.store.allow_mmap: false
    podTemplate:
      spec:
        containers:
        - name: elasticsearch
          env:
          - name: ES_JAVA_OPTS
            value: -Xms512m -Xmx512m

pi@pi4:~ $ kubectl apply -f quickstart-es-nfs.yml        
```  

Let's check if it works. It takes a while to deploy the 3 nodes, rpi's are not as fast as we'd like so, you can go get a cup of tea or whatever you want. Once you're back look if everything is up.

```
pi@pi4:~/github/k8s-volumes/yml/StorageClass $ kc get pv -n default
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM                                                STORAGECLASS   REASON   AGE                                                                              2d16h
pvc-09b7dfdf-dae2-4740-b796-ad4e7fea9a5a   1Gi        RWO            Delete           Bound       default/elasticsearch-data-quickstart-es-default-1   standard                103m
pvc-30eaa7d8-36ae-4632-8df6-ac0fe407ffc8   1Gi        RWO            Delete           Bound       default/elasticsearch-data-quickstart-es-default-0   standard                103m
pvc-ad5546c5-d500-4416-9a5b-1cf94cd55803   1Gi        RWO            Delete           Bound       default/elasticsearch-data-quickstart-es-default-2   standard                103m

pi@pi4:~/github/k8s-volumes/yml/StorageClass $ kc get pvc -n default
NAME                                         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data                           Pending                                                                        standard       105m
elasticsearch-data-quickstart-es-default-0   Bound     pvc-30eaa7d8-36ae-4632-8df6-ac0fe407ffc8   1Gi        RWO            standard       103m
elasticsearch-data-quickstart-es-default-1   Bound     pvc-09b7dfdf-dae2-4740-b796-ad4e7fea9a5a   1Gi        RWO            standard       103m
elasticsearch-data-quickstart-es-default-2   Bound     pvc-ad5546c5-d500-4416-9a5b-1cf94cd55803   1Gi        RWO            standard       103m
```

Looks nice, let's check now the NFS server and see if data is there.

```
pi@pi4:~/github/k8s-volumes/yml/StorageClass $ ssh -tt node03 "du -h -d 1 /extfs/kubedata"
60K     /extfs/kubedata/default-elasticsearch-data-quickstart-es-default-0-pvc-30eaa7d8-36ae-4632-8df6-ac0fe407ffc8
200K    /extfs/kubedata/default-elasticsearch-data-quickstart-es-default-1-pvc-09b7dfdf-dae2-4740-b796-ad4e7fea9a5a
200K    /extfs/kubedata/default-elasticsearch-data-quickstart-es-default-2-pvc-ad5546c5-d500-4416-9a5b-1cf94cd55803
464K    /extfs/kubedata
Connection to node03 closed.
```
Great!! we've a ECK service with 3 elasticsearch nodes using external storage. Remember that NFS is not the best option for Elastic, we just want to learn how to deploy a whole infrastructure with extra options.

### NodeSelector to force ECK pods run on pi4 k3s nodes

We want to label the k3s nodes in order to use NodeSelector. We'll put the LoadBalancer in a new worker node in the future. 

We'll create a label `env` with **elastic** as value on all 3 pi4 nodes, we don't want the elasticpods to run on other nodes than them.

```
$ kubectl label node pi4 env=elastic --overwrite
$ kubectl label node node01 env=elastic --overwrite
$ kubectl label node node02 env=elastic --overwrite
```

After that, we'll add some rules in our `quickstart-es.yml` file, in the `podTemplate` section.

```
  podTemplate:
      spec:
        affinity:
          nodeAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              nodeSelectorTerms:
              - matchExpressions:
                - key: env
                  operator: In
                  values:
                  - elastic
```
You'll have to apply the file again if you want to enable these changes.

### Add a loadBalancer to the ECK service

[//]: #  (Install a loadBalancer, I've used Metallab and followed this [guide](https://www.jurgenallewijn.nl/k3s-kubernetes-dashboard-load-balancer/).)

Add a LoadBalancer in the primary spec section in the quickstart-es.yml file and restart.

```
spec:
  version: 7.10.2
  http:
    service:
      spec:
        type: LoadBalancer
```
Check if it works

```
pi@pi4:~/eck-arm64 $ kc get svc                                                                                                              NAME                      TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE                                                kubernetes                ClusterIP      10.43.0.1       <none>          443/TCP          2d                                                 quickstart-es-default     ClusterIP      None            <none>          9200/TCP         5m42s                                              quickstart-es-http        LoadBalancer   10.43.137.105   192.168.0.181   9200:30945/TCP   5m48s                                              quickstart-es-transport   ClusterIP      None            <none>          9300/TCP         5m48s                                              

pi@pi4:~/eck-arm64 $ kubectl get service quickstart-es-http
NAME                 TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
quickstart-es-http   LoadBalancer   10.43.137.105   192.168.0.181   9200:30945/TCP   7m33s
```
The service will balance between all the IP's of the nodes in the k3s cluster.

### Log in the cluster

You can see that one Pod is in the process of being started:

```
$ kubectl get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=quickstart' 
NAME                      READY   STATUS    RESTARTS   AGE
quickstart-es-default-0   1/1     Running   0          79s
```

#### Get the credentials.

A default user named elastic is automatically created with the password stored in a Kubernetes secret:

```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

#### Request the Elasticsearch endpoint.

If you haven't enabled the loadBalancer, you must port-forward the elastic service to access over it.

Run

```
kubectl port-forward service/quickstart-es-http 9200
```

Then request localhost:

```
$ curl -u "elastic:$PASSWORD" -k "https://localhost:9200" 
{
  "name": "quickstart-es-default-0",
  "cluster_name": "quickstart",
  "cluster_uuid": "60XfH9-xSwqTOqArdPrcgQ",
  "version": {
    "number": "7.10.2",
    "build_flavor": "default",
    "build_type": "docker",
    "build_hash": "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date": "2021-01-13T04:42:47.157277Z",
    "build_snapshot": false,
    "lucene_version": "8.7.0",
    "minimum_wire_compatibility_version": "6.8.0",
    "minimum_index_compatibility_version": "6.0.0-beta1"
  },
  "tagline": "You Know, for Search"
}
```

#### Access the logs for that Pod:

```
kubectl logs -f quickstart-es-default-0
```


## TODO 

* New k3s worker nodes for loadbalance only on those nodes
* secure the service with TLS and certs
* Add metrics to see cluster performance


## References: 

[https://blog.alexellis.io/multi-master-ha-kubernetes-in-5-minutes/](https://blog.alexellis.io/multi-master-ha-kubernetes-in-5-minutes/)

[https://github.com/alexellis/k3sup](https://github.com/alexellis/k3sup)

[https://github.com/alexellis/arkade](https://github.com/alexellis/arkade)

[https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-deploy-elasticsearch.html](https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-deploy-elasticsearch.html)

[https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-storage-recommendations.html](https://www.elastic.co/guide/en/cloud-on-k8s/master/k8s-storage-recommendations.html)

[https://kubernetes.io/docs/concepts/storage/storage-classes/](https://kubernetes.io/docs/concepts/storage/storage-classes/)

[https://github.com/elastic/cloud-on-k8s/blob/1.3/config/samples/elasticsearch/elasticsearch.yaml](https://github.com/elastic/cloud-on-k8s/blob/1.3/config/samples/elasticsearch/elasticsearch.yaml)

[https://levelup.gitconnected.com/how-to-use-nfs-in-kubernetes-cluster-storage-class-ed1179a83817](https://levelup.gitconnected.com/how-to-use-nfs-in-kubernetes-cluster-storage-class-ed1179a83817)


