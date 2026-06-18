 ### *Workloads & Scheduling*

 #### **Deployments**

 **Deployments** are the standard way to tun stateless applications
 
 Rolling Updates - the default strategy, incrementally replaces old Pods with new ones. 

 Controlled by:
 - **maxUnavailable**: the maximum number of Pods that can be unavailable during update
 - **maxSurge**: the maximum number of new Pods that can be created over the desired replica count


**Example**

First create a Deployment using older version on Nginx container:
```yaml
#deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.24.0
        ports:
        - containerPort: 80
```
Apply the Deployment:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f deployment.yaml
deployment.apps/nginx-deployment created
```

The easiest way to trigger update is to change container`s version:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl set image deployment/nginx-deployment nginx=nginx:1.25.0
deployment.apps/nginx-deployment image updated
```

Check the status of the update:
```bash
deploy@lab12-kub-master-1:~/training$ kubectl rollout status deployment/nginx-deployment
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "nginx-deployment" rollout to finish: 1 old replicas are pending termination...
deployment "nginx-deployment" successfully rolled out
```


Check rollout history:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl rollout history deployment/nginx-deployment
deployment.apps/nginx-deployment
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

Undo the last rollback
```bash
deploy@lab12-kub-master-1:~/training$ kubectl rollout undo deployment/nginx-deployment
deployment.apps/nginx-deployment rolled back
```

Or go to the specific rollback:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl rollout undo deployment/nginx-deployment --to-revision=1
```

See the details of revisions:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl rollout history deployment/nginx-deployment --revision=2
deployment.apps/nginx-deployment with revision #2
Pod Template:
  Labels:       app=nginx
        pod-template-hash=6cfdddf5f5
  Containers:
   nginx:
    Image:      nginx:1.25.0
 
deploy@lab12-kub-master-1:~/training$ kubectl rollout history deployment/nginx-deployment --revision=3
deployment.apps/nginx-deployment with revision #3
Pod Template:
  Labels:       app=nginx
        pod-template-hash=7fc58ffbbb
  Containers:
   nginx:
    Image:      nginx:1.24.0
    
```

