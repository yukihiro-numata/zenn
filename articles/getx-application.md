---
title: "GetXの運用が辛くなったので乗り換え始めた話"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "GetX"]
published: false
---

# GetXについて

私が現在開発に携わっているアプリはFlutterで開発されており、その根幹部分に大きく依存しているパッケージがGetXです。GetXは、主に状態管理の選択肢の一つとして挙がるパッケージですが、それ以外にも多くの機能を提供しています。特に「状態管理、DI管理（依存性注入）、ルーティング管理」の3つが主要な機能です。

他にも `isBlank` や `isEmail` などの便利関数、ロケールを使った多言語対応、Key-Valueストアの管理なども提供されています。

GetXは、Flutter初心者にとって難しいBuildContextを上手く隠蔽し（GetXを使用すると基本的にcontextにアクセスする必要がなくなります）、簡単にDIを行えるため、知見が少ない状態でも恩恵を多く享受できるパッケージと言えます。実際、私のチームではFlutter経験者やアプリ開発経験者が全くいなかったため、その利点を考慮してGetXを状態管理に選択しました。

約3年前の選定当時は、現在ほどRiverpodが成熟しておらず、Providerが一般的でしたが、デファクトスタンダードが確立されていなかったという背景もあります（これは確実な情報ではなく、私の印象に基づくものです）。

そんなGetXを使って開発を進めてきましたが、機能拡充に伴い使いづらさを感じることが多くなり、やむなく他のパッケージへの乗り換えを決断しました。

# 乗り換えに至った理由や経緯

GetXの使用において扱いづらさを感じた点は以下の3つです：

1. DIの管理が難しい
2. グローバルなステートにおけるRxの扱いが難しい
3. GetX自体のメンテナンス頻度が低い

これらの理由について、それぞれ具体的に説明します。

## 1. DIの管理が難しい

GetXのDIは `Get.put()` を使って依存性注入を行い、 `Get.find()` で注入されたインスタンスを取得するというシンプルな仕組みです。アプリケーションがシンプルでDI対象が少ない場合は問題ありませんでしたが、アプリケーションが複雑化してグローバルなステートを管理する必要が増えるにつれ、問題が顕在化しました。

具体的には、依存注入の忘れによるバグ（ `Get.put()` の忘れで `Get.find()` が失敗する）が多発するようになりました。典型的な問題例として以下があります：

```dart
class SomeService {
  final OtherService otherService;

  SomeService(): otherService = Get.find<OtherService>();
}

void main() {
  // この一行を忘れると、 `Get.find()` が失敗する
  // Get.put(OtherService());

  final service = SomeService();  // ここでエラーになる
}
```

このようなDIのし忘れは、プロジェクトが大規模になるほど頻発し、デバッグが難しくなります。GetXでDIを管理する場合、手動で依存注入を行う必要があり依存関係が明確でないと容易に壊れやすい状態になってしまうため、綿密なアプリケーション設計が必要となります。

## 2. グローバルなステートにおけるRxの扱いが難しい

GetXはRxをベースとしています。変数の更新を容易に監視できる一方で、変数がどこで使用されているかを把握するのが難しく感じました。特にグローバルなステートにおいては、どこで使用されているかが不明瞭で、軽微な修正が関係のない箇所に影響を与えることがありました。以下の例が典型的です：

```dart
class GlobalStateController extends GetxController {
  var count = 0.obs;
}

// あるコンポーネント
class ComponentA extends StatelessWidget {
  final GlobalStateController controller = Get.find();

  @override
  Widget build(BuildContext context) {
    return Obx(() => Text('${controller.count}'));
  }
}

// 別のコンポーネント
class ComponentB extends StatelessWidget {
  final GlobalStateController controller = Get.find();

  @override
  Widget build(BuildContext context) {
    return Obx(() => Text('${controller.count}'));
  }
}

// グローバルステートの変更
void incrementCounter() {
  final controller = Get.find<GlobalStateController>();
  controller.count++;
}
```

このように、どこで状態が変更されるかわかりづらく、意図しない副作用が発生しやすい状況です。依存関係を明確にする厳密なアプリケーション設計が必要でしたが、うまく運用できていませんでした。

## 3. GetX自体のメンテナンス頻度が低い

GetXの最終リリース日は `2023年9月8日` です。機能のアップデートがないだけならまだしも、不具合修正も行われないため非常に問題となっています。GetXに起因する不具合がいくつかあり、数ヶ月経っても修正されていない状況です。例えば、以下の問題はタイミングによって `Get.find()` が失敗するという致命的な不具合です。

> Ref: [Issue](https://github.com/jonataslaw/getx/issues/3112)

こうした状況から、ライブラリ選定の重要性を痛感しました。

# 今後について

まずは、状態管理の実装をRiverpodに置き換える予定です。GetXの反省を活かし、導入前からアプリケーションの設計を行い、慎重に移行作業を進めています。状態管理以外にもGetXに依存している部分が多くあるため、少しずつ移行を進めていく予定です。Riverpodの選定理由や移行後のアプリケーション設計については、また後日文書化したいと思います。
