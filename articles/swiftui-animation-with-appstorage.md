---
title: "[SwiftUI] AppStorageの値の変更はデフォルトだとアニメーションされない"
emoji: "▶️"
type: "tech"
topics: ["swift", "swiftui"]
published: true
---

SwiftUIでの開発中に、さっきまでアニメーションしていたものがアニメーションしなくなって困ったのでメモ。

## 原因

`@SceneStorage`だったものを`@AppStorage`に変更したからだった。

`@AppStorage`はデフォルトではアニメーションしてくれないらしい。UserDefaultsに書き込む/読み込むものだしその制約はわかる。

## 解決方法

`.animation(_:value:)`を使えば問題ない

```sh
strucut ContentView: View {
	@AppStorage("isScaled") var isScaled: Bool = false
	var body: some View {
		VStack {
			Text("Hello, World!")
				.scale(scaled ? 2 : 1)
				.animation(.spring, value: isScaled)
			Toggle("toggle isScaled", $isScaled)
				.frame(width: 150)
		}
	}
}
			
```