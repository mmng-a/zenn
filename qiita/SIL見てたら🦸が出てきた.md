# SILを見てたら🦸が出てきた

## SIL

Swiftをコンパイルする時、まずコンパイラはソースをSILと呼ばれる中間言語に翻訳する。これは最適化に適している言語であり、Swiftに限らず多くの言語はこうした中間言語を持っている。SILは最適化を受けた後、LLVMIRと呼ばれる形になり、LLVMというローレベルコンパイラにてバイナリにされる。LLVMは言語に依存しないコンパイラであり、SwiftがSwiftとしての最適化が行えるのがSILという訳だ。Swift固有の機能がコンパイラ内部でどう処理されるかを確認したい場合はSILを確認すると良い。

## Opaque Return Type

Swift 5.1で追加されたopaque return typeという機能がある。opaque typeはジェネリクスの一種で、ジェネリクスの中でも単純なものをより簡単に書けるようにしたものである。

ジェネリクスは汎用型と訳されるが、こと関数の返り値におけるジェネリクスは汎用型というより実装を抽象化するためのものである。

例えばある型が`getHash`という関数を持っていた時、大抵の場合`getHash`の返す型が`Int`であるか`String`であるかはたまた`Array<Float>.Iterator.Element`であるかは必要な情報ではない。`getHash`がある型を返し、それがハッシュ値として機能することが分かれば良いのだ。

ここで出てくるのがジェネリクスである。ジェネリクスを用いると「protocol Hashableを実装しているなんらかの型」を返す関数を作成できる。例えばこのように。

```swift
func getHash() -> <T: Hashable> T { // Intを返す }
```

この例において`getHash`は`Int`を返すが、`getHash`を呼び出す側は返ってくる値が`Int`だとはわからない。`Hashable`に適合したなんらかの値が返ってくることだけがわかる。

```swift
let x = getHash()  // xの型はIntではない。あくまでsome Hashableである。
```

現在のSwiftではこのような書き方（返り値をジェネリクスにすることをリバースジェネリクスという）は許されていないが、opaque return typeを使うことによって同じ効果を得ることができる（今回は問題ないが、リバースジェネリクスの下位互換であることに注意）。opaque return typeはSwiftUIで効果的に使えるため（SwiftUIでViewを記述する時、bodyプロパティの型は非常に細かく膨大になものになってしまう。詳細な型情報は必要ないためジェネリクスの出番というわけだ。）、Appleがリバースジェネリクスの一歩先にopaque return typeを導入したのだ。opaque return typeはこう記述する。

```swift
func getHash() -> some Hashable { ... }
```

これは「とあるHashableな値を返す」という意味であり、someというキーワードを使用してそれを表す。

## SILでのOpaque Return Type

ところでopaque return typeの本当の型はコンパイラだけが知っている。これがSwiftの中間言語であるSILでどのように表現されているか気になるだろう。swiftcコマンドにはSILを見るオプションが備わっている。`swiftc ファイル.swift -emit-sil`だ。先程の`getHash`をSILにしてみよう。中身を書かないわけにはいかないので、常に`1`を返すようにした。一体なんのハッシュだとか、このハッシュ衝突しかしないやん、というツッコミは気にしないでおこう。

<details><summary>SIL（まあまあ長いので畳んでみた。）</summary>

（わかりやすいようにSwiftのシンタックスハイライトを適用している。）

