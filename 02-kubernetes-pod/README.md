# Kubernetes: Deploying and Configuring Pods

This exercise has the following objectives:
- Create a simple local kubernetes cluster
- Understand what a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) is
- Deploy a Pod with your flask app container image to your cluster
- [Port forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) to the Pod to check that it is working
- Inspect Logs of the container within your pod
- Add [Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/) to the container in your Pod
- Run commands on a container for debugging
- Add [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) to your Pod
- [Configure Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for the container in the Pod
- Add an [initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) to your Pod

## Pre-requisites
In addition to docker, this exercise requires kubectl and minikube. Details on installing both are available [here](https://kubernetes.io/docs/tasks/tools/)

## Create a simple local kubernetes cluster
- `minikube start`

## Deploy a Pod with your flask app container image to your cluster
- Populate a pod.yaml file which:
  - references our flask app container image
  - exposes the 9900 port of the container
```
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.1.0
    ports:
    - containerPort: 9900
```
- Create the pod: `kubectl create -f <path to pod.yaml>`

## Port forward to the Pod to check that it is working
- Port forward to the container port: `kubectl port-forward pod/my-flask-app 9900:9900`
- Running the above command means that port 9900 on your localhost now maps to port 9900 on your pod
- Go to `localhost:9900` and `localhost:9900/author` on your browser

## Inspect Logs of the container within your pod
- To see your container logs: `kubectl logs my-flask-app`

## Add Environment Variables to the container in the Pod
We are going to set two environment variables here:
- Set the `LOG_LEVEL` environment variable to `DEBUG`
- Set the `AUTHOR` environment variable to `Pod`

To do this:
- Delete the current running pod: `kubectl delete pod my-flask-app`
- Edit your pod.yaml file to be similar to the below (i.e. add the env portion)
```
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.1.0
    ports:
    - containerPort: 9900
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: AUTHOR
      value: "Pod"
```
- Recreate the pod `kubectl create -f <path to pod.yaml>`
- (Optional) Repeat the port forward and log inspection steps to see differences

## Run commands on a container for debugging
This is a very useful tool for debugging. To do this, we need to open a shell within the pod.
- Run `kubectl exec -it my-flask-app -- sh`
- Type `exit` and press enter when done to close the shell

## Add Probes to your Pod
There are three main probes in Kubernetes which serve to show different levels of progress for a Pod or workload:
- Liveness probes are used to know when an application restart might be needed i.e. the app is running but not responding/progressing (not alive)
- Readiness probes are used to know when an application is ready for traffic
- Startup probes are used to know when an application has finish starting up/booting
Each of these probes could be commands to be executed, http requests or tcp sockets. Probes are also continuously running so things like intervals, initial delays, failure/success thresholds can be set too

For this, we are going to add:
- A startup probe that checks the /author path is accessible (http request)
- Readiness and liveness probes that both check the 9900 port is available (tcp socket)

To do this:
- Delete the current running pod: `kubectl delete pod my-flask-app`
- Edit your pod.yaml file to be similar to the below (i.e. add the env portion)
```
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.1.0
    ports:
    - containerPort: 9900
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: AUTHOR
      value: "Pod"
    startupProbe:
      httpGet:
        path: /author
        port: 9900
      failureThreshold: 30
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 15
      periodSeconds: 20
```
- Recreate the pod `kubectl create -f <path to pod.yaml>`
- Run `kubectl get pods -o wide` and pay attention to the `READINESS GATES` column especially