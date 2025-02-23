# [SwiftUI] 親Viewの内容を子Viewから操作する

こういう画面下部にPickerなどがあるViewって結構ありますよね？　これをSwiftUIで作ること自体は簡単です。

<details><summary>毎回折りたたみ方忘れる……。</summary><div>

```swift
import SwiftUI

struct HalfModalView<Main: View, Second: View>: View {
    
    @Binding var isPresented: Bool
    
    var mainView: Main
    var secondView: () -> Second
    
    init(_ isPresented: Binding<Bool>, @ViewBuilder main: () -> Main, @ViewBuilder second: @escaping () -> Second) {
        _isPresented = isPresented
        mainView = main()
        secondView = second
    }
    
    var body: some View {
        VStack(spacing: 0) {
            mainView
            
            if isPresented {
                VStack {
                    closeButton
                    secondView()
                }
                .background(backgroundView)
                .transition(.move(edge: .bottom))
            }
        }
    }
    
    var backgroundView: some View {
        let color = Color(.secondarySystemGroupedBackground)
        return  RoundedRectangle(cornerRadius: 25)
            .foregroundColor(color)
            .edgesIgnoringSafeArea(.vertical)
            .shadow(radius: 10)
    }
    
    var closeButton: some View {
        HStack {
            Button(systemImage: "xmark.circle.fill") {
                withAnimation {
                    isPresented = false
                }
            }
            .font(.title)
            .foregroundColor(.gray)
            Spacer()
        }
        .padding(10)
    }
}

struct HalfModalView_Previews: PreviewProvider {
    
    struct ContentView: View {
        @State var isPresented = true
        @State var color = Color.green
        var body: some View {
            HalfModalView($isPresented) {
                ZStack {
                    RoundedRectangle(cornerRadius: 25.0)
                        .foregroundColor(color)
                        .padding()
                    Button(isPresented ? "Hide" : "Show") {
                        withAnimation { isPresented.toggle() }
                    }
                    .foregroundColor(.white)
                    .font(.largeTitle)
                }
            } second: {
                Picker("color", selection: $color) {
                    Text("Red").tag(Color.red)
                    Text("Green").tag(Color.green)
                    Text("Blue").tag(Color.blue)
                }
            }
        }
    }
    
    static var previews: some View {
        ContentView()
    }
}

extension Button where Label == Image {
    init(systemImage: String, action: @escaping () -> Void) {
        self = Button(action: action) {
            Image(systemName: systemImage)
        }
    }
}
```

</div></details>

面倒くさいのでPreviewごとコピペしてきました。一部iOS 14/macOS 11以降のAPIも使用してますが、適当に直せばiOS 13でも動くと思います。

関係ないですが、最近Appleのアプリには右上にXマークがあるViewも多いですね。HIG等で説明されていれば倣ってもいいかなと思うのですが、されていない以上断固として左上Xマークを使っていきたいです。この場合は右上Doneもセオリーですが、Modelessな操作（HlafModalなんて名前ですがこのViewはModelessです。インスペクタのようなものなので。）にDoneは似合わないと思っているので、Xマークを使用しています。
