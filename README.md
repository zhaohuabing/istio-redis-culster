This tutorial shows how to use Istio to enable Envoy Redis Cluster support, including data sharding, read/write splitting, and traffic mirroring, all the magics are done by Istio and Envoy proxy, without any awareness at the client side.

# Deploy Istio 

If you're using a newer Istio version where the following PR has already been incorporated, you can just follow the Istio install guide and you're good to go.

Implement REPLACE operation for EnvoyFilter patch  https://github.com/istio/istio/pull/27426/

At the time of writing, the latest Istio version is 1.7.3, in which the EnvoyFilter REPLACE operation is not supported yet, so I build a customized pilot image to enable it. We need to use zhaohuabing/pilot:1.7.3-enable-ef-replace instead of the default pilot image to make this demo work.

```bash
$ cd istio-1.7.3/bin
$ ./istioctl install --set components.pilot.hub=zhaohuabing --set components.pilot.tag=1.7.3-enable-ef-replace
```

# Deploy Redis Cluster

We will install the demo in the 'redis' namespace, please create one if you don't have this namespace in your cluster.

```bash
$ kubectl create ns redis
namespace/redis created
```

Deploy the statefulset and configmap.

```bash
$ kubectl apply -f k8s/redis-cluster.yaml -n redis
configmap/redis-cluster created
statefulset.apps/redis-cluster created
service/redis-cluster created
```

## Verify the Deployment

Check that the Redis nodes are up and running:

```bash
$ kubectl get pod -n redis
NAME              READY   STATUS    RESTARTS   AGE
redis-cluster-0   2/2     Running   0          4m25s
redis-cluster-1   2/2     Running   0          3m56s
redis-cluster-2   2/2     Running   0          3m28s
redis-cluster-3   2/2     Running   0          2m58s
redis-cluster-4   2/2     Running   0          2m27s
redis-cluster-5   2/2     Running   0          117s
```

## Create the Redis Cluster

