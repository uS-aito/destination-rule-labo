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

## 検証3. 500を返すpodと200を返すpodを混在させた場合の動作
検証2ではhttpbinで代用するため、/status/500にリクエストを投げてサーキットブレイキングさせた後、別のpodを追加したが、混在した場合の挙動も確認する。
### 環境構築
httpresponsetesterを開発したので、500を返すpodを一つ、200を返すpodを3つデプロイするdeploymentをそれぞれ作成する。
次に、これらのpodへのアクセスを提供するclusterip serviceを作成する。このclusterip serviceにアクセスした場合、3/4の確率で200、1/4の確率で500が得られるはずである。
### 動作検証
まず、DestinationRuleを設定しない状態でfortioからアクセスを複数回行う。結果として、200が3/4、500が1/4の割合で得られる。
```
yunoMacBook-Air:destination-rule-labo yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
05:11:38 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:11:38 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 194.637212ms : 100 calls. qps=513.78
Aggregated Function Time : count 100 avg 0.0019456428 +/- 0.0004728 min 0.00136473 max 0.00338328 sum 0.19456428
# range, mid point, percentile, count
>= 0.00136473 <= 0.002 , 0.00168236 , 58.00, 58
> 0.002 <= 0.003 , 0.0025 , 96.00, 38
> 0.003 <= 0.00338328 , 0.00319164 , 100.00, 4
# target 50% 0.00191084
# target 75% 0.00244737
# target 90% 0.00284211
# target 99% 0.00328746
# target 99.9% 0.0033737
Sockets used: 26 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 75 (75.0 %)
Code 500 : 25 (25.0 %)
Response Header Sizes : count 100 avg 123.75 +/- 71.45 min 0 max 165 sum 12375
Response Body/Total Sizes : count 100 avg 177 +/- 15.59 min 168 max 204 sum 17700
All done 100 calls (plus 0 warmup) 1.946 ms avg, 513.8 qps
yunoMacBook-Air:destination-rule-labo yu$
```

次に、DestinationRuleを設定し、一度でも500を出力したらサーキットブレークを起こすよう設定する。結果として、500が一度だけ記録され、他は全て200になっている。
```
yunoMacBook-Air:labo3 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
05:16:08 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
05:16:08 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 286.345756ms : 100 calls. qps=349.23
Aggregated Function Time : count 100 avg 0.0028623633 +/- 0.004604 min 0.001383843 max 0.039637026 sum 0.286236332
# range, mid point, percentile, count
>= 0.00138384 <= 0.002 , 0.00169192 , 61.00, 61
> 0.002 <= 0.003 , 0.0025 , 89.00, 28
> 0.003 <= 0.004 , 0.0035 , 94.00, 5
> 0.005 <= 0.006 , 0.0055 , 95.00, 1
> 0.009 <= 0.01 , 0.0095 , 96.00, 1
> 0.012 <= 0.014 , 0.013 , 97.00, 1
> 0.014 <= 0.016 , 0.015 , 98.00, 1
> 0.02 <= 0.025 , 0.0225 , 99.00, 1
> 0.035 <= 0.039637 , 0.0373185 , 100.00, 1
# target 50% 0.00188704
# target 75% 0.0025
# target 90% 0.0032
# target 99% 0.025
# target 99.9% 0.0391733
Sockets used: 2 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 99 (99.0 %)
Code 500 : 1 (1.0 %)
Response Header Sizes : count 100 avg 163.36 +/- 16.42 min 0 max 166 sum 16336
Response Body/Total Sizes : count 100 avg 168.38 +/- 3.682 min 168 max 205 sum 16838
All done 100 calls (plus 0 warmup) 2.862 ms avg, 349.2 qps
yunoMacBook-Air:labo3 yu$ 
```

