### アプリ名
HANDOFF

### 1. システム構成
AIと人間が同じタスクリストを参照・更新するシステム。以下の3コンポーネントで構成する。

- **REST APIサーバー**: タスクのCRUD操作を提供するAPIを GCP Cloud Run 上にデプロイする（Node.js / コンテナ）。サーバーはステートレスに保ち、データ永続化はすべて Firestore に委ねる。
- **Firestoreデータストア**: Cloud Firestore（ネイティブモード）で2つのコレクションを管理する。
  - `board`: 処理中タスク。1タスク＝1ドキュメント。
  - `archive`: 完了タスク。1タスク＝1ドキュメント。
  - ドキュメントIDは Firestore 自動採番（内部参照のみに使用）。activityログは各ドキュメント内の配列フィールドとして保持する。
- **ディスパッチャー**: OSのスケジューラー（macOS: launchd、Linux: cron）で定時実行する処理スクリプト。運用者のローカル機で動作し、REST API経由でタスクを読み書きする（Firestoreへ直接アクセスはしない）。条件に合致するタスクを1件ずつ処理する。

### 2. 認証

2層で保護する。

- **API認証**: すべてのAPIエンドポイントはリクエストヘッダー `X-Board-Token: {TOKEN}` を必須とする。トークン不一致の場合は 403 Forbidden を返す。トークンは Cloud Run の環境変数（または Secret Manager）で管理し、コードにハードコードしない。AIエージェントの種類ごとに別トークンを発行し、操作主体を activity ログで追跡できるようにする。
- **Firestoreアクセス**: Cloud Run から Firestore へはサービスアカウント（Application Default Credentials）経由でアクセスする。Firestore Security Rules はクライアントからの直接アクセスを全面拒否（deny-all）に設定し、サーバー（Admin SDK）経由のアクセスのみを許可する。

### 3. APIエンドポイント仕様

| メソッド | パス | 処理内容 |
| --- | --- | --- |
| GET | /api/board | `board` コレクション全件取得 |
| GET | /api/board/{id} | `board/{id}` 取得（存在しない場合は 404） |
| POST | /api/board | タスク新規作成（IDは Firestore 自動採番。生成IDをレスポンスで返す） |
| PATCH | /api/board/{id} | タスク部分更新（トランザクションで updated_at と activity ログを自動追記） |
| POST | /api/board/{id}/complete | トランザクションで `archive/{id}` を作成し `board/{id}` を削除する（同一IDを維持。activity 配列も一緒に移動する）。対象が存在しない場合は 404（二重呼び出しに対して冪等に扱う） |
| GET | /api/archive | `archive` コレクション全件取得 |

### 4. タスクスキーマ

```json
{
  "id": "Firestore自動採番のドキュメントID（内部参照用。連番ではない）",
  "title": "タスクの名称（自由記述）",
  "status": "needs-ai | needs-human | in-progress | done | blocked",
  "owner": "human | ai-batch | ai-interactive",
  "priority": "P0 | P1 | P2 | P3",
  "action_type": "content | research | review | publish | setup | other",
  "handoff_note": "前の担当者が何をやり、次の担当者は何をやるべきか、なぜか",
  "blocked_reason": "停滞理由（statusがblockedの場合のみ。それ以外はnull）",
  "tags": ["任意のタグ文字列"],
  "created_at": "2026-01-01T07:30:00Z",
  "updated_at": "2026-01-02T09:00:00Z",
  "activity": [
    {
      "timestamp": "2026-01-01T07:30:00Z",
      "actor": "ai-batch",
      "action": "処理内容の自由記述"
    }
  ]
}
```

- `board` か `archive` のどちらのコレクションに属するかで「処理中／完了」を区別するため、別途の archived フラグは持たない。
- ownerの値は自分の環境のAIエージェント名に合わせて変更してよい（例: claude-code / cowork など）。priorityの基準: P0=即時対応 / P1=本日中 / P2=今週中 / P3=いつでも。
- activity は配列インラインで保持する。1ドキュメントの上限は約1MiBで、現実的なログ件数（数件〜数十件）では十分に収まる前提とする。

### 5. ステータス遷移ルール

```
needs-ai
  → in-progress（バッチ型AIが処理開始）
    → needs-human（人間の判断が必要）
    → done（処理完了）

needs-human
  → in-progress（人間または対話型AIが処理開始）
    → needs-ai（AI処理に戻す）
    → done（処理完了）

done → /completeエンドポイント呼び出し → board/{id} を archive/{id} へ移動（board から削除）

blocked → 人間がblocked_reasonを確認・解消後、needs-ai または needs-human に戻す
```

最終更新から72時間を超えてステータスが変化しないタスクは、ディスパッチャーが自動で blocked に変更し blocked_reason に「72時間変化なし」を記録する。

### 6. ディスパッチャー実装仕様

```markdown
# 実行スケジュール
毎日07:30（macOS launchd または Linux cron で設定）

# 処理フロー
1. GET /api/board で board コレクションの全タスクを取得する
2. status=needs-ai かつ owner=ai-batch のタスクのみ抽出する
3. tags フィールドに "dispatcher-lock" を含むタスクを除外する
   （ディスパッチャー自身の設定を変更するタスクを自動実行する無限ループを防ぐため）
4. priorityの高い順（P0→P1→P2→P3）で並べ、先頭の1件だけを処理する
5. タスク処理を実行する
6. 完了後、PATCH /api/board/{id} で以下を更新する:
   - handoff_note: 実施内容と次担当者への引継ぎ内容
   - status: needs-human または done
   - activity: 実施ログを1件追記
7. 処理中にエラーが発生した場合:
   - status → blocked
   - blocked_reason → エラー内容を記録
   - activity: エラーログを追記

# 72時間ルールの判定方法
board は処理中タスクのみで常に小さいため、GET /api/board で全件を取得し、
updated_at をクライアント側でフィルタして blocked へ更新する。
（Firestore で「updated_at < now-72h」等の複合条件をサーバー側クエリで行う場合は
 複合インデックスの作成が必要になる。board が小さい前提ではクライアント側判定の方が
 インデックス管理を避けられて簡潔。）

# なぜ1件ずつ処理するか
複数タスクを並列処理するとエラー発生時の原因特定が困難になる。
安定稼働を確認した後に並列化を検討する。
```

### 7. セキュリティ要件

- `board` / `archive` への直接アクセスは禁止する。Firestore Security Rules を deny-all に設定し、Cloud Run のサービスアカウント（Admin SDK）経由のアクセスのみを許可する。
- AIエージェントへの指示には「やってはいけないこと」のリストを必ず含める。
- AIは指示しなければ「最小権限で動く」は実行しない。許可範囲は運用者が明示する。

### 8. 拡張ガイド

- **複数ユーザー対応**: ownerフィールドにユーザーIDを追加。API認証をユーザー別トークンに拡張する。さらに踏み込む場合は Firebase Authentication 連携を検討。
- **ストレージのスケール**: Firestore はドキュメント増加に対してネイティブにスケールするため、移行先の検討は不要。関心は次の3点に移る。
  - `archive` の肥大化: TTLポリシーや、古いタスクを定期的に Cloud Storage へエクスポートして退避する運用を検討。
  - 読み取り量: `GET /api/archive` は件数が増えたら limit + startAfter カーソルでページネーションする。
  - 検索要件の拡張: status やタグでの絞り込みクエリを増やす場合は複合インデックスを設計する。
- **並列処理の追加**: 安定稼働後、P0タスクのみ並列実行を追加検討。エラーハンドリングとロールバック機構を先に設計する。
