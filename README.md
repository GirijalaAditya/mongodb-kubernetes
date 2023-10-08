# MongoDB Kubernetes

```t
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  db_host: mongodb-service
---
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  username: dXNlcm5hbWU=
  password: cGFzc3dvcmQ=

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
              
---
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
  
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        - name: ME_CONFIG_MONGODB_SERVER 
          valueFrom: 
            configMapKeyRef:
              name: mongodb-configmap
              key: db_host
---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer  
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000

```


# MongoDB Helm Chart

## helm repo add bitnami https://charts.bitnami.com/bitnami

### mongodb-values.yml

```t
architecture: replicaset
auth:
	rootPassword: cGFzc3dvcmQ=
replicaCount: 3
persistence:
	storageClass: managed-csi-premium
```

## helm install mongodb-helm --values mongodb-values.yml bitnami/mongodb

```t
root@myvm:~/mongodb# kubectl get all
NAME                         READY   STATUS    RESTARTS   AGE
pod/mongodb-helm-0           1/1     Running   0        4m41s
pod/mongodb-helm-1           1/1     Running   0        4m5s
pod/mongodb-helm-2           1/1     Running   0        3m38s
pod/mongodb-helm-arbiter-0   1/1     Running   0        4m41s

NAME                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)     AGE
service/kubernetes ClusterIP   10.0.0.1     <none>        443/TCP     26m
service/mongodb-helm-arbiter-headless   ClusterIP   None         <none>        27017/TCP   4m41s
service/mongodb-helm-headless           ClusterIP   None         <none>        27017/TCP   4m41s

NAME                                    READY   AGE
statefulset.apps/mongodb-helm           3/3     4m42s
statefulset.apps/mongodb-helm-arbiter   1/1     4m42s

root@myvm:~/mongodb# kubectl get secret
NAME                                 TYPE                 DATA   AGE
mongodb-helm                         Opaque               2      9m36s
sh.helm.release.v1.mongodb-helm.v1   helm.sh/release.v1   1      9m36s

root@myvm:~/mongodb# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS          REASON   AGE
pvc-14123351-fae2-4c0e-a1ea-23ec002a47f1   8Gi        RWO            Delete           Bound    default/datadir-mongodb-helm-2   managed-csi-premium            8m54s
pvc-5a4ed135-9f13-413f-bddc-4043166c0d0e   8Gi        RWO            Delete           Bound    default/datadir-mongodb-helm-1   managed-csi-premium            9m21s
pvc-d42d19ec-8be2-4422-9235-c55356d7bc71   8Gi        RWO            Delete           Bound    default/datadir-mongodb-helm-0   managed-csi-premium            9m56s
root@myvm:~/mongodb#
root@myvm:~/mongodb# kubectl get pvc
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
datadir-mongodb-helm-0   Bound    pvc-d42d19ec-8be2-4422-9235-c55356d7bc71   8Gi        RWO            managed-csi-premium   10m
datadir-mongodb-helm-1   Bound    pvc-5a4ed135-9f13-413f-bddc-4043166c0d0e   8Gi        RWO            managed-csi-premium   9m29s
datadir-mongodb-helm-2   Bound    pvc-14123351-fae2-4c0e-a1ea-23ec002a47f1   8Gi        RWO            managed-csi-premium   9m2s

```

## Mongo Express

```t
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports: 
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          value: root
        - name: ME_CONFIG_MONGODB_SERVER
          value: mongodb-0.mongodb-headless
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom: 
            secretKeyRef:
              name: mongodb
              key: mongodb-root-password


---
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081

```
