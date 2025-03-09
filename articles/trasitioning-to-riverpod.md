---
title: "GetXの課題とriverpodへの移行part2"
emoji: "🚀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "GetX", "riverpod"]
published: false
---

# 概要

```mermaid
graph LR
    Page --> Controller
    Controller --> ViewModel
    ViewModel --> State
    State --> Action
    Action --> API
```

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

# モジュール化、カプセル化の必要性を議論する

テスト時にモック化
テストを書くことが目的になってしまっていた。
flagment colocationによって問題が顕在化
各クラス層の責務を再定義

# 認知負荷、コード記述量の多さを減らすため、よりシンプルなクラス構成にする。
StateApiResponseの削除

# 依存関係の明確化
クラス内部でGet.find()するのをやめる。
