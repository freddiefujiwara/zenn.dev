---
title: "話題沸騰ポットに対して Property based testing を試してみる(5)未知の問題を探す"
emoji: "😽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["テスト","property-based-testing","話題沸騰ポット","fast-check","model-based-testing"]
published: true
---

# はじめに
[話題沸騰ポットに対して Property based testing を試してみる (4)沸騰/保温 - 状態遷移に合わせて実装を変更する](https://qiita.com/freddiefujiwara/items/bc9ae3dd8970cd595c14)に引き続き
[fast-check](https://github.com/dubzzz/fast-check)を使ってProperty based testingを試しています。
題材として
[テスト設計コンテストU-30クラス](http://aster.or.jp/business/contest/rulebooku30.html)でテストベースに指定されている
[「話題沸騰ポット要求仕様書 (GOMA‐1015型) 第７版](http://www.sessame.jp/workinggroup/WorkingGroup2/POT_Specification.htm)にすることにしました.
fast-checkは状態をランダムウォークさせて失敗するテストケースを見つける方法([Model based testing](https://www.guru99.com/model-based-testing-tutorial.html))が[実装されてる](https://github.com/dubzzz/fast-check/tree/master/example/004-stateMachine/musicPlayer)みたいですのでいよいよ今日はそれを試してみたいと思います。

# Model based testingとは
ここでいうModel based testingは下記の流れです。
1.モデルを作りにはテスト対象が振る舞うであろう動作とその状態を定義します。
2.テスト対象の動作とモデルによって予測された結果を比較して確認します。

# fast-checkのModel based testing
[Model Based Testing Tutorial: What is, Tools & Example](https://www.guru99.com/model-based-testing-tutorial.html)にはモデルには色々あると記載されていますが

- データフロー
- 制御フロー
- 依存関係グラフ
- 意思決定テーブル
- 状態遷移マシン

fast-checkではModel based testingは、そのうち
状態遷移のモデルをつくりテストをすることを目指しています。
#状態遷移図
[話題沸騰ポットに対して Property based testing を試してみる (3)沸騰/保温 - 状態遷移を考える](https://qiita.com/freddiefujiwara/items/076cbe57cab936b54fc3)で考えた状態遷移図をみていきましょう
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1817/373dba8d-4b24-707e-478e-a3a71ab61716.png)

複雑なようですが、要は状態として
下記6状態

```JavaScript
OFF_CLOSE = -1,
OFF_OPEN, //0
ON_IDLE, //1
ON_OPEN, //2
ON_ACTIVE_BOIL, //3
ON_ACTIVE_KEEP, //4
```

アクションとして

- open() 蓋を開ける
- close() 蓋を締める
- fill() 水を入れる
- dispense() 給湯する
- reboil() 再沸騰させる
- boil to keep 沸騰するまでまつ

を考えたいと思います

#モデルを作成
モデルを作成します、fast-checkではアクションはCommandという形で定義するので、
[Goma1015](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/master/src/lib/index.ts)でpublicとして見える, state,waterとtemperatureを定義し、
constructerでGoma1015と同様に初期化します。

```JavaScript
import fc from 'fast-check'

import { Goma1015, State } from '../src/lib/index'

/** Class Goma1015Model */
export class Goma1015Model {
  /** to manage state transition */
  public state: number
  /** to manage water volume */
  public water: number
  /** to manage temperature */
  public temperature: number
  constructor() {
    this.state = State.OFF_CLOSE
    this.water = 0
    this.temperature = 25
  }
}

export type Goma1015Command = fc.Command<Goma1015Model, Goma1015>
```

# アクションを作成
基本的には未知の問題を見つけたいので各動作は全状態で実行可能にしておきます
各modelの状態と実際のインスタンスを比較していきます。

## open() 蓋を開ける 動作条件:全状態で起動
確認ポイントとしては下記です

- OFF_CLOSE/OFF_OPEN でopen()した場合 OFF_OPENに遷移するか
- それ以外の状態でopen()した場合 ON_OPENに遷移するか
- 中の水の量は変化しないか
- 蓋を開けると水の温度は25度になるのでそうなっているか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/OpenCommand.ts)
## close() 蓋を締める 動作条件:全状態で起動
確認ポイントとしては下記です

- OFF_OPEN でclose()した場合 OFF_CLOSEに遷移するか
- ON_OPEN状態でclose()した場合 ON_IDLEかON_ACTIVE_BOILに遷移するか
 - 水の量が10ml以上の場合 ON_ACTIVE_BOILに遷移するか
    - 1秒後温度は25以上になっていること
 - 水の量が10ml以下の場合 ON_IDLEに遷移するか
- 中の水の量は変化しないか
- 蓋を開けると水の温度は25度になるのでそうなっているか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/CloseCommand.ts)
## fill() 水を入れる 動作条件:全状態で起動
確認ポイントとしては下記です

- OFF_OPEN状態 もしくは ON_OPEN状態以外の状態でErrorが発動するか
- 水を溢れさせるとでErrorが発動するか
- 上記以外でOFF_OPEN状態 もしくは ON_OPEN状態で水を入れた場合ちゃんとポットに水が溜まるか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/FillCommand.ts)
## dispense() 給湯ボタンを押す 動作条件:全状態で起動
確認ポイントとしては下記です

- ON_IDLE状態 もしくは ON_ACTIVE_KEEP状態以外の状態でErrorが発動するか
- ON_ACTIVE_KEEPで給湯した結果水の量が10ml以下になったらON_IDLEに遷移するか
- 上記以外で ON_IDLE状態 もしくは ON_ACTIVE_KEEP状態以外の状態で給湯ボタンを押下した場合ちゃんとポットからお湯、水が指定量でるか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/DispenseCommand.ts)
## reboil() 再沸騰ボタンを押す 動作条件:全状態で起動
確認ポイントとしては下記です

- ON_ACTIVE_KEEP状態以外の状態でErrorが発動するか
- ON_ACTIVE_KEEP状態でreboil()再沸騰ボタンを押下するとON_ACTIVE_BOILに遷移するか
- 中の水の量は変化しないか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/ReboilCommand.ts)
## boil to keep 沸騰するまで待つ 動作条件:沸騰中状態のみ起動(ユーザのinputではないため)
確認ポイントとしては下記です

- 1秒後温度は25以上になっていること
- 十分な時間が立ったら水の温度が100度になりON_ACTIVE_KEEPに遷移
- 中の水の量は変化しないか

[ソース](https://raw.githubusercontent.com/freddiefujiwara/goma-1015/feature/model-based/model_based/BoilToKeepCommand.ts)

# 起動
実際の状態遷移はこんな感じで試行されてるみたいです。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1817/e14bacd8-11dd-124a-6b9d-dd39876a4345.png)

めっちゃいろんな状態が試されてますね

１万回状態遷移をさせましたが　すべてパスしました！！
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/1817/4668dd3f-2f0b-7398-9725-07f381a9d6ff.png)


# 最後に
実際の話題沸騰ポットにはタイマーや保温機能も複数あったりもっともっと状態は複雑ですが、
今回はできるだけボトムアップで一歩一歩作りました、
今後はUIをつけたり上記の機能を追加したりしたいと思いますが
目的のfast-checkでmodel-based testingを試せたので満足です。
一旦シリーズとしては終わりにしたいと思います
また、コードにもし間違いありましたら[Pull request](https://github.com/freddiefujiwara/goma-1015)は大歓迎です。
全5連載　最後まで読んでいただきありがとうございました。
