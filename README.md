# Define a liveness HTTP request 

Another kind of liveness probe uses an HTTP GET request. Here is the configuration file for a Pod that runs a container based on the registry.k8s.io/e2e-test-images/agnhost image.

### Pod Configuration for Liveness HTTP-GET

```yaml
apiVersion: v1
kind: Pod
metadata:
 name: health2
 labels:
  app: v1
spec:
 containers:
  - name: health2
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    imagePullPolicy: IfNotPresent
    args:
     - liveness
    livenessProbe:
     httpGet:
      path: /healthz
      port: 8080
      httpHeaders:
       - name: Custom-Header
         value: Awesome
     initialDelaySeconds: 10
     periodSeconds: 5
```
In the configuration file, you can see that the Pod has a single container. The periodSeconds field specifies that the kubelet should perform a liveness probe every 3 seconds. The initialDelaySeconds field tells the kubelet that it should wait 3 seconds before performing the first probe. To perform a probe, the kubelet sends an HTTP GET request to the server that is running in the container and listening on port 8080. If the handler for the server's /healthz path returns a success code, the kubelet considers the container to be alive and healthy. If the handler returns a failure code, the kubelet kills the container and restarts it.
Any code greater than or equal to 200 and less than 400 indicates success. Any other code indicates failure.

You can see the source code for the server in [server.go](https://github.com/kubernetes/kubernetes/blob/master/test/images/agnhost/liveness/server.go).

For the first 10 seconds that the container is alive, the /healthz handler returns a status of 200. After that, the handler returns a status of 500.
```go
http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
    duration := time.Now().Sub(started)
    if duration.Seconds() > 10 {
        w.WriteHeader(500)
        w.Write([]byte(fmt.Sprintf("error: %v", duration.Seconds())))
    } else {
        w.WriteHeader(200)
        w.Write([]byte("ok"))
    }
})
```
After 10 seconds, view Pod events to verify that liveness probes have failed and the container has been restarted:
```
kubectl describe pod health2
```

The kubelet starts performing health checks 3 seconds after the container starts. So the first couple of health checks will succeed. But after 10 seconds, the health checks will fail, and the kubelet will kill and restart the container.

To try the HTTP liveness check, create a Pod:
```
kubectl apply -f https://k8s.io/examples/pods/probe/http-liveness.yaml
```
In releases after v1.13, local HTTP proxy environment variable settings do not affect the HTTP liveness probe.

