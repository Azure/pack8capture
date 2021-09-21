# Capture Manager

## Installation

On a Kubernetes cluster:

```console
# install the capture manager operator
helm upgrade --install capture-manager charts/capture-manager -n capture-system --create-namespace 
# install the capture stream server
helm upgrade --install capture-stream-server charts/capture-stream-server -n capture-system --create-namespace
```

## Deploy Sample Application

> We use `serviceAccount=capture-system-sa` in-order to be able to GET/WATCH the ConfigMap in the namespace `capture-system`.

```console
# install the sample application that includes the capture agent sidecar
kubectl create -f echoserver-deploy.yaml -n capture-system
```

## Start a capture by deploying the Rule CR

```console
kubectl create -f echoserver-capture-rule.yaml -n capture-system
```

Send a request to the echoserver container using the Load Balancer IP:

```console
# jq is optional for pretty printing the output
curl http://$(kubectl get svc -n capture-system echoserver -ojsonpath="{.status.loadBalancer.ingress[0].ip}") -d '{"hello":"world"}' -H "Content-type:application/json" | jq 
```

```json
{
  "host": {
    "hostname": "20.72.94.248",
    "ip": "::ffff:10.240.0.4",
    "ips": []
  },
  "http": {
    "method": "POST",
    "baseUrl": "",
    "originalUrl": "/",
    "protocol": "http"
  },
  "request": {
    "params": {
      "0": "/"
    },
    "query": {},
    "cookies": {},
    "body": {
      "hello": "world"
    },
    "headers": {
      "host": "20.72.94.248",
      "user-agent": "curl/7.77.0",
      "accept": "*/*",
      "content-type": "application/json",
      "content-length": "17"
    }
  },
  "environment": {
    "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
    "HOSTNAME": "echoserver-5dcc4f9685-5gjqp",
    "NODE_VERSION": "14.17.1",
    "YARN_VERSION": "1.22.5",
    "PORT": "80",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_SERVICE_HOST": "10.0.9.85",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_PORT_443_TCP": "tcp://10.0.9.85:443",
    "KUBERNETES_PORT": "tcp://10.0.0.1:443",
    "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
    "CAPTURE_STREAM_SERVER_SVC_SERVICE_PORT": "9890",
    "CAPTURE_STREAM_SERVER_SVC_PORT": "tcp://10.0.5.35:9890",
    "CAPTURE_STREAM_SERVER_SVC_PORT_9890_TCP": "tcp://10.0.5.35:9890",
    "CAPTURE_STREAM_SERVER_SVC_PORT_9890_TCP_PROTO": "tcp",
    "CAPTURE_MANAGER_METRICS_SERVICE_PORT_HTTP_METRICS": "8383",
    "CAPTURE_MANAGER_METRICS_PORT_8686_TCP_PORT": "8686",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_PORT_443_TCP_ADDR": "10.0.9.85",
    "ECHOSERVER_SERVICE_HOST": "10.0.144.37",
    "KUBERNETES_PORT_443_TCP": "tcp://10.0.0.1:443",
    "CAPTURE_MANAGER_METRICS_SERVICE_PORT": "8383",
    "ECHOSERVER_PORT": "tcp://10.0.144.37:80",
    "ECHOSERVER_PORT_80_TCP_PORT": "80",
    "KUBERNETES_SERVICE_PORT": "443",
    "CAPTURE_MANAGER_METRICS_SERVICE_HOST": "10.0.53.254",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_PORT": "tcp://10.0.9.85:443",
    "KUBERNETES_SERVICE_PORT_HTTPS": "443",
    "CAPTURE_MANAGER_METRICS_PORT_8383_TCP_PORT": "8383",
    "CAPTURE_MANAGER_METRICS_PORT_8686_TCP": "tcp://10.0.53.254:8686",
    "CAPTURE_MANAGER_METRICS_PORT_8686_TCP_ADDR": "10.0.53.254",
    "KUBERNETES_SERVICE_HOST": "10.0.0.1",
    "CAPTURE_MANAGER_METRICS_SERVICE_PORT_CR_METRICS": "8686",
    "CAPTURE_MANAGER_METRICS_PORT": "tcp://10.0.53.254:8383",
    "CAPTURE_MANAGER_METRICS_PORT_8686_TCP_PROTO": "tcp",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_PORT_443_TCP_PROTO": "tcp",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_PORT_443_TCP_PORT": "443",
    "ECHOSERVER_SERVICE_PORT": "80",
    "ECHOSERVER_PORT_80_TCP": "tcp://10.0.144.37:80",
    "ECHOSERVER_PORT_80_TCP_PROTO": "tcp",
    "ECHOSERVER_PORT_80_TCP_ADDR": "10.0.144.37",
    "KUBERNETES_PORT_443_TCP_PORT": "443",
    "KUBERNETES_PORT_443_TCP_ADDR": "10.0.0.1",
    "CAPTURE_STREAM_SERVER_SVC_PORT_9890_TCP_PORT": "9890",
    "CAPTURE_STREAM_SERVER_SVC_PORT_9890_TCP_ADDR": "10.0.5.35",
    "CAPTURE_MANAGER_METRICS_PORT_8383_TCP": "tcp://10.0.53.254:8383",
    "CAPTURE_STREAM_SERVER_SVC_SERVICE_HOST": "10.0.5.35",
    "CAPTURE_MANAGER_METRICS_PORT_8383_TCP_PROTO": "tcp",
    "CAPTURE_MANAGER_METRICS_PORT_8383_TCP_ADDR": "10.0.53.254",
    "CAPTUREMANAGER_ADMISSION_SERVER_SERVICE_SERVICE_PORT": "443",
    "CAPTURE_STREAM_SERVER_SVC_SERVICE_PORT_SERVER_GRPC": "9890",
    "HOME": "/root"
  }
}
```
## Adding the capture-agent to a pod

