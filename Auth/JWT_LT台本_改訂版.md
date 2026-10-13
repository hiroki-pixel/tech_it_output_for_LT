# JWTを理解する ― セッション認証との違いから考える

## 想定発表時間

10〜15分

---

# 0. はじめに【約1分】

前回は「セッションを用いた認証」について話しました。

HTTPはステートレスなので、ログインしたユーザーの状態をそのまま覚えておくことはできません。

そこでセッション認証では、

```text
ログイン成功
↓
サーバー側にセッション情報を保存
↓
クライアントに session_id を渡す
↓
以降は session_id を使ってユーザーを特定
```

という仕組みを使っていました。

例えば、

```text
session_id = abc123
↓
user_id = 100
```

という対応関係を、サーバー側のDBやRedisなどに持ちます。

では、ここで一つ疑問があります。

> **毎回サーバー側のセッション情報を確認しなくても、  
> 「このユーザーは誰なのか」を判断する方法はないのか？**

このような場面で使われる仕組みの一つが **JWT** です。

ただし最初に重要なことを言うと、

> **JWTは「本人確認の方法」そのものではありません。**

ID・Passwordやパスキーなどで本人確認した**後**に、

> **認証・認可に必要な情報を、どう持ち回るか**

という話です。

---

# 1. 「セッション認証」と「JWT認証」は何が違う？【約1分30秒】

よく「セッション認証」「JWT認証」と並べて説明されます。

ただし厳密には、どちらも最初の本人確認自体は同じようにできます。

```text
ID / Password
↓
サーバーで照合
↓
本人確認OK
```

違いは、その後です。

## セッションの場合

```text
本人確認OK
↓
session_id を発行
↓
サーバー側に認証状態を保存
↓
各リクエストで Session Store を参照
```

## JWTを使う場合

```text
本人確認OK
↓
署名付きJWTを発行
↓
クライアントがJWTを持つ
↓
各APIサーバーがJWTを検証
```

つまり、

> **セッションは「サーバー側に保存した状態」を信頼する。  
> JWTは「署名された情報」を検証して信頼する。**

という違いです。

今回の発表では、この違いを中心にJWTを見ていきます。

---

# 2. JWTとは何か【約1分】

JWTは、**JSON Web Token** の略です。

RFC 7519で定義されている、情報をやり取りするためのトークン形式です。

JWTは、見た目としては次のようになっています。

```text
xxxxx.yyyyy.zzzzz
```

`.` で3つの部分に分かれています。

```text
Header.Payload.Signature
```

それぞれ、

```text
Header
→ JWTをどう署名したか

Payload
→ 誰なのか、有効期限などの情報

Signature
→ 内容が改ざんされていないことを確認するための署名
```

という役割があります。

---

# 3. Header / Payload / Signature【約2分】

## Header

例えば、

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

のような情報が入ります。

`alg` は、署名に使うアルゴリズムです。

## Payload

例えば、

```json
{
  "sub": "user123",
  "exp": 1791200000,
  "role": "user"
}
```

のような情報を持たせることができます。

```text
sub
→ このJWTが誰を表しているか

exp
→ 有効期限

role
→ ユーザーの権限
```

こうした情報を **Claim（クレーム）** と呼びます。

ここで重要なのが、

> **Payloadは暗号化されているわけではない**

ということです。

JWTを持っていれば、Payloadの中身自体は読むことができます。

なので、

```text
パスワード
クレジットカード番号
秘密情報
```

のようなものを入れる場所ではありません。

## Signature

ここがJWTの重要な部分です。

イメージとしては、

```text
Header
  +
Payload
  +
秘密鍵など
  ↓
署名アルゴリズム
  ↓
Signature
```

となります。

Signatureによって、

> **HeaderやPayloadが途中で書き換えられていないか**

を確認できます。

---

# 4. Payloadが読めるのに、なぜ信用できるのか【約1分30秒】

例えば、元のPayloadが、

```json
{
  "sub": "user123",
  "role": "user"
}
```

だったとします。

攻撃者が、

```json
{
  "sub": "user123",
  "role": "admin"
}
```

に書き換えたとします。

Payload自体を書き換えることはできます。

しかし、Payloadが変わると正しいSignatureも変わります。

攻撃者は署名に必要な秘密情報を持っていないので、

