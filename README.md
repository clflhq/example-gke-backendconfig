# README

GKEでCDNを有効にする作成手順

ref. https://cloud.google.com/kubernetes-engine/docs/how-to/cdn-backendconfig?hl=ja

## 作成手順
1. CDNキャッシュを有効にしたNginxコンフィグを作成する
```
# default.conf
server {
  listen 80 default_server;
  server_name  _;

  root /usr/share/nginx/html;
  index index.html index.htm;

  location / {
    try_files $uri $uri/ =404;
  }

  error_page 500 502 503 504 /50x.html;

  # CDNのキャッシュを有効にするために、Nginxに以下の設定を行う
  # 今回は/images/以下をキャッシュするようにする
  # ref. https://cloud.google.com/cdn/docs/caching?hl=ja
  location ~ ^/images/(.*)$ {
    expires max;
    add_header Cache-Control "public";
  }
}
```

2. default.confのConfigMapを作成する
```
% kubectl create configmap --dry-run --save-config nginx-configmap --from-file=default.conf --output yaml > configmap.yaml
```

3. Deploymentを作成する
```
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.16.1-alpine
          volumeMounts:
            - name: default-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
      volumes:
        - name: default-conf
          configMap:
            name: nginx-configmap
            items:
              - key: default.conf
                path: default.conf
```

4. BackendConfigを作成する (ここにCDNの設定を書く)
```
# backendconfig.yaml
apiVersion: cloud.google.com/v1beta1
kind: BackendConfig
metadata:
  name: nginx-backendconfig
spec:
  cdn:
    enabled: true
    cachePolicy:
      includeHost: true
      includeProtocol: true
      includeQueryString: false
```

5. Serviceを作成する
```
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  annotations:
    beta.cloud.google.com/backend-config: '{"ports": {"80": "nginx-backendconfig"}}'    # <= BackendConfigを指定
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      protocol: "TCP"
  selector:
    app: nginx
```

6. Ingressを作成する
```
# ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
  labels:
    app: nginx
spec:
  backend:
    serviceName: nginx-service
    servicePort: 80
```

7. GKEにapplyする
```
% kubectl apply -f .
% kubectl get po,deploy,svc,ing
NAME                                    READY   STATUS    RESTARTS   AGE
pod/nginx-deployment-859fcfb65d-5pxbh   1/1     Running   0          76s

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/nginx-deployment   1/1     1            1           76s

NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/kubernetes      ClusterIP   10.3.240.1    <none>        443/TCP        2m33s
service/nginx-service   NodePort    10.3.243.77   <none>        80:30302/TCP   74s

NAME                               HOSTS   ADDRESS          PORTS   AGE
ingress.extensions/nginx-ingress   *       34.107.211.101   80      75s
```

8. 初回作成時はIngressが起動するまで暫く待つ
```
% curl -I http://$(kubectl get --no-headers ing|awk '{print $3}')
HTTP/1.1 200 OK                     # <= 200が返ってくればOK
:
```

9. サンプル画像をPodにコピーする
```
% kubectl cp images $(kubectl get po -o=name|sed -e 's#pod/##'):/usr/share/nginx/html/.
```

10. 確認
```
% curl -s --dump-header - -o /dev/null http://$(kubectl get --no-headers ing|awk '{print $3}')/images/sample.jpg
HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Wed, 11 Mar 2020 03:46:14 GMT
Content-Type: image/jpeg
Content-Length: 28522
Last-Modified: Wed, 11 Mar 2020 03:43:30 GMT
ETag: "5e685e62-6f6a"
Expires: Thu, 31 Dec 2037 23:55:55 GMT
Accept-Ranges: bytes
Via: 1.1 google
Cache-Control: max-age=315360000,public
Age: 4    # <= コンテンツがキャッシュされてからの経過時間を示す
```

以上
