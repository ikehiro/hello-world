# Snowflake キーペア認証（パスフレーズ付き秘密鍵）× AWS コンテナ ― JWT エラー対応メモ

AWS 上のコンテナイメージから Snowflake へ、キーペア認証（パスフレーズ付き秘密鍵）で接続する際に
`JWT token is invalid` (390144) が発生するケースの切り分けと、鍵の取り扱いに関する整理。
コネクタは **snowflake-connector-python / dbt** を前提とする。

-----

## 1. まず押さえる大前提：JWT エラー＝鍵の復号は成功している

- パスフレーズが間違っていれば、その手前で `cryptography` の `load_pem_private_key` が
  `ValueError`（“Bad decrypt” 等）を投げて止まり、**JWT 生成まで到達しない**。
- 逆に `JWT token is invalid` まで進んでいるなら、鍵のロード自体は成功している。
  → 問題は次のどちらか：
1. **復号後の秘密鍵が、Snowflake 登録済みの公開鍵と一致していない**（フィンガープリント不一致）
1. **account / user の指定が JWT クレームと噛み合っていない**

|出ているエラー               |切り分け                                   |
|----------------------|---------------------------------------|
|鍵の復号エラー（Bad decrypt 等）|パスフレーズ／鍵文字列の破損を疑う（§3）                  |
|`JWT token is invalid`|フィンガープリント不一致 or account/user 指定（§2, §6）|

-----

## 2. 最短の切り分け：フィンガープリントの突き合わせ

「実際にロードされた鍵」から公開鍵フィンガープリントを計算し、`DESC USER` の値と比較する。

### Snowflake 側

```sql
DESC USER <username>;
-- RSA_PUBLIC_KEY_FP の値を確認（SHA256:... 形式）
```

### コンテナ内（実際に使っている鍵で）― Python

```python
import base64, hashlib
from cryptography.hazmat.primitives import serialization

passphrase = os.environ["SNOWFLAKE_PRIVATE_KEY_PASSPHRASE"]  # strip 注意（§3）
with open("rsa_key.p8", "rb") as f:
    p_key = serialization.load_pem_private_key(f.read(), password=passphrase.encode())

pub_der = p_key.public_key().public_bytes(
    encoding=serialization.Encoding.DER,
    format=serialization.PublicFormat.SubjectPublicKeyInfo,
)
print("SHA256:" + base64.b64encode(hashlib.sha256(pub_der).digest()).decode())
```

### openssl 版

```bash
openssl rsa -pubout -in rsa_key.p8 -passin pass:<passphrase> 2>/dev/null | \
  openssl pkey -pubin -outform DER 2>/dev/null | \
  openssl dgst -sha256 -binary | openssl enc -base64
```

**判定：**

- 一致しない → 鍵の取り違え（古い鍵の残存／`RSA_PUBLIC_KEY` と `RSA_PUBLIC_KEY_2` の混同／ローテーション後の不整合）
- 一致する → account / user 側の問題（§6）へ

-----

## 3. コンテナ + Secrets Manager 特有の落とし穴

- **パスフレーズの末尾改行・空白**：Secrets Manager の JSON シークレットから取り出した値は
  トリム必須。`.strip()` を入れる。
- **秘密鍵 PEM の改行破損**：プレーン文字列で格納すると改行が消えて 1 行化したり
  `\n` リテラルに化けたりして、鍵としてパースできなくなる。
  → 対策：**PEM を丸ごと base64 化して格納 → 取り出し時にデコード**（§5）。

-----

## 4. PEM とは

PEM は「鍵そのもの」ではなく、**鍵（中身はバイナリ＝DER）をテキストとして表現するための入れ物
（エンコード形式）**。バイナリを base64 テキストに変換し、前後に目印の行を付けたもの。

```
-----BEGIN ENCRYPTED PRIVATE KEY-----
MIIFLTBXBgkqhkiG9w0BBQ0wSjApBgkqhkiG9w0BBQwwHAQI...
（base64 のランダムな文字列が複数行続く）
...kZ9Qz3vH2bN8mP1aS5dF7gH==
-----END ENCRYPTED PRIVATE KEY-----
```

- **PEM ＝ テキスト版 / DER ＝ バイナリ版**。中身の鍵としては同じ。
- PEM は `PEM = base64(DER) + BEGIN/END 行`。

### BEGIN 行の種類で鍵の種類がわかる

|先頭行                                    |意味                 |Snowflake 用途     |
|---------------------------------------|-------------------|-----------------|
|`-----BEGIN ENCRYPTED PRIVATE KEY-----`|PKCS#8・パスフレーズ**付き**|◎ パスフレーズ付きならこれが正解|
|`-----BEGIN PRIVATE KEY-----`          |PKCS#8・パスフレーズ**なし**|○                |
|`-----BEGIN RSA PRIVATE KEY-----`      |PKCS#1（旧形式）        |△ 形式変換が必要なことがある  |

