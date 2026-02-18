# Lab: Deploy nginx into 3 worker nodes.
File: nginx-allin.yaml will perform following
- Create namespace "nginx-demo"
- Create PVC from storage class infoscale-concat-sc
- Deploy nginx
- Create Service

Command:  *#oc apply -f nginx-allin.yaml*

The content of "nginx-deploy.yaml"
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: nginx-demo
  labels:
    app: nginxsrv
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginxsrv
  template:
    metadata:
      labels:
        app: nginxsrv
    spec:
      securityContext:
        runAsNonRoot: true
        seccompProfile:
          type: RuntimeDefault

      # ?? Init container to ensure index.html exists
      initContainers:
        - name: init-web
          image: busybox:1.36
          command: ["sh", "-c"]
          args:
            - |
              set -eu
              dst=/usr/share/nginx/html/index.html
              if [ ! -s "$dst" ]; then
                echo '<!doctype html><title>Hello</title><h1>?? Welcome to NGINX</h1><p>Auto-created by initContainer</p>' > "$dst"
                echo "Created default index.html"
              else
                echo "index.html already exists � skipping"
              fi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: webdata
              mountPath: /usr/share/nginx/html
              readOnly: false

      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:stable-alpine
          ports:
            - containerPort: 8080
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: webdata
              mountPath: /usr/share/nginx/html
              readOnly: false
          readinessProbe:
            httpGet:
              path: /index.html
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 20
          livenessProbe:
            httpGet:
              path: /index.html
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 30
      volumes:
        - name: webdata
          persistentVolumeClaim:
            claimName: nginx-pvc

```


# Creating PVC
*#oc apply -f nginx-pvc.yaml*


The content of "nginx-pvc.yaml"

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-pvc
  namespace: nginx-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: infoscale-concat-sc
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi 
```

delete pvc
oc delete pvc nginx-vol -n nginx-demo



create deploy
oc apply -f nginx-deploy.yaml

delete deploy
oc delete -f nginx-deploy.yaml -n nginx-demo



create service
oc apply -f nginx-service.yaml

delete service
oc delete svc nginx-service -n nginx-demo

get service
oc get svc nginx-service -n nginx-demo



Test by browser
1. oc -n nginx-demo expose svc nginx-service
2. oc -n nginx-demo get route
3. Open browser with URL nginx-service-nginx-demo.apps.xxx

Get pod
oc -n nginx-demo get pod -o wide

Test edit html from pods
oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-1 at $(date)</h1>" > /usr/share/nginx/html/index.html'

oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-2 at $(date)</h1>" > /usr/share/nginx/html/index.html'

oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-3 at $(date)</h1>" > /usr/share/nginx/html/index.html'



Edit html
echo '<!doctype html><title>Hello</title><h1>Updated from shell</h1><p>Time: '$(date)'</p>' > /usr/share/nginx/html/index.html


To show all pod mount same pvc


df -k    look for /usr/share/nginx/html