```diff
apiVersion: apps/v1                                             apiVersion: apps/v1
kind: Deployment                                                kind: Deployment
metadata:                                                       metadata:
  name: echoserver                                                name: echoserver
spec:                                                           spec:
  replicas: 3                                                     replicas: 3
  selector:                                                       selector:
    matchLabels:                                                    matchLabels:
      app: echoserver                                                 app: echoserver
  template:                                                       template:
    metadata:                                                       metadata:
      labels:                                                         labels:
        app: echoserver                                                 app: echoserver
    spec:                                                           spec:
                                                              >       serviceAccount: capture-system-sa
      containers:                                                     containers:
                                                              >       - args:
                                                              >         - --stream_cap_svr=capture-stream-server-svc.capture-
                                                              >         command:
                                                              >         - /capture-agent
                                                              >         env:
                                                              >         - name: CNA_POD_IP
                                                              >           valueFrom:
                                                              >             fieldRef:
                                                              >               fieldPath: status.podIP
                                                              >         - name: CNA_POD_UID
                                                              >           valueFrom:
                                                              >             fieldRef:
                                                              >               apiVersion: v1
                                                              >               fieldPath: metadata.uid
                                                              >         - name: CNA_NAMESPACE
                                                              >           valueFrom:
                                                              >             fieldRef:
                                                              >               apiVersion: v1
                                                              >               fieldPath: metadata.namespace
                                                              >         - name: CNA_APP_LABEL
                                                              >           value: echoserver
                                                              >         image: nmalhotra/capture-agent:latest
                                                              >         imagePullPolicy: Always
                                                              >         name: capture-agent
                                                              >         securityContext:
                                                              >           capabilities:
                                                              >             add:
                                                              >             - NET_ADMIN
                                                              >             - NET_RAW
                                                              >             drop:
                                                              >             - ALL
      - image: ealen/echo-server:latest                               - image: ealen/echo-server:latest
        imagePullPolicy: IfNotPresent                                   imagePullPolicy: IfNotPresent
        name: echoserver                                                name: echoserver
        ports:                                                          ports:
        - containerPort: 80                                             - containerPort: 80
        env:                                                            env:
        - name: PORT                                                    - name: PORT
          value: "80"                                                     value: "80"
---                                                             ---
apiVersion: v1                                                  apiVersion: v1
kind: Service                                                   kind: Service
metadata:                                                       metadata:
  name: echoserver                                                name: echoserver
spec:                                                           spec:
  ports:                                                          ports:
    - port: 80                                                      - port: 80
      targetPort: 80                                                  targetPort: 80
      protocol: TCP                                                   protocol: TCP
  type: LoadBalancer                                          |   type: LoadBalancer
  selector:                                                       selector:
    app: echoserver                                                 app: echoserver
```

## Creating a new capture `Rule` CR

From the sample echoserver capture rule:

```yaml
apiVersion: capturemanager.affirmednetworks.com/v1alpha1
kind: Rule
metadata:
  name: http-rule
  namespace: capture-system
spec:
# appLabel must select the app label of the application to be captured where the app label is of the format `app=echoserver`. This is a limitation in the capture manager that requires the `app=<app-name>` label in-order to watch for the capture rule configmap.
  appLabel: echoserver
  params:
    # filter in BPF format
    filter: "tcp port 80"
    # interface to capture on (e.g. eth0), To capture on all interfaces, use "any"
    interface: eth0
    # Capture limit like num bytes, num packets, num seconds
    limits:
      duration:
        # example captures for a duration of 5 minutes
        runTime: 5m
    # PCAP file management in the storage
    pcap:
      # Rollover cap files when the specified number is reached
      maxBackups: 5
      # close the file when the specified bytes are reached 
      maxSize: 2MB
      # Unique prefix to identify the capture in the capture directory on the capture stream server
      prefix: echoserver
```

### Lookup the capture files by exec-ing into the capture stream server container

```console
kubectl exec -it $(kubectl get pods -n capture-system --selector app=capture-stream-server -ojsonpath='{.items[0].metadata.name}') -n capture-system -- bash

[root@capture-stream-server-765fbf4dfb-mlnv9 /]# ls /captures
http-rule

[root@capture-stream-server-765fbf4dfb-mlnv9 /]# ls captures/http-rule/
echoserver-73c7edbb-7357-41cb-95e9-98aeafe642dd.pcap  echoserver-9dcc3735-5983-4845-adaf-983b2ab1f136.pcap  echoserver-c920fee0-1ae1-4912-b102-e4b9c0f5d960.pcap  echoserver-cc0f3a48-b1bc-4334-98a2-58b66d2a90d4.pcap
```

> This repo has been populated by an initial template to help get you started. Please
> make sure to update the content to build a great experience for community-building.

As the maintainer of this project, please make a few updates:

- Improving this README.MD file to provide a great experience
- Updating SUPPORT.MD with content about this project's support experience
- Understanding the security reporting process in SECURITY.MD
- Remove this section from the README

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft 
trademarks or logos is subject to and must follow 
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
