---
title: "GetXの課題とriverpodへの移行part2"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "GetX", "riverpod"]
published: false
---

# 概要
この記事は下記記事の続きです。
https://zenn.dev/mixi/articles/transitioning-from-getx

前回の記事では、なぜGetXを乗り換えるに至ったかについて記載しました。
今回は、乗り換え先としてriverpodを選択した理由とGetXでの"失敗"を踏まえてどのような工夫を行なったかを説明します。

# GetXを使って構成していたアーキテクチャ
riverpodの選択理由は「現状からの乗り換えやすさ」が1番の理由です。
そのため、まずはGetXを使ってどのようなアーキテクチャを構成していたかについて説明します。

最初のアーキテクチャ構成は下記のような図になっていました。
```mermaid
graph LR
    Page --> Controller
    Controller --> ViewModel
    ViewModel --> State
    State --> Action
    Action --> API
```

Reducerがないですが、Reduxな思想を持ったアーキテクチャです。
ControllerがPage(UI)のロジックを担う責務を持ち、ViewModelがUIの状態を保持します。
Actionは必要に応じてControllerから実行され、API呼び出しを行なってStateにレスポンスを格納し、必要に応じてViewModelがStateを参照します。

## アーキテクチャの問題点について
プロジェクト立ち上げ当初はこのアーキテクチャを敷いていましたが、プロダクトが成熟するにつれ、問題がいくつか発生するようになりました。
このアーキテクチャで抱えていた問題は、GetXの辛さだけが理由ではありませんでした。
以下に詳細を説明します。

### 問題点①：コード記述量の多さによるコストパフォーマンスの悪さ
このアーキテクチャでは、どれほどシンプルなページであっても、必ずPage、Controller、ViewModel、State、Actionの5つのクラスファイルを用意
する必要があります。
コストパフォーマンスが悪いと感じるケースとして、例えばAPIを叩いてそのレスポンスを表示するだけのシンプルな画面を考えてみます。
この場合、以下のようなコードを書く必要があります。

```dart
import 'package:flutter/material.dart';
import 'package:get/get.dart';

class SamplePage extends GetView<SampleController> {
  const SamplePage({super.key});

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Obx(
        () => Text(
          controller.viewModel.response.toString(),
        ),
      ),
    );
  }
}

class SampleController extends GetxController {
  SampleViewModel get viewModel => Get.find<SampleViewModel>();

  @override
  void onInit() {
    super.onInit();
    _init();
  }

  Future<void> _init() async {
    await Get.find<SampleAction>().fetch();
  }
}

class SampleViewModel {
  SampleResponse get response => Get.find<SampleState>().response.value;
}

class SampleState {
  final Rxn<SampleResponse> response = Rxn();
}

class SampleAction {
  Future<void> fetch() async {
    // ここでAPIリクエストを行う
    final response = SampleResponse();
    Get.find<SampleState>().response.value = response;
  }
}

class SampleResponse {}
```

これだけでもコード記述量の多さ、要は面倒さが分かると思います。

さらに、我々のチームでは「ネイティブアプリ＝不具合が発生してもすぐに修正ができない」という点を踏まえて、テストカバレッジを上げてリスクヘッジを図る方針も採用していたため、上記のコードに加えてテストコードも実装する必要がありました。
我々のチームは小規模ですが、ネイティブアプリだけでなくWebアプリやサーバーサイドアプリの開発も行っているため、このコストパフォーマンスの悪さが大きなボトルネックとなっていました。（アプリの実装は時間がかかるから、、という会話が頻繁に聞かれるようにもなってしまっていました）
また、このアーキテクチャはネイティブアプリ独自の構成であったため、構成変更によるスイッチングコストも大きく、改善を迫られることになりました。

### 問題点②：コードの保守性と再現性に関する問題
ロジックのモジュール化やカプセル化が適切に行われていなかったため、コードのシンプルさや保守性が損なわれていた。

分かりやすい例として、下記のようなコードを考えます。

```dart
class SampleResponse {
  final String time;
  final bool notShowTime;

  const SampleResponse({
    required this.time,
    required this.notShowTime,
  });
}

class SampleHelper {
  DateTime _stringToDatetime(String time) {
    return DateTime.parse(time);
  }

  DateTime? formatTime(SampleResponse sample) {
    if (sample.notShowTime) {
      return null;
    }

    return _stringToDatetime(sample.time);
  }
}
```

時刻と、時刻を表示するかどうかを示すフラグを持つオブジェクトがある場合で、画面にどう表示するかをHelperクラスが担っています。
このオブジェクトを扱う画面が複数ある場合で、ロジックを共通化するために
このような実装になったものなのですが、本来であれば、こうした処理はモデルクラスを作成し、そのgetterメソッドとして実装すべきです。
しかし、わざわざHelperというクラスを作成し、依存関係を増やしてしまっていました。
また、依存関係が増えることで、GetXの扱いづらさとしてあげた依存管理の難しさも助長させてしまっていました。

このような問題は言語化が難しく、ベストな実装方法もケースバイケースであるため、コードレビューでしか防ぐことができず、レビュワーが気づかなければ問題視もされないという状況になってしまっていました。
さらに我々のプロジェクトでは、APIにGraphQLを採用しており、型の自動生成を行っていますが、Fragment Colocationを導入することで、ページごとに型が異なる状態になり、ロジックの共有化が現状のアーキテクチャではより難しくなるなど問題がさらに顕在化しました。

## 抱えていた問題に対する工夫について
行なったことは2点あります。どちらも単純な工夫ではありますが、それなりに効果のあるものでした。
1. State層の廃止
2. 無理な共通化をやめる

まず1についてですが、APIのレスポンスをカプセル化するメリットよりも、クラス層が増えてコード記述量が増えてしまうデメリットの方が大きいという結論に至りました。
多くの場合においてState層はAPIレスポンスを保持しているだけで、結局ViewModel層を介してアクセスするだけなので、ViewModel層がState層のラッパークラスのような扱いになってしまい、恩恵をあまり受けられなかったという理由があります。

2については、そもそもの問題がケースバイケースで解決法が異なるということもあり、やや定性よりな工夫です。
独自実装が必ずしも悪いとは限らない、既存の共通ロジックを無理に使おうとしなくてもよい、という意識をもつことをまずは持つようにしました。
これにより、無理に共通ロジックを適用しようとして発生していた複雑さを軽減し、シンプルで理解しやすいコードを維持することができるようになりました。

# riverpodへの乗り換えと新しいアーキテクチャについて
上述の旧アーキテクチャが抱えていた問題を解決しつつ、なるべく乗り換えのコストがかからないと判断できたのがriverpodでした。
riverpodはある程度実装のルールがriverpod側で定義されています。このルールに従うことでコードのシンプルさが一定保たれますし、属人的な知識も要らないため、実装やレビューに迷うことも少なくなります。
GetXはRxプログラミングであるためstreamを使うような実装は習熟コストが高いと考えたのも理由の1つですし、riverpodの人気の高さも理由の1つでした。

アーキテクチャについてはレイヤード型のアーキテクチャを採用し、下記のような構成をとりました。

```mermaid
graph TD;
    subgraph Presentation Layer
        A[Widget] --> B[Controller]
        B --> C[State]
    end

    subgraph Domain Layer
        D[Service] --> E[Model]
    end

    subgraph Data Layer
        F[Repository] --> G[Dto]
        G --> H[DataSource]
    end

    B --> F
    D --> F
    B --> D
```

