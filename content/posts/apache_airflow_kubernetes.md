---
title: "Running Apache Airflow locally on Kubernetes (minikube)"
date: 2020-04-12T19:29:05+01:00
draft: false
toc: true
autoCollapseToc: true
tags: ["apache airflow", "airflow", "kubernetes"]
categories: ["airflow", "kubernetes"]
---

The goal of this guide is to show how to run Airflow entirely on a Kubernetes
cluster. This means that all Airflow componentes (i.e. webserver, scheduler and workers)
would run within the cluster.

<!--more-->

## Before we begin...
What does this article covers?
* How to define Kubernetes components to run Airflow and why we need them
* Deploy Airflow components and run a DAG
* Explain necessary K8s components like their definitions and learn why we need them

What doesn't this article cover?
* Doesn't explain Airflow in detail
* Doesn't explain Kubernetes in detail

Important: this deployment guide is not suitable for production environments

Source code can be found in: [https://github.com/ipeluffo/airflow-on-kubernetes](https://github.com/ipeluffo/airflow-on-kubernetes).

## Prerequisites
* [minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/)
* [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

At the moment of writing, I'm using the following versions on macOS 10.14.6:
```bash
➜ minikube version
minikube version: v1.8.2
commit: eb13446e786c9ef70cb0a9f85a633194e62396a1

➜ kubectl version
Client Version: version.Info{Major:"1", Minor:"17", GitVersion:"v1.17.4", GitCommit:"8d8aa39598534325ad77120c120a22b3a990b5ea", GitTreeState:"clean", BuildDate:"2020-03-12T23:40:44Z", GoVersion:"go1.14", Compiler:"gc", Platform:"darwin/amd64"}
```

## Introduction
[Airflow](https://airflow.apache.org/) is described on its website as:
> Airflow is a platform created by the community to programmatically author, schedule and monitor workflows.

[Kubernetes](https://kubernetes.io/) is described on its website as:
> Kubernetes (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.

### What are we going to build?
![](/images/airflow_on_kubernetes_schema.png "Diagram of Pods running in the cluster")

### Why Airflow on Kubernetes?
Airflow offers a very flexible toolset to programmatically create workflows of any complexity.
In order to run the individual tasks Airflow uses an [executor](https://airflow.apache.org/docs/stable/executor/index.html)
to run them in different ways like locally or using Celery.

In version 1.10.0, Airflow introduced a new executor called [KubernetesExecutor](https://airflow.apache.org/docs/stable/executor/kubernetes.html)
to dynamically run tasks on [Kubernetes pods](https://kubernetes.io/docs/concepts/workloads/pods/pod/).
In this way, Airlfow is able to run tasks creating Pods on demand without wasting resources 
with idle workers as it would happen with other executors.

## Docker image
In order to be able to run Airflow's components, we need a Docker image that Kubernetes will
use in order to run pods inside the cluster.

For this tutorial we'll use the image [puckel/docker-airflow](https://hub.docker.com/r/puckel/docker-airflow) 
(Github: [https://github.com/puckel/docker-airflow](https://github.com/puckel/docker-airflow)), more specifically image's tag 1.10.9 which 
provides a flexible Airflow configuration image using the last version of Airflow at the 
moment of writing (1.10.9).

The only issue is that docker-airflow image doesn't provide support for `KubernetesExecutor`: 
[https://github.com/puckel/docker-airflow/blob/1.10.9/README.md#usage](https://github.com/puckel/docker-airflow/blob/1.10.9/README.md#usage)

However, we can take advantage of some flexibilities of the image definition that will allow us to use `KubernetesExecutor`.

### Customizations
#### Airflow dependency
One key thing that is not present in the image is the extra Kubernetes dependencies from Airflow:
[https://github.com/puckel/docker-airflow/blob/1.10.9/Dockerfile#L62](https://github.com/puckel/docker-airflow/blob/1.10.9/Dockerfile#L62)

Despite the Dockerimage allows us to add more dependencies setting the `AIRFLOW_DEPS` 
argument, in this tutorial we're not going to create our own custom docker image, so we're 
using another customization available in the entrypoint script of the image:
[https://github.com/puckel/docker-airflow/blob/1.10.9/script/entrypoint.sh#L26-L29](https://github.com/puckel/docker-airflow/blob/1.10.9/script/entrypoint.sh#L26-L29)

Thanks to that part of the entrypoint script, we're able to add a `requirements.txt` file that 
will be used dynamically by the entry point when starting the container. The downside of this 
is that any new pod will need to install this dependency before starting it, making the 
startup time slower. A better solution would be to design a custom and optimal Docker image 
for our purposes that it's out of the scope of this guide.

#### Environment variables
Related to the need of a customized docker image, we should also customize Airflow's
configuration in order to use the executor.

Similar to what was described above, we have the option to create our own configuration file 
and load it in our custom image or we can override configuration's setting by [using environment variables](https://airflow.apache.org/docs/1.10.9/howto/set-config.html#setting-configuration-options).

## Kubernetes objects
### `ConfigMap`
From [Kubernetes site](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/):
> ConfigMaps allow you to decouple configuration artifacts from image content to keep containerized applications portable.

In our case, we're going to create two `ConfigMap` objects described below.

#### `ConfigMap`: `requirements.txt`
As described above, we need to add a requirements file in order to install Airflow's Kubernetes dependency (i.e. `apache-airflow[kubernetes]`).

Although there are different alternatives to implement to create the file, in this case we'll use a 
`ConfigMap` that will be later mounted as a `Volume` in order to create the necessary file in the pod's 
file system. The objective of this guide is not only to show Airflow running on Kubernetes but use and 
learn different tools that Kubernetes provides us.

Below is the `ConfigMap` definition:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: requirements-configmap
data:
  requirements.txt: |
    apache-airflow[kubernetes]==1.10.9
```

#### `ConfigMap`: environment variables
In order to customise Airflow's configuration, we'll set environment variables that override the
file configuration. To achieve this, we can define the env vars within the Kubernetes object definition or we can also create a `ConfigMap` and just configure the object to set the env vars from it.

Below is the `ConfigMap` for our custom environment variables:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: airflow-envvars-configmap
data:
  EXECUTOR: Kubernetes
  POSTGRES_HOST: postgres
  POSTGRES_USER: airflow
  POSTGRES_PASSWORD: airflow
  POSTGRES_DB: airflow
  POSTGRES_PORT: "5432"
  LOAD_EX: "y"
  AIRFLOW__KUBERNETES__KUBE_CLIENT_REQUEST_ARGS: '{"_request_timeout": [60,60]}'
  AIRFLOW__KUBERNETES__WORKER_CONTAINER_REPOSITORY: puckel/docker-airflow
  AIRFLOW__KUBERNETES__WORKER_CONTAINER_TAG: "1.10.9"
  AIRFLOW__KUBERNETES__DAGS_VOLUME_HOST: /mnt/airflow/dags
  AIRFLOW__KUBERNETES__LOGS_VOLUME_CLAIM: airflow-logs-pvc
  AIRFLOW__KUBERNETES__ENV_FROM_CONFIGMAP_REF: airflow-envvars-configmap
```

Following is the explanation for each of the env vars:
* `EXECUTOR`: we need this one to dynamically set the Airflow's executor. The [docker image entrypoint script uses this env var](https://github.com/puckel/docker-airflow/blob/1.10.9/script/entrypoint.sh#L13) 
to set the Airflow executor configuration.
* `POSTGRES_`: these env vars are needed since our deployment needs a Postgres server running to which our Airflow components will connect to store information about DAGs and Airflow such as [connections](https://airflow.apache.org/docs/stable/concepts.html#connections), [variables](https://airflow.apache.org/docs/stable/concepts.html#variables) and DAGs' information such as tasks' state.
* `LOAD_EX`: this env var is used to load Airflow's example DAGs. Feel free to disable it if you want to not see all default DAGs.
* `AIRFLOW__KUBERNETES__KUBE_CLIENT_REQUEST_ARGS`: when developing this guide, I found that Airflow failed 
to parse the configuration file and apparently this value was causing some issues because of the double 
brackets in the [configuration value in the docker image](https://github.com/puckel/docker-airflow/blob/1.10.9/config/airflow.cfg#L934).
* `AIRFLOW__KUBERNETES__WORKER_CONTAINER_REPOSITORY`: all env vars with prefix `AIRFLOW__KUBERNETES__` are
specifically for Kubernetes integration on Airflow. As the name suggest, this env var is to specify the
docker image to be used for workers. In the context of Kubernetes, workers will be run on a Pod.
* `AIRFLOW__KUBERNETES__WORKER_CONTAINER_TAG`: this env var is used to specify the docker image tag.
* `AIRFLOW__KUBERNETES__DAGS_VOLUME_HOST`: we'll see this in more detail later. For now, this specifies the 
path of the volume in the host (i.e. cluster node) where DAGs file are stored.
* `AIRFLOW__KUBERNETES__LOGS_VOLUME_CLAIM`: this env var specifies the Kubernetes volume claim to use to
store and read logs. We'll talk about this in more detail later.
* `AIRFLOW__KUBERNETES__ENV_FROM_CONFIGMAP_REF`: this specifies the name of the `ConfigMap` that stores the 
env vars which actually is this one. This will allow workers to load env vars from this `ConfigMap` when 
running.

### Volumes
For each Airflow component (i.e. Kubernetes pod) we're going to setup three volumes for different purposes using multiple Kubernetes tools:
1. Volume for Logs
2. Volume for requirements file
3. Volume for DAGs

#### Volume: Logs
There are multiple alternatives to save Airflow's logs on a Kubernetes deployment. In this guide we'll 
define a Volume that we'll allow us to persist logs from all Airflow's components. If we decide to not set
a volume, then Airflow's workers logs would be lost after they finish.

To achieve this, we need to create a `PersistenVolumeClaim` object:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: airflow-logs-pvc
  labels:
    app: airflow-k8s
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 512Mi
```

This volume claim will allow pods to create a volume that will be attached to this volume claim. For more 
information about to `PersistentVolume` and `PersistentVolumeClaim`: [https://kubernetes.io/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

Important things about this object:
* `metadata.name`: this is the value used in the env var used to set the name of the persistent volume 
claim to be used by Airflow's workers.
* `accessModes`: since the volume claim will be read and write by multiple pods, we need to set a proper 
access mode to allow them to use it.

#### Volume: requirements file
As mentioned in the section of `ConfigMap`s, the requirements file is declared as a `ConfigMap` which will
be mounted as a file using a volume. This volume is define within a Kubernetes object and looks like:
```yaml
- name: requirements-configmap
  configMap:
    name: requirements-configmap
```

We need to set `configMap.name` the same value that we use in the config map definition.

#### Volume: DAGs directory
For the DAGs directory, we'll set something only suitable for local deployments that gives us a lot of 
flexibility when doing tests when we need to test changes on DAGs.

The definition of this volume looks like the definition below:
```yaml
- name: dags-host-volume
  hostPath:
    path: /mnt/airflow/dags
    type: Directory
```

In this case we mount a volume of type [`hostPath`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath).
This means that the pod's volume is attached to a path (could be a file or a directory) within the cluster
node. This last part is important, *the path should exists in the cluster node and not in the host machine*.

Then, to make this works we can put the files into a directory in the host machine or mount a host directory into the Kubernetes node (in our case the `minikube` cluster).

Since this a test environment where we'd like to easily change DAGs and run tests, we'll mount a host 
folder into the minikube cluster running the following command:
```bash
minikube mount ./dags/:/mnt/airflow/dags
```

When running this command you should see an output like this:
```bash
➜ minikube mount ./dags/:/mnt/airflow/dags
📁  Mounting host path ./dags/ into VM as /mnt/airflow/dags ...
    ▪ Mount type:   <no value>
    ▪ User ID:      docker
    ▪ Group ID:     docker
    ▪ Version:      9p2000.L
    ▪ Message Size: 262144
    ▪ Permissions:  755 (-rwxr-xr-x)
    ▪ Options:      map[]
    ▪ Bind Address: 192.168.64.1:50240
🚀  Userspace file server: ufs starting
✅  Successfully mounted ./dags/ to /mnt/airflow/dags

📌  NOTE: This process must stay alive for the mount to be accessible ...
```

Finally, having the mount running, pods will be able to mount cluster node's folder where they'll be able to read and write files which will be written into our host machine.

### PostgreSQL
In this and the following sections, we'll define the necessary Kubernetes objects to run the 
pods that we need to run Airflow.

First, we'll define a `Deployment` and a `Service` to run a PostgreSQL instance where Airflow 
will use. We can also define a `Pod` object, but in this case, they'll be automatically created
when we create the `Deployment`.

If you're not familiar with thes Kubernetes concepts, I recommend having a quick read to the
links below:
* `Deployment`: [https://kubernetes.io/docs/concepts/workloads/controllers/deployment/](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
* `Service`: [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)

`Service` definition:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

This Service object will allow us to expone the pod port to be able to connect to the 
PostgreSQL instance. Kubernetes cluster runs a DNS service that will allow other pods to
connect to this service using its name (i.e. `postgres`).

`Deployment` definition:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  selector:
    matchLabels:
      app: postgres

  replicas: 1

  template:
    metadata:
      labels:
        app: postgres

    spec:
      containers:
      - name: postgres
        image: postgres:12
        resources:
          limits:
            memory: 128Mi
            cpu: 500m
        ports:
        - containerPort: 5432        
        env:
        - name: POSTGRES_PASSWORD
          value: airflow
        - name: POSTGRES_USER
          value: airflow
        - name: POSTGRES_DB
          value: airflow
```

For the deployment, we specify that the number of replicas should be one and the necessary env 
vars to setup the default user and database to be created when the container starts.

### Airflow webserver
Now, we're going to go through the definition of the `Service` and `Deployment` objects for
the Airflow webserver. In this case, we need a `Service` since we need to connect to the
webserver from our machine and for that we'll need to expose the port from the cluster.

`Service` definition:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: airflow-webserver
  labels:
    app: airflow-k8s
spec:
  type: NodePort
  selector:
    app: airflow-webserver
  ports:
  - port: 8080
```

Most important things from the definition:
* `selector`: this will be used by the service to identify which pods should receive traffic 
sent to the service.
* `type: NodePort`: this will allow us to expose the service port so we're able to connect to
the service from our machine. For more info: [https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types](https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types)
* `port`: port to expose to connect to the webserver.

`Deployment` full definition:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airflow-webserver
  labels:
    app: airflow-k8s

spec:
  selector:
    matchLabels:
      app: airflow-webserver

  replicas: 1

  template:
    metadata:
      labels:
        app: airflow-webserver

    spec:
      containers:
      - name: airflow-webserver
        image: puckel/docker-airflow:1.10.9
        envFrom:
          - configMapRef:
              name: airflow-envvars-configmap
        resources:
          limits:
            memory: "2Gi"
        ports:
          - containerPort: 8080
        volumeMounts:
          - name: requirements-configmap
            subPath: "requirements.txt"
            mountPath: "/requirements.txt"
          - name: dags-host-volume
            mountPath: /usr/local/airflow/dags
          - name: logs-persistent-storage
            mountPath: /usr/local/airflow/logs
      volumes:
        - name: requirements-configmap
          configMap:
            name: requirements-configmap
        - name: dags-host-volume
          hostPath:
            path: /mnt/airflow/dags
            type: Directory
        - name: logs-persistent-storage
          persistentVolumeClaim:
              claimName: airflow-logs-pvc
```

Few comments about this definition:
* `volumes` section: as mentioned on the `Volumes` section of this guide, we need to define the volumes to be used by the pods. So in this case we define one volume for the logs, one for the requirements file and one for the logs persistent volume claim.
* `volumeMounts`: to be able to use the volumes, we need to specify how they should be mounted on within the pod and this section is used for that.
* `ports`: define the pod's port to expose.
* `resources`: in this example we just specified the maximum amount of memory allowed to be used by the pod.

### Airflow scheduler
In the case of the scheduler, we only need to create a deployment since it doesn't expose any
service that other workers need to connect to it. However, scheduler is responsible of creating
workers, in this case pods, to run Airflow's tasks so we need to give the needed permissions
to the scheduler to be able to manage pods on the cluster like creating and deleting them. To 
achieve this, we need to define a `ClusterRole` and `ClusterRoleBinding` Kubernetes objects.

`Deployment` definition:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: airflow-scheduler
  labels:
    app: airflow-k8s

spec:
  selector:
    matchLabels:
      app: airflow-scheduler

  replicas: 1

  template:
    metadata:
      labels:
        app: airflow-scheduler

    spec:
      containers:
      - name: airflow-scheduler
        image: puckel/docker-airflow:1.10.9
        args: ["scheduler"]
        envFrom:
          - configMapRef:
              name: airflow-envvars-configmap
        resources:
          limits:
            memory: "512Mi"
        volumeMounts:
          - name: requirements-configmap
            subPath: "requirements.txt"
            mountPath: "/requirements.txt"
          - name: dags-host-volume
            mountPath: /usr/local/airflow/dags
          - name: logs-persistent-storage
            mountPath: /usr/local/airflow/logs
      volumes:
        - name: requirements-configmap
          configMap:
            name: requirements-configmap
        - name: dags-host-volume
          hostPath:
            path: /mnt/airflow/dags
            type: Directory
        - name: logs-persistent-storage
          persistentVolumeClaim:
              claimName: airflow-logs-pvc
```

The only difference with the definition of the webserver deployment is the `args` setting 
which overrides the Docker image command that will be run on from the [entrypoint script](https://github.com/puckel/docker-airflow/blob/1.10.9/script/entrypoint.sh#L119-L122).

Airflow scheduler permissions:
```yaml
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pods-permissions
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "delete"]

--- 

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: pods-permissions
subjects:
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  kind: ClusterRole
  name: pods-permissions
  apiGroup: rbac.authorization.k8s.io
```

It's easy to see that the `ClusterRole` gives specific permissions to manage pods.

## Running Airflow in Kubernetes
To make it easier to create and delete all resources from the Kubernetes cluster, I created two scripts:
* `script-apply.sh`: creates all Kubernetes objects.
* `script-delete.sh`: deletes all objects, it can take some time to delete the persistent volume claim.

After starting `minikube`, if we're not running anything in the cluster, we should see something like this:
```bash
➜ kubectl get pods,deploy,svc,pv,pvc
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   25d
```

If we run `script-apply.sh` script:
```bash
➜ ./script-apply.sh
persistentvolumeclaim/airflow-logs-pvc created
clusterrole.rbac.authorization.k8s.io/pods-permissions created
clusterrolebinding.rbac.authorization.k8s.io/pods-permissions created
service/postgres created
deployment.apps/postgres created
configmap/requirements-configmap created
configmap/airflow-envvars-configmap created
service/airflow-webserver created
deployment.apps/airflow-webserver created
deployment.apps/airflow-scheduler created
```

Then, we should be able all objects created by the script:
```bash
➜ kubectl get pods,deploy,svc,pv,pvc
NAME                                     READY   STATUS    RESTARTS   AGE
pod/airflow-scheduler-5c85cb5c9c-qmz4x   1/1     Running   0          7s
pod/airflow-webserver-744ddfcf6-4gfgc    1/1     Running   0          7s
pod/postgres-58fb56c4cb-tcdx8            1/1     Running   0          7s

NAME                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/airflow-scheduler   1/1     1            1           9s
deployment.apps/airflow-webserver   1/1     1            1           9s
deployment.apps/postgres            1/1     1            1           9s

NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/airflow-webserver   NodePort    10.100.119.225   <none>        8080:32044/TCP   9s
service/kubernetes          ClusterIP   10.96.0.1        <none>        443/TCP          25d
service/postgres            ClusterIP   10.106.95.57     <none>        5432/TCP         10s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                      STORAGECLASS   REASON   AGE
persistentvolume/pvc-830cbd78-8603-44cf-8f59-56e1fe16ec73   512Mi      RWX            Delete           Bound    default/airflow-logs-pvc   standard                10s

NAME                                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/airflow-logs-pvc   Bound    pvc-830cbd78-8603-44cf-8f59-56e1fe16ec73   512Mi      RWX            standard       10s
```

To access Airflow webserver:
```bash
➜ minikube service airflow-webserver
|-----------|-------------------|-------------|---------------------------|
| NAMESPACE |       NAME        | TARGET PORT |            URL            |
|-----------|-------------------|-------------|---------------------------|
| default   | airflow-webserver |             | http://192.168.64.4:32044 |
|-----------|-------------------|-------------|---------------------------|
🎉  Opening service default/airflow-webserver in default browser...
```

And we should see Airflow UI homepage:
![Airflow web UI](/images/airflow_capture.png "Airflow up and running!")

Let's t try running a DAG:
```bash
➜ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
airflow-scheduler-5c85cb5c9c-qmz4x   1/1     Running   0          4m17s
airflow-webserver-744ddfcf6-4gfgc    1/1     Running   0          4m17s
postgres-58fb56c4cb-tcdx8            1/1     Running   0          4m17s
```

If we run `example_complex` DAG and we wait a few seconds, we should see the DAG tasks start to be run:
![Airflow complex DAG example](/images/airflow_example_complex.png "example_complex DAG running")

And if we check the pods on Kubernetes:
```bash
➜ kubectl get pods
NAME                                                                     READY   STATUS              RESTARTS   AGE
airflow-scheduler-5c85cb5c9c-qmz4x                                       1/1     Running             0          6m26s
airflow-webserver-744ddfcf6-4gfgc                                        1/1     Running             0          6m26s
examplecomplexcreateentrygcs-a66431daa1c646e3a22ae7da97371047            1/1     Running             0          6s
examplecomplexcreateentrygroupresult-11450ba8cbd04e93b8d89f75b256b232    1/1     Running             0          2s
examplecomplexcreateentrygroupresult2-52663896be3c4bcc9e0d5f7293525d4d   0/1     ContainerCreating   0          0s
examplecomplexgetentrygroup-2243f00d53e44d75b6f511e8bfc272da             1/1     Running             0          4s
postgres-58fb56c4cb-tcdx8                                                1/1     Running             0          6m26s
```

Let's check the logs of one successful task:
![](/images/airflow_example_complex_logs.png "Task logs")

However, if we wait the DAG to finish, we'll see that it fails:
![](/images/airflow_example_complex_fail.png "DAG failed")

And we can check why the task failed:
![](/images/airflow_example_complex_fail_log.png "Failed task logs")

If we check pods in the cluster:
```bash
➜ kubectl get pods
NAME                                 READY   STATUS    RESTARTS   AGE
airflow-scheduler-5c85cb5c9c-qmz4x   1/1     Running   0          10m
airflow-webserver-744ddfcf6-4gfgc    1/1     Running   0          10m
postgres-58fb56c4cb-tcdx8            1/1     Running   0          10m
```

We can see that all pods running Airflow's tasks have finished and were removed.

Lastly, we can clean the Kubernetes cluster removing all objects:
```bash
➜ ./script-delete.sh
clusterrole.rbac.authorization.k8s.io "pods-permissions" deleted
clusterrolebinding.rbac.authorization.k8s.io "pods-permissions" deleted
service "postgres" deleted
deployment.apps "postgres" deleted
configmap "requirements-configmap" deleted
configmap "airflow-envvars-configmap" deleted
service "airflow-webserver" deleted
deployment.apps "airflow-webserver" deleted
deployment.apps "airflow-scheduler" deleted
persistentvolumeclaim "airflow-logs-pvc" deleted
```

```bash
➜ kubectl get pods,deploy,svc,pv,pvc
NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   25d
```

## Conclusion
We reached the end of this tutorial and we could see that running a whole Airflow deployment
on a local Kubernetes cluster was straightforward.

As mentioned in the beginning, the objective of this guide was to use several tools from 
Kubernetes and many things should be changed for a deployment in a production environment.
This deployment allow to quickly do tests, if you have a DAG you should be able to put it in the `dags` folder and immediately see it on the UI.

So that's it for the tutorial, I hope it was useful and help you to continue learning about Kubernetes and Apache Airflow.