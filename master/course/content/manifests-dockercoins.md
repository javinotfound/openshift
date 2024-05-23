```
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: app
spec:
  tls:
    insecureEdgeTerminationPolicy: Redirect
    termination: edge
  to:
    kind: Service
    name: webui
```
```
apiVersion: v1
kind: Service
metadata:
  name: webui
spec:
  ports:
  - 
    port: 80
    targetPort: 9000
  selector:
    app: webui
  type: ClusterIP
```
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: webui
data:
  webui.js: |
    var express = require('express');
    var app = express();
    var redis = require('redis');
    
    var client = redis.createClient(6379, 'redis');
    client.on("error", function (err) {
        console.error("Redis error", err);
    });
    
    app.get('/', function (req, res) {
        res.redirect('/index.html');
    });
    
    app.get('/json', function (req, res) {
        client.hlen('wallet', function (err, coins) {
            client.get('hashes', function (err, hashes) {
                var now = Date.now() / 1000;
                res.json( {
                    coins: coins,
                    hashes: hashes,
                    now: now
                });
            });
        });
    });
    
    app.use(express.static('files'));
    
    var server = app.listen(9000, function () {
        console.log('WEBUI running on port 9000');
    });
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webui
spec:
  replicas: 6
  selector:
    matchLabels:
      app: webui
  template:
    metadata:
      labels:
        app: webui
    spec:
      containers:
      - 
        command:
        - node
        - webui.js
        image: docker.io/academiaonline/dockercoins-webui:2.0
        name: webui
        ports:
        -
          containerPort: 9000
          protocol: TCP
        resources:
          limits:
            cpu: 40m
            memory: 40M
          requests:
            cpu: 20m
            memory: 20M
        securityContext:
          readOnlyRootFilesystem: true
        volumeMounts:
        - 
          mountPath: /var/data/webui.js
          name: webui
          readOnly: true
          subPath: webui.js
        - 
          mountPath: /var/data/files/
          name: files
          readOnly: true
        workingDir: /var/data/
      initContainers:
      -
        command:
        - sh
        - -c
        - set -x;git clone -b main --single-branch https://github.com/sebastian-colomar/dockercoins;cp -rv cchillida/webui/files/* /var/data/files/
        image: docker.io/bitnami/git:latest
        name: init
        volumeMounts:
        -
          mountPath: /var/data/files/
          name: files
          readOnly: false
        workingDir: /var/data/
      topologySpreadConstraints:
      -
        labelSelector:
          matchLabels:
            app: webui
        maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
      -
        labelSelector:
          matchLabels:
            app: webui
        maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: DoNotSchedule
      volumes:
      - 
        configMap:
          defaultMode: 0400
          items: 
          -
            key: webui.js
            mode: 0400
            path: webui.js
          name: webui
        name: webui
      -
        emptyDir:
          medium: Memory
          sizeLimit: 1Mi
        name: files
```