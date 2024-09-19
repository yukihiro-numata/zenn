---
title: "Widgetを適切に使い分けて読みやすいコードにする"
emoji: "🧑‍💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "リファクタリング", "リーダブルコード"]
published: true
---

単純なUIであればよいのですが、複雑なUIになってネストが深くなるとかなり読みづらくなってしまうのがFlutterを実装していて辛いなと感じる部分です。
Widgetを適切に使い分けることでネストを浅くすることができその辛さを解消できます。

具体的に以下のようなコードがあるとします。
```dart
import 'package:flutter/material.dart';

class Sample extends StatelessWidget {
  const Sample({super.key});

  @override
  Widget build(BuildContext context) {
    return Container(
      margin: const EdgeInsets.only(top: 20),
      width: double.infinity,
      child: DecoratedBox(
        decoration: BoxDecoration(
          border: Border(
            top: BorderSide(
              color: Colors.white.withOpacity(0.3),
              width: 0.5,
            ),
          ),
        ),
        child: Container(
          margin: const EdgeInsets.only(bottom: 20),
          child: Column(
            children: [
              const Text(
                'タイトル',
                style: TextStyle(
                  fontSize: 20,
                  fontWeight: FontWeight.w400,
                ),
              ),
              Container(
                margin: const EdgeInsets.only(top: 8),
                child: Row(
                  children: [
                    Image.asset('assets/images/hoge.png'),
                    Container(
                      margin: const EdgeInsets.only(left: 8),
                      child: const Text(
                        'テキスト',
                        style: TextStyle(
                          fontSize: 14,
                          fontWeight: FontWeight.w400,
                        ),
                      ),
                    ),
                  ],
                ),
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

個人的にはこれでもだいぶしんどいです。
これはmarginの取り方やボーダーの付け方を工夫することで以下のように実装することができます。
```dart
import 'package:flutter/material.dart';

class Sample extends StatelessWidget {
  const Sample({super.key});

  @override
  Widget build(BuildContext context) {
    return SizedBox(
      width: double.infinity,
      child: Column(
        children: [
          const SizedBox(height: 20),
          Divider(
            color: Colors.white.withOpacity(0.3),
            thickness: 0.5,
          ),
          const Text(
            'タイトル',
            style: TextStyle(
              fontSize: 20,
              fontWeight: FontWeight.w400,
            ),
          ),
          const SizedBox(height: 8),
          Row(
            children: [
              Image.asset('assets/images/hoge.png'),
              const SizedBox(width: 8),
              const Text(
                'テキスト',
                style: TextStyle(
                  fontSize: 14,
                  fontWeight: FontWeight.w400,
                ),
              ),
            ],
          ),
          const SizedBox(height: 20),
        ],
      ),
    );
  }
}
```

修正したのは以下の箇所です
- `Container(margin: XXX)` → `SizedBow(height: XXX)` or `SizedBow(width: XXX)`
- `DecoratedBox(decoration: BoxDecoration(border: YYY))` → `Divider(YYY)`

この修正によってネストがかなり浅くなり、またマージンやボーダーがどこに配置されるのか分かりやすくなりました。
ケースバイケースで必ずしも適用できるわけではないですが、適用できないか立ち止まって再考するといいと思います。
