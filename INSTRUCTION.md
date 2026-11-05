# ToDo app: DaemonSet and CronJob

Both new manifests live in the `mateapp` namespace and call the ToDo app through its ClusterIP service `todoapp-service` in the `todoapp` namespace, using the full DNS name `todoapp-service.todoapp.svc.cluster.local`.

## 1. Deploy the ToDo app and its ClusterIP service

```bash
kubectl apply -f .infrastructure/namespace.yml
kubectl apply -f .infrastructure/deployment.yml
kubectl apply -f .infrastructure/clusterIp.yml
```

Wait until the app pods are ready (the readiness probe passes ~30 seconds after start):

```bash
kubectl get pods -n todoapp
```

## 2. Deploy the DaemonSet and the CronJob

```bash
kubectl create namespace mateapp
kubectl apply -f .infrastructure/daemonset.yml
kubectl apply -f .infrastructure/cronjob.yml
```

If the `mateapp` namespace already exists, `kubectl create namespace` prints `AlreadyExists`: this is fine, continue with the `apply` commands.

Check that both objects were created:

```bash
kubectl get daemonset,cronjob -n mateapp
```

## 3. Validate the DaemonSet

The DaemonSet runs one `busyboxplus:curl` pod on every node. Every 5 seconds it calls `/api/health` of the ToDo app through the ClusterIP service and prints the time and the answer.

```bash
kubectl get pods -n mateapp -l app=todoapp-curl -o wide
kubectl logs -n mateapp daemonset/todoapp-curl --tail=10
```

Expected log lines, one every 5 seconds:

```
Thu Nov  5 12:40:00 UTC 2026 Health OK
Thu Nov  5 12:40:05 UTC 2026 Health OK
```

Follow the logs live:

```bash
kubectl logs -n mateapp daemonset/todoapp-curl -f
```

## 4. Validate the CronJob

The CronJob runs every 4 minutes (`*/4 * * * *`) and calls `/api/health` once. Settings:

- `successfulJobsHistoryLimit: 10`: keeps the last 10 successful jobs;
- `failedJobsHistoryLimit: 5`: keeps the last 5 failed jobs;
- `concurrencyPolicy: Allow`: a new run starts even if the previous one is still running.

Wait up to 4 minutes for the first run, then check the jobs:

```bash
kubectl get cronjob,jobs -n mateapp
```

Read the logs of the latest job (replace `<JOB_NAME>` with a name from the previous command, for example `todoapp-health-29371840`):

```bash
kubectl logs -n mateapp job/<JOB_NAME>
```

Expected output: `Health OK`. The job status is `Complete` and `COMPLETIONS` is `1/1`.

To check the CronJob without waiting 4 minutes, start a job from it manually:

```bash
kubectl create job --from=cronjob/todoapp-health todoapp-health-manual -n mateapp
kubectl logs -n mateapp job/todoapp-health-manual
```
