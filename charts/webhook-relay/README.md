# webhook-relay

[concrnt/webhook-relay](https://github.com/concrnt/webhook-relay) をデプロイするチャートです。
任意のサービスからのwebhook POSTをテンプレートで整形し、concrnt v2のタイムラインへ投稿します。

`values.yaml` の `webhooks[]` はそのまま webhook-relay の `config.yaml` に出力されます。
各キーの意味は webhook-relay 側の README を参照してください。

秘密鍵は既定では ConfigMap に平文で出力されます。`useSecret: true` にすると ConfigMap には出力せず、
Secret `webhook-relay-secret` の `PRIVATE_KEY` キーを環境変数として読みます。