```bash
$ kubectl exec -it redis-cluster-0 -n redis -- redis-cli --cluster create --cluster-replicas 1 $(kubectl get pods -l app=redis-cluster -o jsonpath='{range.items[*]}{.status.podIP}:6379 ' -n redis)
Defaulting container name to redis.
Use 'kubectl describe pod/redis-cluster-0 -n redis' to see all of the containers in this pod.
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 172.16.0.72:6379 to 172.16.0.138:6379
Adding replica 172.16.0.201:6379 to 172.16.1.52:6379
Adding replica 172.16.0.139:6379 to 172.16.1.53:6379
M: 8fdc7aa28a6217b049a2265b87bff9723f202af0 172.16.0.138:6379
   slots:[0-5460] (5461 slots) master
M: 4dd6c1fecbbe4527e7d0de61b655e8b74b411e4c 172.16.1.52:6379
   slots:[5461-10922] (5462 slots) master
M: 0b86a0fbe76cdd4b48434b616b759936ca99d71c 172.16.1.53:6379
   slots:[10923-16383] (5461 slots) master
S: 94b139d247e9274b553c82fbbc6897bfd6d7f693 172.16.0.139:6379
   replicates 0b86a0fbe76cdd4b48434b616b759936ca99d71c
S: e293d25881c3cf6db86034cd9c26a1af29bc585a 172.16.0.72:6379
   replicates 8fdc7aa28a6217b049a2265b87bff9723f202af0
S: ab897de0eca1376558e006c5b0a49f5004252eb6 172.16.0.201:6379
   replicates 4dd6c1fecbbe4527e7d0de61b655e8b74b411e4c
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
.
>>> Performing Cluster Check (using node 172.16.0.138:6379)
M: 8fdc7aa28a6217b049a2265b87bff9723f202af0 172.16.0.138:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 4dd6c1fecbbe4527e7d0de61b655e8b74b411e4c 172.16.1.52:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: 94b139d247e9274b553c82fbbc6897bfd6d7f693 172.16.0.139:6379
   slots: (0 slots) slave
   replicates 0b86a0fbe76cdd4b48434b616b759936ca99d71c
M: 0b86a0fbe76cdd4b48434b616b759936ca99d71c 172.16.1.53:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: ab897de0eca1376558e006c5b0a49f5004252eb6 172.16.0.201:6379
   slots: (0 slots) slave
   replicates 4dd6c1fecbbe4527e7d0de61b655e8b74b411e4c
S: e293d25881c3cf6db86034cd9c26a1af29bc585a 172.16.0.72:6379
   slots: (0 slots) slave
   replicates 8fdc7aa28a6217b049a2265b87bff9723f202af0
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

## Verify Cluster Deployment

Check the cluster details and the role of each member.

```bash
$ kubectl exec -it redis-cluster-0 -n redis -- redis-cli cluster info 
Defaulting container name to redis.
Use 'kubectl describe pod/redis-cluster-0 -n redis' to see all of the containers in this pod.
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:206
cluster_stats_messages_pong_sent:210
cluster_stats_messages_sent:416
cluster_stats_messages_ping_received:205
cluster_stats_messages_pong_received:206
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:416
```

## Some useful Redis commands

* redis-cli --scan  Getting a list of keys
* redis-cli monitor
* redis-cli cluster info
* redis-cli cluster nodes

# Use Istio to enable Envoy Redis Cluster support

This is where the real magic happens. We make the Istio and Envoy do all the dirty work, so the client is not aware of the topo of the Redis cluster behind Envoy proxy.

## Apply the EnvoyFilter to create a Cluster of "envoy.clusters.redis" type

```bash
$ kubectl apply -f istio/envoyfilter-custom-redis-cluster.yaml
envoyfilter.networking.istio.io/custom-redis-cluster created
```

## Update EnvoyFilter CRD to enable REPLACE operation

```bash
$ kubectl apply -f istio/envoyfilter-crd.yaml 
customresourcedefinition.apiextensions.k8s.io/envoyfilters.networking.istio.io configured
```

## Apply the EnvoyFilter to replace the default tcp proxy with our redis proxy

```bash
$ sed -i .bak "s/\${REDIS_VIP}/`kubectl get svc redis-cluster -n redis -o=jsonpath='{.spec.clusterIP}'`/" istio/envoyfilter-redis-proxy.yaml
$ kubectl apply -f istio/envoyfilter-redis-proxy.yaml
envoyfilter.networking.istio.io/add-redis-proxy created
```

## Deploy a test client

```bash
$ kubectl apply -f k8s/redis-client.yaml -n redis
deployment.apps/redis-client created
```

## Verify the Envoy Redis proxy

With the configuration pushed from Istio in the form of EnvoyFilter, the Envoy Redis proxy should be able to discover the topology of the backend Redis Cluster automatically and distribute the keys in the client requests to the correct server accordingly.

From the output of the previous Redis cluster create command, we can figure out the topology of this Redis Cluster. The cluster has three shards, and each shard has one master node and one slave node (replica).

```text
Shard[0] Master[0]  redis-cluster-0 172.16.0.138:6379   replica[0]  redis-cluster-4 172.16.0.72:6379  -> Slots 0 - 5460 
Shard[1] Master[1]  redis-cluster-1 172.16.1.52:6379    replica[1]  redis-cluster-5 172.16.0.201:6379 -> Slots 5461 - 10922
Shard[2] Master[2]  redis-cluster-2 172.16.1.53:6379    replica[2]  redis-cluster-3 172.16.0.139:6379 -> Slots 10923 - 16383
```

Please note that the exact topology of the Redis Cluster and key distribution among shards in the following steps may be different when you try to deploy this demo in your cluster, but the basic idea is the same.

Send some requests with different keys to the Rdeis Cluster:

```bash
$ kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -n redis -- redis-cli -h redis-cluster
Defaulting container name to redis-client.
Use 'kubectl describe pod/redis-client-56f767d9d9-n22m9 -n redis' to see all of the containers in this pod.
redis-cluster:6379> set a a
OK
redis-cluster:6379> set b b
OK
redis-cluster:6379> set c c
OK
redis-cluster:6379> set d d
OK
redis-cluster:6379> set e e
OK
redis-cluster:6379> set f f
OK
redis-cluster:6379> set g g
OK
redis-cluster:6379> set h h
OK
```

So far so good, it looks fine from the client side. Let's check the server side.

Shard[0], in which the master is redis-cluster-0 and the slave is redis-cluster-4
```bash
$ kubectl exec redis-cluster-0 -c redis -n redis -- redis-cli --scan
b
f
$ kubectl exec redis-cluster-4 -c redis -n redis -- redis-cli --scan
f
b
```

Shard[1], in which the master is redis-cluster-1 and the slave is redis-cluster-5
```bash
$ kubectl exec redis-cluster-1 -c redis -n redis -- redis-cli --scan
c
g
$ kubectl exec redis-cluster-5 -c redis -n redis -- redis-cli --scan
g
c
```

Shard[2], in which the master is redis-cluster-12 and the slave is redis-cluster-3
```bash
$ kubectl exec redis-cluster-2 -c redis -n redis -- redis-cli --scan
a
e
d
h
$ kubectl exec redis-cluster-3 -c redis -n redis -- redis-cli --scan
h
e
d
a
```

We can see that the keys have been distributed to the three shards in the Redis Cluster. It's automatically done by the Envoy Redis Proxy without any awareness of the cluster topology at the client side. From the client's point of view, it's just talking to a single Redis node.

## Verify Redis Cluster read policy

We have set the read policy to 'REPLICA' in the EnvoyFilter, which means all the 'get' requests should only be sent to the slave node. Let's check it:

Use the following commands to verify the traffic mirroing policy:

Client:

```bash
$ kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster

redis-cluster:6379> get b
"b"
redis-cluster:6379> get b
"b"
redis-cluster:6379> get b
"b"
redis-cluster:6379> set b bb
OK
redis-cluster:6379> get b
"bb"
redis-cluster:6379> 
```

Master node:

```bash
$ kubectl exec redis-cluster-0 -c redis -n redis -- redis-cli monitor
```

Slave node:

```bash
$ kubectl exec redis-cluster-4 -c redis -n redis -- redis-cli monitor
```

![](img/redis-cluster-read-policy.png)

Note that there's only one slave node in each shard in this demo. You can deploy more slave nodes to share the client traffic if there're heavy read loads.

## Enable traffic mirroring

Create a single node redis as the mirror server:

```bash
$ kubectl apply -f k8s/redis-mirror.yaml -n redis 
deployment.apps/redis-mirror created
service/redis-mirror created
```

Apply the envofilter to enable traffic mirroring at the Envoy proxy.

```bash
$ sed -i .bak "s/\${REDIS_VIP}/`kubectl get svc redis-cluster -n redis -o=jsonpath='{.spec.clusterIP}'`/" istio/envoyfilter-redis-proxy-with-mirror.yaml
$ kubectl apply -f istio/envoyfilter-redis-proxy-with-mirror.yaml
envoyfilter.networking.istio.io/add-redis-proxy configured
```
## Verify traffic mirroring

Use the following commands to verify the traffic mirroing policy:

Client:

```bash
$ kubectl exec -it `kubectl get pod -l app=redis-client -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-client -n redis -- redis-cli -h redis-cluster
redis-cluster:6379> get b
"b"
redis-cluster:6379> get b
"b"
redis-cluster:6379> get b
"b"
redis-cluster:6379> set b bb
OK
redis-cluster:6379> get b
"bb"
redis-cluster:6379> set b bbb
OK
redis-cluster:6379> get b
"bbb"
redis-cluster:6379> get b
"bbb"
```

Master node:

```bash
$ kubectl exec redis-cluster-0 -c redis -n redis -- redis-cli monitor
```

Slave node:

```bash
$ kubectl exec redis-cluster-4 -c redis -n redis -- redis-cli monitor
```

Mirror node:

```bash
$ kubectl exec -it `kubectl get pod -l app=redis-mirror -n redis -o jsonpath="{.items[0].metadata.name}"` -c redis-mirror -n redis -- redis-cli monitor
```
From the output of these comands, we can see that all the 'set' commands have also been sent to the mirror node.
![](img/redis-cluster-mirror-policy.png)

# References

* https://rancher.com/blog/2019/deploying-redis-cluster
* https://medium.com/@fr33m0nk/migrating-to-redis-cluster-using-envoy-93a87ae79dc3
* [Implement REPLACE operation for EnvoyFilter patch](https://github.com/istio/istio/pull/27426/)