## 検証4. maxEjectionPercentの動作検証
maxEjectionPercentの値を変化させ、このパラメータがどのような動作に影響を及ぼすのかを明らかにする。
### 環境構築
httpresponsetesterを使用し、500を返すpodを5つ、200を返すpodを5つデプロイするdeploymentを作成する。
次に、これらのpodへのアクセスを提供するclusterip serviceを作成する。このserviceにアクセスすると、200と500がそれぞれ半分の割合で得られる。
このマニフェストは./labo4/labo4-deployments.yamlを参照。
### 動作検証
まずDestinationRuleを設定しない状態でforitoからアクセスを複数回行う。結果として、200と500が約半々の割合で得られる。
```
yunoMacBook-Air:destination-rule-labo yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
05:40:01 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:01 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
05:40:02 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 604.764073ms : 100 calls. qps=165.35
Aggregated Function Time : count 100 avg 0.006043145 +/- 0.009005 min 0.001426411 max 0.040122245 sum 0.604314504
# range, mid point, percentile, count
>= 0.00142641 <= 0.002 , 0.00171321 , 40.00, 40
> 0.002 <= 0.003 , 0.0025 , 76.00, 36
> 0.003 <= 0.004 , 0.0035 , 80.00, 4
> 0.005 <= 0.006 , 0.0055 , 81.00, 1
> 0.011 <= 0.012 , 0.0115 , 82.00, 1
> 0.012 <= 0.014 , 0.013 , 84.00, 2
> 0.014 <= 0.016 , 0.015 , 90.00, 6
> 0.025 <= 0.03 , 0.0275 , 96.00, 6
> 0.03 <= 0.035 , 0.0325 , 99.00, 3
> 0.04 <= 0.0401222 , 0.0400611 , 100.00, 1
# target 50% 0.00227778
# target 75% 0.00297222
# target 90% 0.016
# target 99% 0.035
# target 99.9% 0.04011
Sockets used: 50 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 51 (51.0 %)
Code 500 : 49 (49.0 %)
Response Header Sizes : count 100 avg 84.25 +/- 82.58 min 0 max 166 sum 8425
Response Body/Total Sizes : count 100 avg 185.82 +/- 17.98 min 168 max 205 sum 18582
All done 100 calls (plus 0 warmup) 6.043 ms avg, 165.4 qps
yunoMacBook-Air:destination-rule-labo yu$
```

次に、DestinationRuleをmaxEjectionPercentを100に設定した状態で設定する。結果として、500が5回得られ、それ以外は全て200になる。
-> ならない。大体200:500=6:4よりやや500よりの結果が得られる。同じDestinationruleでtarget側を200を4、500を1にすると検証3と同じ結果になったのでDestinationRuleの問題ではない。
500側のログを見ると1つのpodは1回のみリクエストを処理し、その後は一切リクエストを受けていなかったが、残りはその後もリクエストを受け続けているようである。
また、受け付けているリクエストの数は以下の通りで、極端な差異はないように思える。
10,1,11,10,10
一つを除き四つのpodが概ね10リクエストを受け付けている。これは一つのpodがプールから排除された後、残りのpodはしばらくはロードバランシングプールに残り続けたことを意味すると思われる。
クールダウンタイムがあるのか？
fortioにリクエストごとのクールダウンタイムを設定する機能がないので、確認できない。あれば10sごとに1リクエストに設定することで確認できたかもしれない。
スクリプトを書けばできるが時間がかかるので後に回す。少なくとも、最終的にはすべての500podがロードバランシングプールから排除され、100%200が帰っている

次に、DestinationRuleをmaxEjectionPercentを10に設定した状態でデプロイする。結果として、200と500が約6:4の割合で得られる。
```
yunoMacBook-Air:labo4 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 90 -loglevel Warning http://target:8000
06:27:51 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 90 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 90 calls (90 per thread + 0)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:27:51 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 235.23739ms : 90 calls. qps=382.59
Aggregated Function Time : count 90 avg 0.0026078969 +/- 0.001122 min 0.001590839 max 0.010920733 sum 0.23471072
# range, mid point, percentile, count
>= 0.00159084 <= 0.002 , 0.00179542 , 28.89, 26
> 0.002 <= 0.003 , 0.0025 , 78.89, 45
> 0.003 <= 0.004 , 0.0035 , 94.44, 14
> 0.004 <= 0.005 , 0.0045 , 97.78, 3
> 0.005 <= 0.006 , 0.0055 , 98.89, 1
> 0.01 <= 0.0109207 , 0.0104604 , 100.00, 1
# target 50% 0.00242222
# target 75% 0.00292222
# target 90% 0.00371429
# target 99% 0.0100921
# target 99.9% 0.0108379
Sockets used: 40 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 50 (55.6 %)
Code 500 : 40 (44.4 %)
Response Header Sizes : count 90 avg 91.666667 +/- 81.99 min 0 max 165 sum 8250
Response Body/Total Sizes : count 90 avg 184 +/- 17.89 min 168 max 204 sum 16560
All done 90 calls (plus 0 warmup) 2.608 ms avg, 382.6 qps
yunoMacBook-Air:labo4 yu$ 
```

