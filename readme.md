# Lab: Deploy nginx into 3 worker nodes.
File: nginx-allin.yaml will perform following
- Create namespace "nginx-demo"
- Create PVC from storage class infoscale-concat-sc
- Deploy nginx
- Create Service

Command:  *#oc apply -f nginx-allin.yaml*

The content of "nginx-allin.yaml"
```
apiVersion: v1
kind: Namespace
metadata:
  name: nginx-demo
  labels:
    kubernetes.io/metadata.name: nginx-demo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nginx-vol
  namespace: nginx-demo
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: infoscale-concat-sc
  volumeMode: Filesystem
  resources:
    requests:
      storage: 5Gi
---
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
      initContainers:
        - name: init-web
          image: busybox:1.36
          command: ["sh","-c"]
          args:
            - |
              if [ ! -f /work/index.html ]; then
                echo "<h1>Hello from PVC - InfoScale RWX</h1>" > /work/index.html
              fi
          volumeMounts:
            - name: webdata
              mountPath: /work
      containers:
        - name: nginx
          image: nginxinc/nginx-unprivileged:1.27-alpine
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: webdata
              mountPath: /usr/share/nginx/html
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 20
      volumes:
        - name: webdata
          persistentVolumeClaim:
            claimName: nginx-vol
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: nginx-demo  
spec:
  type: ClusterIP
  selector:
    app: nginxsrv
  ports:
    - port: 80
      targetPort: 8080

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

#### deleting pvc
```
oc delete pvc nginx-vol -n nginx-demo
```



#### Create nginx-deploy
```
oc apply -f nginx-deploy.yaml
```

#### Deleting ngin-deploy
```
oc delete -f nginx-deploy.yaml -n nginx-demo
```



#### Createing nginx-service
```
oc apply -f nginx-service.yaml
```

#### Deleting then  nginx-service
```
oc delete svc nginx-service -n nginx-demo
```

#### Get the nginx-service
```
#oc get svc nginx-service -n nginx-demo
```


## Tesing nginx 3x worknode by deployment nginx-allin.yaml. include
- Create namespace: nginx-demo
- Create PVC: nginx-vol 5Gi
- Create Service: nginx-service 
- 

#### Deploy the nginx-allin.yaml
```
oc apply -f nginx-allin.yaml 
```


#### Testing by expose nginx-service and route then using the browser access to the nginx
1. oc -n nginx-demo expose svc nginx-service
2. oc -n nginx-demo get route
3. Open browser with URL nginx-service-nginx-demo.apps.xxx

#### Get pod information
```
oc -n nginx-demo get pod -o wide
```

#### Test edit html from pods
```
oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-1 at $(date)</h1>" > /usr/share/nginx/html/index.html'

oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-2 at $(date)</h1>" > /usr/share/nginx/html/index.html'

oc -n nginx-demo exec -it <pod-name> -c nginx -- sh -c 'echo "<h1>Changed from POD-3 at $(date)</h1>" > /usr/share/nginx/html/index.html'
```

#### Modify index.html
```
echo '<!doctype html><title>Hello</title><h1>Updated from shell</h1><p>Time: '$(date)'</p>' > /usr/share/nginx/html/index.html
```

## To show all pod mount same pvc
```
df -k    look for /usr/share/nginx/html
```



## Storage Consumtion

Ex. using vxdg -g <dg_name> -u h free
```
#oc exec -it infoscale-sds-1000-5b8c8147c47d39ff-nn78n -n infoscale-vtas -- vxdg -g vrts_kube_dg-1000 -u h free
```

```
DISK         DEVICE       TAG          OFFSET    LENGTH    FLAGS
ibm_shark0_0 ibm_shark0_0 ibm_shark0_0 8456448   5.93g     -
ibm_shark0_1 ibm_shark0_1 ibm_shark0_1 8456448   5.93g     -
ibm_shark0_2 ibm_shark0_2 ibm_shark0_2 6359296   6.93g     -
ibm_shark0_3 ibm_shark0_3 ibm_shark0_3 4961280   7.60g     -
ibm_shark0_4 ibm_shark0_4 ibm_shark0_4 3631104   8.23g     -
ibm_shark0_5 ibm_shark0_5 ibm_shark0_5 3631104   8.23g     -
ibm_shark0_6 ibm_shark0_6 ibm_shark0_6 3631104   8.23g     -
ibm_shark0_7 ibm_shark0_7 ibm_shark0_7 699136    682.62m   -
ibm_shark0_7 ibm_shark0_7 ibm_shark0_7 6291456   6.96g     -
ibm_shark0_8 ibm_shark0_8 ibm_shark0_8 3563264   8.26g     -
ibm_shark0_9 ibm_shark0_9 ibm_shark0_9 3563264   8.26g     -
```