```swift
swiftsil_stage canonical

import Builtin
import Swift
import SwiftShims

func getHash() -> some Hashable


// main
sil @main : $@convention(c) (Int32, UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>) -> Int32 {
bb0(%0 : $Int32, %1 : $UnsafeMutablePointer<Optional<UnsafeMutablePointer<Int8>>>):
  %2 = integer_literal $Builtin.Int32, 0          // user: %3
  %3 = struct $Int32 (%2 : $Builtin.Int32)        // user: %4
  return %3 : $Int32                              // id: %4
} // end sil function 'main'

// getHash()
sil hidden @$s4main7getHashQryF : $@convention(thin) () -> @out @_opaqueReturnTypeOf("$s4main7getHashQryF", 0) 🦸 {
// %0                                             // user: %3
bb0(%0 : $*Int):
  %1 = integer_literal $Builtin.Int64, 1          // user: %2
  %2 = struct $Int (%1 : $Builtin.Int64)          // user: %3
  store %2 to %0 : $*Int                          // id: %3
  %4 = tuple ()                                   // user: %5
  return %4 : $()                                 // id: %5
} // end sil function '$s4main7getHashQryF'

// Int.init(_builtinIntegerLiteral:)
sil public_external [transparent] [serialized] @$sSi22_builtinIntegerLiteralSiBI_tcfC : $@convention(method) (Builtin.IntLiteral, @thin Int.Type) -> Int {
// %0                                             // user: %2
bb0(%0 : $Builtin.IntLiteral, %1 : $@thin Int.Type):
  %2 = builtin "s_to_s_checked_trunc_IntLiteral_Int64"(%0 : $Builtin.IntLiteral) : $(Builtin.Int64, Builtin.Int1) // user: %3
  %3 = tuple_extract %2 : $(Builtin.Int64, Builtin.Int1), 0 // user: %4
  %4 = struct $Int (%3 : $Builtin.Int64)          // user: %5
  return %4 : $Int                                // id: %5
} // end sil function '$sSi22_builtinIntegerLiteralSiBI_tcfC'
```

</details>

SILの説明まですると日が暮れてしまうし、何より僕もそんなに知らないので簡単になってしまうが、中程に`// getHash()`という行があり、ここから10行程度先ほどの関数がSILで書かれている。（ちなみに上にある`main`はこのプログラムのエントリーポイントを表しており、今回は0、つまり成功を返している。下の`Int.init(_builtinIntegerLiteral:)`は`1`というリテラルを`Int`型に変換している。）

`$s4main7getHashQryF`というのは`getHash`をマングリングしたもの（一意な表現にした関数名）で、その型が`:`の後に書いてある。この関数の型は`@convention(thin) () -> @out @_opaqueReturnTypeOf("s4main6opaqueQryF", 0) 🦸`だ。今回注目すべきは`@_opaqueReturnTypeOf`というアノテーション。この中に関数名と`0`が記述されている。

### `@_opaqueReturnTypeOf("s4main6opaqueQryF", 0)`

opaque return typeは各関数では一つの型だが、同じ`some Hashable`を返す関数でもそれが違う関数であれば同じ型とは言えない。例えば、

```swift
func getValue() -> some Equatable { Bool.random() }
let x = getValue()
let y = getValue()
print(x == y)
```

は可能だが、

```swift
func getValue() -> some Equatable { Bool.random() }
func getAnotherValue() -> some Equatable { Int.random(1...10) }
let x = getValue()
let y = getAnotherValue()
print(x == y)
```

はコンパイルできない。この例だと当たり前に見えるかもしれないが、

```swift
func getValue() -> some Equatable { Bool.random() }
func getAnotherValue() -> some Equatable { Bool.random() }
let x = getValue()
let y = getAnotherValue()
print(x == y)
```

も出来ない。これはopaque return typeの型は関数ごとに決まっており、なんらかの型で表すことは不可能であることを意味する。よってSILでの表現も`@_opaqueReturnTypeOf("s4main6opaqueQryF", 0)`と関数名を含んだ表現になっているのだろう。

（`0`はTupleの添字を表しているのではないかと思っているが、正確なところはわからない。`func f() -> (some Equatable, Int)`を試してみたが、コンパイラに怒られた。どうも今のところTupleの中でopaque return typeは使えないようだ。将来的にサポートされるのかも不明である。）

### 🦸 

しかし🦸とはなんなのだろうか。Unicode名はSUPERHEROらしい。opaque return type以外のところで見たことはない。そもそも型情報の一部なのかも疑問である。`:`より後、`{`より前にあるのでそう判断したが、何もかもがわからない。Swiftレポジトリをきちんと読むしかないのだろうか。