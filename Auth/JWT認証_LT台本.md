# JWT認証を初心者向けに理解する

## 0. はじめに【約1分】

今回は「JWT」について話します。

前回は、セッションを使った認証について説明しました。

セッション認証では、ログインするとサーバー側に、

```text
session_id = abc123
↓
user_id = 100
```

のような情報を保持します。

クライアントはCookieなどで `session_id` を送り、サーバーがセッションストアを確認することで、

> 「この人はログイン済みのuser100だ」

と判断できます。

では、

> **サーバー側に毎回セッション情報を問い合わせずに、ユーザーを確認することはできないのか？**

この考え方を理解するうえで登場するのがJWTです。

---

## 1. JWTとは何か【約1分30秒】

JWTは、

**JSON Web Token**

の略です。

厳密には「認証方式そのもの」ではなく、

> **情報を安全にやり取りするためのトークン形式**

です。

RFC 7519という標準仕様で定義されています。

JWTを見ると、

```text
xxxxx.yyyyy.zzzzz
```

という文字列になっています。

実際には3つの部分に分かれています。

```text
Header.Payload.Signature
```

です。

---

## 2. Header / Payload / Signature【約2分】

まずHeader。

例えば、

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

のように、

> 「どのアルゴリズムで署名したのか」

などの情報が入っています。

次にPayload。

```json
{
  "sub": "user123",
  "exp": 1791200000,
  "role": "user"
}
```

例えば、

```text
sub  → このJWTが誰のものか
exp  → 有効期限
role → ユーザーの権限
```

などを入れられます。

こうした情報を **Claim（クレーム）** と呼びます。

ここで重要なのが、

> **Payloadは暗号化されているとは限らない**

ということです。

JWTのPayloadは簡単にデコードできます。

なので、パスワードなどの秘密情報を入れるものではありません。

最後がSignatureです。

これがJWTの重要な部分です。

```text
Header
  +
Payload
  +
秘密鍵など

↓ 署名

Signature
```

Signatureによって、

> 「このPayloadが途中で改ざんされていない」

ことを確認できます。

---

## 3. JWTはなぜ改ざんできないのか【約1分30秒】

例えばPayloadに、

```json
{
  "sub": "user123",
  "role": "user"
}
```

と入っていたとします。

悪意のある人が、

```json
{
  "sub": "user123",
  "role": "admin"
}
```

に書き換えたとします。

Payload自体を書き換えることはできます。

しかし、Payloadが変われば正しいSignatureも変わります。

攻撃者は署名に必要な秘密鍵を持っていないので、

```text
Payloadを改ざん
↓
Signatureと一致しない
↓
サーバーが拒否
```

となります。

つまり、

> **Payloadが読めないから安全なのではなく、改ざんを検知できるから信頼できる**

というのがJWTのポイントです。

---

## 4. JWTを使ったAPI呼び出し【約2分】

ここから実際の流れを見ます。

まずユーザーがログインします。

```text
Client
   │
   │ ID / Password
   ▼
認証サーバー
```

認証に成功すると、認証サーバーがTokenを発行します。

```text
認証サーバー
   │
   │ Access Token
   ▼
Client
```

APIを呼び出すときには、例えばHTTPのAuthorizationヘッダーを使います。

```http
Authorization: Bearer <Access Token>
```

Bearer Tokenは、

> **そのTokenを持っている人が利用できるToken**

という考え方です。

そのため、Tokenそのものを盗まれると攻撃者にも利用される可能性があります。

APIサーバーでは、

```text
Access Tokenを受信
↓
署名を確認
↓
expなどを確認
↓
問題なければAPI実行
```

となります。

ここでセッションとの違いが出てきます。

---

## 5. セッションと何が違うのか【約1分30秒】

セッションの場合は、

```text
Client
   │ session_id
   ▼
API Server
   │
   ▼
Session Store

「このsession_idは誰？」
```

と確認します。

一方、JWTをAccess Tokenとして使う構成では、

```text
Client
   │ JWT
   ▼
API Server
   │
   ▼
署名検証

「正規に発行されたJWTだ」
```

とAPIサーバー自身で検証できます。

つまりJWTの大きなメリットの一つは、

> **各APIサーバーがTokenを自分で検証しやすい**

ことです。

複数のAPIやマイクロサービスが存在する構成では、特にメリットが出てきます。

---

## 6. Access TokenとRefresh Token【約2分】

ただし、JWTには問題があります。

一度JWTを発行すると、

> **サーバー側だけで即座に無効化するのが難しい**

ケースがあります。

そこでAccess Tokenには短い有効期限を設定します。

例えば、

```text
Access Token
有効期限：15分
```

とします。

ただ、15分ごとにユーザー名とパスワードを入力するのは大変です。

そこで登場するのが、

**Refresh Token**

です。

イメージとしては、

