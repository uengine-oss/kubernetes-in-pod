# Kubernetes in a Pod

KIND (Kuberntes in Docker) is very useful when you only need to create a cluster for testing or education purposes. If you provide them in a Pod, you can easily provide kubernetes clusters for many developers or operators for debugging/education/testing purposes. 

Create following pod firstly:
```
apiVersion: v1
kind: Pod
metadata:
  name: kind-cluster
  labels:
    app: kind-cluster
spec:
  containers:
  - image: ghcr.io/jinyoung/kind-cluster:v4
    imagePullPolicy: Always
    name: kind-cluster
    stdin: true
    tty: true
    args:
    - /bin/bash
    env:
    - name: API_SERVER_ADDRESS
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
    volumeMounts:
    - mountPath: /var/lib/docker
      name: varlibdocker
    - mountPath: /lib/modules
      name: libmodules
      readOnly: true
    securityContext:
      privileged: true
    ports:
    - containerPort: 30001
      name: api-server-port
      protocol: TCP
    - containerPort: 80
      name: service-port
      protocol: TCP
    - containerPort: 443
      name: secure-service-port
      protocol: TCP
    - containerPort: 3000
      name: grafana-port
      protocol: TCP
    readinessProbe:
      failureThreshold: 15
      httpGet:
        path: /healthz
        port: api-server-port
        scheme: HTTPS
      initialDelaySeconds: 120
      periodSeconds: 20
      successThreshold: 1
      timeoutSeconds: 1
  volumes:
  - name: varlibdocker
    emptyDir: {}
  - name: libmodules
    hostPath:
      path: /lib/modules
```

Expose the pod as LoadBalancer:
```
kubectl expose po kind-cluster --port=80 --type=LoadBalancer
```
* Keep a note the external ip for later use.

After creating the Pod, attach to the bash:
```
kubectl exec -it kind-cluster -- /bin/bash
```

Wait until the Kind cluster is up by trying to set the Kubectl context connect to the Kind:
```
root@kind-cluster:/# kubectl cluster-info --context kind-kind
error: context "kind-kind" does not exist
root@kind-cluster:/# kubectl cluster-info --context kind-kind
error: context "kind-kind" does not exist
root@kind-cluster:/# kubectl cluster-info --context kind-kind
error: context "kind-kind" does not exist
root@kind-cluster:/# kubectl cluster-info --context kind-kind
error: context "kind-kind" does not exist
root@kind-cluster:/# kubectl cluster-info --context kind-kind

Kubernetes master is running at https://10.36.10.137:30001
KubeDNS is running at https://10.36.10.137:30001/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

```

To access the service outside the pod, you need to install Ambassador Ingress Provider firstly:
```
kubectl apply -f https://github.com/datawire/ambassador-operator/releases/latest/download/ambassador-operator-crds.yaml
kubectl apply -n ambassador -f https://github.com/datawire/ambassador-operator/releases/latest/download/ambassador-operator-kind.yaml
kubectl wait --timeout=180s -n ambassador --for=condition=deployed ambassadorinstallations/ambassador    # skippable
```

Create an Ingress for accessing the service:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: "www.service.com"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: <service name>
          servicePort: 80
```

After that, we need to annotate the Ingress so that the Ambassador pick up the service to it's operator:

```
kubectl annotate ingress example-ingress kubernetes.io/ingress.class=ambassador
```

Make sure that you have to configure the hosts file before you access to the URL www.service.com:


```
# vi /etc/hosts

<the external ip obtained from kind-cluster service>   www.service.com   
```


If you use Istio, you only need to expose the istio-ingressgateway:
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  namespace: istio-system
spec:
  rules:
  - host: "www.service.com"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: istio-ingressgateway
          servicePort: 80
```

And annotate the ingress to make effect for Ambassador:
```
kubectl annotate ingress example-ingress kubernetes.io/ingress.class=ambassador -n istio-system      # for istio-system
```

Finally, go to following URL with a browser:

```
www.service.com/<service paths>
```

Example configuration for istio and istio-related telemetry services:

- ingress.yaml
```
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: example-ingress
  namespace: istio-system
spec:
  rules:
  - host: "www.service.com"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: istio-ingressgateway
          servicePort: 80

  - host: "tracing.service.com"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: tracing
          servicePort: 80


  - host: "grafana.service.com"
    http:
      paths:
      - path: "/"
        backend:
          serviceName: grafana
          servicePort: 3000

```

- /etc/hosts file
```
34.146.166.15   www.service.com
34.146.166.15   tracing.service.com
34.146.166.15   grafana.service.com
```

# RoadMap
Integration with Theia-IDE, for providing an integrated development environment for complete kubernetes development.
