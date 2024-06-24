---
title: "Go (Echo) サーバで Firebase Auth 由来の JWT の認可ミドルウェアをぱぱっと作る"
emoji: "🫥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [go, firebase, firebaseauth, jwt, echo]
published: true
---

Go (Echo) の API サーバで, Firebase Auth が発給した JWT による認可を行います.

## 0. (準備) フロント側で認証 (Authentication) を完了して JWT を取得する

ここは, 今回の本題ではないですが, フロントやモバイルアプリにおいて, Firebase の各種 SDK を用いることにより, 認証を完了させておきます.

React, React Native, Flutter, Swift, Kotlin など, それぞれのやり方で, 認証まで終わらせておいてください.

言語により異なりますが, 例えば以下のような関数で, JWT Token が取得できるようになれば完璧です.

```typescript
await auth.currentUser.getIdToken();
```

ちなみに, 余談ですが, 私は今回のプロジェクトでは, クライアントに React Native/Expo を用いているので, `react-native-firebase` ライブラリを用いています. こちらの実装も, バージョン間の変更が多く, 情報が少なくて大変だったので, いずれ記事にしようかなと思います. (が, React Native の情報がどれくらい需要があるか微妙なので後回しになっています)

Auth (認証認可) 全体のフロー図としては以下のような感じです.

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-06_at_11.53.57.png)

<blockquote class="twitter-tweet"><p lang="ja" dir="ltr">創ってるアプリの認証認可フロー図<br><br>Firebaseは完全に『JWT発給専門機関』としてだけ使うアーキテクチャで、JS SDKだけで完結できるのでios/android個別コードを書かなくてすむ<br>FirestoreはAPIサーバなしで直接ならありなのかもだけど、サーバ介すなら同じマネージドNoSQLならMongoDB Atlasとかが好き <a href="https://t.co/bcZYBJ7J4k">pic.twitter.com/bcZYBJ7J4k</a></p>&mdash; Taniii (@taniiicom) <a href="https://twitter.com/taniiicom/status/1798324234047136056?ref_src=twsrc%5Etfw">June 5, 2024</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## 1. (準備) API リクエストの際に, ヘッダに `Authorization: Bearer <token>` を付加する

これも, 各種クライアント側から API を叩くときに, Header に付与します.

ここでは一例として, React や React Native の場合は, `axios-jwt` というライブラリがあり, JWT の保持, リクエストへの付加, リフレッシュを担ってくれます. API を叩くときに自動で, `Authorization: Bearer <token>` を加えた状態でリクエストしてくれるので, 個人的に愛用しています. ただ, 自前で実装しても数行で実現できるので, お好みです.

https://www.npmjs.com/package/axios-jwt

https://github.com/jetbridge/axios-jwt

https://qiita.com/chanuu/items/b1ae3939e149ec12a166

## 2. Go 用の Firebase Admin SDK を導入する

ここから本題です.

JWT 自体は共通規格で, Google の公開鍵も公開されていますので, Firebase Admin SDK を使うのは必須ではありません. このあたりが, コンテキストがカプセル化している利点ですね.

ただ, Admin SDK を使うといろいろ便利なことがある (かつ, 自前で実装するのは気をつけないと脆弱性の入り込む余地が多分にある) ので, 今回は使うことにします.

汎用の JWT として解析, 認可している方法については, 以下などが参考になります.

https://note.com/pharmax/n/nacbf8afb3b42

では, こちらは, Admin SDK を導入してみます.

```bash
go get firebase.google.com/go
go get google.golang.org/api/option
```

## 3. Firebase サービスアカウントを発行

[https://console.firebase.google.com/project/[project-id]/settings/serviceaccounts/adminsdk?hl=ja](https://console.firebase.google.com/project/[project-id]/settings/serviceaccounts/adminsdk?hl=ja)

- `[project-id]`: Firebase プロジェクトの id

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-24_at_12.56.38.png)

これによって得られた, json ファイルをプロジェクトルートなどに設置します.

## 4. 認可ミドルウェア (auth-middleware) の作成

いよいよ, 認可ミドルウェア (auth-middleware) を作ります.

Echo のミドルウェア機能を使います.

```bash
go get github.com/labstack/echo/v4/middleware
```

\> `/middleware/auth.go`

