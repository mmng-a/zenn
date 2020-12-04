---
title: "[Swift] コードカバレッジをXcodeを使わずに確認する"
emoji: "💯"
type: "tech" # tech: 技術記事 / idea: アイデア記事
topics: ["swift"]
published: true
---

:::message
この記事は[Life is Tech ! Members Advent Calendar 2020](https://qiita.com/advent-calendar/2020/life-is-tech-members)4日目の記事です。昨日はれんの[中3の頃のコードを高2目線でリファクタリングする](https://zenn.dev/renren/articles/5333457c6b6153)でした。
本当は他のことをしようと思っていたのですが、ちょっとエラーに悩まされてましてプランB（普通に投稿して良かったけど万一のために残してた記事を使う作戦）で行きます。幸い枠には空きがあるようなのでプランAもどこかで出します。
:::

普通に[「Swift コードカバレッジ」で検索する](https://duckduckgo.com/?q=Swift+コードカバレッジ)とXcodeでコードカバレッジを有効にする方法しか出てきませんが、もちろん`swift`コマンドでもコードカバレッジを計測できます。

## コードカバレッジを有効にしてテストする

テストしたいSwift Packageで`swift test --enable-code-coverage`と打つだけです。そういえばSwiftコマンドは`swift package completion-tool generate-YOURSHELL-script`で補完を生成できるのですが、あまり知られていない気がします。

しかしこのオプションをつけて実行しても標準出力はただの`swift test`となんら変わりません。一体生成物はどこにあるのでしょうか？

## 結果を確認する

まあ普通に「Swift test --enable-code-coverage」と検索すれば世の中には先人というものがぎょーさんおる訳です。日本語の記事はなさそうだったのでインターネットの民として情報を複製していきましょう。

### llvmを入れる（てもいい）

Linuxとかだと（多分）ないので入れる必要があります。Macでは（Xcodeを入れていると？）`xcrun llvm-cov`で`llvm-cov`を呼べるので次章以降適宜代用してください。

brewだとかaptを使って適当に入れて、

```
export PATH="/usr/local/opt/llvm/bin:$PATH"
```

をzshrcにでも書いてパスを通しましょう。

### `llvm-cov report`

このコマンドを実行するだけです。もちろん`myProjectPackageTests`は適当にテストターゲットの名前を入れてください。

```sh
llvm-cov report \
  .build/x86_64-apple-macosx/debug/myProjectPackageTests.xctest/Contents/MacOS/myProjectPackageTests \
  -instr-profile=.build/x86_64-apple-macosx/debug/codecov/default.profdata \
  -ignore-filename-regex=".build|Tests" \
  -use-color
```

色付きで結構見やすく表示されましたね。

### `llvm-cov show`

次にどこがテストされてるかを確認しましょう。

```sh
llvm-cov report \
  .build/x86_64-apple-macosx/debug/myProjectPackageTests.xctest/Contents/MacOS/myProjectPackageTests \
  -instr-profile=.build/x86_64-apple-macosx/debug/codecov/default.profdata \
  -use-color
```

`-instr-profile`オプションより後ろにソースコードを指定することで特定のファイルのみを見ることもできます。

## 適当にシェルスクリプトに

8割コピペですが。`find-root-path`というのは`Sources`等にいるときでも呼べるように`Package.swift`のあるパスを取得するコマンドです。

### swift-cov-report

```sh
#!/bin/env sh

PACKAGE_PATH=`find-root-path Package.swift`

if [[ -z $PACKAGE_PATH ]]; then
	echo "$(pwd) is not swift package"
	exit 1
fi

BIN_PATH=`swift build --show-bin-path`
XCTEST_PATH=`find ${BIN_PATH} -name '*.xctest'`

COV_BIN=$XCTEST_PATH
if [[ "$OSTYPE" == "darwin"* ]]; then
	f=`basename $XCTEST_PATH .xctest`
	COV_BIN=$COV_BIN/Contents/MacOS/$f
fi

cd $PACKAGE_PATH

llvm-cov report $COV_BIN \
	-instr-profile=.build/debug/codecov/default.profdata \
	-ignore-filename-regex=".build|Tests"
```

### swift-cov-show

ファイルを渡してあげることでそのファイルのみを表示します。

```sh
#!/bin/env sh

PACKAGE_PATH=`find-root-path Package.swift`

if [[ -z $PACKAGE_PATH ]]; then
	echo "$(pwd) is not swift package"
	exit 1
fi

BIN_PATH=`swift build --show-bin-path`
XCTEST_PATH=`find $BIN_PATH -name '*.xctest'`

COV_BIN=$XCTEST_PATH
if [[ "$OSTYPE" == "darwin"* ]]; then
	f=`basename $XCTEST_PATH .xctest`
	COV_BIN=$COV_BIN/Contents/MacOS/$f
fi

llvm-cov show $COV_BIN \ｈ
	-instr-profile=$PACKAGE_PATH/.build/debug/codecov/default.profdata \
	-use-color \
	$@
```

### find-root-path

```sh
#!/usr/bin/env sh

if [ $# != 1 -o $1 == "-h" -o $1 == "--help" ]; then
	echo "USAGE: find-root-path FILENAME"
	exit 1
fi

while [[ `pwd` != "/" ]]; do
	ls -A | grep -w $1 > /dev/null && pwd && exit 0
	cd ..
done

exit 1
```

`swift-cov-show`は`-use-color`を使っているのでパイプに渡しづらいことに注意してください。lessで見るときは`less -R`でANSIコードを解釈してくれます。`swift-cov-report`は`-use-color`は使っていませんが、標準出力とパイプとで出力を分けてくれます。

シェルスクリプトは初心者もいいところなので、特に`find-root-path`とかもっといい書き方があればご教示していただくと幸いです。

## 参考
<https://theswiftdev.com/code-coverage-for-swift-package-manager-based-apps/>
<http://www.koganemaru.co.jp/cgi-bin/mroff.cgi?sect=1&cmd=&lc=1&subdir=man&dir=jpman-11.3.2%2Fman&subdir=man&man=llvm-cov>
