# ToDo app: DaemonSet and CronJob

All manifests used in this task live in the `mateapp` namespace: the ToDo app Deployment, its ClusterIP service `todoapp-service`, the DaemonSet and the CronJob. The DaemonSet and the CronJob call the app through the service DNS name `todoapp-service.mateapp.svc.cluster.local`.

### Why `ikulyk404/busyboxplus:curl`

The task asks for the `busyboxplus:curl` container. There is no official `busyboxplus` image on Docker Hub, and the original `radial/busyboxplus:curl` is stored in the old Docker manifest v1 format, which containerd v2.1+ (used by current Docker Desktop / kind clusters) refuses to pull (`ErrImagePull`). `ikulyk404/busyboxplus:curl` is a busybox + curl image published in the current format (it pulls and runs on containerd v2.1+), and it is already used by `.infrastructure/busybox.yml` in this repository.

## 1. Deploy the ToDo app and its ClusterIP service

If the `mateapp` namespace already exists, `kubectl create namespace` prints `AlreadyExists`: this is fine, continue with the `apply` commands.

```bash
kubectl create namespace mateapp
kubectl apply -f .infrastructure/deployment.yml
kubectl apply -f .infrastructure/clusterIp.yml
```

Wait until the app pods are ready (the readiness probe passes ~30 seconds after start):

```bash
kubectl get pods -n mateapp -l app=todoapp
```

## 2. Deploy the DaemonSet and the CronJob

```bash
kubectl apply -f .infrastructure/daemonset.yml
kubectl apply -f .infrastructure/cronjob.yml
```

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
