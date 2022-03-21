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
 
nikto -h www -p 80 -nossl
  ... 
```

## Verify the nginx modsec logs
```
kubectl logs nginx-modsec 

2022/03/21 21:33:00 [error] 27#27: *161 [client 10.0.1.171] ModSecurity: Access denied with code 403 (phase 2). Matched "Operator `Ge' with parameter `5' against variable `TX:ANO
MALY_SCORE' (Value: `15' ) [file "/etc/modsecurity.d/owasp-crs/rules/REQUEST-949-BLOCKING-EVALUATION.conf"] [line "80"] [id "949110"] [rev ""] [msg "Inbound Anomaly Score Exceede
d (Total Score: 15)"] [data ""] [severity "2"] [ver "OWASP_CRS/3.3.2"] [maturity "0"] [accuracy "0"] [tag "modsecurity"] [tag "application-multi"] [tag "language-multi"] [tag "pl
atform-multi"] [tag "attack-generic"] [hostname "10.0.1.48"] [uri "/modules/vWar_Account/includes/functions_common.php"] [unique_id "1647898380"] [ref ""], client: 10.0.1.171, se
rver: www, request: "GET /modules/vWar_Account/includes/functions_common.php?vwar_root2=http://cirt.net/rfiinc.txt? HTTP/1.1", host: "nginx-modsec"
10.0.1.171 - - [21/Mar/2022:21:33:00 +0000] "GET /modules/vWar_Account/includes/functions_common.php?vwar_root2=http://cirt.net/rfiinc.txt? HTTP/1.1" 403 153 "-" "Mozilla/5.00 (N
ikto/2.1.5) (Evasions:None) (Test:005467)" "-"
{"transaction":{"client_ip":"10.0.1.171","time_stamp":"Mon Mar 21 21:33:00 2022","server_id":"1a60b8944f7a3429eacd57e46a84aa26c646ca7f","client_port":55926,"host_ip":"10.0.1.48",
"host_port":80,"unique_id":"1647898380","request":{"method":"GET","http_version":1.1,"uri":"/modules/vWar_Account/includes/functions_common.php?vwar_root2=http://cirt.net/rfiinc.
txt?","headers":{"Connection":"Keep-Alive","User-Agent":"Mozilla/5.00 (Nikto/2.1.5) (Evasions:None) (Test:005467)","Host":"nginx-modsec"}},"response":{"body":"<html>\r\n<head><ti
tle>403 Forbidden</title></head>\r\n<body>\r\n<center><h1>403 Forbidden</h1></center>\r\n<hr><center>nginx/1.20.2</center>\r\n</body>\r\n</html>\r\n","http_code":403,"headers":{"
Server":"nginx/1.20.2","Date":"Mon, 21 Mar 2022 21:33:00 GMT","Content-Length":"153","Content-Type":"text/html","Connection":"keep-alive"}},"producer":{"modsecurity":"ModSecurity
...

```

