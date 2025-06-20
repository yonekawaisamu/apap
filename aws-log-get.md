承知いたしました！**CloudWatch Logs API** を採用する方向で進めましょう。
ログの保存場所をCloudWatchに一元化できる、とてもスマートなアーキテクチャですね。

実装は少し複雑になりますが、一つ一つ手順を踏んでいけば必ず完成できますので、ご安心ください。一緒に頑張りましょう！

### 実装の全体像

これから行う作業は、大きく分けて以下の3ステップです。

1.  **準備**: LaravelからAWSのサービスを操作するためのSDK（Software Development Kit）を導入し、認証設定を行います。
2.  **ログ取得処理の作成**: CloudWatch Logsから操作ログを取得するためのロジックを、再利用しやすい「サービスクラス」として作成します。
3.  **画面表示**: 作成したサービスクラスをコントローラから呼び出し、取得したログを画面に表示します。

---

### ステップ1：AWS SDKのインストールと設定

まずは、LaravelプロジェクトでAWSの機能を使うための準備をします。

#### 1. AWS SDK for PHPのインストール

ターミナルで、Laravelプロジェクトのルートディレクトリに移動し、以下のComposerコマンドを実行します。

```bash
composer require aws/aws-sdk-php-laravel
```

これにより、LaravelでAWSの各サービスを簡単に使えるようになります。

#### 2. AWS認証情報の設定

Laravelアプリケーションが、あなたの代わりにAWSへAPIリクエストを送信するには、認証情報が必要です。これにはいくつかの方法がありますが、**IAMロールを使用する方法を強く推奨します**。

* **本番環境（推奨）**: EC2インスタンスやECSタスクに**IAMロール**を割り当てます。この方法が最も安全で、コードに認証情報を埋め込む必要がありません。アプリケーションは自動的に割り当てられたロールの権限を使用します。
* **開発環境（ローカル）**: `.env` ファイルにアクセスキーを設定します。

`.env`ファイルに以下のように追記してください。

```ini
# .env

AWS_ACCESS_KEY_ID=YOUR_AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY=YOUR_AWS_SECRET_ACCESS_KEY
AWS_DEFAULT_REGION=ap-northeast-1  # 例: 東京リージョン
AWS_SESSION_TOKEN= # IAMロールの一時的な認証情報を使う場合は設定
```

#### 3. 必要なIAM権限

アプリケーションに割り当てるIAMロール（またはIAMユーザー）には、CloudWatch Logsからログを検索・取得するための最低限の権限が必要です。以下の権限を許可するIAMポリシーを作成・アタッチしてください。

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:StartQuery",
                "logs:GetQueryResults"
            ],
            "Resource": "arn:aws:logs:REGION:ACCOUNT_ID:log-group:YOUR_LOG_GROUP_NAME:*"
        }
    ]
}
```
* `REGION`, `ACCOUNT_ID`, `YOUR_LOG_GROUP_NAME` はご自身の環境に合わせて書き換えてください。
* `YOUR_LOG_GROUP_NAME` は、Laravelアプリケーションのログが保存されているCloudWatchのロググループ名です。

---

### ステップ2：ログ取得サービスクラスの作成

CloudWatch Logsとの複雑なやり取りは、専用のクラスにまとめて管理しやすくしましょう。

#### 1. サービスクラスの作成

`app` フォルダの下に `Services` というフォルダを作成し、さらにその中に `CloudWatchLogService.php` というファイルを作成します。

```
app/
├── Http/
├── Models/
└── Services/  <-- このフォルダを作成
    └── CloudWatchLogService.php  <-- このファイルを作成
```

#### 2. サービスクラスの実装

作成した `CloudWatchLogService.php` に、以下のコードを記述します。

```php
<?php

namespace App\Services;

use Aws\CloudWatchLogs\CloudWatchLogsClient;
use Illuminate\Support\Facades\Auth;

class CloudWatchLogService
{
    protected CloudWatchLogsClient $client;
    protected string $logGroupName;

    public function __construct()
    {
        // AWS SDKのクライアントを初期化
        $this->client = app('aws')->createClient('cloudwatchlogs');
        
        // .envファイルからロググループ名を取得
        $this->logGroupName = env('CLOUDWATCH_LOG_GROUP');
    }

    /**
     * 現在ログイン中のユーザーの操作ログを取得する
     *
     * @param int $days 取得する日数（例: 7日前から）
     * @return array
     */
    public function getOperationLogsForCurrentUser(int $days = 7): array
    {
        $userName = Auth::user()->name; // 現在のユーザー名を取得

        // CloudWatch Logs Insightsのクエリ
        // ログイン中のユーザー名（user.name）でフィルタリングする
        $queryString = "
            fields @timestamp, @message
            | filter log_type = 'operation_log' and user.name = '{$userName}'
            | sort @timestamp desc
            | limit 200
        ";

        try {
            // クエリの開始
            $startQuery = $this->client->startQuery([
                'logGroupName' => $this->logGroupName,
                'startTime' => now()->subDays($days)->timestamp, // 検索開始時間
                'endTime' => now()->timestamp,                  // 検索終了時間
                'queryString' => $queryString,
            ]);

            $queryId = $startQuery->get('queryId');

            // クエリが完了するまで待機（最大30秒）
            $status = '';
            $waitTime = 0;
            while ($status !== 'Complete' && $waitTime < 30) {
                sleep(1); // 1秒待機
                $waitTime++;
                $results = $this->client->getQueryResults(['queryId' => $queryId]);
                $status = $results->get('status');
            }
            
            if ($status !== 'Complete') {
                // タイムアウトまたはエラー
                return ['error' => 'ログの取得に失敗しました。時間をおいて再度お試しください。'];
            }

            // 取得した結果を整形する
            return $this->formatResults($results->get('results'));

        } catch (\Throwable $e) {
            // AWS API呼び出しでエラーが発生した場合
            report($e); // エラーをログに記録
            return ['error' => 'ログの取得中にエラーが発生しました。'];
        }
    }

