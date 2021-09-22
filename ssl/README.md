# SSL Section

## Create Self-Signed Cert for Wildcard Domain

`openssl.cnf` is uncommented to include the `alt_names` below:
```
[alt_names]
DNS.1 = ingress.demo
DNS.2 = *.ingress.demo
```

Use the following steps to generate the self-signed cert:
```
openssl genrsa -out ingress.demo.key 2048
openssl rsa -in ingress.demo.key -out ingress.demo.key.pem
openssl req -new -key ingress.demo.key.pem -out ingress.demo.request.csr
openssl x509 -req -extensions v3_req -days 730 -in ingress.demo.request.csr -signkey ingress.demo.key.pem -out ingress.demo.crt -extfile openssl.cnf
```

Generate the yaml file for the secret and apply to the appropriate namespaces:
```
kubectl create secret tls ingress-demo --key ingress.demo.key --cert ingress.demo.crt --dry-run=client -o=yaml > ingress-demo-secret.yaml
kubectl apply -f ingress-demo-secret.yaml
kubectl apply -f ingress-demo-secret.yaml -n monitoring
kubectl apply -f ingress-demo-secret.yaml -n nginx-ingress
```