最後に、DestinationRuleをmaxEjectionPercentを50に設定した状態でデプロイする。結果として、500が5回得られ、それ以外は全て200になる。
```
yunoMacBook-Air:labo4 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
06:29:51 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
Ended after 222.052338ms : 100 calls. qps=450.34
Aggregated Function Time : count 100 avg 0.0022192996 +/- 0.0006436 min 0.001549336 max 0.005354983 sum 0.221929955
# range, mid point, percentile, count
>= 0.00154934 <= 0.002 , 0.00177467 , 48.00, 48
> 0.002 <= 0.003 , 0.0025 , 92.00, 44
> 0.003 <= 0.004 , 0.0035 , 97.00, 5
> 0.004 <= 0.005 , 0.0045 , 99.00, 2
> 0.005 <= 0.00535498 , 0.00517749 , 100.00, 1
# target 50% 0.00204545
# target 75% 0.00261364
# target 90% 0.00295455
# target 99% 0.005
# target 99.9% 0.00531948
Sockets used: 1 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 100 (100.0 %)
Response Header Sizes : count 100 avg 165 +/- 0 min 165 max 165 sum 16500
Response Body/Total Sizes : count 100 avg 168 +/- 0 min 168 max 168 sum 16800
All done 100 calls (plus 0 warmup) 2.219 ms avg, 450.3 qps
yunoMacBook-Air:labo4 yu$ 
```

## 検証5. minHealthPercentの動作検証
ドキュメントによると、ロードバランシングプールのpodのうち、healthy statusのpodの割合が、このパラメータで指定された割合を下回った場合、outerline detectionの機能が停止する。
つまり、サーキットブレイキングが無効になり、エラーを起こしているpodにも正常なpodにもリクエストが送信されるようになる。

### 環境構築
500を返すpodを5、200を返すpodを5デプロイする。maxEjectionPercentを100にする。minHealthPercentを60にする。

