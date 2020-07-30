# Kubernetes Certified Application Developer (CKAD)

Online Performance based exam - Need to know how k8s works + practice needed.

## Components

- API Server
  Acts as a frontend to k8s cluster
- etcd
  Distributed key-value store to store k8s info for nodes
- kubelet
  Agent runs on each node from the cluster
- Container Runtime
  Docker
- Controller
  Responsible for orchastrating
- Scheduler
  Distributing works to nodes

## Master vs Worker Nodes

Worker Node has Container Runtime and kubelet
Master Node has API-server | controller | scheduler

## Pods

A single instance of an application.

## YAML in k8s

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
  labels:
    app: app1
spec:
  containers:
    - name: nginx-controller
      image: nginx
```

https://kubernetes.io/docs/reference/kubectl/conventions/

## Namespaces

Create a namespace
`kubectl create namespace dev`

Set current namespace
`kubectl config set-context ${kubectl config current-context} --namespace=dev`

Getting a list of pods in default namespace
`kubectl get pods --namespace=dev`

Getting a list of pods in all namespaces
`kubectl get pods --all-namespace`

## Commands and Arguments in k8s pod

```dockerfile
FROM Ubuntu

ENTRYPOINT ["sleep"]

CMD ["5"]
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
  labels:
    app: app1
spec:
  containers:
    - name: nginx-sleeper
      image: nginx-sleeper
      command: ["sleep2"] //equivalent to ENTRYPOINT in dockerfile
      args: ["10"] // equivalent to CMD in docker file
```

## Edit a Pod

`kubectl edit pod <pod-name>`

We can not edit specs of an existing Pod other than:

- spec.containers[*].image
- spec.initContainers[*].image
- spec.activeDeadlineSeconds
- spec.tolerations

## Setting an env variable in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
  labels:
    app: app1
spec:
  containers:
    - name: nginx-sleeper
      image: nginx-sleeper
      ports:
        - containerPort: 8080
      env:
        - name: HTTP_ENABLED
          value: true
```

## Create a ConfigMap

```bash
 kubectl create configmap \
    app1-config --from-literal=HTTP_ENABLED=true
```

//ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-config
data:
  HTTP_ENABLED: true
```

//Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
  labels:
    app: app1
spec:
  containers:
    - name: nginx-sleeper
      image: nginx-sleeper
      ports:
        - containerPort: 8080
      envFrom:
        - configMapRef:
            name: app1-config
```

Configuring ConfigMaps in Pods

- ENV

```yaml
envFrom:
  - configMapRef:
      name: app1-config
```

- SINGLE ENV

```yaml
env:
  - name: HTTP_ENABLED
    valueFrom:
      configMapRef:
        name: app1-config
        key: HTTP_ENABLED
```

- VOLUME

```yaml
volumes:
  - name: app1-config-volume
    configMap:
      name: app1-config
```

## Create a Secret

`kubectl create secret generic app1-secret --from-literal=HOST=localhost`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app1-secret
data:
  HOST: localhost
  USER: admin
  PWD: admin
```

To encode:
`echo -n 'admin' | base64`
To decode:
`echo -n 'enc_a_dmin' | base64 -d`

Injecting Secrets in Pods

- ENV

```yaml
envFrom:
  - secretRef:
      name: app1-secret
```

- SINGLE ENV

```yaml
env:
  - name: HOST
    valueFrom:
      secretRef:
        name: app1-secret
        key: HOST
```

- VOLUME

```yaml
volumes:
  - name: app1-secret-volume
    secret:
      name: app1-secret
```

## Security in Docker

Capabilities are only supported at the container level and not at the POD level.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app1-pod
  labels:
    app: app1
spec:
  containers:
    - name: nginx-sleeper
      image: nginx-sleeper
      securityContext:
        runAsUser: 1000
        capabilities:
          add: ["MAC_ADMIN"]
```
