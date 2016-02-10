
# Rough Walkthrough
```sh
$:-> flockerctl --control-service=ec2-52-2-176-38.compute-1.amazonaws.com create -m name=redis-slave -s 10G -n 4b03576b
created dataset in configuration, manually poll state with 'flocker-volumes list' to see it show up.

$:-> flockerctl --control-service=ec2-52-2-176-38.compute-1.amazonaws.com create -m name=flocker-redis-master -s 10G -n 4b03576b
created dataset in configuration, manually poll state with 'flocker-volumes list' to see it show up.

$:-> flockerctl --control-service=ec2-52-2-176-38.compute-1.amazonaws.com list
DATASET                                SIZE     METADATA                     STATUS         SERVER
f0487599-e4d9-45d0-abcb-38b5b141f42a   10.00G   name=redis-slave   attached ✅   4b03576b (10.0.0.231)
1a6c10bd-8628-4f9a-b3e5-72b5ac11a152   10.00G   name=flocker-redis-master    attached ✅   4b03576b (10.0.0.231)
```

# Start Services
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig create -f ../install-flocker/examples/redis/redis-controller.yaml
replicationcontroller "redis-master" created

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig create -f ../install-flocker/examples/redis/redis-service.yaml
service "redis-master" created

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get po
NAME                 READY     STATUS          RESTARTS   AGE
redis-master-py3gn   0/1       ImageNotReady   0          20s

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get po
NAME                 READY     STATUS    RESTARTS   AGE
redis-master-pxvuh   1/1       Running   0          7m
```

Find the IP of the Redis Master
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  describe po | grep IP:
IP:				10.2.51.8

or

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get ep
NAME           ENDPOINTS        AGE
kubernetes     10.0.0.50:443    19h
redis-master   10.2.51.8:6379   11h
```

Verify we have the correct sized volume
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  exec redis-master-n8jop -- df -h | grep redis
/dev/xvdf       9.8G   23M  9.2G   1% /var/lib/redis
```

Inspect the container running redis.
```sh
kubectl --kubeconfig=clusters/my-k8s-cluster-meetup/kubeconfig logs -c redis-master redis-master-472qt
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.4 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[1] 26 Jan 04:10:24.682 # Server started, Redis version 2.8.4
[1] 26 Jan 04:10:24.682 * The server is now ready to accept connections on port 6379
```

Inside the host running the container you can inspect the flocker volume
```sh
ip-10-0-0-234 ~ # ls /flocker/1a6c10bd-8628-4f9a-b3e5-72b5ac11a152/
appendonly.aof  lost+found

ip-10-0-0-234 ~ # cat /flocker/1a6c10bd-8628-4f9a-b3e5-72b5ac11a152/appendonly.aof
ip-10-0-0-234 ~ #
```

# Use Redis Single Master

From KubeCTL
```sh
kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  exec redis-master-n8jop -c redis-slave -it -- /bin/bash
root@redis-master-n8jop:/var/lib/redis# redis-cli -h 10.2.51.8 rpush mylist A
root@redis-master-n8jop:/var/lib/redis# redis-cli -h 10.2.51.8 rpush mylist B
root@redis-master-n8jop:/var/lib/redis# redis-cli -h 10.2.51.8 lpush mylist first
root@redis-master-n8jop:/var/lib/redis# redis-cli -h 10.2.51.8 lpush mylist last
root@redis-master-n8jop:/var/lib/redis# redis-cli -h 10.2.51.8 lrange mylist 0 -1
1) "laste"
2) "first"
3) "A"
4) "B"
root@redis-master-n8jop:/var/lib/redis# exit
exit
$:->
```

Or from the CoreOS node running the service
```sh
ip-10-0-0-234 ~ # docker run -it --rm wallnerryan/redis sh -c 'exec redis-cli -h 10.2.51.8'
10.2.51.3:6379>
10.2.51.3:6379>
10.2.51.3:6379>
10.2.51.3:6379> rpush mylist A
(integer) 1
10.2.51.3:6379> rpush mylist B
(integer) 2
10.2.51.3:6379> lpush mylist first
(integer) 3
10.2.51.3:6379> lpush mylist last
(integer) 3
10.2.51.3:6379> lrange mylist 0 -1
1) "last"
2) "first"
3) "A"
4) "B"
10.2.51.3:6379> exit
```

Now lets check out file again.
```sh
ip-10-0-0-234 ~ # cat /flocker/1a6c10bd-8628-4f9a-b3e5-72b5ac11a152/appendonly.aof
*2
$6
SELECT
$1
0
*3
$5
rpush
$6
mylist
$1
A
*3
$5
rpush
$6
mylist
$1
B
*3
$5
lpush
$6
mylist
$5
first
```

## Start the rest of the services
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig create -f ../install-flocker/examples/redis/redis-slave-controller.yaml
replicationcontroller "redis-slave" created

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster-meetup/kubeconfig create -f ../install-flocker/examples/redis/redis-slave-service.yaml
service "redis-slave" created
```

## Watch Redis Slave SYNC
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get po
NAME                 READY     STATUS    RESTARTS   AGE
redis-master-n8jop   1/1       Running   0          26m
redis-slave-zszm7    1/1       Running   0          1m

ip-10-0-0-233 ~ # docker ps | grep redis
6e6cd7bcd9e3        wallnerryan/redis                           "/usr/bin/redis-serve"   54 seconds ago      Up 53 seconds                           k8s_redis-slave.1aaf8914_redis-slave-24rno_default_9c4dcef6-c3e3-11e5-8692-1219a9fd1aaf_b01fdc63
aa57f4c17bbd        gcr.io/google_containers/pause:0.8.0        "/pause"                 55 seconds ago      Up 53 seconds                           k8s_POD.badc061d_redis-slave-24rno_default_9c4dcef6-c3e3-11e5-8692-1219a9fd1aaf_3a4b5556
ip-10-0-0-233 ~ # docker logs 6e6cd7bcd9e3
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 2.8.4 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in stand alone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