### 動作検証
fortioから100回リクエストを投げさせる。すると全ての500podがunhealthy扱いになるので排除される。従来であれば、この後再度fortioからリクエストすると200が100%になるが、healthyなpodの割合がminHealthPercentを下回るため、outerliner detection自体が無効になり、200と500が50%ずつになるはずである。
```
yunoMacBook-Air:labo5 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
06:43:18 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:18 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 539.652936ms : 100 calls. qps=185.3
Aggregated Function Time : count 100 avg 0.0053929659 +/- 0.002436 min 0.002153531 max 0.018344947 sum 0.539296594
# range, mid point, percentile, count
>= 0.00215353 <= 0.003 , 0.00257677 , 14.00, 14
> 0.003 <= 0.004 , 0.0035 , 33.00, 19
> 0.004 <= 0.005 , 0.0045 , 51.00, 18
> 0.005 <= 0.006 , 0.0055 , 66.00, 15
> 0.006 <= 0.007 , 0.0065 , 79.00, 13
> 0.007 <= 0.008 , 0.0075 , 88.00, 9
> 0.008 <= 0.009 , 0.0085 , 94.00, 6
> 0.009 <= 0.01 , 0.0095 , 97.00, 3
> 0.011 <= 0.012 , 0.0115 , 98.00, 1
> 0.012 <= 0.014 , 0.013 , 99.00, 1
> 0.018 <= 0.0183449 , 0.0181725 , 100.00, 1
# target 50% 0.00494444
# target 75% 0.00669231
# target 90% 0.00833333
# target 99% 0.014
# target 99.9% 0.0183105
Sockets used: 47 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 54 (54.0 %)
Code 500 : 46 (46.0 %)
Response Header Sizes : count 100 avg 89.11 +/- 82.24 min 0 max 166 sum 8911
Response Body/Total Sizes : count 100 avg 184.57 +/- 17.93 min 168 max 204 sum 18457
All done 100 calls (plus 0 warmup) 5.393 ms avg, 185.3 qps
yunoMacBook-Air:labo5 yu$ 
yunoMacBook-Air:labo5 yu$ 
yunoMacBook-Air:labo5 yu$ 
yunoMacBook-Air:labo5 yu$ 
yunoMacBook-Air:labo5 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target:8000
06:43:46 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
06:43:46 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 304.53987ms : 100 calls. qps=328.36
Aggregated Function Time : count 100 avg 0.0030377361 +/- 0.0009783 min 0.001678011 max 0.007202539 sum 0.303773611
# range, mid point, percentile, count
>= 0.00167801 <= 0.002 , 0.00183901 , 7.00, 7
> 0.002 <= 0.003 , 0.0025 , 56.00, 49
> 0.003 <= 0.004 , 0.0035 , 88.00, 32
> 0.004 <= 0.005 , 0.0045 , 94.00, 6
> 0.005 <= 0.006 , 0.0055 , 98.00, 4
> 0.006 <= 0.007 , 0.0065 , 99.00, 1
> 0.007 <= 0.00720254 , 0.00710127 , 100.00, 1
# target 50% 0.00287755
# target 75% 0.00359375
# target 90% 0.00433333
# target 99% 0.007
# target 99.9% 0.00718229
Sockets used: 50 (for perfect keepalive, would be 1)
Jitter: false
Code 200 : 50 (50.0 %)
Code 500 : 50 (50.0 %)
Response Header Sizes : count 100 avg 82.5 +/- 82.5 min 0 max 165 sum 8250
Response Body/Total Sizes : count 100 avg 186 +/- 18 min 168 max 204 sum 18600
All done 100 calls (plus 0 warmup) 3.038 ms avg, 328.4 qps
yunoMacBook-Air:labo5 yu$ 
```

