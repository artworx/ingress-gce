<!--
-----------------NOTICE------------------------
This file is referenced in code as
https://github.com/kubernetes/ingress/blob/master/docs/troubleshooting.md
Do not move it without providing redirects.
-----------------------------------------------
-->

# Troubleshooting


## Authentication to the Kubernetes API Server


A number of components are involved in the authentication process and the first step is to narrow
down the source of the problem, namely whether it is a problem with service authentication or with the kubeconfig file.
Both authentications must work:

```
+-------------+   service          +------------+
|             |   authentication   |            |
+  apiserver  +<-------------------+  ingress   |
|             |                    | controller |
+-------------+                    +------------+

```

__Service authentication__

The Ingress controller needs information from apiserver. Therefore, authentication is required, which can be achieved in two different ways:

1. _Service Account:_ This is recommended, because nothing has to be configured. The Ingress controller will use information provided by the system to communicate with the API server. See 'Service Account' section for details.

2. _Kubeconfig file:_ In some Kubernetes environments service accounts are not available. In this case a manual configuration is required. The Ingress controller binary can be started with the `--kubeconfig` flag. The value of the flag is a path to a file specifying how to connect to the API server. Using the `--kubeconfig` does not requires the flag `--apiserver-host`.
The format of the file is identical to `~/.kube/config` which is used by kubectl to connect to the API server. See 'kubeconfig' section for details.

