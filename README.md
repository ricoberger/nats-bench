# nats-bench

[NATS](https://nats.io) is fast and lightweight and places a priority on performance. NATS provides [tools for measuring performance](https://docs.nats.io/nats-tools/natsbench). In this tutorial you learn how to benchmark and tune NATS on your systems and environment.

## Usage

The following example runs the benchmark as CronJob in a Kubernetes cluster. The cluster was deployed using the [NATS Operator](https://github.com/nats-io/nats-operator):

```yaml
apiVersion: nats.io/v1alpha2
kind: NatsCluster
metadata:
  name: example-nats-cluster
spec:
  size: 3
  version: "2.0.0"
```

The NATS Operator creates two services. The `example-nats-cluster` can be used to connect to the cluster:

```
NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
example-nats-cluster        ClusterIP   10.100.187.55   <none>        4222/TCP                     3d21h
example-nats-cluster-mgmt   ClusterIP   None            <none>        6222/TCP,8222/TCP,7777/TCP   3d21h
```

Then create the following CronJob and trigger the CronJob manually:

```sh
kubectl apply -f cronjob.yaml
kubectl create job --from=cronjob/nats-bench nats-bench-manual
```

```yaml
---
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: nats-bench
  namespace: nats-io
  labels:
    app: nats-bench
spec:
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: nats-bench
        spec:
          containers:
            - name: nats-bench
              image: ricoberger/nats-bench
              imagePullPolicy: IfNotPresent
              args:
                - -s=nats://example-nats-cluster.nats-io.svc.cluster.local:4222
                - -np=5
                - -ns=5
                - -n=100000
                - -ms=16
                - foo
          restartPolicy: Never
  schedule: "0 0 * * *"
```

To view the benchmarking results you have to view the logs of the `nats-bench` container:

```sh
kubectl logs -l app=nats-bench -f
```

```
Starting benchmark [msgs=100000, msgsize=16, pubs=5, subs=5]
NATS Pub/Sub stats: 2,705,785 msgs/sec ~ 41.29 MB/sec
 Pub stats: 501,353 msgs/sec ~ 7.65 MB/sec
  [1] 501,139 msgs/sec ~ 7.65 MB/sec (20000 msgs)
  [2] 402,345 msgs/sec ~ 6.14 MB/sec (20000 msgs)
  [3] 348,178 msgs/sec ~ 5.31 MB/sec (20000 msgs)
  [4] 233,843 msgs/sec ~ 3.57 MB/sec (20000 msgs)
  [5] 352,378 msgs/sec ~ 5.38 MB/sec (20000 msgs)
  min 233,843 | avg 367,576 | max 501,139 | stddev 86,648 msgs
 Sub stats: 2,263,262 msgs/sec ~ 34.53 MB/sec
  [1] 509,800 msgs/sec ~ 7.78 MB/sec (100000 msgs)
  [2] 497,363 msgs/sec ~ 7.59 MB/sec (100000 msgs)
  [3] 488,543 msgs/sec ~ 7.45 MB/sec (100000 msgs)
  [4] 469,722 msgs/sec ~ 7.17 MB/sec (100000 msgs)
  [5] 460,120 msgs/sec ~ 7.02 MB/sec (100000 msgs)
  min 460,120 | avg 485,109 | max 509,800 | stddev 18,071 msgs
```
