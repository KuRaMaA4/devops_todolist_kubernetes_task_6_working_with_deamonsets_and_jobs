# ToDo app: DaemonSet and CronJob

All manifests used in this task live in the `mateapp` namespace: the ToDo app Deployment, its ClusterIP service `todoapp-service`, the DaemonSet and the CronJob. The DaemonSet and the CronJob call the app through the service DNS name `todoapp-service.mateapp.svc.cluster.local`.

### Note about the `busyboxplus:curl` image

Both manifests use `busyboxplus:curl`, as required by the task. Without a registry prefix Kubernetes looks for the official image `docker.io/library/busyboxplus:curl`, which does not exist, so the pods fail to start. Real events from `kubectl describe pod -n mateapp -l app=todoapp-curl`:

```
Normal   Pulling  kubelet  Pulling image "busyboxplus:curl"
Warning  Failed   kubelet  Failed to pull image "busyboxplus:curl": failed to pull and unpack image "docker.io/library/busyboxplus:curl": failed to resolve reference "docker.io/library/busyboxplus:curl": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
Warning  Failed   kubelet  Error: ErrImagePull
Normal   BackOff  kubelet  Back-off pulling image "busyboxplus:curl"
Warning  Failed   kubelet  Error: ImagePullBackOff
```

The original community image `radial/busyboxplus:curl` does not help on current clusters either: it is stored in the old Docker manifest v1 format, which containerd v2.1+ (current Docker Desktop / kind) refuses to pull.

To make the DaemonSet and the CronJob work, use `ikulyk404/busyboxplus:curl`, a busybox + curl image in the current format, already used by `.infrastructure/busybox.yml` in this repository. Change it in the running cluster:

```bash
kubectl set image daemonset/todoapp-curl busybox=ikulyk404/busyboxplus:curl -n mateapp
kubectl patch cronjob todoapp-health -n mateapp --type='json' \
  -p='[{"op":"replace","path":"/spec/jobTemplate/spec/template/spec/containers/0/image","value":"ikulyk404/busyboxplus:curl"}]'
```

or in the files before `kubectl apply` (macOS; on Linux use `sed -i` without `''`):

```bash
sed -i '' 's#image: busyboxplus:curl#image: ikulyk404/busyboxplus:curl#' .infrastructure/daemonset.yml .infrastructure/cronjob.yml
```

With `ikulyk404/busyboxplus:curl` the DaemonSet logs show `Health OK` every 5 seconds and the CronJob runs complete successfully (see the validation sections below).

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
