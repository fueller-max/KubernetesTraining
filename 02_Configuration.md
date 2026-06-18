###  Configuration
 ####   **ConfigMaps and Secrets**

 * **ConfigMaps**: for non-sensitive data in key-value pairs 
 * **Secrets**: for sensitive data like password, API keys, TLS cers

The convenient way in production is to create a file with configuration:

```yaml
#configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-declarative
data:
  # property-like keys; each key maps to a simple value
 database.url: "jdbc:mysql://db.example.com:3306/mydb"
 ui.theme: "dark"
```

```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f configmap.yaml
configmap/app-config-declarative created
```

Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.

```yaml
#secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: api-key
type: Opaque
stringData:  # used for plain text
  key: "api-key"
```

```bash
deploy@lab12-kub-master-1:~/training$ deploy@lab12-kub-master-1:~/training$ kubectl apply -f secret.yaml
secret/api-key created
```

**Example**

Create a secret with db password:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:
  password: "db_password"
```

```bash
deploy@lab12-kub-master-1:~/training$ deploy@lab12-kub-master-1:~/training$ kubectl apply -f db-credentials.yaml
secret/db-credentials created
```

Create a Pods which uses ConfigMaps and Secrets to inject data:

```yaml
#pod-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
spec:
  containers:
  - name: demo-container
    image: busybox
    command: ["/bin/sh", "-c", "env && sleep 3600"]
    env:
        #Inject a value from Config Map:
      - name: THEME
        valueFrom:
          configMapKeyRef:
            name: app-config-declarative # Config Map`s name
            key: ui.theme                # the key to pull the value from
        #Inject a value from Secret:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials         # the Secret`s name
            key: password                # the key to pull the value from
  restartPolicy: Never
```

Check whether the values have been applied:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl logs config-demo-pod
KUBERNETES_SERVICE_PORT=443
KUBERNETES_PORT=tcp://10.96.0.1:443
HOSTNAME=config-demo-pod
SHLVL=1
HOME=/root
WEB_APP_SERVICE_SERVICE_HOST=10.103.255.137
WEB_APP_SERVICE_SERVICE_PORT=80
WEB_APP_SERVICE_PORT=tcp://10.103.255.137:80
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
WEB_APP_SERVICE_PORT_80_TCP_ADDR=10.103.255.137
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
WEB_APP_SERVICE_PORT_80_TCP_PORT=80
WEB_APP_SERVICE_PORT_80_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
WEB_APP_SERVICE_PORT_80_TCP=tcp://10.103.255.137:80
DB_PASSWORD=db_password
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
THEME=dark
```

> **WEB_APP_SERVICE_SERVICE_HOST and others variable appears because Kubernetes automatically injects environment variables for every active Service into every new Pod created in the same namespace. This built-in service discovery mechanism is independent of your pod's specific YAML configuration**

Lets`s start pod in its own namespace:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl create namespace config-demo-pod
namespace/config-demo-pod created
```

Add the namespace in the config:

```yaml
#pod-config.yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo-pod
  namespace: config-demo-pod
spec:
  containers:
  - name: demo-container
    image: busybox
    command: ["/bin/sh", "-c", "env && sleep 3600"]
    env:
        #Inject a value from Config Map:
      - name: THEME
        valueFrom:
          configMapKeyRef:
            name: app-config-declarative # Config Map`s name
            key: ui.theme                # the key to pull the value from
        #Inject a value from Secret:
      - name: DB_PASSWORD
        valueFrom:
          secretKeyRef:
            name: db-credentials         # the Secret`s name
            key: password                # the key to pull the value from
  restartPolicy: Never
```

> **Important!**
Kubernetes cannot cross namespace boundaries to fetch ConfigMaps or Secrets for a Pod. Because the Pod is starting in a different namespace, it cannot find the app-config-declarative ConfigMap or the db-credentials Secret left behind in the default namespace. 

Copy configMap and secret to out new namespace and verify:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl get secret db-credentials --namespace=default -o yaml | sed '/namespace:/d' | kubectl apply --namespace=config-demo-pod -f -
secret/db-credentials created
deploy@lab12-kub-master-1:~/training$ kubectl get configmap,secret -n config-demo-pod
NAME                               DATA   AGE
configmap/app-config-declarative   2      39s
configmap/kube-root-ca.crt         1      14m

NAME                    TYPE     DATA   AGE
secret/db-credentials   Opaque   1      9s
```

Now it works as supposed in a new namespace:

```
deploy@lab12-kub-master-1:~/training$ kubectl logs config-demo-pod -n config-demo-pod
KUBERNETES_PORT=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT=443
HOSTNAME=config-demo-pod
SHLVL=1
HOME=/root
KUBERNETES_PORT_443_TCP_ADDR=10.96.0.1
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP=tcp://10.96.0.1:443
KUBERNETES_SERVICE_PORT_HTTPS=443
DB_PASSWORD=db_password
KUBERNETES_SERVICE_HOST=10.96.0.1
PWD=/
THEME=dark
```

####   **Volumes**

To make config data appear as files inside the containers we can use Volumes.

Lets create a properties file with config data and create a config map from an entire properties file:
```bash
deploy@lab12-kub-master-1:~/training$ echo "retires = 3" > config.properties
deploy@lab12-kub-master-1:~/training$ kubectl create configmap app-config-file --from-file=config.properties -n volume-demo-pod
configmap/app-config-file created
```

Use a Volume to inject the file inside the container:

```yaml
#pod-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo-pod
  namespace: volume-demo-pod
spec:
  containers:
  - name: demo-container
    image: busybox
    command: ["/bin/sh", "-c", "cat /etc/config/config.properties && sleep 3600"]
    volumeMounts:
    - name: config-volume        # A reference to the volume defined below
      mountPath: /etc/config     # The directory inside the container
  volumes:
  - name: config-volume
    configMap:
      name: app-config-file      # The source of the data is ConfigMap  
  restartPolicy: Never
```

Start and check the pod`s logs

```bash
deploy@lab12-kub-master-1:~/training$ kubectl apply -f pod-volume.yaml
pod/volume-demo-pod created

deploy@lab12-kub-master-1:~/training$ kubectl logs volume-demo-pod -n volume-demo-pod
retires = 3
```

Also check inside the running container:

```bash
deploy@lab12-kub-master-1:~/training$ kubectl exec -n volume-demo-pod --stdin --tty volume-demo-pod -- sh

/ # ls /etc/config
config.properties

/ # cat /etc/config/config.properties
retires = 3
```