```text
Access Token
＝ APIを利用するための入場券

Refresh Token
＝ 入場券を再発行するための引換券
```

です。

```text
Access Token期限切れ
        ↓
Refresh Tokenを認証サーバーへ送る
        ↓
Refresh Tokenを確認
        ↓
新しいAccess Tokenを発行
```

こうすることで、

```text
Access Token → 短命

Refresh Token → 長命
```

という役割分担ができます。

ただしRefresh Tokenを盗まれると、新しいAccess Tokenを発行される危険があります。

そのため、Refresh TokenはAccess Token以上に厳重に管理する必要があります。

---

## 7. JWTならセッションより安全なのか？【約1分30秒】

ここは誤解しやすいところです。

JWTにはSignatureがあるので、

```text
JWTを書き換える
↓
Signature検証で検知
```

できます。

しかし、

```text
JWTそのものを盗む
↓
そのまま利用
```

は可能です。

つまり、

> **Signatureが防いでいるのはTokenの盗難ではなく、主に改ざん**

です。

これはセッションIDも同じです。

```text
session_idを盗まれる
↓
なりすまし可能

Access Tokenを盗まれる
↓
なりすまし可能
```

なので、

> 「JWTだからセッションより安全」

とは単純には言えません。

むしろ、

```text
セッション
→ サーバー側で状態管理しやすい
→ 即時失効させやすい

JWT Access Token
→ 各APIが自律的に検証しやすい
→ ステートレスな構成を作りやすい
→ その代わり失効管理などを考える必要がある
```

というトレードオフがあります。

---

## 8. JWTを使えば完全にステートレスなのか【約1分】

JWTについて、

> 「JWTはステートレス認証です」

と説明されることがあります。

これは半分正しいのですが、少し注意が必要です。

Access Tokenだけを見ると、

```text
署名検証
↓
認証情報を判断
```

できるため、ステートレスに処理できます。

しかし実際のシステムでは、

```text
Refresh Token
失効情報
Token Rotation
```

などをDBやRedisで管理することがあります。

つまり、

> **JWTを使えば何も状態管理しなくていい**

という意味ではありません。

---

## 9. まとめ【約1分】

最後にまとめます。

JWTは、

```text
Header.Payload.Signature
```

という構造を持つTokenです。

Payloadに、

```text
誰なのか
いつまで有効なのか
どんな権限なのか
```

といったClaimを持たせます。

Signatureによって、

> **情報が改ざんされていないことを検証できます。**

そしてJWTをAccess Tokenとして使うことで、

```text
Client
↓
JWT
↓
API Server
↓
署名検証
```

のように、APIサーバー自身でTokenを検証しやすくなります。

ただし、

```text
JWTだから安全
JWTだから完全ステートレス
JWTだからセッションより優れている
```

というわけではありません。

今日一番覚えてほしいのは、

> **セッションは「サーバー側に保存した状態」を信頼する。  
> JWTは「署名された情報」を検証して信頼する。**

という違いです。

この違いが分かれば、JWTが何をする技術なのかはかなり理解できると思います。

---

## 補足：発表時に入れると分かりやすいデモ

JWT.ioにサンプルJWTを貼り付けて、

```text
Header
Payload
Signature
```

が分解されるところを見せます。

その後Payloadの、

```json
"role": "user"
```

を、

```json
"role": "admin"
```

に変更し、

> 「Payload自体は簡単に変更できる。でもSignature Verificationが通らなくなる」

と見せます。

1分程度で、JWTの仕組みをかなり直感的に伝えられます。

---

# 参考文献・参考サイト

## RFC 7519 - JSON Web Token (JWT)

JWTそのものの標準仕様。Header、Claim、JWTの基本を確認するときの一次資料。

https://www.rfc-editor.org/rfc/rfc7519.html

## RFC 8725 - JSON Web Token Best Current Practices

JWTを安全に利用するためのBest Current Practice。

https://www.rfc-editor.org/rfc/rfc8725.html

## RFC 6750 - OAuth 2.0 Bearer Token Usage

Bearer Access TokenをHTTPでどのように利用するかを定義。

https://www.rfc-editor.org/rfc/rfc6750.html

## RFC 9068 - JWT Profile for OAuth 2.0 Access Tokens

OAuth 2.0のAccess TokenをJWT形式にする場合の標準。

https://www.rfc-editor.org/rfc/rfc9068.html

## RFC 9700 - Best Current Practice for OAuth 2.0 Security

Access Token・Refresh Token・Token Rotationなど、OAuth 2.0のセキュリティ推奨事項を確認する資料。

https://www.rfc-editor.org/rfc/rfc9700.html

## OWASP JSON Web Token Cheat Sheet

JWT実装時の脆弱性や対策を実践的に確認できる。

https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_Cheat_Sheet.html

## JWT.io

JWTを実際にデコードしてHeader / Payload / Signatureを見るための学習・デモ用サイト。

https://jwt.io/