[6] 26 Jan 05:11:38.894 # Server started, Redis version 2.8.4
[6] 26 Jan 05:11:38.894 * The server is now ready to accept connections on port 6379
[6] 26 Jan 05:11:38.894 * Connecting to MASTER 10.3.0.60:6379
[6] 26 Jan 05:11:38.894 * MASTER <-> SLAVE sync started
[6] 26 Jan 05:11:38.903 * Non blocking connect for SYNC fired the event.
[6] 26 Jan 05:11:38.904 * Master replied to PING, replication can continue...
[6] 26 Jan 05:11:38.904 * Partial resynchronization not possible (no cached master)
[6] 26 Jan 05:11:38.905 * Full resync from master: 552a27f5bb5f2284c436f278c64c16dad04c95ad:3417
[6] 26 Jan 05:11:38.937 * MASTER <-> SLAVE sync: receiving 60 bytes from master
[6] 26 Jan 05:11:38.938 * MASTER <-> SLAVE sync: Flushing old data
[6] 26 Jan 05:11:38.938 * MASTER <-> SLAVE sync: Loading DB in memory
[6] 26 Jan 05:11:38.938 * MASTER <-> SLAVE sync: Finished with success
[6] 26 Jan 05:11:38.939 * Background append only file rewriting started by pid 9
[9] 26 Jan 05:11:38.957 * SYNC append only file rewrite performed
[9] 26 Jan 05:11:38.958 * AOF rewrite: 6 MB of memory used by copy-on-write
[6] 26 Jan 05:11:38.997 * Background AOF rewrite terminated with success
[6] 26 Jan 05:11:38.997 * Parent diff successfully flushed to the rewritten AOF (0 bytes)
[6] 26 Jan 05:11:38.997 * Background AOF rewrite finished successfully

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  describe po redis-slave-ynpx4 | grep IP:
IP:				10.2.64.6

or 

$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get ep
NAME           ENDPOINTS        AGE
kubernetes     10.0.0.50:443    19h
redis-master   10.2.51.8:6379   11h
redis-slave    10.2.64.6:6379   11h
```

# Data is now on our Slave Redis Container, but also saved to persistent disk.

From KubeCTL
```sh
kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  exec redis-slave-ynpx4 -c redis-slave -it -- /bin/bash
root@redis-slave-ynpx4:/var/lib/redis# redis-cli -h 10.2.64.6 lrange mylist 0 -1
1) "last"
2) "first"
3) "A"
4) "B"
root@redis-slave-ynpx4:/var/lib/redis# exit
exit
$:->
```

Or from the CoreOS node where the redis-slave is running.
```sh
ip-10-0-0-231 ~ # docker run -it --rm wallnerryan/redis sh -c 'exec redis-cli -h 10.2.64.6'
10.2.64.6:6379> lrange mylist 0 -1
1) "last"
2) "first"
3) "A"
4) "B"
10.2.64.6:6379>
```


## Testing Migrations

### Label a node
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig label nodes ip-10-0-0-235.ec2.internal servertype=production
node "ip-10-0-0-235.ec2.internal" labeled
```

```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get no
NAME                         LABELS                                                                    STATUS    AGE
ip-10-0-0-231.ec2.internal   kubernetes.io/hostname=ip-10-0-0-231.ec2.internal                         Ready     9h
ip-10-0-0-232.ec2.internal   kubernetes.io/hostname=ip-10-0-0-232.ec2.internal                         Ready     9h
ip-10-0-0-233.ec2.internal   kubernetes.io/hostname=ip-10-0-0-233.ec2.internal                         Ready     9h
ip-10-0-0-234.ec2.internal   kubernetes.io/hostname=ip-10-0-0-234.ec2.internal                         Ready     9h
ip-10-0-0-235.ec2.internal   kubernetes.io/hostname=ip-10-0-0-235.ec2.internal,servertype=production   Ready     9h
```

### Add nodeSelector
```
spec:
  containers: 
    - name: "redis-slave"
    image: "wallnerryan/redis-slave"
    env:
    - name: GET_HOSTS_FROM
      value: env
    ports: 
      - name: "redis-server"
        containerPort: 6379
    volumeMounts:
      - mountPath: "/var/lib/redis"
        name: redis-master-data2
  nodeSelector:
    servertype: "production"
```

### Stop and Start your POD with the new Selector
```sh
kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig delete -f ../install-flocker/examples/redis/redis-slave-controller.yaml

(Make your change to redis-slave-controller.yaml)

kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig create -f ../install-flocker/examples/redis/redis-slave-controller.yaml
```

View our volume go to the "production" node (235)
```sh
$:-> kubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  get no
NAME                         LABELS                                                                    STATUS    AGE
...<snip>...
ip-10-0-0-235.ec2.internal   kubernetes.io/hostname=ip-10-0-0-235.ec2.internal,servertype=production   Ready     20h

$:-> flockerctl --control-service=ec2-52-2-176-38.compute-1.amazonaws.com list
DATASET                                SIZE     METADATA                     STATUS         SERVER
f0487599-e4d9-45d0-abcb-38b5b141f42a   10.00G   name=flocker-redis-master2   attached ✅   327e5227 (10.0.0.235)
```

Now that our node is on our production node we want to check it has its data
```sh
ubectl --kubeconfig=clusters/my-k8s-cluster/kubeconfig  exec redis-slave-d8qdh -c redis-slave -- redis-cli -h 10.2.7.4 lrange mylist 0 -1
last
first
A
B
```
