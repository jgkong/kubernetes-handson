# Deployment control

## Create frontend
- `vue init pwa frontend`
- `cd frontend`
- `npm install`
- `npm run dev`
- http://localhost:8080 확인
- `cat src/components/Hello.vue`
-  ...
```
cat > src/components/Hello.vue << _EOF_
<template>
  <div class="hello">
    <h1>Frontend version {{ frontVer }}</h1>
    <h1>Backend version {{ backVersion }}</h1>
  </div>
</template>

<script>
export default {
  name: 'hello',
  data () {
    return {
      frontVer: '1.0.0',
      backVersion: '0.0.0',
      hostname: 'computer'
    }
  }
}
</script>

<style>
</style>
_EOF_
```
- `cat src/components/Hello.vue`
- `npm run build`
- `ls dist`
- ...
```
cat > Dockerfile << _EOF_
FROM nginx:alpine
COPY dist/ /usr/share/nginx/html/
_EOF_
```
- `cat Dockerfile`
- `docker build -t registry.au-syd.bluemix.net/jgkong/frontend:1 .`
- `docker run -d --rm --name frontend1 -p 8080:80 registry.au-syd.bluemix.net/jgkong/frontend:1`
- http://localhost:8080 확인
- `docker stop frontend1`
- `docker push registry.au-syd.bluemix.net/jgkong/frontend:1`
- `cd ..`

## Create backend server
- `mkdir backend`
- `cd backend`
- `echo "{}" > package.json`
- `npm install --save express`
- ...
```
cat > app.js << _EOF_
let express = require('express')
let app = express()

app.get('/api', (req, res) => {
  res.send('This is backend base API server')
})

app.listen(3000, () => {
  console.log('app is running')
})
_EOF_
```
- `cat app.js`
-  `node app.js`
- http://localhost:3000/api 접속
- `CTRL-C`

```
cat > Dockerfile << _EOF_
FROM node:alpine
WORKDIR /app
EXPOSE 3000
CMD ["node", "app.js"]
COPY . .
_EOF_
```
- `cat Dockerfile`
- `docker build -t registry.au-syd.bluemix.net/jgkong/baseapi:1 .`
- `docker run -d --name baseapi1 -p 3000:3000 registry.au-syd.bluemix.net/jgkong/baseapi:1`
- http://localhost:3000/api 접속
- `docker stop baseapi1`
- `docker push registry.au-syd.bluemix.net/jgkong/baseapi:1`
- `cd ..`

## Deploy to Kubernetes
- ...
```
cat > myhomepage.yaml << _EOF_
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: frontend
  name: frontend
  namespace: default
spec:
  selector:
    matchLabels:
      run: frontend
  template:
    metadata:
      labels:
        run: frontend
    spec:
      containers:
      - image: registry.au-syd.bluemix.net/jgkong/frontend:1
        name: frontend
        ports:
        - containerPort: 80
          protocol: TCP
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  labels:
    run: baseapi
  name: baseapi
  namespace: default
spec:
  selector:
    matchLabels:
      run: baseapi
  template:
    metadata:
      labels:
        run: baseapi
    spec:
      containers:
      - image: registry.au-syd.bluemix.net/jgkong/baseapi:1
        name: baseapi
        ports:
        - containerPort: 3000
          protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: frontend
  name: frontend
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: frontend
  type: ClusterIP
---
apiVersion: v1
kind: Service
metadata:
  labels:
    run: baseapi
  name: baseapi
spec:
  ports:
  - port: 3000
    protocol: TCP
    targetPort: 3000
  selector:
    run: baseapi
  type: ClusterIP
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: myhomepage
spec:
  rules:
  - host: myhomepage.jgkong.seo01.containers.appdomain.cloud
    http:
      paths:
      - path: /
        backend:
          serviceName: frontend
          servicePort: 80
      - path: /api
        backend:
          serviceName: baseapi
          servicePort: 3000
_EOF_
```
- `cat myhomepage.yaml`
- `kubectl apply -f myhomepage.yaml`
- http://myhomepage.jgkong.seo01.containers.appdomain.cloud/ 확인
- http://myhomepage.jgkong.seo01.containers.appdomain.cloud/api/version 확인


## Update baseapi server
- `cd baseapi`
- app.js 추가
```
app.get('/api/version', (req, res) => {
  res.send('2.0.0')
})
```
- `perl -pi -e 's/1\.0\.0/2.0.0/' app.js`
- `docker build -t registry.au-syd.bluemix.net/jgkong/baseapi:2 .`
- `docker push registry.au-syd.bluemix.net/jgkong/baseapi:2`
- `kubectl set image --record deployment baseapi baseapi=registry.au-syd.bluemix.net/jgkong/baseapi:2`
- `kubectl rollout status deployment baseapi`
- `kubectl rollout history deployment baseapi`
- http://myhomepage.jgkong.seo01.containers.appdomain.cloud/api/version 확인
- `cd ..`

## Update frontend
- `cd frontend`
- src/components/Hello.vue 추가
```
  created () {
    fetch('/api/version')
      .then(res => res.text())
      .then(res => {
        this.backVersion = res
      })
  }
```
- `npm run build`
- `docker build -t registry.au-syd.bluemix.net/jgkong/frontend:2 .`
- `kubectl rollout undo deployment baseapi`
- `kubectl set image --record deployment frontend frontend=registry.au-syd.bluemix.net/jgkong/frontend:2`
- `kubectl rollout status deployment frontend` 후 `CTRL`-`C`
- `kubectl get deployment`
- `kubectl get replicaset`
- `kubectl get pod`
- `kubectl rollout history deployment frontend`
- `kubectl rollout undo deployment frontend`
- `kubectl rollout status deployment frontend`
- `kubectl rollout history deployment frontend`
- `kubectl get deployment`
- `kubectl get replicaset`
- `kubectl get pod`

## Modify RollingUpdateStrategy
- `kubectl describe deployment frontend`
- `kubectl get deployment frontend -o json`
- `kubectl patch deployment frontend -p='{"spec": {"strategy": {"rollingUpdate": {"maxUnavailable": 0}}}}'`
- `kubectl describe deployment frontend`
- `kubectl set image --record deployment frontend frontend=registry.au-syd.bluemix.net/jgkong/frontend:2`
- `kubectl get replicaset`
- `docker push registry.au-syd.bluemix.net/jgkong/frontend:2`

## Rollout by kubectl apply
- src/components/Hello.vue 추가
```
    <h2>at {{ hostname }}</h2>
```
```
    fetch('/api/hostname')
      .then(res => res.text())
      .then(res => {
        this.hostname = res
      })
```
- `perl -pi -e 's/2\.0\.0/3.0.0/' src/components/Hello.vue`
- `npm run build`
- `docker build -t registry.au-syd.bluemix.net/jgkong/frontend:3 .`
- `docker push registry.au-syd.bluemix.net/jgkong/frontend:3`
- `cd ../backend/`
- app.js 추가
```
app.get('/api/hostname', (req, res) => {
  res.send(require('os').hostname())
})
```
- `perl -pi -e 's/2\.0\.0/3.0.0/' app.js`
- `docker build -t registry.au-syd.bluemix.net/jgkong/baseapi:3 .`
- `docker push registry.au-syd.bluemix.net/jgkong/baseapi:3`
- `cd ..`
- `perl -pi -e 's/:1$/:3/' myhomepage.yaml`
- `kubectl apply -f myhomepage.yaml`
