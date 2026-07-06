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