```text
Payloadを変更
↓
Signatureと一致しなくなる
↓
サーバーがJWTを拒否
```

となります。

つまり、

> **JWTは「Payloadが読めないから安全」なのではなく、  
> Signatureによって改ざんを検知できるから信頼できる**

という仕組みです。

---

# 5. JWTを使ったAPIアクセス【約1分30秒】

ログイン後、認証サーバーからJWT形式のAccess Tokenが発行されたとします。

```text
Client
  │
  │ ID / Password
  ▼
認証サーバー
  │
  │ Access Token
  ▼
Client
```

その後APIを呼び出すときには、例えばHTTPのAuthorizationヘッダーでTokenを送ります。

```http
Authorization: Bearer <Access Token>
```

APIサーバーでは、

```text
Access Tokenを受け取る
↓
Signatureを検証
↓
expなどのClaimを確認
↓
問題なければAPI実行
```

となります。

セッションの場合は、

```text
session_id
↓
Session Storeを検索
↓
誰なのかを確認
```

でした。

一方JWTの場合は、

```text
JWT
↓
署名を検証
↓
Payloadから誰なのかを確認
```

できます。

そのため、

> **各APIサーバーが認証サーバーやSession Storeに毎回問い合わせずに、Tokenを検証しやすい**

という特徴があります。

複数のAPIやマイクロサービスが存在する構成では、特に使いやすい考え方です。

---

# 6. JWTの弱点 ― 一度発行すると無効化しづらい【約1分】

ただし、JWTにも困るところがあります。

例えば、

```text
Access Token
有効期限：30日
```

というTokenを発行したとします。

JWTをステートレスに検証している場合、

```text
署名OK
有効期限OK
↓
利用可能
```

となるので、

> **サーバー側から途中で「このJWTだけ今すぐ無効」とするのが難しい**

という問題があります。

そこで、Access Tokenの有効期限を短くします。

例えば、

```text
Access Token
有効期限：15分
```

のようにします。

ただしこれでは、15分ごとにID・Passwordを入力する必要が出てしまいます。

そこで登場するのが **Refresh Token** です。

---

# 7. Access TokenとRefresh Token【約2分】

まず役割を簡単に言うと、

```text
Access Token
= APIを使うための入場券

Refresh Token
= 入場券を再発行してもらうための引換券
```

です。

ログインすると、

```text
Access Token
→ 有効期限が短い

Refresh Token
→ 有効期限が長い
```

という2種類のTokenを受け取る構成があります。

普段APIを呼ぶときに使うのはAccess Tokenです。

```text
Client
↓ Access Token
API Server
```

Access Tokenの期限が切れたら、

```text
Access Token期限切れ
↓
Refresh Tokenを認証サーバーへ送る
↓
Refresh Tokenを確認
↓
新しいAccess Tokenを発行
```

という流れになります。

つまりRefresh Tokenは、

> **APIを直接使うためのTokenではなく、  
> 新しいAccess Tokenを発行するためのToken**

です。

---

# 8. ではRefresh Tokenを盗まれたら？【約1分】

ここで、

> 「Refresh Tokenが長期間有効なら、セッションIDを盗まれるのと結局同じでは？」

という疑問が出てきます。

これはかなり重要です。

結論としては、

> **Refresh Tokenも盗まれたら危険です。**

Refresh Tokenを盗まれると、新しいAccess Tokenを発行される可能性があります。

そのため、

```text
Access Token
→ 短命にする

Refresh Token
→ より厳重に保管する
→ 必要に応じてDB等で失効管理する
→ Rotationを行う
```

といった設計をします。

つまりJWTを使ったからといって、

> **完全にサーバー側の状態管理がなくなるわけではありません。**

---

# 9. Signatureがあるなら、JWTは盗まれても安全？【約1分】

ここも誤解しやすいポイントです。

Signatureが防いでいるのは、

```text
JWTを改ざんする
```

ことです。

一方、

```text
正規のJWTそのものを盗む
```

ことを防いでいるわけではありません。

```text
JWTを改ざん
↓
Signature不一致
↓
拒否できる
```

しかし、

```text
正規のAccess Tokenを盗む
↓
そのままAPIへ送信
↓
有効期限内なら利用される可能性がある
```

となります。

つまり、

