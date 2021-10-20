# Circit break labo
## 目的
istioを利用したサーキットブレイキングの挙動を確認する。サーキットブレイクの条件として500エラーの発生を使用する。
outlinerDetectionとbaseEjectionTimeを設定し、500エラーが発生したpodに対しリクエストが到達しないことを確認する。
(エラーが発生したpodをサービスの対象外にするのはReadiness Probeの設定ではないか？->似たようなことをIstio側がやってくれる)
## 検証1: DestinationRuleの動作確認
### 環境構築
Istio公式にCircuit BreakingのGetting startedがあるので参考にする。
https://istio.io/latest/docs/tasks/traffic-management/circuit-breaking/

まず、httpbinを使用してサービスを作成する。Istioのsampleにいいマニフェストがあるので利用する。
```
kubectl apply -f samples/httpbin/httpbin.yaml
```
httpbinのdeploymentとclusteridがデプロイされる。なお、このhttpbinのdeploymentのreplicasは1。

次にクライアントを作成する。Istioのsampleにいいマニフェストがあるので利用する。
```
kubectl apply -f samples/httpbin/sample-client/fortio-deploy.yaml
export FORTIO_POD=$(kubectl get pods -l app=fortio -o 'jsonpath={.items[0].metadata.name}')
```

最後に、DestinationRuleを作成する。Istio公式のGetting startedを参考に`sample-destinationrule.yaml`を作成した。
このDestinationRuleではhttpbinへのアクセスが、10sに1回でも500になると3分間サーキットブレイクする。

### 動作検証
httpbinは/status/<status code>にアクセスすると任意のステータスコードを返却する。この機能を利用し、恣意的に500エラーを発生させることで、httpbin clusteripへのアクセスが落とされることを確認する。
まず現在疎通していることを確認する。
```
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/200
HTTP/1.1 200 OK
server: envoy
date: Wed, 20 Oct 2021 07:15:05 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 8
$ 
```
次に、500エラーを発生させる。
```
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/500
07:18:40 E commonflags.go:131> Error status 500 : HTTP/1.1 500 Internal Server Error\r\nserver: envoy\r\ndate: Wed, 20 Oct 2021 07:18:40 GMT\r\ncontent-type: text/html; charset=utf-8\r\naccess-control-allow-origin: *\r\naccess-control-allow-credentials: true\r\ncontent-length: 0\r\nx-envoy-upstream-service-time: 12\r\n\r\n
HTTP/1.1 500 Internal Server Error
server: envoy
date: Wed, 20 Oct 2021 07:18:40 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 12

command terminated with exit code 1
$
```
最後に、再度疎通を確認する。
```
 kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/200
07:18:46 E commonflags.go:131> Error status 503 : HTTP/1.1 503 Service Unavailable\r\ncontent-length: 19\r\ncontent-type: text/plain\r\ndate: Wed, 20 Oct 2021 07:18:46 GMT\r\nserver: envoy\r\n\r\nno healthy upstream
HTTP/1.1 503 Service Unavailable
content-length: 19
content-type: text/plain
date: Wed, 20 Oct 2021 07:18:46 GMT
server: envoy

no healthy upstreamcommand terminated with exit code 1
$
```
アクセス先が/status/200なので200 OKが帰ってくるはずだが、503 Service Unavailableが帰ってきた。
3分経過後再度アクセスすると、疎通が確認できた。
```
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/200
HTTP/1.1 200 OK
server: envoy
date: Wed, 20 Oct 2021 07:24:40 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 3
$
```

### 結論
500エラーが規定回数発生した場合をトリガーとするサーキットブレイクについて検証できた。
任意のステータスコードを発生させるサービスを作成し、DestinationRuleで指定した回数の500エラーを発生させたところ、接続が遮断されることを確認した。
また、DestinationRuleで指定した時間が経過すると再度接続できることを確認した。

### 余談
consecutive5xxErrorsの値を変更し、指定回数500エラーを発生させた場合の検証もしたが、上記と同じように指定回数500エラーを発生させると503になった。


### memo
サーキットブレイク発生前のログ
yunoMacBook-Air:destination-rule-labo yu$ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 4294967295
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 24
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 33
yunoMacBook-Air:destination-rule-labo yu$

サーキットブレイク発生後のログ
yunoMacBook-Air:destination-rule-labo yu$ kubectl exec "$FORTIO_POD" -c istio-proxy -- pilot-agent request GET stats | grep httpbin | grep pending
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.remaining_pending: 4294967295
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.default.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.circuit_breakers.high.rq_pending_open: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_active: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_failure_eject: 0
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_overflow: 24
cluster.outbound|8000||httpbin.default.svc.cluster.local.upstream_rq_pending_total: 35
yunoMacBook-Air:destination-rule-labo yu$ 

## 検証2: エラーを出力したpodをロードバランシングプールから外す
### 環境構築
serviceのバックエンドに常に500を出力するpodを紛れ込ませてもいいが、開発が手間なのでできればhttpbinで代用したい。
このため、まずhttpbinのdeploymentのreplicasを1に設定してcircuit breakingを発生させ、その後kubectl editコマンドでdeploymentのreplicasを2に変更する。
こうすることで、バックエンドプールに新しいpodが追加され、そちらのpodでリクエストが処理されるはずである。

### 動作検証
まずhttpbinのpodが一つだけ動いていることを確認する。
```
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
fortio-deploy-687945c6dc-lbf44   2/2     Running   0          19h
httpbin-74fb669cc6-r252c         2/2     Running   0          19h
$
```
次に、このpodで500エラーを発生させ、サーキットブレイキングを起こす。
```
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/500
HTTP/1.1 500 Internal Server Error
server: envoy
date: Wed, 20 Oct 2021 08:13:39 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 3

08:13:39 E commonflags.go:131> Error status 500 : HTTP/1.1 500 Internal Server Error\r\nserver: envoy\r\ndate: Wed, 20 Oct 2021 08:13:39 GMT\r\ncontent-type: text/html; charset=utf-8\r\naccess-control-allow-origin: *\r\naccess-control-allow-credentials: true\r\ncontent-length: 0\r\nx-envoy-upstream-service-time: 3\r\n\r\n
command terminated with exit code 1
yunoMacBook-Air:destination-rule-labo yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/500
HTTP/1.1 503 Service Unavailable
content-length: 19
content-type: text/plain
date: Wed, 20 Oct 2021 08:13:45 GMT
server: envoy

no healthy upstream08:13:45 E commonflags.go:131> Error status 503 : HTTP/1.1 503 Service Unavailable\r\ncontent-length: 19\r\ncontent-type: text/plain\r\ndate: Wed, 20 Oct 2021 08:13:45 GMT\r\nserver: envoy\r\n\r\nno healthy upstream
command terminated with exit code 1
$
```
その後、httpbinのdeploymentを変更し、二つ目のpodを立てる。
```
$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
fortio-deploy-687945c6dc-lbf44   2/2     Running   0          19h
httpbin-74fb669cc6-2gj2q         2/2     Running   0          7s
httpbin-74fb669cc6-r252c         2/2     Running   0          19h
$
```
最後に、リクエストをhttpbin clusteripに送信し、接続できることを確認する。
```
$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio curl -quiet http://httpbin:8000/status/200
HTTP/1.1 200 OK
server: envoy
date: Wed, 20 Oct 2021 08:14:15 GMT
content-type: text/html; charset=utf-8
access-control-allow-origin: *
access-control-allow-credentials: true
content-length: 0
x-envoy-upstream-service-time: 25

$
```
200が帰ってきているので、新しいpodがロードバランサープールに追加され、そちらで処理されたと考えられる。