    /**
     * CloudWatchからの結果を画面表示用に整形する
     *
     * @param array $results
     * @return array
     */
    protected function formatResults(array $results): array
    {
        $formattedLogs = [];
        foreach ($results as $result) {
            $logData = [];
            foreach ($result as $field) {
                // @messageの中身はJSON文字列なのでデコードする
                if ($field['field'] === '@message') {
                    $logData['details'] = json_decode($field['value'], true);
                }
                // @timestampはそのまま利用
                if ($field['field'] === '@timestamp') {
                    $logData['timestamp'] = $field['value'];
                }
            }
            $formattedLogs[] = $logData;
        }
        return $formattedLogs;
    }
}
```

**設定の追加:**
`.env` ファイルに、ロググループ名を追加するのを忘れないでください。

```ini
# .env
CLOUDWATCH_LOG_GROUP=your-log-group-name-here
```

---

### ステップ3：コントローラとビューの実装

最後に、作成したサービスクラスを使って画面にログを表示します。

#### 1. コントローラとルートの作成

操作履歴を表示するためのコントローラと、そこへアクセスするためのルートを定義します。

```bash
# コントローラを作成
php artisan make:controller LogHistoryController
```

`routes/web.php` にルートを追加します。

```php
// routes/web.php
use App\Http\Controllers\LogHistoryController;

Route::get('/log-history', [LogHistoryController::class, 'index'])
    ->middleware('auth') // ログイン必須にする
    ->name('log.history');
```

#### 2. コントローラの実装

作成した `app/Http/Controllers/LogHistoryController.php` を編集し、サービスクラスを呼び出します。

```php
<?php

namespace App\Http\Controllers;

use App\Services\CloudWatchLogService; // 作成したサービスクラスをインポート

class LogHistoryController extends Controller
{
    public function index(CloudWatchLogService $logService) // 依存性の注入でインスタンスを受け取る
    {
        // サービスクラスのメソッドを呼び出してログを取得
        $logs = $logService->getOperationLogsForCurrentUser();

        // ビューにデータを渡して表示
        return view('log-history', ['logs' => $logs]);
    }
}
```

#### 3. ビューの作成

最後に、取得したログを表示するビューを作成します。
`resources/views/log-history.blade.php` というファイルを作成し、以下のように記述します。

```blade
{{-- resources/views/log-history.blade.php --}}

<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <title>操作履歴</title>
    <style>
        body { font-family: sans-serif; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .error { color: red; }
    </style>
</head>
<body>
    <h1>操作履歴</h1>

    @if(isset($logs['error']))
        <p class="error">{{ $logs['error'] }}</p>
    @elseif(empty($logs))
        <p>操作履歴はありません。</p>
    @else
        <table>
            <thead>
                <tr>
                    <th>日時</th>
                    <th>画面</th>
                    <th>操作内容</th>
                    <th>IPアドレス</th>
                </tr>
            </thead>
            <tbody>
                @foreach($logs as $log)
                    <tr>
                        <td>{{ \Carbon\Carbon::parse($log['timestamp'])->setTimezone('Asia/Tokyo')->format('Y-m-d H:i:s') }}</td>
                        <td>{{ $log['details']['action']['screen'] ?? 'N/A' }}</td>
                        <td>{{ $log['details']['action']['operation'] ?? 'N/A' }}</td>
                        <td>{{ $log['details']['request']['ip_address'] ?? 'N/A' }}</td>
                    </tr>
                @endforeach
            </tbody>
        </table>
    @endif
</body>
</html>
```

これで全ての実装が完了です！
ブラウザで `/log-history` にアクセスすると、ログイン中のユーザーの操作ログがCloudWatchから取得され、表示されるはずです。

### 注意点・今後の改善点

* **パフォーマンス**: `sleep(1)` でクエリ完了を待つ方法はシンプルですが、ユーザーを待たせてしまいます。より高度な実装として、JavaScriptを使って非同期でログを取得し、「読み込み中...」のような表示を出すと、ユーザー体験が向上します。
* **エラーハンドリング**: 現在は単純にエラーメッセージを表示するだけですが、より詳細なエラーハンドリングを実装すると、さらに堅牢なシステムになります。

まずはこの基本形を動かしてみて、わからない点やもっと良くしたい点があれば、いつでもご質問ください！
