# 英語プロンプトのほうが良いって、ほんとに？
ChatGPTに対して「日本語で指示するより英語で指示したほうが性能が良い」という話をよく見かけます。しかし、その事例の多くは一般的な知識に関しての質問応答で、日本語の（典型的な？）NLPタスクを解かせるという文脈ではありません（すでに検証している方がいたり、論文なんかがすでに出ていたらごめんなさい）。パーソナルアシスタントというよりかは、業務の（半）自動化ツールで将来的に使うことを想定すると、日本語のNLPタスクではどうか？についてはきちんと検証しておきたいところです。

私の仮説は、「日本語の言語的な知識」を強く要求する場合なら、日本語で指示したほうが良い、というものです。日本語固有の事情を加味しないといけない場合、下手に別言語の空間を参照させようとすると、性能が下がるのでは？

そういう動機があって、今回は読み推定タスクを例に、日本語指示と英語指示での性能差を、gpt-3.5-turbo でちょっと試しに確かめてみました。結論から言うと、試した範囲（たった150文ですが）では英語指示のほうが性能が良かったです[^1]。マジかー。


# 検証
読み推定をGPT-3.5-turboに行わせます。
入出力はこんな感じです：
```
input:
犬が兎を追う。
output:
いぬがうさぎをおう。
```

