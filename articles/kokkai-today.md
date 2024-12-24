---
title: "アニメーションで国会の発言を可視化した個人開発がネットニュースになった話"
emoji: "⛲️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

[#国会 Today](https://kokkai.today) ([kokkai.today](https://kokkai.today)) の制作にこめた想いや技術的な解説を書いていきます.

[GDGoC Japan Advent Calendar 2024 > 6 日目](https://qiita.com/advent-calendar/2024/gdgoc-japan)

## #国会 Today とは

[#国会 Today](https://kokkai.today) は 1 日の国会の衆参の全発言を可視化する Web サービスです.

![]()

国会図書館の国会議事録 API, 衆議院インターネット審議中継, 参議院インターネット審議中継の 3 つのデータソースから取得した, 1 日の国会の全発言を形態素解析, キーワード抽出し, 言葉が湧き出る泉のようなアニメーションで可視化しています.

ブラックボックスになりがちな国会で今日何が話されたのか, 可視化により国会をより身近に感じてもらい, 国会の議論に参加しやすくすることを目指しています.

公開後たくさんの反響いただき, Twitter ([x.com/taniiicom/status/1853418731864739900](https://x.com/taniiicom/status/1853418731864739900)) で広く拡散していただいたり, ネットニュース ([internet.watch.impress.co.jp/docs/yajiuma/1637438.html](https://internet.watch.impress.co.jp/docs/yajiuma/1637438.html)) に取り上げていただいたりしました.

また, より様々な話題を, ついつい見てしまうような UI で可視化できるように, アニメーション UI の実装を含めた全てのソースコードを OSS として公開しています.

https://x.com/taniiicom/status/1853418731864739900

https://x.com/taniiicom/status/1853684697366413804

https://internet.watch.impress.co.jp/docs/yajiuma/1637438.html

## 技術について

ソースコードは以下のリポジトリで公開しています.

https://github.com/taniiicom/kokkai-today

大きく, `tasks` と `front` に分かれており, `tasks` はデータ取得, 解析, キーワード抽出, `front` はアニメーション UI を含むフロントエンド (と DB アクセスのための薄いバックエンドを含む) を担当しています.

### `tasks`: データ取得, 解析, キーワード抽出

議事録を取得し, 形態素解析, キーワード抽出を行う部分です.

処理の内容的に, python を使っています.

- 議事録の取得: 国会図書館の API を使って取得
- 形態素解析: `janome` を使って**並行処理**で形態素解析
- キーワード抽出: 形態素解析の結果から, 名詞を抽出のうえ, 後処理で一部**ドメインに特化したヒューリスティックなルール**を適用してキーワードを抽出

一般的な形態素解析によるキーワード抽出の流れを踏襲していますが, ちょっとした工夫として, **平行処理** と **ヒューリスティックなルール** を取り入れているので, その部分について詳細を書きます.

#### 形態素解析の並行処理

1 日の全ての発言を形態素解析する都合上, そこそこ時間がかかります.

なので, 形態素解析の部分は平行処理を入れています.

以下のように, `ThreadPoolExecutor` を使って, 複数のスレッドで形態素解析を行いつつ, A

```python
from concurrent.futures import ThreadPoolExecutor

def process_speeches(date):
    # スピーチデータを指定した日付から取得
    speeches = fetch_speeches(date, start_record=1, maximum_records=100)
    all_word_counts = Counter()  # 全スピーチの単語出現回数を格納するカウンタ

    print(f"Starting text parsing and word count aggregation for {len(speeches)} speeches...")

    # ThreadPoolExecutor を使用して並行処理を実行
    with ThreadPoolExecutor() as executor:
        # parse_text 関数を各スピーチのテキストデータに対して並行して適用
        results = executor.map(parse_text, [speech['speech'] for speech in speeches])

        # 並行処理の結果を順次取得し, 単語の出現回数を集計
        for idx, word_count in enumerate(results, start=1):
            all_word_counts.update(word_count)  # 各スピーチの単語出現回数を合計

            # 進捗を 10 件ごと, またはすべて処理した時点で表示
            if idx % 10 == 0 or idx == len(speeches):
                print(f"Processed {idx}/{len(speeches)} speeches.")

    # 集計した単語の出現回数をデータベースに保存
    save_to_postgres(date, all_word_counts)
```

#### キーワード抽出のヒューリスティックなルール

![](/images/kokkai-today/img1.png)

![](/images/kokkai-today/img2.png)

\*\* _筆者の 2024-11-25 の発表資料より_

上のように, 国会では「閉会中審査」「議院運営委員会」などの複合語が多く登場し, これらの単語を 1 つのキーワードとして抽出することが重要です.

例えば, 「公職選挙法」というキーワードであれば, 公職選挙法に関する議論が行われているということがわかります. しかし, このような複合語を細かく解析してしまうと, 「公職」「選挙」「法」の 3 単語に分かれてしまい, 議論の話題を特定するには一般的すぎる単語になってしまいます.

そこで, ヒューリスティックなルールとして,

- **連続する名詞を 1 単語とみなす後処理**

を導入しています.

簡易的ではあるものの強力な方法で, 日本語において, 「1 文節 1 自立語」であり, 複合語以外で名詞が連続することが少ないという特徴を活かしています.

この方法では, 少数ではあるものの, 一定数, 意図しない複合語が出力されるのですが, このような文の構造上たまたま生まれた単語は, ほとんどが 1 回しか生成されない単語であるので, 今回の可視化のような単語の出現回数を使って可視化する場合には, その影響が現れないことがほとんどです.

このように, データマイニングでは, ミクロ的にミスが生じうる手法であっても, マクロ的には大きな影響にならず意味のある結果を得られることも多いので, 局所的な精度にとらわれず, コストと全体の精度のバランスを考えて適切な手法を考えるのが大切なんじゃないかなと思っています.

![](/images/kokkai-today/img3.png)

\*\* _筆者の 2024-11-25 の発表資料より_

```python
def parse_text(text):
    tokenizer = Tokenizer()
    words = []
    temp_word = ""

    # テキストから人名部分と不要な行を除去
    text = remove_speaker_name_and_skip_lines(text)

    for token in tokenizer.tokenize(text):
        # 名詞であれば一時的に保存し、次の名詞に連結
        if token.part_of_speech.startswith("名詞"):
            temp_word += token.surface
        else:
            # 名詞の連続が終わった場合、保存してリセット
            if temp_word:
                words.append(temp_word)
                temp_word = ""

    # 最後の名詞の連続を処理
    if temp_word:
        words.append(temp_word)

    # 出現回数をカウントして返す
    return Counter(words)
```

### `front`: アニメーション UI を含むフロントエンド (と DB アクセスのための薄いバックエンドを含む)

可視化において, 一番重要なのは, どのようにデータを表現するかです.

今回のアプリにおいて目標としたのは, **『ぼーっとつい眺めたくなる』**.

なので, フロントエンドの実装についても, ここでは, アニメーション UI の実装について詳しく書きます.

## 制作の想い

国会では, 1 日に, 衆議院・参議院, 本会議・委員会で様々な議論がされています.

しかし, メディアがニュースとして取り上げるのは, その中のほんの一握りの話題のみです.

しかも, 一部の大きな政治的議題を除けば, 多くの法案は本会議の採決の段階になってようやくニュースに取り上げられるのです.

これでは, 議題について, 世論として意思表示をするのが遅れてしまいます. 採決よりも前の段階で, 議題を見て, 国民が議論に参加することができれば, より民主的な政治が実現できるのではないかと考えました.

国会での話題を知る方法には, ニュースに頼らずとも, 一次情報として国会議事録が国会図書館によって提供されています.

これを

このアプリは, もともと東京都立大学のみやこ祭

## ワードクラウドならぬ言葉の泉『ワードスプリングス』で国会の発言を可視化した話