## 検証6. consecutiveGatewayErrorsの動作検証
consecutiveGatewayErrorsの動作を検証する。ドキュメントによると502,503,504エラーの発生回数をカウントするものとある。
### 環境構築と検証方針
500を返すpodと502を返すpodを用意する。DestinationRuleではconsecutiveGatewayErrorsを設定する。500の方はcircuit breakingが発生せず、502の方は発生するはずである。
### 検証方法
500を返すpodをロードバランシングプールに持つserviceを立てる。fortioで100回リクエストを投げる。Desinationruleに引っかからないので、常に500が帰ってくる。
次に、502を返すpodをロードバランシングプールに持つserviceを立てる。fortioで100回リクエストを投げる。DestinationRuleに引っかかるので、1回502が帰ってきた後、残りは503になる。
```
yunoMacBook-Air:labo6 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target-500:8000
08:01:12 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target-500:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
08:01:12 W http_client.go:693> Parsed non ok code 500 (HTTP/1.1 500)
Ended after 340.528046ms : 100 calls. qps=293.66
Aggregated Function Time : count 100 avg 0.0034030605 +/- 0.001834 min 0.001947716 max 0.019727251 sum 0.34030605
# range, mid point, percentile, count
>= 0.00194772 <= 0.002 , 0.00197386 , 1.00, 1
> 0.002 <= 0.003 , 0.0025 , 48.00, 47
> 0.003 <= 0.004 , 0.0035 , 85.00, 37
> 0.004 <= 0.005 , 0.0045 , 94.00, 9
> 0.005 <= 0.006 , 0.0055 , 98.00, 4
> 0.006 <= 0.007 , 0.0065 , 99.00, 1
> 0.018 <= 0.0197273 , 0.0188636 , 100.00, 1
# target 50% 0.00305405
# target 75% 0.00372973
# target 90% 0.00455556
# target 99% 0.007
# target 99.9% 0.0195545
Sockets used: 100 (for perfect keepalive, would be 1)
Jitter: false
Code 500 : 100 (100.0 %)
Response Header Sizes : count 100 avg 0 +/- 0 min 0 max 0 sum 0
Response Body/Total Sizes : count 100 avg 204 +/- 0 min 204 max 204 sum 20400
All done 100 calls (plus 0 warmup) 3.403 ms avg, 293.7 qps
yunoMacBook-Air:labo6 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 1 -qps 0 -n 100 -loglevel Warning http://target-502:8000
08:01:18 I logger.go:127> Log level is now 3 Warning (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 100 calls: http://target-502:8000
Starting at max qps with 1 thread(s) [gomax 2] for exactly 100 calls (100 per thread + 0)
08:01:18 W http_client.go:693> Parsed non ok code 502 (HTTP/1.1 502)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
08:01:18 W http_client.go:693> Parsed non ok code 503 (HTTP/1.1 503)
Ended after 111.45047ms : 100 calls. qps=897.26
Aggregated Function Time : count 100 avg 0.0011138617 +/- 0.001787 min 0.000530179 max 0.018656117 sum 0.111386165
# range, mid point, percentile, count
>= 0.000530179 <= 0.001 , 0.00076509 , 66.00, 66
> 0.001 <= 0.002 , 0.0015 , 98.00, 32
> 0.002 <= 0.003 , 0.0025 , 99.00, 1
> 0.018 <= 0.0186561 , 0.0183281 , 100.00, 1
# target 50% 0.000884352
# target 75% 0.00128125
# target 90% 0.00175
# target 99% 0.003
# target 99.9% 0.0185905
Sockets used: 100 (for perfect keepalive, would be 1)
Jitter: false
Code 502 : 1 (1.0 %)
Code 503 : 99 (99.0 %)
Response Header Sizes : count 100 avg 0 +/- 0 min 0 max 0 sum 0
Response Body/Total Sizes : count 100 avg 153.32 +/- 3.184 min 153 max 185 sum 15332
All done 100 calls (plus 0 warmup) 1.114 ms avg, 897.3 qps
yunoMacBook-Air:labo6 yu$ 
```

## 検証7. splitExternalLocalOriginErrorsの動作検証
一応、公式ドキュメントにはこのような記載があるが、どのような動作をするのか、ユースケースは何か、全くわからない。
> This is especially useful when the upstream service explicitly returns a 5xx for some requests and you want to ignore those responses from upstream service while determining the outlier detection status of a host. Defaults to false.
> (これは、アップストリームサービスが一部のリクエストに対して明示的に5xxを返しており、ホストの異常値検出ステータスを決定する際にアップストリームサービスからのこれらの応答を無視したい場合に特に有効です。デフォルトは false です。)
サービス側が500を出力しているときに、ホスト側に異常があるかどうか判定するために有効と言っているのか？ローカル起因の異常値の例としてfailure to connect, timeout while connectingなどを挙げているが、これらが発生した場合はそもそもstatus codeを得られてないと思う。
-> DestinationRuleやVirtualService側でtimeout等の設定があるようなのでこれに抵触した場合を言っているか？
タイムアウトまたは接続失敗を発生させる。
### 検証7.1 タイムアウト
以下のような設定でDestinationRuleを設定する。

