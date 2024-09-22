# What is Minio?

MinIO's High Performance Object Storage is Open Source, Amazon S3 compatible, Kubernetes Native and is designed for cloud native workloads like AI.

## Deploying Minio on OpenShift

### 1. Create **PVC** to persist Minio data

```yml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: minio-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
  volumeMode: Filesystem
```

### 2. Create **Secret** to secure minio credentials

```yml
kind: Secret
apiVersion: v1
metadata:
  name: minio-secret
stringData:
  minio_root_user: <your_secure_user>
  minio_root_password: <your_secure_password>
```

### 3. Create **Deployment** to deploy your minio instance

```yml
kind: Deployment
apiVersion: apps/v1
metadata:
  name: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          env:
            - name: MINIO_ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_user
            - name: MINIO_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-secret
                  key: minio_root_password
          ports:
            - containerPort: 9000
              protocol: TCP
            - containerPort: 9090
              protocol: TCP
          volumeMounts:
            - name: data
              mountPath: /data
              subPath: minio
          image: >-
            quay.io/minio/minio:RELEASE.2023-06-19T19-52-50Z
          args:
            - server
            - /data
            - --console-address
            - :9090
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-pvc
```

### 4. Create **Service** to expose minio to the cluster network

```yml
kind: Service
apiVersion: v1
metadata:
  name: minio-service
spec:
  ports:
    - name: api
      protocol: TCP
      port: 9000
      targetPort: 9000
    - name: ui
      protocol: TCP
      port: 9090
      targetPort: 9090
  type: ClusterIP
  selector:
    app: minio
```

### 5. Create **Routes** to expose your service/application to external traffic

```yml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-api
spec:
  to:
    kind: Service
    name: minio-service
    weight: 100
  port:
    targetPort: api
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: minio-ui
spec:
  to:
    kind: Service
    name: minio-service
    weight: 100
  port:
    targetPort: ui
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```
