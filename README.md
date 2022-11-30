# trino_gke
## prepair PD
- Check your gcloud evniorments
  - accounts
  - projects
  - default regions and others
- Disk Volume for PV
```
% gcloud compute disks create data-disk --type=pd-ssd --size=500GB
Created [https://www.googleapis.com/compute/v1/projects/XXXX-XXXX/zones/asia-northeast1-a/disks/data-disk].
NAME       ZONE               SIZE_GB  TYPE    STATUS
data-disk  asia-northeast1-a  500      pd-ssd  READY
```
## PV, PVC
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
  labels:
    app: data-pv
spec:
  capacity:
    storage: 500Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  gcePersistentDisk:
    pdName: data-disk
    fsType: ext4
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
  labels:
    app: data-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 500Gi
  selector:
    matchLabels:
      app: data-pv
EOF
```
## configmap-catalog
```
kubectl apply -f - <<EOF
# Source: trino/templates/configmap-catalog.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcb-trino-catalog
  labels:
    app: trino
    chart: trino-0.3.0
    release: tcb
    heritage: Helm
    role: catalogs
data:
  localfile.properties: |
    connector.name=localfile
EOF
```
## configmap-coordinator
```
kubectl apply -f - <<EOF
# Source: trino/templates/configmap-coordinator.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tcb-trino-coordinator
  labels:
    app: trino
    chart: trino-0.3.0
    release: tcb
    heritage: Helm
    component: coordinator
data:
  node.properties: |
    node.environment=production
    node.data-dir=/data/trino
    plugin.dir=/usr/lib/trino/plugin

  jvm.config: |
    -server
    -Xmx8G
    -XX:+UseG1GC
    -XX:G1HeapRegionSize=32M
    -XX:+UseGCOverheadLimit
    -XX:+ExplicitGCInvokesConcurrent
    -XX:+HeapDumpOnOutOfMemoryError
    -XX:+ExitOnOutOfMemoryError
    -Djdk.attach.allowAttachSelf=true
    -XX:-UseBiasedLocking
    -XX:ReservedCodeCacheSize=512M
    -XX:PerMethodRecompilationCutoff=10000
    -XX:PerBytecodeRecompilationCutoff=10000
    -Djdk.nio.maxCachedBufferSize=2000000

  config.properties: |
    coordinator=true
    node-scheduler.include-coordinator=true
    http-server.http.port=8080
    query.max-memory=4GB
    query.max-memory-per-node=1GB
    # query.max-total-memory-per-node=2GB
    memory.heap-headroom-per-node=1GB
    discovery-server.enabled=true
    discovery.uri=http://localhost:8080

  log.properties: |
    io.trino=INFO
EOF
```
## Service
```
kubectl apply -f - <<EOF
# Source: trino/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: tcb-trino
  labels:
    app: trino
    chart: trino-0.3.0
    release: tcb
    heritage: Helm
spec:
  type: ClusterIP
  ports:
    - port: 8080
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: trino
    release: tcb
    component: coordinator
EOF
```
## deployment-coordinator
```
kubectl apply -f - <<EOF
# Source: trino/templates/deployment-coordinator.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tcb-trino-coordinator
  labels:
    app: trino
    chart: trino-0.3.0
    release: tcb
    heritage: Helm
    component: coordinator
spec:
  selector:
    matchLabels:
      app: trino
      release: tcb
      component: coordinator
  template:
    metadata:
      labels:
        app: trino
        release: tcb
        component: coordinator
    spec:
      securityContext:
        fsGroup: 2000
        runAsNonRoot: true
        runAsUser: 1000
        # runAsGroup: 1000
      volumes:
        - name: config-volume
          configMap:
            name: tcb-trino-coordinator
        - name: catalog-volume
          configMap:
            name: tcb-trino-catalog
        - name: data-volume
          persistentVolumeClaim:
            claimName: data-pvc
      imagePullSecrets:
        - name: registry-credentials
      containers:
        - name: trino-coordinator
          image: "trinodb/trino:latest"
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - mountPath: /etc/trino
              name: config-volume
            - mountPath: /etc/trino/catalog
              name: catalog-volume
            - name: data-volume
              mountPath: /data/trino
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /v1/info
              port: http
          readinessProbe:
            httpGet:
              path: /v1/info
              port: http
          resources:
            {}
EOF
```
