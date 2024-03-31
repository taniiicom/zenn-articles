---
title: "VScode 上での Git の GUI 操作でもコミットへの GPG 署名を可能にする"
emoji: "🔏"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git", "github", "gpg", "gnupg", "vscode"]
published: true
---

コミットメッセージが複数行書きやすかったり, 最近だとコミットメッセージ書く時に Copilot のアシストが効くので, 開発の時は git commit を VScode のサイドバー (左パネル) の Git (Source Control) を使って GUI ですることが多いのですが, コミットに GPG 署名をつける方法が一癖あり, 調べても全然情報がなかったので, 残しておきます.

## commit に GPG 署名をつける

- commit に GPG 署名をつけることで, 自分がそのコミットをしたことを証明できます
- 逆にいうと, 署名をしていない場合, コミットが自分のものかどうか判断するのは user.email や user.name でしかできないので, なりすましなどをされてしまう恐れがあります
- GitHub では, GPG key を, SSH key と同様に登録することができ, そのコミットが正しくそのユーザによるものだと確認がとれたコミットには, 左横に `Verified` と表示されます
  - GitHub 上で行われる merge などは, とくに設定しなくても, GitHub の鍵を用いて署名が行われています

![Screenshot 2024-03-31 at 19.31.45.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_19.31.45.png)

![Screenshot 2024-03-31 at 19.33.33.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_19.33.33.png)

GPG key を生成し, 署名を有効にする方法については, 以下の記事などが参考になります

cf.

https://qiita.com/noraworld/items/43cd1dd8c28931a766c1

mac の場合の gpg インストールは, brew で簡単にできます

```bash
brew install gpg
```

cf.

https://keruu.netlify.app/875efc6dd03473e3e5b2bffd7df9d3f3/

## VScode の Git 拡張の GUI 上で署名付きコミットをする

本題です.

まず, vscode の設定から, `Enable commit signing with GPG, X.509, or SSH.` を `true` にします.

![Screenshot 2024-03-31 at 19.46.39.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_19.46.39.png)

しかし, これだけでは以下のようなエラーが生じます.

![Screenshot 2024-03-31 at 19.49.17.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_19.49.17.png)

CLI から, `git commit -S` は問題なくできる場合でも, VScode の GUI からコミットした場合には秘密鍵が見つからないとなり, commit に失敗します.

CLI で実行する場合には, 以下のように `.zshrc` で, テレタイプデバイスを指定しているので, `passphrase` の要求などのインタラクティブな動作を行うことができます.

`.zshrc`

```bash
export GPG_TTY=$(tty)
```

そのため, おそらくこれは, VScode 内部で実行する時に秘密鍵の `passphrase` を渡せていないことにより生じているものなので, passphrase を渡す必要があるのですが, これを上手くやってくれる拡張機能を見つけました.

ついでに, 毎回 passphrase を入力するのも面倒なので, この拡張機能には, passphrase をキャッシュする機能があり非常に便利です.

https://marketplace.visualstudio.com/items?itemName=wdhongtw.gpg-indicator

https://github.com/wdhongtw/vscode-gpg-indicator

インストール後, VScode の下部に, 鍵のアイコンが追加されるので, それをクリックします.

そこで, passphrase を入力することで, 鍵を使用できるようになり, 右下で passphrase をキャッシュするか聞かれるので, 有効にすることができます.

![Screenshot 2024-03-31 at 20.21.50.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_20.21.50.png)

![Screenshot 2024-03-31 at 20.25.57.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_20.25.57.png)

これで, 無事 commit することができるようになりました.

![Screenshot 2024-03-31 at 20.43.03.png](/images/git-commit-gpg-signature-vscode/Screenshot_2024-03-31_at_20.43.03.png)

## **epilogue**

雑談です.

先日, 2 月ごろに, Mastodon や Misskey の ActivityPub プロトコルの Fediverse 界隈では, 大規模な DoS やメンションを悪用した荒らしなどの攻撃があり, 大きな問題になりました.

その中で, サーバインスタンスの連絡用メールアドレスを取得され悪用されるといった被害もあり, 多くの鯖缶が被害を受けたのですが, 私のサーバも被害を受けました.

この件, commit の署名もそうですが, `Verified` とついてなくても, その人のアイコンや名前が表示されていたら, その人がしたものと信じて疑わないことになりがちですが, 常に本当にそうなのかと立ち止まって考えるように, 情報セキュリティを成立させるには, 発信者だけでなく, 受け手側の意識もすごく大事だなと思ったりしました.