確認コマンド：

```bash
head -1 rsa_key.p8
```

### 公開鍵との関係

- Snowflake に登録するのは公開鍵側（`-----BEGIN PUBLIC KEY-----`）。
- `ALTER USER ... SET RSA_PUBLIC_KEY='...'` に貼るときは、**BEGIN/END 行を外した base64 本体だけ**を渡す。
- 秘密鍵（手元で署名）と公開鍵（Snowflake 登録）がペア。対応が崩れると JWT エラー。

-----

## 5. base64 でくるむときのデコード回数

base64 の層は 2 つ存在するが、**自分で明示的にデコードするのは 1 回だけ**。
「自分で 1 回、ライブラリが内部でもう 1 回」という分担になる。

- **内側の base64**：PEM 形式そのものに含まれる（BEGIN/END の間の文字列）。
  → `load_pem_private_key` が内部で自動デコードする。
- **外側の base64**：Secrets Manager に改行を壊さず入れるために、自分でくるんだもの。
  → これだけを自分でデコードする。

```python
import base64
from cryptography.hazmat.primitives import serialization

# 1. Secret から取り出す（中身は base64(PEM) の文字列）
secret_str = get_secret_from_secrets_manager()

# 2. 外側の base64 を「自分で」1回デコード → PEM テキストが復元される
pem_bytes = base64.b64decode(secret_str)

# 3. PEM をそのまま渡す。内側の base64 は load_pem_private_key が自動でデコード
p_key = serialization.load_pem_private_key(pem_bytes, password=passphrase.encode())
```

> 手順 2 を忘れて base64(PEM) のままを `load_pem_private_key` に渡すと PEM として読めずエラーになる。

### さらにシンプルにする選択肢：DER を base64 化して保存

PEM の BEGIN/END 行を扱わずに済み、改行破損の心配がなくなる。

```python
der_bytes = base64.b64decode(secret_str)               # 自分でのデコードは1回
p_key = serialization.load_der_private_key(der_bytes, password=passphrase.encode())
```

パスフレーズ付き DER（PKCS#8 ENCRYPTED）はそのまま base64 化でき、暗号化を維持したまま保存可能。

-----

## 6. 接続コードと account 識別子

### 推奨：復号済み鍵を DER バイトにして `private_key` に渡す

```python
pkb = p_key.private_bytes(
    encoding=serialization.Encoding.DER,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption(),
)
con = snowflake.connector.connect(
    account="ORG-ACCOUNT",   # 形式に注意（下記）
    user="SVC_ELT",
    private_key=pkb,
    ...
)
```

### dbt（profiles.yml）

```yaml
outputs:
  prod:
    type: snowflake
    account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
    user: "{{ env_var('SNOWFLAKE_USER') }}"
    private_key_path: /run/secrets/rsa_key.p8
    private_key_passphrase: "{{ env_var('SNOWFLAKE_PRIVATE_KEY_PASSPHRASE') }}"
    role: ...
    warehouse: ...
    database: ...
    schema: ...
```

### account 識別子の注意

- JWT の `iss` / `sub` は account と user を**大文字**で組み立てる。
- `account` に `xxxxx.snowflakecomputing.com` を**含めない**（含めると JWT が壊れる）。
- `ORG-ACCOUNT` 形式かアカウントロケータ形式（例 `xy12345.ap-northeast-1.aws`）を、
  末尾ドメインなしで指定。リージョンまたぎ構成ではロケータ形式の指定ミスが JWT エラーの定番。

-----

## 7. チェックリスト

- [ ] `head -1 rsa_key.p8` で `ENCRYPTED PRIVATE KEY`（PKCS#8 パスフレーズ付き）になっているか
- [ ] ロード済み鍵のフィンガープリントが `DESC USER` の `RSA_PUBLIC_KEY_FP` と一致するか
- [ ] `RSA_PUBLIC_KEY` / `RSA_PUBLIC_KEY_2` のどちらを使うか、登録と一致しているか
- [ ] Secrets Manager 由来のパスフレーズを `.strip()` しているか
- [ ] PEM の改行が壊れていないか（base64 でくるむ or DER 化で回避）
- [ ] `account` に末尾ドメインを含めていないか／形式は正しいか
- [ ] コンテナの時刻ずれ（JWT の iat/exp 由来の失効）がないか
- [ ] エラー文言の括弧内（`JWT token is invalid [account.user]`）で account/user を確認したか