## 検証8. maxConnectionsの検証
maxConnectionsの設定を検証する。複数のpodからmaxConnectionsを1にしたserviceにtelnetで接続を行う。
→ TCP自体は普通に接続できてしまった
→ fortio load -c 10とかにしても通った。
istioはある程度の余裕を許容するそうなので、それを考えると厳密な結果を得るのは難しいかもしれない。
インターネット上の使用例を調べると、サンプルとしてはmaxConnectionsとhttp1MaxPendingRequests、maxRequestsPerConnectionを全て1にしている例が多い。
どれがどのくらい影響しているのか？http/1.1は1TCPあたり1リクエストのはずなので、maxRequestsPerConnectionがいくつでも関係ないはず。試す。
**maxRequestsPerConnection:1**
```
yunoMacBook-Air:labo8 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 5 -qps 0 -n 1000 -loglevel Error http://httpbin:8000/get
07:12:01 I logger.go:127> Log level is now 4 Error (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 1000 calls: http://httpbin:8000/get
Aggregated Function Time : count 1000 avg 0.0036879818 +/- 0.004588 min 0.000379697 max 0.034176044 sum 3.68798183
# target 50% 0.00234783
# target 75% 0.0032551
# target 90% 0.008
# target 99% 0.025
# target 99.9% 0.032784
Sockets used: 876 (for perfect keepalive, would be 5)
Jitter: false
Code 200 : 126 (12.6 %)
Code 503 : 874 (87.4 %)
All done 1000 calls (plus 0 warmup) 3.688 ms avg, 1170.1 qps
yunoMacBook-Air:labo8 yu$
```
**maxRequestsPerConnection:10**
```
yunoMacBook-Air:labo8 yu$ kubectl exec "$FORTIO_POD" -c fortio -- /usr/bin/fortio load -c 5 -qps 0 -n 1000 -loglevel Error http://httpbin:8000/get
07:11:07 I logger.go:127> Log level is now 4 Error (was 2 Info)
Fortio 1.11.3 running at 0 queries per second, 2->2 procs, for 1000 calls: http://httpbin:8000/get
Aggregated Function Time : count 1000 avg 0.0039736575 +/- 0.004068 min 0.000404827 max 0.030141552 sum 3.97365754
# target 50% 0.00231923
# target 75% 0.00475
# target 90% 0.0104667
# target 99% 0.0192
# target 99.9% 0.025
Sockets used: 723 (for perfect keepalive, would be 5)
Jitter: false
Code 200 : 281 (28.1 %)
Code 503 : 719 (71.9 %)
All done 1000 calls (plus 0 warmup) 3.974 ms avg, 1098.3 qps
yunoMacBook-Air:labo8 yu$
```
影響があった。原因がいまいち理解できないのでStackOverflowに質問させてもらった。→回答がない
tcp.maxConnectionsを2、http.http1MaxPendingRequestsを1にしてfortioから2つのリクエストを同時に投げると、大概200と503が帰ってくる(そうでない時もある)。
逆に、tcp.maxConnectionsを1、http.http1MaxPendingRequestsを2にして同じことをすると、確実に200がふたつ帰ってくる。
これらの事象はhttp.http1MaxPendingRequestsの方がこの場合は支配的であると解釈できる。
tcp.maxConnectionsの値を1と2にして、/delay/5にリクエストをふたつ平行で投げた場合の結果を見たところ、以下のようになった。
(http以下の設定は未設定、つまりデフォルト値)

**tcp.maxConnections:1**
Aggregated Function Time : count 2 avg 7.5227777 +/- 2.503 min 5.020252076 max 10.025303248 sum 15.0455553
**tcp.maxConnections:2**
Aggregated Function Time : count 2 avg 5.0254922 +/- 8.017e-05 min 5.025412056 max 5.025572394 sum 10.0509845

tcp.maxConnectionsの値が1だと最長で10秒かかっている。httpbinは/delay/NにリクエストするとN秒後にレスポンスするので、一つのリクエストが処理されるのを待ってからふたつ目のリクエストの処理が行われたと考えられる。つまり、tcp.maxConnectionsの値を超えると、tcp接続が遮断されるのではなく、後から来たほうがペンディングされると思われる。
tcp.maxConnectionsを5(十分に大きい値)にして、http.http1MaxPendingRequestsを1にした上で二つのリクエストを同時に投げると大体200と503が帰ってくる(時々200のみ)。このことから、現在処理中のリクエストもPendingRequestsに含んでいると思われる。ただし、http1MaxPendingRequestsはある程度余裕を持って処理される。

