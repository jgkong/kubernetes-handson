# Build and run

## Create simple page
- `mkdir web`
- `cd web`
- `echo "<h1>Hello Kubernetes</h1>" > index.html`
- `cat index.html`
- ...
```
cat > Dockerfile << _EOF_
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
_EOF_
```
- `cat Dockerfile`
- `docker build -t registry.au-syd.bluemix.net/jgkong/simplehome .`
- `docker run -d --rm --name simplehome1 -p 8080:80 registry.au-syd.bluemix.net/jgkong/simplehome`
- http://localhost:8080 확인
- `docker stop simplehome1`

## Push image to IBM Cloud Container Registry
- `bx login`
- `bx cr region-set ap-south`
- `bx cr login`
- `docker push registry.au-syd.bluemix.net/jgkong/simplehome`

## Run on IBM Cloud Kubernetes Service
- `bx cs region-set ap-north`
- `bx cs clusters`
- `bx cs cluster-config jgkong`
- `bx cs workers jgkong`
    - 위에서 worker Public IP 확인
- `kubectl get pod`
- `kubectl get deployment`
- `export KUBECONFIG=/Users/arcturus/.bluemix/plugins/container-service/clusters/jgkong/kube-config-seo01-jgkong.yml`
- `kubectl run simplehome --image=registry.au-syd.bluemix.net/jgkong/simplehome --port 80`
- `kubectl get deployment`
- `kubectl describe deployment`
- `kubectl get pod`
- `kubectl describe pod`
- `kubectl delete service simplehome`
- `kubectl delete service simplehome-nodeport`
- `kubectl delete deployment simplehome`

## Expose service
- `kubectl expose deployment simplehome`
- `kubectl get service`
- `kubectl expose --name simplehome-nodeport deployment simplehome --type=NodePort`
- `kubectl get service`
- http://<worker Public IP>:<NodePort번호>/ 확인

# Run with yaml
- ...
```
cat > simplehome.yaml << _EOF_
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: simplehome
  name: simplehome
  namespace: default
spec:
  selector:
    matchLabels:
      run: simplehome
  template:
    metadata:
      labels:
        run: simplehome
    spec:
      containers:
      - image: registry.au-syd.bluemix.net/jgkong/simplehome
        name: simplehome
        ports:
        - containerPort: 80
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: simplehome
  name: simplehome
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: simplehome
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: simplehome
  name: simplehome-nodeport
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: simplehome
  type: NodePort
_EOF_
```
- `kubectl apply -f simplehome.yaml`
- `kubectl get deployment`
- `kubectl get pod`
- `kubectl get service`
- http://<worker Public IP>:<NodePort번호>/ 확인
- `kubectl delete -f simplehome.yaml`
- `kubectl apply -f simplehome.yaml`
