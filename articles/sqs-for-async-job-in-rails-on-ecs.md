---
title: "ECS上のRailsアプリケーションで非同期ジョブにAmazon SQSを採用した話"
emoji: "🚀"
type: "tech"
topics: ["rails", "aws", "ecs", "sqs", "eventbridge"]
published: true
---

# はじめに

ECS（Elastic Container Service）上で稼働するRailsアプリケーションにおいて、非同期ジョブの処理基盤としてAmazon SQSを採用しました。本記事では、なぜSQSを選択したのか、そしてどのようにシステムを構築したのかについて共有します。

# なぜSQSを採用したのか

## 検討した選択肢

非同期ジョブの処理基盤として、以下の3つの選択肢を検討しました。

### 1. Sidekiq
インフラコストと運用コストが増えるので不採用としました：
- Redisサーバーの運用が必要
- Redis自体の可用性確保（レプリケーション、バックアップ等）が必要

### 2. Solid Queue
Solid QueueはRails 8からデフォルトの非同期ジョブアダプタとなった、DBベースのジョブキューシステムです。
デフォルトであることから、導入は非常に簡単で、実際に試してみましたが、以下の懸念から不採用としました：
- アプリケーションサーバーへの負荷増加
- データベースインスタンスへの負荷増加
- ジョブ処理のためにデータベースへのポーリングが必要
- 大量のジョブ処理時のパフォーマンスへの懸念

### 3. Amazon SQS（採用）
最終的にAmazon SQSを採用した理由は以下の通りです：

#### マネージドサービスの利点
- サーバーの管理が不要
- 自動的なスケーリング
- インフラ運用コストの削減

#### 充実したリトライ制御機構
- デッドレターキュー（DLQ）のネイティブサポート
- リトライ回数の柔軟な設定
- 失敗したメッセージの自動隔離
- メッセージの可視性タイムアウト制御

#### AWSエコシステムとの親和性
- IAMによる細かなアクセス制御
- CloudWatchによる監視・アラート
- EventBridge Pipesとの連携による柔軟なジョブ実行

# SQSを使った非同期ジョブ実行の仕組み

## システム構成の概要

構築したシステムは以下のような流れで動作します：

1. Railsアプリケーションからジョブをエンキュー
2. SQSがメッセージを受信
3. EventBridge PipesがSQSのメッセージを検知
4. ECSタスクを起動してジョブを処理

## 実装の詳細

### 1. Railsアプリケーションからのエンキュー

RailsアプリケーションからSQSへのジョブエンキューには、`aws-activejob-sqs` gemを使用しました。

```ruby
# Gemfile
gem 'aws-activejob-sqs'
```

Active Jobの設定：

```ruby
# config/application.rb
config.active_job.queue_adapter = :sqs
```

ジョブクラスの実装例：

```ruby
class UserNotificationJob < ApplicationJob
  queue_as :default

  def perform(user_id)
    user = User.find(user_id)
    # 通知処理の実装
    NotificationService.new(user).send_welcome_email
  end
end
```

ジョブのエンキュー：

```ruby
UserNotificationJob.perform_later(user.id)
```

### 2. EventBridge Pipesを使用したECSタスクの起動

SQSにメッセージが送信されると、EventBridge Pipesがそれを検知し、ECSタスクを起動します。

#### EventBridge Pipesの設定

```yaml
# CloudFormationでの設定例
JobProcessorPipe:
  Type: AWS::Pipes::Pipe
  Properties:
    Name: rails-job-processor
    Source: !GetAtt JobQueue.Arn
    Target: !GetAtt ECSCluster.Arn
    TargetParameters:
      EcsTaskParameters:
        TaskDefinitionArn: !Ref JobProcessorTaskDefinition
        LaunchType: FARGATE
        NetworkConfiguration:
          AwsvpcConfiguration:
            Subnets: !Ref PrivateSubnets
            SecurityGroups: !Ref SecurityGroup
        Overrides:
          ContainerOverrides:
            - Name: job-processor
              Command:
                - "bundle"
                - "exec"
                - "rails"
                - "runner"
                - "JobProcessor.process_message(ENV['MESSAGE_BODY'])"
              Environment:
                - Name: MESSAGE_BODY
                  Value: $.body
```

#### ジョブ処理用のECSタスク

ジョブ処理専用のコンテナイメージを用意し、受信したメッセージを処理します：

```ruby
# lib/job_processor.rb
class JobProcessor
  def self.process_message(message_body)
    message = JSON.parse(message_body)
    job_class = message['job_class'].constantize
    job_args = message['arguments']

    # Active Jobのジョブを実行
    job_class.perform_now(*job_args)
  rescue => e
    Rails.logger.error "Job processing failed: #{e.message}"
    raise # リトライのために例外を再発生
  end
end
```

### 3. エラーハンドリングとリトライ

SQSのデッドレターキューを活用して、失敗したジョブを適切に処理します：

```yaml
# SQSキューの設定
JobQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: rails-job-queue
    VisibilityTimeout: 300  # 5分
    MessageRetentionPeriod: 1209600  # 14日
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt DeadLetterQueue.Arn
      maxReceiveCount: 3  # 3回失敗したらDLQへ

DeadLetterQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: rails-job-dlq
    MessageRetentionPeriod: 1209600  # 14日
```

## 監視とアラート

CloudWatchを使用して、ジョブの処理状況を監視します：

```yaml
# CloudWatchアラームの設定例
DLQAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: rails-job-dlq-messages
    AlarmDescription: Alert when messages are in DLQ
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Statistic: Maximum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    TreatMissingData: notBreaching
    Dimensions:
      - Name: QueueName
        Value: rails-job-dlq
```

# まとめ

ECS上のRailsアプリケーションにおいて、非同期ジョブ処理基盤としてAmazon SQSを採用することで、以下のメリットが期待できます：

- インフラ運用コストの削減（Redisサーバーが不要）
- 高い可用性とスケーラビリティ
- 充実したリトライ機構とエラーハンドリング
- AWSエコシステムとのシームレスな連携

特に、EventBridge Pipesを使用したECSタスクの起動により、サーバーレスなジョブ実行環境を実現できます。

一方で、SQSのポーリング間隔によるレイテンシや、分散システムゆえのデバッグの複雑さには注意が必要です。

同様のアーキテクチャを検討されている方の参考になれば幸いです。