> **Signatureが防ぐのは「盗難」ではなく「改ざん」**

です。

これはセッションIDでも同じです。

```text
session_idを盗まれる
↓
なりすまし可能

Access Tokenを盗まれる
↓
なりすまし可能
```

なので、

> **JWTだからセッションより安全**

とは単純には言えません。

---

# 10. セッションとJWTの比較【約1分】

| 観点 | セッション | JWTをAccess Tokenとして利用 |
| --- | --- | --- |
| 本人確認 | ID/Passwordなど | ID/Passwordなど |
| 認証後に持つもの | session_id | JWT |
| 認証情報の主な置き場所 | サーバー側 | Token側 |
| リクエスト時 | Session Storeを参照 | JWTを検証 |
| 即時失効 | 比較的しやすい | ステートレス構成では難しい |
| 複数APIでの利用 | Session Store共有などが必要 | 各APIで検証しやすい |
| 盗まれた場合 | 危険 | 危険 |

つまり、どちらが絶対に優れているという話ではありません。

用途によって使い分ける必要があります。

---

# 11. まとめ【約1分】

今回のポイントは4つです。

### 1. JWTは本人確認方法そのものではない

```text
ID / Password
パスキー
MFA
```

などで本人確認した**後**の情報を持ち回るために利用できます。

### 2. JWTは3つの要素で構成される

```text
Header.Payload.Signature
```

### 3. Signatureが守るのは改ざん

```text
Payloadを書き換える
↓
Signature検証に失敗
```

ただしTokenそのものを盗まれれば利用される可能性があります。

### 4. セッションとの大きな違い

> **セッションは「サーバー側に保存した状態」を信頼する。  
> JWTは「署名された情報」を検証して信頼する。**

JWTは、

> **認証後の情報をどう持ち回るか**

という問題に対する一つのアプローチです。

---

# 補足：LTで入れると分かりやすいデモ

JWT.ioを使って、実際のJWTを表示します。

```text
Header
Payload
Signature
```

に分解されるところを見せます。

そのうえでPayloadの、

```json
"role": "user"
```

を、

```json
"role": "admin"
```

に変更します。

Payload自体は変更できますが、Signature Verificationが失敗します。

ここで、

> **「JWTは読めないTokenではなく、改ざんを検知できるToken」**

ということを視覚的に説明できます。

---

# 参考文献・参考サイト

## RFC 7519 - JSON Web Token (JWT)

JWTそのものを定義している標準仕様。

https://www.rfc-editor.org/rfc/rfc7519.html

## RFC 8725 - JSON Web Token Best Current Practices

JWTを安全に利用するためのBest Current Practice。

https://www.rfc-editor.org/rfc/rfc8725.html

## RFC 6750 - OAuth 2.0 Bearer Token Usage

`Authorization: Bearer <token>` など、Bearer Tokenの利用方法を定義。

https://www.rfc-editor.org/rfc/rfc6750.html

## RFC 9068 - JWT Profile for OAuth 2.0 Access Tokens

JWTをOAuth 2.0のAccess Tokenとして利用する場合の仕様。

https://www.rfc-editor.org/rfc/rfc9068.html

## RFC 9700 - Best Current Practice for OAuth 2.0 Security

Access Token、Refresh Token、Refresh Token Rotationなどのセキュリティ上の推奨事項。

https://www.rfc-editor.org/rfc/rfc9700.html

## OWASP JSON Web Token Cheat Sheet

JWT利用時のセキュリティ上の注意点を実践的に確認できる資料。

https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html

## JWT.io

JWTをデコードし、Header / Payload / Signatureを確認できる学習・デモ用サイト。

https://jwt.io/

## 【2025年版】JWT認証を完全マスター｜初心者エンジニア向け

JWTの構造、Access Token / Refresh Token、認証フローなどを初心者向けに整理した解説記事。

https://soft.takinoko.net/2025/10/04/%E3%80%902025%E5%B9%B4%E7%89%88%E3%80%91jwt%E8%AA%8D%E8%A8%BC%E3%82%92%E5%AE%8C%E5%85%A8%E3%83%9E%E3%82%B9%E3%82%BF%E3%83%BC%E5%88%9D%E5%BF%83%E8%80%85%E3%82%A8%E3%83%B3%E3%82%B8%E3%83%8B%E3%82%A2/
