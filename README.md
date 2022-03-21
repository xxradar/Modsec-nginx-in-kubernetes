# Modsecurity w/ OWASP CRS Nginx demo in kubernetes
## Setting up a webserver
Spin up a nginx webserver
```
kubectl run --image nginx www
```
Expose the webserver
```
kubectl expose po www --port 80
```

## Deploy a Nginx Modsec proxy
```
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-modsec
  labels:
    purpose: owasp-modsec
spec:
  containers:
  - name: nginx-modsec
    image: owasp/modsecurity-crs:3.3.2-nginx
    env:
    - name: PROXY_SSL
      value: "on"
    - name: BACKEND
      value: http://www.default.svc.cluster.local
    - name: MODSEC_AUDIT_ENGINE
      value: "On"
    - name: DNS_SERVER
      value: 10.96.0.10
    - name: SERVER_NAME
      value: www
EOF
```
Expose the Nginx-Modsec proxy
```
kubectl expose po nginx-modsec --port 80
```
## Testing
```
kubectl run -it --rm --image xxradar/hackon test
```
Inside the pod 
```
curl -H "Host: www" http://nginx-modsec   #200 OK
 ....
 
curl -H "Host: www" 'http://nginx-modsec/?test=<script>alert("Hacked")</script>' #403 blocking
 ...
```

