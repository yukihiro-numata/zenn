---
title: "GetXの課題とriverpodへの移行part2"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "GetX", "riverpod"]
published: false
---

```mermaid
graph TD
    GQL --> Action1
    GQL --> Action2
    GQL --> Action3
    Action1 --> State1
    Action2 --> State2
    Action3 --> State3
    State1 --> ViewModel1
    State2 --> ViewModel2
    State3 --> ViewModel3
    ViewModel1 --> Controller1
    ViewModel2 --> Controller2
    ViewModel3 --> Controller3
    Controller1 --> Page1
    Controller2 --> Page2
    Controller3 --> Page3
```

# モジュール化、カプセル化の必要性を議論する

テスト時にモック化
テストを書くことが目的になってしまっていた。
flagment colocationによって問題が顕在化
各クラス層の責務を再定義

# 認知負荷、コード記述量の多さを減らすため、よりシンプルなクラス構成にする。
StateApiResponseの削除

# 依存関係の明確化
クラス内部でGet.find()するのをやめる。