```go
package middleware

import (
	"context"
	"fmt"
	"net/http"
	"strings"

	firebase "firebase.google.com/go"
	"github.com/labstack/echo/v4"
	"google.golang.org/api/option"
)

var firebaseApp *firebase.App

func InitFirebase() error {
	opt := option.WithCredentialsFile("[firebase-service-account-key.json]")
	app, err := firebase.NewApp(context.Background(), nil, opt)
	if err != nil {
		return err
	}
	firebaseApp = app
	return nil
}

func AuthMiddleware(next echo.HandlerFunc) echo.HandlerFunc {
	return func(c echo.Context) error {
		authHeader := c.Request().Header.Get("Authorization")
		if authHeader == "" {
			return echo.NewHTTPError(http.StatusUnauthorized, "missing or invalid token")
		}

		token := strings.TrimPrefix(authHeader, "Bearer ")
		ctx := context.Background()
		client, err := firebaseApp.Auth(ctx)
		if err != nil {
			return echo.NewHTTPError(http.StatusInternalServerError, "failed to initialize Firebase Auth client")
		}

		decodedToken, err := client.VerifyIDToken(ctx, token)
		if err != nil {
			return echo.NewHTTPError(http.StatusUnauthorized, "invalid token")
		}

		fmt.Println(decodedToken)

		// クレームの内容を検証する
		uid := decodedToken.UID
		if uid == "" {
			return echo.NewHTTPError(http.StatusUnauthorized, "invalid token claims")
		}

		email, ok := decodedToken.Claims["email"].(string)
		if !ok || email == "" {
			return echo.NewHTTPError(http.StatusUnauthorized, "invalid token claims")
		}

		emailVerified, ok := decodedToken.Claims["email_verified"].(bool)
		if !ok || !emailVerified {
			return echo.NewHTTPError(http.StatusUnauthorized, "invalid token claims")
		}

		// ユーザー情報をコンテキストに設定
		c.Set("uid", uid)
		c.Set("email", email)

		return next(c)
	}
}
```

- `[firebase-service-account-key.json]`: さっきプロジェクトルートに置いたサービスアカウントの鍵ファイルを指定してください (実際のプロジェクトでは `.env` などで適宜環境変数にするなどしてください)

\> `/main.go`

```go
package main

import (
	"log"
	"net/http"

	"github.com/labstack/echo/v4"
	echomiddleware "github.com/labstack/echo/v4/middleware"
	"github.com/taniiicom/example/controller"
	"github.com/taniiicom/example/domain"
	"github.com/taniiicom/example/middleware"
	"github.com/taniiicom/example/repository"
	"github.com/taniiicom/example/usecase"
)

func main() {
	e := echo.New()

	// middleware
	if err := middleware.InitFirebase(); err != nil {
		log.Fatalf("Failed to initialize Firebase: %v", err)
	}
	e.Use(echomiddleware.Logger())
	e.Use(echomiddleware.Recover())
	// 認可ミドルウェアの適用
	e.Use(middleware.AuthMiddleware)
	// -

	// ...

    e.GET("/", func(c echo.Context) error {
        return c.String(http.StatusOK, "Hello, World!")
    })

    e.Logger.Fatal(e.Start(":8080"))
}
```

- ☆ **認可により, 保証された `uid` や `email` などは, `context` に含めておきます. これを, 各ルートで利用します.**

## 動作例

実際に, Go サーバを立ち上げて, insomnia で確かめてみます.

使う JWT は, フロントの実装でログインしたのをデバッグ環境で拝借して使います.

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-24_at_13.39.19.png)

まずは, `Authorization: Bearer <token>` を付加せずに実行してみると...

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-24_at_13.33.46.png)

正しく, `401 Unauthorized` になります.

では, insomnia の `Auth > Bearer Token` を選択し, Firebase Auth が発行した JWT を `TOKEN` 欄に入力します.

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-24_at_13.34.02.png)

![](/images/go-firebase-auth-jwt/Screenshot_2024-06-24_at_13.37.48.png)

正しく表示されました!

今回はこれで完成です.

## epilogue

最近, `トークン認証方式` をソフトウェアエンジニア以外の人に説明するときに, **『パスポート』**を例にあげて説明しています.

パスポートも, 偽造されていないということをいろいろな偽造防止技術でなんとか保証することで, それをみた人は, 基本的に発給元にいちいち問い合わせることなく, そこに書かれている情報を信用しています.

なぜ, トークン認証が可能なのか, 直感的にわかりやすいんじゃないかなと思っています.