3. _Using the flag `--apiserver-host`:_ Using this flag `--apiserver-host=http://localhost:8080` it is possible to specify an unsecure api server or reach a remote kubernetes cluster using [kubectl proxy](https://kubernetes.io/docs/user-guide/kubectl/kubectl_proxy/).
Please do not use this approach in production.

In the diagram below you can see the full authentication flow with all options, starting with the browser
on the lower left hand side.
```

Kubernetes                                                  Workstation
+---------------------------------------------------+     +------------------+
|                                                   |     |                  |
|  +-----------+   apiserver        +------------+  |     |  +------------+  |
|  |           |   proxy            |            |  |     |  |            |  |
|  | apiserver |                    |  ingress   |  |     |  |  ingress   |  |
|  |           |                    | controller |  |     |  | controller |  |
|  |           |                    |            |  |     |  |            |  |
|  |           |                    |            |  |     |  |            |  |
|  |           |  service account/  |            |  |     |  |            |  |
|  |           |  kubeconfig        |            |  |     |  |            |  |
|  |           +<-------------------+            |  |     |  |            |  |
|  |           |                    |            |  |     |  |            |  |
|  +------+----+      kubeconfig    +------+-----+  |     |  +------+-----+  |
|         |<--------------------------------------------------------|        |
|                                                   |     |                  |
+---------------------------------------------------+     +------------------+
```


## Service Account
If using a service account to connect to the API server, Dashboard expects the file
`/var/run/secrets/kubernetes.io/serviceaccount/token` to be present. It provides a secret
token that is required to authenticate with the API server.

Verify with the following commands:

```shell
# start a container that contains curl
$ kubectl run test --image=tutum/curl -- sleep 10000

# check that container is running
$ kubectl get pods
NAME                   READY     STATUS    RESTARTS   AGE
test-701078429-s5kca   1/1       Running   0          16s

# check if secret exists
$ kubectl exec test-701078429-s5kca ls /var/run/secrets/kubernetes.io/serviceaccount/
ca.crt
namespace
token

# get service IP of master
$ kubectl get services
NAME         CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.0.0.1     <none>        443/TCP   1d

# check base connectivity from cluster inside
$ kubectl exec test-701078429-s5kca -- curl -k https://10.0.0.1
Unauthorized

# connect using tokens
$ TOKEN_VALUE=$(kubectl exec test-701078429-s5kca -- cat /var/run/secrets/kubernetes.io/serviceaccount/token)
$ echo $TOKEN_VALUE
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3Mi....9A
$ kubectl exec test-701078429-s5kca -- curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt -H  "Authorization: Bearer $TOKEN_VALUE" https://10.0.0.1
{
  "paths": [
    "/api",
    "/api/v1",
    "/apis",
    "/apis/apps",
    "/apis/apps/v1alpha1",
    "/apis/authentication.k8s.io",
    "/apis/authentication.k8s.io/v1beta1",
    "/apis/authorization.k8s.io",
    "/apis/authorization.k8s.io/v1beta1",
    "/apis/autoscaling",
    "/apis/autoscaling/v1",
    "/apis/batch",
    "/apis/batch/v1",
    "/apis/batch/v2alpha1",
    "/apis/certificates.k8s.io",
    "/apis/certificates.k8s.io/v1alpha1",
    "/apis/extensions",
    "/apis/extensions/v1beta1",
    "/apis/policy",
    "/apis/policy/v1alpha1",
    "/apis/rbac.authorization.k8s.io",
    "/apis/rbac.authorization.k8s.io/v1alpha1",
    "/apis/storage.k8s.io",
    "/apis/storage.k8s.io/v1beta1",
    "/healthz",
    "/healthz/ping",
    "/logs",
    "/metrics",
    "/swaggerapi/",
    "/ui/",
    "/version"
  ]
}
```

If it is not working, there are two possible reasons:

1. The contents of the tokens are invalid. Find the secret name with `kubectl get secrets | grep service-account` and
delete it with `kubectl delete secret <name>`. It will automatically be recreated.

2. You have a non-standard Kubernetes installation and the file containing the token
may not be present. The API server will mount a volume containing this file, but
only if the API server is configured to use the ServiceAccount admission controller.
If you experience this error, verify that your API server is using the ServiceAccount
admission controller. If you are configuring the API server by hand, you can set
this with the `--admission-control` parameter. Please note that you should use
other admission controllers as well. Before configuring this option, you should
read about admission controllers.

More information:

* [User Guide: Service Accounts](http://kubernetes.io/docs/user-guide/service-accounts/)
* [Cluster Administrator Guide: Managing Service Accounts](http://kubernetes.io/docs/admin/service-accounts-admin/)

## Kubeconfig
If you want to use a kubeconfig file for authentication, create a deployment file similar to the one below:

*Note:* the important part is the flag `--kubeconfig=/etc/kubernetes/kubeconfig.yaml`.


```
kind: Service
apiVersion: v1
metadata:
  name: nginx-default-backend
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io
spec:
  ports:
  - port: 80
    targetPort: http
  selector:
    app: nginx-default-backend

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nginx-default-backend
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io
spec:
  replicas: 1
  template:
    metadata:
      labels:
        k8s-addon: ingress-nginx.addons.k8s.io
        app: nginx-default-backend
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - name: default-http-backend
        image: gcr.io/google_containers/defaultbackend:1.0
        volumeMounts:
        - mountPath: /etc/kubernetes
          name: kubeconfig
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        resources:
          limits:
            cpu: 10m
            memory: 20Mi
          requests:
            cpu: 10m
            memory: 20Mi
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP

---

kind: ConfigMap
apiVersion: v1
metadata:
  name: ingress-nginx
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io

---

kind: Service
apiVersion: v1
metadata:
  name: ingress-nginx
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io
spec:
  type: LoadBalancer
  selector:
    app: ingress-nginx
  ports:
  - name: http
    port: 80
    targetPort: http
  - name: https
    port: 443
    targetPort: https

---

kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: ingress-nginx
  labels:
    k8s-addon: ingress-nginx.addons.k8s.io
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: ingress-nginx
        k8s-addon: ingress-nginx.addons.k8s.io
    spec:
      terminationGracePeriodSeconds: 60
      containers:
      - image: gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.14
        name: ingress-nginx
        imagePullPolicy: Always
        ports:
          - name: http
            containerPort: 80
            protocol: TCP
          - name: https
            containerPort: 443
            protocol: TCP
        livenessProbe:
          httpGet:
            path: /healthz
            port: 10254
            scheme: HTTP
          initialDelaySeconds: 30
          timeoutSeconds: 5
        env:
          - name: POD_NAME
            valueFrom:
              fieldRef:
                fieldPath: metadata.name
          - name: POD_NAMESPACE
            valueFrom:
              fieldRef:
                fieldPath: metadata.namespace
        args:
        - /nginx-ingress-controller
        - --default-backend-service=$(POD_NAMESPACE)/nginx-default-backend
        - --configmap=$(POD_NAMESPACE)/ingress-nginx
        - --kubeconfig=/etc/kubernetes/kubeconfig.yaml
      volumes:
      - name: "kubeconfig"
        hostPath:
          path: "/etc/kubernetes/"
```
