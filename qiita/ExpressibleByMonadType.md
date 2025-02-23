
# [妄想] ExpressibleByMonadType ~SwiftのOptional~

Evolにあった[Make Result expressible by literals when its Success type conforms to expressible by literal protocols](https://forums.swift.org/t/make-result-expressible-by-literals-when-its-success-type-conforms-to-expressible-by-literal-protocols/36794)を見ていて思いついたOptionalのサブタイプをなくして代入可能を一般化しようぜって話。というか妄想。

しばしばOptionalのあり方は議論の対象となる。untaggedなUnionやtagged Unionが一般的だが、Swiftはいいとこ取りをしようとした結果、tagged Unionにサブタイプを導入した。しかしこれはコンパイラマジックを使わざるを得ないし、いくつかの問題も引き起こす。

これが採用される日が来るとは思えないし、そもそもEvolに投稿しないが、案としては悪くはないのではないか。

## 概要

```swift
struct A<T>: ExpressibleByMonadType {
	var value: T
	init(wrappedValue value: T) {
		self.value = value
	}
}
```

というのがあったとして、

```swift
let a: A<Int> = 1
```
と宣言できるようになる。


## 引き続き可能な構文

### 代入

```swift
let a: Int?  = 1
let b: Int?? = 1
let c: Int?  = someInt()
```

### 反変

```swift
let d: Int? = 1
print(d == 1)
```

### `[T?]`のリテラルによる宣言

ちなみに`[]`で書いてるのはArrayに限らないから。

```swift
let e: [Int?] = [1, 2]
```

## 不能となる構文

### `[T?]`への`[T]`の代入

```swift
let f: [Int] = [1, 2, 3]
let g: [Int?] = f
```

ただし、このような場合はあまりないと思う。

```swift
func h(_: [Int?]) { ... }
```

というAPIを作る意味はまずなく、`[Int]`を引数にすれば良い。`[T?]`を必要とする場合もあるが（eg：リバーシ）、そういった場合は常に`[T?]`で取り回していることが多く問題にならない気がする。何分検索しづらいのでどの程度使われてるかわからなかった。

### 共変

欲しくはある。が、あまり使われてるのを見ないような？（要検証。検索できない。実装して試した方が早いまである。）　`NonEmptyArray`などは不可能となる。

```swift
class A {
	func f() -> Int? { ... }
}
class B: A {
	func f() -> Int { ... }
}
```

### 関数の代入

あんま使わんから忘れてた。なくなっても問題ないやろ。

```swift
let f: () -> Int = { 1 }
let g: () -> Int? = f
```

```swift
let f: (Int?) -> Void = { _ in }
let g: (Int) -> Void = f
```

## バグ修正

[Unexpected casting result in Array/Optional.](https://bugs.swift.org/plugins/servlet/mobile#issue/SR-6126)がそもそもできなくなる。

## その他

- `Optional`に関するコンパイラのマジックを減らせる。
- ABI的に大丈夫なのかは知らない。ソース互換性はアウト。
- 例えば`Result`を`Success`の値を書くだけで生成できたりする。（いいかは別として）。
- 表現可能以上のことができるので`ExpressibleBy`より良い名前はありそう。
- 悪用できそう。

## 実装

```swift
protocol Monad {
	associatedtype Wrapped
}

protocol ExpressibleByMonadType: Monad {
	init(wrappedValue: Wrapped)
}

extension Optional: ExpressibleByMonadType {
	typealias Wrapped = Wrapped
	init(wrappedValue value: Wrapped) {
		self = .some(value)
	}
}
```

とコンパイラ。言うは易し行うは難し。