## 検証9. http2MaxRequestsの検証
http2MaxRequests動作を検証する。fortioは普通にするとhttp/2をしゃべってくれないので、gRPCを喋らせることでhttp/2を使わせる。
→ gRPCだとhttp/2だけでなくて、その上に別のものも載せることになるので、http/2のロードテスターを探したところ、h2loadが見つかったのでそれを使う。
→ http2MaxRequestsの値を10にして、h2loadのパラメータを`-p h2c -n 15`にしたところ、全て200だった。
→ そもそもhttp2におけるリクエストとはなんぞや？を調べると、一つのTCPコネクション上で複数のストリームがやり取りする、その一つ一つのやり取りがリクエストであるとわかった。従って、上記設定だと一度に1リクエストを15回繰り返している。
→ 同時に許容するリクエスト数と考え、`-p h2c -c 11 -n 11`としてリクエストをすると、2xxが10、5xxが1になった。何回繰り返しても同じ結果になったので、http1系と違い結構厳密に見てくれるらしい。
→ h2loadは5xxが出たのは教えてくれるが具体的なナンバーが出てないので、もしかすると別のエラーの可能性もあるはある


## 検証10. maxRequestsPerConnectionの検証
http1と2で意味合いが異なるようによめる。http1の場合、keep-aliveがONの時、一つのTCPコネクションで何回リクエストできるかを指定すると思われる。http2の場合、一つのTCPコネクション上の全てのストリームで行ったリクエストの総数と思われる。
→ http/2の準備が整ってるのでこちらから始める。`maxRequestsPerConnection`を5にして、`-p h2c -c 1 -m 10 -n 10`を指定する。
→ 全て通った。他に可能性を思い付かないので、http/2に影響がないと思われる。http/2で接続数を制限したい場合はhttp2MaxRequestsを指定すると思われる。
→ http1で再度試す。TELNETでhttpリクエストを送信し、動作を確認する。5回手動でリクエストするのは混乱を招きうるので、`maxRequestsPerConnection`を2にして、3回リクエストする。
→ 何回リクエストしても閉じない。
→ `maxRequestsPerConnection`を1にしてリクエストする。一度で切られるはず。
→ きられない。Connection: closeを指定してリクエストすると普通に切られた。fortioで`-c 1 -qps 0 -n 10`あたりを指定してリクエストしてみる。
→ 通った。通るな。
Envoyのドキュメントを読むと、http/2に関してはconnection数を制限する？ようによめる。
https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/connection_pooling
→ `maxRequestsPerConnection`を1にして、`-p h2c -c 10 -m 1 -n 10`を指定してみる。
→ 通った。`-p h2c -c 10 -m 10 -n 100`を試す(一リクエストないでのストリーム数を増やす)
→ 通った。
tcp.maxConnectionsと組み合わせていないため、逐次新しいTCPセッションが開かれている？しかしfortioで投げた時、sockets usedは常に1だったし、telnetで投げた時も何回投げても切断されなかった。本格的にパケットキャプチャが必要かもしれない。

# 検証11. h2UpgradePolicyの検証
DestinationRuleの有無を変更してリクエストを送信し、istio-proxyの動作を確認した。
```
[2021-11-18T09:24:17.367Z] "GET /get HTTP/2" 200 - via_upstream - "-" 0 603 28 24 "-" "curl/7.74.0" "a60e926c-0a8f-96c8-99b3-d24a6ce42dd0" "httpbin:8000" "10.72.0.14:80" inbound|80|| 127.0.0.6:43121 10.72.0.14:80 10.72.0.21:42870 outbound_.8000_._.httpbin.default.svc.cluster.local default
[2021-11-18T09:24:46.655Z] "GET /get HTTP/1.1" 200 - via_upstream - "-" 0 603 2 2 "-" "curl/7.74.0" "76c189de-f910-907d-adc1-b521002713f8" "httpbin:8000" "10.72.0.14:80" inbound|80|| 127.0.0.6:53279 10.72.0.14:80 10.72.0.21:43102 outbound_.8000_._.httpbin.default.svc.cluster.local default
```
上がDestinationRuleあり、下がなし。サーバにリクエストが到達した時点でhttp/2になっているので、クライアント側のEnvoryが変換していると思われる。
Envoryのページを参照すると、websocketを利用した場合の例でclient側のenvoyがhttp/2に変換しているらしい絵が確認できる。下記のような記述があり、WebsocketやConnect(おそらくConnect Method)以外については断言しきれないこと、http_chainなどIstio側からは見えないリソースがあることから検証したが、望み通りの結果になった。
> Envoy Upgrade support is intended mainly for WebSocket and CONNECT support, but may be used for arbitrary upgrades as well.