コーパスには [KWDLC](https://nlp.ist.i.kyoto-u.ac.jp/?KWDLC) を利用させていただきました。
検証で使ったコードは[ここ](https://github.com/0x9D28/gpt_use_cases_for_nlp)に置いておきます。実験の出力全体については、ごめんなさい、データセットのライセンスがよく分からないので、共有は控えます（非営利・研究調査用途ならよいのでしょうけど、いちおう）。もし気になる人がいれば拙いコードですが手元で試してください。


## プロンプト
出力はひらがなと約物のみと指示ししています。また、数字を数字のまま出す場合が多そうだったので、数字もひらがなにしろと言い含めます。
日本語指示：
```
入力文の読み方をひらがな（あいうえお など）および約物（、。！？「」" など）のみで出力してください。

### 条件
- 数字の読み方もひらがなで書いてください。
- 使っていい文字は、ひらがなと約物だけです。
```

英語指示：
```
Output the reading of the input sentence using only hiragana (such as あいうえお) and punctuation marks (such as 、。！？「」" and so on).

### Conditions
- Write the reading of numbers in hiragana as well.
- Only use hiragana and punctuation marks for allowed characters.
```

zero-shot の場合と few-shot の場合で検証します。
few-shot 設定では、次の形式で参考事例を与えます。
```
### Examples:
input:
<surface text>
output:
<yomi text>
```

## データセット
コーパスには KWDLC を利用します。選定基準は「読みアノテーションが付与されている」「それなりに多様なジャンルが含まれている」「無料で入手しやすい」です。読み推定に利用するのは目的外[^2]という感じはしますが、使えるのでありがたく使わせていただきます。

適切さはいったんおいて、とりあえずファイル（id）を文字列昇順に並び替えて最初の4つを few-shot 用、続く50ファイルを検証用のデータとしました。1ファイル3文あるから、参考例は12文、検証対象は150文です。

### 実験結果
改行や半角空白を除去したうえで、各出力と正解文字列との編集距離を、ドキュメント単位で測りました。評価対象ドキュメントは50個です。

#### zero-shot の場合
日本語プロンプトの勝ち： 6
英語プロンプトの勝ち　：23
引き分け　　　　　　　：21

正解と完全一致した数：
日本語プロンプト：13
英語プロンプト　：18

#### few-shot の場合
日本語プロンプトの勝ち： 4
英語プロンプトの勝ち　：13
引き分け　　　　　　　：33

正解と完全一致した数：
日本語プロンプト：17
英語プロンプト　：18

### 定量分析
zero-shot の場合、大きく英語に差をつけられて日本語プロンプトが負けています。引き分けにさえなっていない場合が多いです。
few-shot でも、日本語プロンプトの性能はけっこう上がっています（正解と完全一致の場合が増えています）が、英語に勝てていない状態です。

### 定性分析
日本語プロンプトの場合で特に、カタカナをひらがなに変換できていない例が多々見られました。これは、日本語に依存する部分が大きいタスクであっても、英語のほうが細かい制御が効きやすい（場合がある）ことを示唆します。ただし、同様の現象は、英語でも見られました。ただ、やはり傾向としては英語のほうが細かい制御がしやすい感はあります。


### 事例分析
いくつかの固有名詞の読み誤りが、zero-shot & 日本語指示の場合だけで起きていました。面白いので抜粋して紹介します[^2]。


```
入力:
「朝鮮日報記事情報」は、韓国で最大の発行部数を誇る日刊紙「朝鮮日報」の日本語訳記事データベースです。

日本語プロンプト:
「あさかんにっぽうきじじょうほう」は、かんこくでさいだいのはっこうぶすうをほこるにっかんし「あさかんにっぽう」のにほんごやくききデータベースです。
英語プロンプト:
「ちょうせんにっぽうきじじょうほう」は、かんこくでさいだいのはっこうぶすうをほこるにっかんし「ちょうせんにっぽう」のにほんごやくききデータベースです。
```
「朝鮮」を「あかさん」と読んでしまっています。あとついでに言うと、「データベース」がカタカナのままになっていて、指示に違反しています。few-shot & 英語プロンプトの場合でだけ、ここの「データベース」を指示通りに「でーたべーす」とひらがなで書くことができていました。

```
入力:
龍馬の姉・千鶴の夫で龍馬の義兄にあたります。

日本語プロンプト：
りゅうまのあね・ちづるのおっとでりゅうまのぎきょうだいにあたります。
英語プロンプト：
りょうまのあね、ちづるのおっとでりょうまのぎけいにあたります。
```
「龍馬」を「りゅうま」と読んでしまっています。また、「義兄」を「ぎきょうだい」と思い込んでしまっています。

固有名詞については日本語由来の知識が特に役に立つだろうから、日本語指示のほうが良いことを期待していましたが、けっこう変な読み誤りをしてしまっています。もしかすると、日本語のデータだといちいち読み仮名なんて明記されていない、というバイアスがあるのかもしれません。

あと面白かったのがこちら。これも zero-shot 日本語プロンプトの事例です。
```
入力：
猫と同じ空間でくつろげ、猫に癒されることを目的とする。

日本語プロンプト：
にゃんことおなじくうかんでくつろげ、にゃんこにいやされることをもくてきとする。
英語プロンプト：
ねことおなじくうかんでくつろげ、ねこにいやされることをもくてきとする。
```
先の固有表現の誤り事例の話ともつながりますが、あえて日本語で「猫」に読み仮名をつけるとしたら、「にゃんこ」である、と学習していたのかもしれません。

上で挙げた誤りはいずれも few-shot 設定では正解できています。事例を与えないとこのように変な誤り方をするため、明確に正解があるようなタスクにおいて zero-shot で何かさせるのは危険でしょう。


# まとめ
今回、読み推定タスクにおいて、GPT-3.5-turbo の性能を、日本語プロンプト・英語プロンプトで比較しました。その結果、少なくとも今回の設定・範囲においては、細かい条件設定に従順だった英語プロンプトのほうが良い結果を残せました。加えて zero-shot の場合、日本語プロンプトでは日本語コーパス由来と見られる奇妙なバイアスが入りやすいことも分かりました。


[^1]: もちろんタスクによるでしょうし、対象テキストのドメインにもよるでしょう。今回試したのは少し硬めのスタイルの150文だけですので、まあ話半分で読んでもらえると。また、2023-03-26現在、評価についても検証が足りていない部分が多いです。追記できるときに追記します。
[^2]:  ライセンス的な問題があるかもしれません。マズければ当該箇所は削除します。