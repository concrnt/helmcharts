# concrnt2

concrnt v2 専用の Helm チャートです。v1 のコンポーネント (ccapi / ccgateway /
ccwebui / activitypub bridge / 旧DB) は含みません。

## concrnt2-migration からの移行

このチャートは `concrnt2-migration` チャートの後継です。v2 側のリソース名
(Deployment `concrnt2` / `world-app`、StatefulSet `v2db` / `v2-redis`、
Deployment `v2-memcached` / `mediaserver2`、ConfigMap `concrnt2-config` /
`concrnt-static`) と接続先のデフォルト値をそのまま維持しているため、
同じリリースに対して `helm upgrade` するだけで v1 コンポーネントだけが削除され、
v2 の StatefulSet と PVC (データ) はそのまま引き継がれます。

移行チャートで v1 が提供していた `/api/v1` `/ap` `/storage` などの
legacy パスのプロキシは含まれないので、v1 を退役させてから乗り換えてください。

**注意:** アップグレードすると v1 用の PVC (`postgres-varlib` / `redis-data`)
も削除されます (v1 の DB データが消えます)。v2 側のデータは StatefulSet の
volumeClaimTemplates が作った PVC (`postgres-varlib-v2db-0` /
`redis-data-v2-redis-0`) にあり、これらは Helm 管理外なのでそのまま残ります。

## 追加の .well-known

`concrnt2.additionalWellKnown` に書いた内容がそのまま concrnt の config
(`additionalWellKnown`) に渡され、`/.well-known/<name>` で返されます
(additionalWellKnown に対応した concrnt が必要です)。

```yaml
concrnt2:
  additionalWellKnown:
    my-service:
      type: string        # text/plain
      value: "hello"
    my-json:
      type: json          # application/json
      value:
        links:
          - rel: self
            href: https://example.com/
```

## マルチレプリカ (cluster モード)

```yaml
concrnt2:
  replicas: 3
  cluster:
    enabled: true
```

- 全 Pod が公開 API (:8000) を提供し、自由にスケールできます。
- concrnt 本体は Kubernetes に依存しません。クラスタ協調は k8s-elector
  サイドカー (同一リポジトリでビルドされる別イメージ) が担い、
  coordination.k8s.io の Lease でリーダーを選出し、headless service
  (`concrnt2-headless`) で ピアを発見して、
  `GET /status -> {"isLeader", "leaderUrl", "peers"}` を localhost:8002 で
  提供します (このために ServiceAccount + Role/RoleBinding を作成します)。
- Lease を持つ Pod のサイドカーがリーダーとなり、その Pod だけが
  シングルトンワーカー (federation subscriber / chunkline cache updater /
  web push reactor) を実行します。
- :8001 は internal (運用) リスナーです。/health・/ready のプローブと、
  レプリカ間の購読協調 API (リーダーが各 Pod の :8001/internal/demand を
  集約し、:8001/internal/subscriptions で応答) に使われます。
- **:8001 と :8002 はクラスタ内部専用です。Ingress や LoadBalancer で
  公開しないでください。** 公開するのは Service `concrnt2` (:80 → :8000) のみです。

cluster を無効 (デフォルト) にした場合はサイドカーや RBAC は作られず、
従来どおりの単一レプリカ構成になります。この場合も /health・/ready は
:8001 で提供されるため、cluster 対応版以降のイメージが必要です。

## atproto (Bluesky) ブリッジ

[concrnt/atproto](https://github.com/concrnt/atproto) を `atproto.enabled: true`
で同梱します。ActivityPub ブリッジ (`activitypub2`) と同じ位置づけで、
concrnt2 の `services` に自動登録され、ゲートウェイ経由で
`/atproto/*` (管理 API・認証あり) と `/xrpc/*` (公開 PDS 面・`noAuth`) が
公開されます。PDS の serviceEndpoint は `concrnt2.fqdn` そのものなので
専用ドメインやワイルドカード DNS/TLS は不要です。
ブリッジの Service (`atproto:8010`) はクラスタ内部専用で、Ingress 等で
直接公開してはいけません (管理 API はゲートウェイが付ける
`cc-requester` ヘッダを信頼します)。

```yaml
atproto:
  enabled: true
  useSecret: true
  image: ghcr.io/concrnt/atproto:v0.1.0
  ccid: con1...            # conctl operation create-account で作成したサービスアカウント
  storage: 4Gi             # carstore / blob / firehose ログ (StatefulSet の PVC)
```

`useSecret: true` の場合は Secret `atproto-secret` を用意し、
`secret.yaml` キーに秘匿値だけを部分 YAML で書きます
(ConfigMap の `config.yaml` の後に読まれ、深いマージで上書きされます):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: atproto-secret
stringData:
  secret.yaml: |
    concrnt:
      privateKey: <サービスアカウントの秘密鍵 64桁hex>
    atproto:
      masterRotationKey: <K256 秘密鍵 (multibase か hex)。全ブリッジ DID の PLC rotation key>
      keyEncryptionKey: <32バイト hex。アカウント毎の署名鍵の保存時暗号化キー>
```

`masterRotationKey` が漏洩すると全ブリッジアカウントの DID を乗っ取れます。
`keyEncryptionKey` を変えると既存アカウントの署名鍵が復号できなくなります。
どちらも PVC と同様にバックアップ対象にしてください。

DB は既定で v2 コアと同じ `concrnt` データベースを共有します
(テーブルは全て `at_` プレフィックス)。Redis は**必ず**コアと同じインスタンスを
指してください (投稿イベントを PSUBSCRIBE で購読します)。

firehose (`wss://<fqdn>/xrpc/com.atproto.sync.subscribeRepos`) は
ゲートウェイの ReverseProxy を WebSocket で通ります。デプロイ後に疎通を
確認してください。`observability.createPodMonitors` が有効なら
ブリッジの `/metrics` を pod ポート直接で scrape する PodMonitor も作られます。
