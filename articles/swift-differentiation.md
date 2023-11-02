---
title: "Swiftで自動微分"
emoji: "➗"
type: "tech
topics: ["swift"]
published: false
---

:::message
この記事は[Life is Tech ! Members Advent Calendar 2020](https://qiita.com/advent-calendar/2020/life-is-tech-members) 11日目の記事です。あれ今日何日だっけ？
:::

## Introduction

何か小洒落た導入でも書こうかと思ったのですが、僕にそんな能力はなかったので普通に始めます。今回はSwiftの自動微分のお話です。当方特に微積分に精通しているというわけではないので悪しからず。

### Swift for TensorFlow

[Swift for TensorFlow](https://www.tensorflow.org/swift/)は2018年3月にTensorFlowチームが発表したプロジェクトです。TensorFlow for Swiftではなく、Swift for TensorFlowです。SwiftコンパイラにTensorFlowのための機能を追加することで、機械学習をする際の体験を高めることを目的としています。

Swift for TensorFlowの利点として、

- 安全
- 速い
- 新しく整備されたAPI
- Python等との連携
- 第一級の自動微分

が挙げられています（適当にかいつまんで来たので本当にそう言われているか怪しい）。

上2つは普段Swiftを使用している方なら説明する必要はないでしょう。APIは表現力豊かなSwift版としてさらに使いやすいものになるでしょう。Pythonとの連携には[PythonKit](https://github.com/pvieito/PythonKit)などが存在し、自然な形でSwiftからPythonのAPIを呼び出すことができます。

:::message
今回はTensorFlowに関しては扱いません。これについて詳しく知りたい人は[公式チュートリアル](https://www.tensorflow.org/swift/tutorials/model_training_walkthrough)などを参照してください。
:::

### 自動微分

さて、気になるのは第一級の自動微分です。これがSwiftコンパイラを拡張する必要がある理由なわけですが、その実装は概ね完了しており、Swift Forumsに[Differentiable programming for gradient-based machine learning](https://forums.swift.org/t/differentiable-programming-for-gradient-based-machine-learning/42147)というPitch（Proposalの前段階）も上がっています。いつ正式に導入されるかわかりませんが、そう遠くない未来には使えるようになっているかもしれません。

この提案では

- `@differentiable(reverse)`という関数宣言用のアトリビュート
- `@differentiable(reverse)`という関数の型（e.g., `@escaping`）
- `Differentiation`モジュール
	- `Differentiable`プロトコル
	- `gradient(of:)`などの導関数を計算するための演算子（Swiftの上では関数ですが数学では演算子Δなので提案では微分演算子として説明されています。Swiftの演算子と被ってしまうのでややこしい）

を追加し、Swiftを*自動微分機能を持つ最初の汎用プログラミング言語*にします。自動微分が導入されることで、

- 型安全な機械学習
- 微積は楽しい（は？）
- アニメーションのイージング関数
- ゲーム
- シミュレーション
- ロボット工学、機械工学
- レンダリングとレイトレーシング

をSwiftで行うのが簡単になります。~~2番目はさておき~~これらが実現したとき、Swiftの活用場所がさらに広がりそうでワクワクします。ピッチには他にも自動微分のアプローチが長々と説明されていますが、私には理解できそうにないので興味のある方は原文を確認してください。

`@differentiable`ではなく`@differentiable(reverse)`が追加される理由ですが、自動微分（というか微分一般）には准方向の微分と逆方向の微分が存在し、計算結果はどちらも同じですが、計算量が異なり、場合によって最適な計算方法が異なります（[Reverse-mode automatic differentiation: a tutorial](https://rufflewind.com/2016-12-30/reverse-mode-automatic-differentiation)が参考になりそう）。また、`@differentiable(linear)`なども存在するらしく、後の拡張性のために逆方向の微分は`@differenstiable(reverse)`として導入されます。[^reverse]

[^reverse]: 複雑さの低減のため初期は逆方向の微分のみサポートされるようですが、`reverse`だけを導入することは逆効果なのではという指摘もあったのでここら辺もまだ変わりそうです。Forumを見ている限りAPIの名前は他にも変更されそうな予感がします。

## 試す

### インストール

さきほど実装は完了していると述べましたが、Xcodeに入ってるSwiftなどでは試すことができません。[`main`ブランチ](https://github.com/apple/swift)で`import _Differentiation`と書くことで使用できます。もしくは[TensorFlow/swift](https://github.com/tensorflow/swift)から[ツールチェインをインストール](https://github.com/tensorflow/swift/blob/master/Installation.md)して同様に`import _Differentiation`することでも可能ですが、`TensorFlow`ライブラリもついてくる代わりに、REPLが使えません。今回は自動微分が使えればいいですし、Swift Playgroundで確認して見たかったので[SwiftのDevelopment Snapshot](https://swift.org/download/#snapshots)をダウンロードして使います。

:::message
と思ってたのですが、最新のSnapshotに入れ替えて直して動かしたら`dyld: Library not loaded: @rpath/Python3.framework/Versions/3.8/Python3`と言われて色々してたのですが、結局解決できなかったので、今回は元から入ってたTensorFlowの方をコマンドラインから使ってます。
:::

### 簡単な微分

```swift
import _Differentiation

let grad = gradient(at: 3.0) { x in
  x.squareRoot()
}
print(grad)
```

とりあえず適当に簡単そうな微分を試して見ます。`0.2886751345948129`と出力されました。正しいですね。さて、どうやって計算しているのでしょうか？

## プロポーザルの詳細

### `Differentiable`プロトコル

今回微分している型は`Double`ですが、標準ライブラリの型では`FloatingPoint`に準拠している型全て、それに加え`SIMD`、`Optional`、`Array`は`Differentiable`プロトコルに適合しています。その`Differentiable`プロトコルの宣言は以下のようになっています。

```swift
public protocol Differentiable {
    associatedtype TangentVector: Differentiable & AdditiveArithmetic
        where TangentVector == TangentVector.TangentVector
        
    mutating func move(by direction: TangentVector)
}
```

- `TangentVector`: プロポーザルにはリーマン多様体がなんだと書いてありましたが僕にはわかりませんでした。接線ベクトルです（それはただの翻訳）。微分をしたことがある方なら誰でもわかると思いますが、定数を部分した際`0`が出てきます。そのため、`AdditiveArithmetic`が関連型の要件となっています。（加法も必要みたいなことが書いてあった。）

- `move(by:)`: 接戦ベクトルに沿って指定された値だけ値を変化させます。`TangentVector`が`Self`の時はデフォルト実装が提供されます。うん、`TangentVector == Self`だと想像しやすいですね。

```swift
public extension Differentiable where Self == TangentVector {
    mutating func move(along direction: TangentVector) {
        self += direction
    }
}
```

また、機械学習などでは多くの場面で`Differentiable`に適合した型を書くことになると思いますが、`Equatable`、`Hashable`、`Codable`おなじく`Differentiable`の実装を書く必要はありません。コンパイラが自動で生成します。つまり、

```swift
struct Perceptron: Differentiable {
  var x: Float
  var y: Float
  @noDerivative var bias: Float
}
```

これだけで`Differentiable`に適合した型を作成できます。（詳細はピッチを見てください。ここで`@noDerivative`についても説明されています。）

### `@differentiable(reverse)`

`@differentiable(reverse)`を使用することで任意の関数、`subscript`、computed propertyなどが微分可能な関数であることをコンパイラに伝えます。

```swift
@differentiable(reverse)
func silly(_ x: Float, _ n: Int) -> Float {
    sin(cos(x))
}
```

この例ではnは微分できないためxで微分されます。明示的に何で微分されるか記述するには`@differentiable(reverse, wrt: x)`と書きます。

しかし[squareRootの定義](https://github.com/apple/swift/blob/main/stdlib/public/core/FloatingPoint.swift#L765)には`@differentiable(reverse)`はついていません。そもそも微分する際の基本の関数たち（`*`とか`sin`とか）は自動で微分することはできないからです。ではそれらの導関数はどう定義されているのでしょうか。

### `@differentiable(reverse)`型

その前に新しい関数の型です。`@escaping`のようなやつですね。上記の方法で宣言した関数は全て`@differentiable(reverse)`つきの関数です。`gradient(of:)`などは`<T: Differentiable> @differentiable(reverse) (T) -> T`型（など）の引数をとりますが、`@differentiable(reverse)`属性をつけずに宣言した関数も、コンパイラが`@differentiable(reverse)`型に変換可能か判断するため、引数に渡すことができます。また、サブタイピングにより通常の関数へ変換できます。

### `@derivative(of:)`

Differentiationモジュールを見ると[_vjpSquareRoot](https://github.com/apple/swift/blob/main/stdlib/public/Differentiation/FloatingPointDifferentiation.swift.gyb#L279)という関数があり、`@derivative(of: squareRoot)`という属性がついています。

```swift
extension FloatingPoint where Self: Differentiable, TangentVector == Self {
  @derivative(of: squareRoot)
  func _vjpSquareRoot() -> (value: Self, pullback: (Self) -> Self) {
    let y = squareRoot()
    return (y, { v in v / (2 * y) })
  }
}
```

この`@derivative(of: f)`という属性でそれが関数`f`の導関数であることを示すのです。

`@deriviative`をつけられた関数の返り値は`value`と`pullback`の2値からなるタプルを返す必要があります。`value`は元の関数の返り値です。多くの場合導関数を計算する過程でも関数の値は使用されるため、まとめることでパフォーマンスを向上さています（インライン化すれば最適化できますが、常にされるとは限りません）。`pullback`は導関数そのものです。`squareRoot`は$(x^\frac{1}{2})' = \frac{1}{2}x^{-\frac{1}{2}} = \frac{x}{2*\sqrt{x}}$ですので正しい導関数を返していますね（それはそう）。

また、一般に関数は複数の微分可能なパラメータを受け取ることができます。そのそれぞれに対応する接戦ベクトルを返す必要があるので、微分するパラメータの個数と`pullback`の返り値の個数は一致している必要があります。ある関数`f`に渡される引数の全てを微分に使用するわけではない場合`wrt`というオプションをつけることで微分するパラメータを指定できます。（多分普通にwith reference toの略だと思います。withとかでいいのでは……？）

```
func foo<T: Differentiable>(_ x: T, _ y: T, _ z: T) -> T { ... }

// 全てのパラメータの微分
@derivative(of: foo)
func derivativeOfFoo<T: Differentiable>(_ x: T, _ y: T, _ z: T) -> (
    value: T,
    pullback: (T.TangentVector) -> (T.TangentVector, T.TangentVector, T.TangentVector)
) {
    ...
}

// `x`と`z`の微分
@derivative(of: foo, wrt: (x, z))
func derivativeOfFoo<T: Differentiable>(_ x: T, _ y: T, _ z: T) -> (
    value: T,
    pullback: (T.TangentVector) -> (T.TangentVector, T.TangentVector)
) {
    ...
}
```

この`derivative`属性は元の関数に関係なく定義できるので、例えばGlibcの`expf`などにも同関数を定義することができます。便利ですね。試しに`Float.sin`の微分関数を用意してあげましょう。[^floatSin]

```swift
extension Float {
  @derivative(of: sin)
  public static func __derivativeOfSin(
    _ x: Float
  ) -> (value: Float, pullback: (Float) -> Float) {
    return (sin(x), { dx in cos(x) * dx })
  }
}
```

[^floatSin]: `sin`関数は`FloatingPoint`ではなく`ElementaryFunctions`にある関数です。んなもんあったんかと[SE-246](https://github.com/apple/swift-evolution/blob/master/proposals/0246-mathable.md)を見てみたら型チェックなどに問題があるらしくStandard Libraryには入ってないそうです。僕が使えたってことはTensorFlowのToolchainには入ってるっぽいですね。[Swift Numerics](https://github.com/apple/swift-numerics)には`Real`protocolの中に似た関数群が入っているので実用にはそちらがお勧めです。

### `gradient(at:in:)`

`gradient(of:)`という関数がいわゆるΔであり渡された関数の導関数を返し、`gradient(at:in:)`はその導関数のxでの値を返します。他にも元の結果と微分した結果を同時に取得できる`valueWithGradient(at:in:)`などが用意されています。

さて、ここまでがプロポーザルの内容です。（数学的な内容とか一切触れてませんが。）が、このままではただの翻訳なので、もうちょっと調べてみましょう。……ってなわけでSILを除いてたんですが、いろいろあって適当に自動微分を使った何かを作ることにしました。

## イージング関数を作る

