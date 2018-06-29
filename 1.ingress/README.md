# Ingress and TLS

## Init helm and install Ingress controller
- https://github.com/kubernetes/ingress-nginx
- https://github.com/kubernetes/charts/tree/master/stable/nginx-ingress
- `helm init`
- `helm install stable/nginx-ingress --name=nginx-ingress --namespace=kube-system --set rbac.create=true --set controller.service.externalIPs="{Worker Public IP,Worker Public IP}"`
- ...
```
cat > home-ingress.yaml << _EOF_
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myhome-ingress
spec:
  rules:
  - host: jgkong.seo01.containers.appdomain.cloud
    http:
      paths:
      - path: /
        backend:
          serviceName: simplehome
          servicePort: 80
_EOF_
```
- http://jgkong.seo01.containers.appdomain.cloud/ 에서 확인

## Create TLS certificates
- `mkdir -p letsencrypt/{etc,var/lib}`
- `docker run  -it --rm --name certbot -v "$(pwd)/letsencrypt/etc:/etc/letsencrypt" -v "$(pwd)/letsencrypt/var/lib:/var/lib/letsencrypt" certbot/certbot certonly --server https://acme-v02.api.letsencrypt.org/directory --manual`
- `mkdir -p .well-known/acme-challenge/`
- `echo "1gC_ahidUGBdOxNIhNWoTzbBTdn0K8f-axhIAczf_gc.JiWd-JcRdFtPARc5KelJvzhFDz0EdJm16uBuN0iiLaU" > .well-known/acme-challenge/1gC_ahidUGBdOxNIhNWoTzbBTdn0K8f-axhIAczf_gc`
- `docker build -t registry.au-syd.bluemix.net/jgkong/acme-challenge .`
- `docker push registry.au-syd.bluemix.net/jgkong/acme-challenge`
- `kubectl run acme-challenge --image=registry.au-syd.bluemix.net/jgkong/acme-challenge --port 80`
- `kubectl expose deployment acme-challenge`
- `kubectl edit ingress myhome-ingress`
```
      - backend:
          serviceName: acme-challenge
          servicePort: 80
        path: /.well-known/acme-challenge
```
- `kubectl create secret tls simplehome-tls --key letsencrypt/etc/live/simplehome.arcy.me/privkey.pem --cert letsencrypt/etc/live/simplehome.arcy.me/fullchain.pem`
- `kubectl get secret simplehome-tls`
- `kubectl edit ingress myhome-ingress`
```
  tls:
  - hosts:
    - simplehome.arcy.me
    secretName: simplehome-tls
```


## Cert-manager
- `kubectl edit ingress myhome-ingress`
    - TLS 관련 삭제
- https://github.com/kubernetes/charts/tree/master/stable/cert-manager
- `helm install --name cert-manager --namespace kube-system stable/cert-manager --set ingressShim.defaultIssuerName=letsencrypt-prod --set ingressShim.defaultIssuerKind=ClusterIssuer`
- ...
```
cat > letsencrypt.yaml << _EOF_
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: jgkong@kr.ibm.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging
    # Enable the HTTP-01 challenge provider
    http01: {}

---
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
  namespace: default
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: jgkong@kr.ibm.com
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod
    # Enable the HTTP-01 challenge provider
    http01: {}
_EOF_
```
- `kubectl apply -f letsencrypt.yaml`
- `kubectl patch ingress myhome-ingress -p='{"metadata": {"annotations": {"kubernetes.io/tls-acme": "true"}}}'`
- `kubectl patch ingress myhome-ingress -p='{"spec": {"tls": [{"hosts": ["simplehome.arcy.me"], "secretName": "simplehome-tls"}]}}'`
- `kubectl get secret simplehome-tls`
- `kubectl describe certificate simplehome-tls`

