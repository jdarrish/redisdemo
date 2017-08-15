# Redis + memtier_benchmark (from Redislabs)
This Helm chart creates a redis deployment, exposes it via a service NodePort, and sets it to horizontally autoscale.
It also deploys a memtier_benchmark pod to exercise the redis nodes

* Assuming helm is already initalized...

## Install the helm chart

eg
`$ helm install --namespace=redisdemo redisdemo/ --name jeffredis`

```
$ helm list
NAME          	REVISION	UPDATED                 	STATUS  	CHART          	NAMESPACE
jeffredis     	1       	Mon Aug 14 21:09:38 2017	DEPLOYED	redisdemo-0.1.0	redisdemo
```

Once the helm chart has completely deployed, you should see these resources:

```
$ kubectl get all
NAME                                  READY     STATUS    RESTARTS   AGE
po/memtierbenchmark-849680939-9z0fm   1/1       Running   0          49s
po/redis-master-1386390193-6t450      1/1       Running   0          19s
po/redis-master-1386390193-fbd6v      1/1       Running   0          19s
po/redis-master-1386390193-ztxq0      1/1       Running   0          49s

NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
svc/redissvc   10.21.122.187   <nodes>       6379:30331/TCP   49s

NAME            REFERENCE                 TARGETS           MINPODS   MAXPODS   REPLICAS   AGE
hpa/redis-hpa   Deployment/redis-master   <unknown> / 30%   3         50        1          49s

NAME                      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/memtierbenchmark   1         1         1            1           49s
deploy/redis-master       3         3         3            3           49s

NAME                            DESIRED   CURRENT   READY     AGE
rs/memtierbenchmark-849680939   1         1         1         49s
rs/redis-master-1386390193      3         3         3         49s
```

Next, connect to an interactive shell with the memtierbenchmark pod:
```
$ kubectl exec -it memtierbenchmark-849680939-9z0fm -- /bin/bash
```

Once you are connected you can execute the benchmark utility and point it to the Redis deployment service "redissvc" IP address obtained from the `kubectl get all` output

```
root@memtierbenchmark-849680939-9z0fm:/# /memtier_benchmark -s 10.21.122.187
[RUN #1] Preparing benchmark client...
[RUN #1] Launching threads now...
^CUN #1 1%,  11 secs]  4 threads:       13349 ops,    1186 (avg:    1213) ops/sec, 44.90KB/sec (avg: 46.45KB/sec), 166.74 (avg: 163.54) msec latency
```

Once the workload begins and after ~2-minutes, the Kubernetes Horizontal Autoscaler "HPA" should begin to replicate redis pods.

You can monitor the status of the HPA by running `kubectl get hpa`
```
$ kubectl get hpa
NAME           REFERENCE                 TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
redis-hpa   Deployment/redis-master   83% / 30%   3         50       47         8m
```

* As new redis pods are replicated, the benchmark tool will not initiate net-new connections to the new pods until it is stopped and restarted. Use `ctrl-c` in the memtierbenchmark container to stop the workload and execute the same command `# /memtier_benchmark -s 10.21.122.187` to restart the workload. Kubernetes will automatically load-balance the connections from the benchmark tool across all the new pods. You may need to stop and start the workload a couple times to keep CPU high enough to continue auto-scaling out to 50 pods.