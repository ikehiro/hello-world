はい、**証明書認証**が実質的にキーペア認証です。

サービスプリンシパルの認証方式は3つあります。

- **クライアントシークレット**: パスワード方式。有効期限は最大24か月。
- **証明書**: 公開鍵(.cer/.pem)をアプリ登録に登録し、クライアント側は秘密鍵で署名したJWT(client assertion)で認証します。これがキーペア認証にあたります。
- **フェデレーション資格情報(ワークロードID連携)**: GitHub ActionsやKubernetesなどの外部IdPのトークンと交換する方式。シークレットも鍵も保管不要で、CI/CDでは今の推奨です。

補足です。

- 証明書は自己署名でも使えます。Azure側に登録するのは公開鍵だけで、秘密鍵は渡りません。
- Azure CLIなら `az ad sp create-for-rbac --cert @cert.pem`、または `--create-cert` で作成できます。ログインは `az login --service-principal --username <appId> --tenant <tenant> --certificate <pem>` です(PEMに秘密鍵を含める必要があります)。
- Azure上で動く処理なら、そもそもマネージドIDを使えば鍵の管理自体が不要です。

SnowflakeのRSAキーペア認証のように、公開鍵を登録して秘密鍵で署名する発想は同じです。ただしAzureでは「証明書」という形になります。

用途(CI/CD、Snowflake連携など)を教えてもらえれば、どれが合うかもう少し絞れます。