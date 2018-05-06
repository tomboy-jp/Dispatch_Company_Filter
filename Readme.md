# 派遣会社フィルター
機械学習を使ってSESやエンジニア派遣を行っている企業の求人広告をフィルタリングします。

## 使い方
このリポジトリを「git clone」して、type様のIT系求人広告から適当に募集情報ページのURLを用意します。  
ターミナル上でリポジトリのディレクトリまで移動して、  
```
python flg_checker.py 'https://type.jp/job-1/xxxxx_detail/?companyMessage=false&pathway=4'
```
上記を参考にコマンドを打ち込んでください。  
URLをクオーテーションで囲うのを忘れずに。  

実行すると、
```
会社名: XXXX
年収: 240〜360万円
年収捕捉: 月給20万円〜30万円
派遣フラグ: 1
```
といった感じに結果が返却されます。  
派遣フラグが1ならばそういう会社さんで、0なら自社開発を行っている会社さんということになります。  
だいたい80%くらいの確率で当たります。  

###### 動かなかった場合
動かなかった場合は必要なモジュールが入っていないか、そもそもpython3が入ってないケースが考えられます。  
モジュールが足りない場合は各pyファイルのimport先を見て必要なものをpipなりcondaなりで落として頂ければ。  
MeCabはデプロイが少々複雑なので、もし入っていなければ[ここ](https://github.com/tomboy-jp/MeCab_dep)を参考にしてください。  
Python3が入っていない場合は適当にググって環境構築するところから始めましょう。


## 作り方
・Webクローリング & スクレイピング  
・データクレンジング  
・自然言語処理 & MLモデルの構築  

の三本立てにおまけが一本ついてきます。  

#### ■Webクローリング & スクレイピング
対象pythonファイル: crawling_type.py  

まずはデータ収集です。  
今回はいつもお世話になっている[type](https://type.jp/)様の求人広告を利用させて頂きました。  

1.インデックスページからIT系のURLを取得(get_index)。  
2.詳細ページから必要な情報を抜き取る(get_detail)。  

の二段構えとなっています。  
HTMLコードの取得(クローリング)にrequestsを  
情報の抜き取り(スクレイピング)にre, BeautifulSoup & lxmlを使用しました。  

#### ■データクレンジング
対象pythonファイル: cleansing.py  

pandasを使って不必要な列を削ぎ落としていきます。  
並行して取得したデータのdispatch_flg欄に0か1かのデータ入力をしていきます。  
教師なしでは精度が出ないのため、辛いところではありますが私自身が教師になってデータを補完していきます。  
合計200件のデータを用意しました。  
決して多くはありませんが、簡単な二項分類ですし、充分かと。
非常に地味な作業のため、技術的に特筆すべきところはありません。   

#### ■自然言語処理 & MLモデルの構築  
対象pythonファイル: exe_ml.py

作業の最後にして最も華のあるフェイズです。  

1.出来上がったデータのドキュメント部分をMeCabで分かち書きにしてコーパスを作成(owakati)  
2.tf-idfでベクトル化(nlp)。  
3.グリッドサーチでパラメータを探り、最適化されたモデルを構築(ml_exe)。

###### ・分かち書き
「私はVtuberになりたい」を「私 は Vtuber に なり たい」  
のように品詞ごとにバラすことです。  
MeCabという形態素解析器を使って簡単に行えます。

###### ・tf-idf
tfは「terms frequency」の略語でどれだけの頻度で単語が出てきたかを評価します。  
idfは「invese document frecuency」の略語でどれだけの文章で登場したかを逆評価します。  
この二つのロジックを組み合わせたのがtf-idfです。  
つまり特定の文章で沢山出てきた単語には強い重みが与えられ、登場の少ない単語やあまりに一般的な単語(今回の場合は「年収」「転職」など)は軽視されます。  
sklearnが提供するTfidfVectorizerはこれらに加えて、  
ワンホットエンコーディング(単語や文節ごとを一つの特徴として扱うこと)を自動で行ってくれたり、  
何品詞までを一つの特徴として扱うかの設定が出来たり(今回は1~7品詞までを対象に設定しています)、  
ストップワード(無視する単語)の設定ができたりします。  
至れり尽くせりです。  

###### ・モデルの選定
MLモデル界のいぶし銀こと、ロジスティック回帰を使いました。  
最初は今流行のXGBoostやDeepLearningでイケイケドンドンしようかとも思ったのですが、  
GXBoostは今回のようにコーパスが少ないときは汎化させにくい(過学習を起こしやすい)ため、  
DeepLearningは中身がブラックボックスであるがゆえ、今回のもう一つの目的を果たせないために不採用としました(後記します)。  
パラメータは基本グリッドサーチに一任していますが、L1正規化(重要度の低い特徴をドロップする手法)だけは使わないようにしました(後記します)。  

最終的なスコアとパラメータは以下の通りとなります。  
```
Best Score: 0.787
Test Score: 0.800
Best Parameters:
{'logisticregression__C': 10000,
 'logisticregression__fit_intercept': True,
 'logisticregression__penalty': 'l2',
 'logisticregression__random_state': 0}
```
もっとデータがあって、Deepを使えば0.9は目指せると思います。  
後者を行うのはそう難しくありませんが、前者は辛い & 辛いので多分やらないと思います。

#### ■おまけ
対象pythonファイル: weight_of_words_ranking.py  

Deepを使わなかったのもL1正規化を行わなかったのもこれが見たかったからです。  
みなまで解説しませんが、パラメータが高ければ高いほどアレで低ければ低いほど癒やされる感じのやつです。
```
プロジェクト,9.728777673242941
案件,8.468665248615574
株,6.554591508811359
ます,5.66746407703486
未経験,4.64236526828179
し,4.3805474237348
も,4.192236074872746
エンジニア,3.915318662632371
を 除く,3.8452164455698314
希望,3.7733057417368716
除く,3.7590268425872173
川崎市,3.362387953950279
横浜市 川崎市,3.322560055197111
印刷,3.3054020599293485
て,3.258506356501901
キャリア,3.2382838116126593
な,3.2326256978442385
スキル,3.172424928474536
い ます,3.1074869406107846
い,3.080712593216811
研修,3.075725370761292
金融,3.066323422432494
の プロジェクト,3.045066335992295
常駐,3.0363878038269014
.
.
.
自由,-2.464278812983278
向い て,-2.5121636115585546
向い,-2.5121636115585546
新規,-2.5493468309881893
技術,-2.5877556932077077
自社 開発,-2.605995747991827
提案,-2.691577808820943
マイコン,-2.6971726249915533
自社 サービス,-2.7001705184854936
コンサルタント,-2.7338758403781003
ゲート,-2.786025215462243
国内,-2.8021170590521374
システム の,-2.808839645710968
販売,-2.8742278279417013
受賞,-2.9556238874992316
生産管理,-2.9708138535862583
社会,-3.0718079852900297
自社 で,-3.122537653864612
クラウド,-3.49355245453089
100 自社,-3.731157220486234
開発,-3.793528931120211
に,-4.3090121514171935
自社,-5.315964404630647
google,-5.410429083230298
製品,-5.818014094938781
```
より詳しいデータが気になる方は「coefficients/coefficients.csv」をどうぞ。  

以上です。  
ありがとうございました。
