---
title: "NetworkImageのerrorBuilderがエラーを補足してくれないことがある"
emoji: "🧑‍💻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter"]
published: true
---

ネットワークから画像を取得して表示する際に `Image.network` を利用すると思います。
`Image.network` には `errorBuilder` という引数が生えており、設定することで画像の取得に失敗した場合の処理を記述することができます。

```dart
import 'package:flutter/material.dart';

void main() => runApp(const MyApp());

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    var title = 'Web Images';

    return MaterialApp(
      title: title,
      home: Scaffold(
        appBar: AppBar(
          title: Text(title),
        ),
        body: Image.network(
          'https://picsum.photos/250?image=9',
          errorBuilder: (context, object, stackTrace) {
            // TODO: alt画像を返したりする
          }
        ),
      ),
    );
  }
}
```

そのため、上記のように画像の通信取得に失敗したら代わりの画像を描画するといった実装をすることができるのですが、この `errorBuilder` でキャッチされないエラーがあり、意図した動作にならない場合があります。
具体的に下記のようなエラーが発生します。
`flutter_error_exception	OperationException(linkException: ServerException(originalException: Connection closed before full header was received, XXX…))`

エラー文を見てもなぜキャッチできないのか読み解けないのですが、FlutterのGitHub上でIssueが上がっていました。
https://github.com/flutter/flutter/issues/107416

ワークアラウンドな対策法として画像の通信取得部分を自作する方法があります。

```dart
import 'dart:typed_data';

import 'package:fansta_app/config/settings.dart';
import 'package:fansta_app/pages/components/loading_indicator.dart';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:http/http.dart' as http;

class CustomNetworkImage extends StatelessWidget {
  final Uri imageUri;
  final double width;
  final double height;
  final BoxFit fit;

  const CustomNetworkImage({
    super.key,
    required this.imageUri,
    required this.width,
    required this.height,
    this.fit = BoxFit.cover,
  });

  @override
  Widget build(BuildContext context) {
    return FutureBuilder(
      future: _fetchImage(),
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return SizedBox(
            width: width,
            height: height,
          );
        }

        return snapshot.hasError || snapshot.data == null
            ? Image.asset(
                'assets/images/no_image.png',
                width: width,
                height: height,
                fit: fit,
              )
            : Image.memory(
                snapshot.data!,
                width: width,
                height: height,
                fit: fit,
              );
      },
    );
  }

  Future<Uint8List?> _fetchImage() async {
    final response =
        await http.get(imageUri).timeout(const Duration(seconds: 10));
    if (response.statusCode == 200) {
      return response.bodyBytes;
    } else {
      debugPrint('Failed to load image: ${response.statusCode}');
      // 画像を取得できなかった場合はnullを返してキャッチできるようにする
      return null;
    }
  }
}
```

ただ、このような方法を取る場合だとキャッシュをかませたい場合にキャッシュ機構を自作する必要があったり手間が多いので微妙に思います。。
ちなみにこの問題はCachedNetworkImageを使う場合でも発生します。
https://github.com/Baseflow/flutter_cached_network_image/issues/443

余談ですが、この調査を進めていく中で無駄に通信処理を走らせている部分や無駄にウィジェットのリビルドをかけている箇所を発見できました。
適切にキャッシュをかませたり、Keyを指定して余計なUI破棄を減らしたりすることもエラー回避ないしパフォーマンス向上につなげることができそうです。
