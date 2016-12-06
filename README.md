Note: This repo is an old prototype that I had built and it has subsequently moved to the glusterfs community and is now being maintained there. You can access it at https://github.com/gluster/gluster-kubernetes


# Running GlusterFS inside Kubernetes

I thought it would be interesting to explore a converged scenario where Kubernetes was running the containers for the compute runtimes as well as the containers for the storage runtimes. This is useful in that it allows you to have a single infrastructure for the full stack. I thought I'd write up what I did to get it working given that its a new scenario for Kubernetes and also uses some aspects of Kubernetes that aren't documented in much detail.

## Some Background

For the storage platform example, we'll be using GlusterFS, which is a Clustered Open Source Scale Out Storage platform. Given that it is clustered, GlusterFS is comprised of many nodes/servers which are each attached to local storage. In GlusterFS terms, these servers are called peers. Each peer runs an identical set of processes. The peer processes can be codified within a single Docker image and that image runs on every node that you would like to have as a GlusterFS peer. Each peer requires access to a certain amount of local storage, which in GlusterFS terms is called the "brick". The "cluster" is formed when the GlusterFS peers are able to be made aware of each other via a process called "probing". The GlusterFS term for the cluster is called a Trusted Storage Pool. Once the Trusted Storage Pool is established, GlusterFS Volumes can be created, by aggregating the local storage made available by each peer, and then making those GlusterFS volumes available to the Compute Pods running in Kubernetes using the GlusterFS or NFS Kubernetes volume plugins.

## Architectural Considerations

Given that we have containerized the GlusterFS peer processes into an image, we need to carefully design the Kubernetes Pods to make sure the containers are launched and orchestrated in the correct manner.

The first factor to consider is that each of the GlusterFS Peers will need access to local storage that can be used as the GlusterFS Peer's Brick. It will likely be the case that only certain nodes in the Kubernetes Cluster will have access to local storage so this requires a way of ensuring that the pods only land on suitable Kubernetes Nodes. We achieve this by having a specific pod for each peer. Each of these pods uses a Kubernetes NodeSelector label to ensure that it is always scheduled on a specific Host.

The second factor to consider is networking. This solution works with both Flannel and Docker Host Networking. This example uses Docker Host Networking to keep things a little cleaner and also because at Red Hat we are interested in exploring different methods of providing QoS for Storage Traffic and one approach could involve physically separating the network traffic by putting the compute traffic on eth0 and the storage traffic on eth1. We achieve this by using hostNetwork: true in the Pod and setting the following in the /etc/kubernetes/kubelet on all nodes in the cluster:
```
KUBELET_ARGS="--host-network-sources=*"
```

The third factor is properly identifying the brick and mounting it into the correct location of the GlusterFS Peer container. We achieve this using the HostPath Kubernetes Volume Plugin which allows us to mount local storage into a container from the Host. We assume that the local direct attached storage has already been prepared and made available at a given path on the Host. For our examples, that path is /mnt/brick1.

The fourth factor is that the GlusterFS images need to run in privileged mode. We enable this by using privileged: true in the Pod and setting the following in the /etc/kubernetes/config on all nodes in the cluster:
```
KUBE_ALLOW_PRIV="--allow-privileged=true"
```


## Implementation

To demonstrate this solution, we're going to create a 2 Node GlusterFS Cluster. This means that there is a single GlusterFS peer running on each node.

We begin by submitting 2 unique pods (one for each peer), which can be seen here. Due to the NodeSelector Labels in the pods the Gluster-1 Pod will be scheduled on a Kubernetes Node that has the label name=worker-1 and the Gluster=2 Pod will be scheduled on name=worker-2.

```
[root@master ~]# kubectl create -f gluster-1.yaml
[root@master ~]# kubectl create -f gluster-2.yaml
[root@master ~]# kubectl get pods -o wide
NAME        READY     STATUS    RESTARTS   AGE       NODE
gluster-1   1/1       Running   0          1h        worker-1
gluster-2   1/1       Running   0          1h        worker-2
```

Now that the peers are running, we can shell into the worker-1 node and run "docker ps" to see the container ID for the peer image that was launched. 

```
[root@worker-1 ~]# docker ps
CONTAINER ID        IMAGE                                  COMMAND             CREATED             STATUS              PORTS               NAMES
56830d96667c        gluster/rhgs-3.1.0                     "/usr/sbin/init"    About an hour ago   Up About an hour                        k8s_glusterfs.f6ef5764_gluster-1_default_5541a4c1-56b0-11e5-97e6-000c294da916_2b367cbe   
cabec0a71468        gcr.io/google_containers/pause:0.8.0   "/pause"            About an hour ago   Up About an hour                        k8s_POD.7be6d81d_gluster-1_default_5541a4c1-56b0-11e5-97e6-000c294da916_64d330b7  
```

We can then Docker Exec into the container so that we can probe the other peer pod's container and form the trusted storage pool. Note that the current image has a bug which requires you to reset the Gluster Peer UUID.
```
[root@worker-1 ~]# docker exec -it 56830d96667c bash
[root@worker-1 /]# gluster system uuid reset  
[root@worker-1 /]# gluster peer probe 192.168.58.22
[root@worker-1 /]# gluster peer status
Number of Peers: 1

Hostname: worker-2.fed
Uuid: 1845075d-6ecc-4db6-8aef-7d82e6f3ae7e
State: Peer in Cluster (Connected)
```
Now that the Trusted Storage Pool is established between the worker-1 peer and the worker-2 peer, while staying inside the container, we can create a GlusterFS volume
```
[root@worker-1 /]# gluster volume create newvol replica 2 192.168.58.21:/mnt/brick1/newvol 192.168.58.22:/mnt/brick1/newvol
[root@worker-1 /]# gluster volume start newvol
[root@worker-1 /]# gluster volume status           
Status of volume: newvol
Gluster process                             TCP Port  RDMA Port  Online  Pid
------------------------------------------------------------------------------
Brick 192.168.58.21:/mnt/brick1/newvol      49152     0          Y       322  
Brick 192.168.58.22:/mnt/brick1/newvol      49152     0          Y       335  
NFS Server on localhost                     N/A       N/A        N       N/A  
Self-heal Daemon on localhost               N/A       N/A        Y       360  
NFS Server on worker-2.fed                  N/A       N/A        N       N/A  
Self-heal Daemon on worker-2.fed            N/A       N/A        Y       371  
 
Task Status of Volume newvol
------------------------------------------------------------------------------
There are no active volume tasks
```
Now that the Volume is created, we can then make it accessible to any of the compute containers running in Kubernetes as demonstrated by the [GlusterFS Kubernetes Volume Plugin Documentation](https://github.com/kubernetes/kubernetes/tree/master/examples/glusterfs)
