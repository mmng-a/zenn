---
title: "[SwiftData] SwiftUIのPreviewが表示されない時は"
emoji: "🛢️"
type: "tech"
topics: ["swift", "swiftui", "swiftdata", "coredata"]
published: true
---

SwiftDataとSwiftUIを使って開発していた時、シミュレーターでは動くのにPreviewが表示されない状況に陥ったのでメモ。

## 原因

開発初期段階だったためModelの構造をちょくちょく変えていた。当然スキーマが変わればそれに合わせてマイグレーションしなければ以前のデータが残ったままでは起動できないのだが、開発初期段階であるためシミュレーターのデータ削除→再インストールで対応していた。

PreviewではModelContainerのイニシャライザなどにある`isStoredInMemoryOnly`乃至`inMemory`をtrueにすることで、データを保存しないようにできるのだが、これをオンにし忘れていた箇所があった。

## 解決方法

以下のコマンドでPreviewのデータを削除できる。

```sh
xcrun simctl --set previews delete all
```

## 対策

Previewでは必ず`isStoredInMemoryOnly`をオンにすること。

或いは、以下のようにグローバルな`ModelContainer`を作って使い回すべし。

```swift
let previewContainer: ModelContainer= ...
```


今回はこれもやっていたのだが、表示するデータを変えるため一箇所で`.modelContainer(for:, inMemory:)`を使っていた。が、`inMemory`を呼び忘れた。