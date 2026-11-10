# TODOアプリ V1 設計書 (02_design.md)

## 1. 作るものとファイルの役割

本設計書では、Express + TypeScript + EJSを用いたTODOアプリ（V1）のアーキテクチャおよび各コンポーネントの詳細を定義します。ファイル構成は授業テキストに準拠し、プロジェクト直下の構成に従います。

* **ファイル構成（配置）**:
* `index.ts`, `db.ts`, `package.json`, `tsconfig.json`: 共通基盤および設定ファイル。
* `public/todo/todo.js`: ブラウザ側からAPIを叩き、DOM操作やイベントハンドリングを行うクライアントサイドスクリプト。
* `views/todo/index.ejs`: TODO画面のテンプレート。
* `src/apps/todo/todoRouter.ts`: TODO機能のルーティング定義。
* `src/apps/todo/todoController.ts`: リクエストを受け取り、Modelを呼び出してレスポンスを返すコントローラー。
* `src/apps/todo/todoModel.ts`: データベース（Neon PostgreSQL）とのデータアクセスを担うモデル。
* `sql/todo/020_create_todo_tasks.sql`: TODOテーブル作成用のDDLスクリプト。
* `docs/todo/`: 各種ドキュメント（要件定義、設計、実装計画、テスト結果）。
*※ 実際の詳細なディレクトリ配置は要確認とし、必要に応じて実装時に微調整します。*



---

## 2. リクエストはどこを通るか

リクエストのライフサイクル（ブラウザからDBまで）の流れは以下の通りです。

```
[ブラウザ (EJS / todo.js)] 
       │ (Fetch API)
       ▼
[index.ts (Express App)]
       │ ルーティング委譲
       ▼
[todoRouter.ts]
       │
       ▼
[todoController.ts]
       │
       ▼
[todoModel.ts]
       │ (SQL実行 / db.ts)
       ▼
[Neon PostgreSQL (DB)]

```

1. ユーザーがブラウザ上で操作を行うと、`public/todo/todo.js` 内の `TodoApi` クラス経由で非同期リクエスト（Fetch）が送信されます。
2. リクエストはExpressのアプリケーション（`index.ts`）を通り、`todoRouter.ts` にルーティングされます。
3. `todoRouter.ts` から `todoController.ts` の各ハンドラーへ処理が渡されます。
4. `todoController.ts` はバリデーションや入力値の整形を行った上で、`todoModel.ts` を呼び出します。
5. `todoModel.ts` は共通の `db.ts` を経由してNeon PostgreSQLへSQLを発行し、結果を取得・返却します。

---

## 3. DBの設計

データベースにはNeon PostgreSQLを使用します。

* **テーブル名**: `todos`
* **カラム設計 (`sql/todo/020_create_todo_tasks.sql`)**:
* `id`: `SERIAL PRIMARY KEY` （一意のタスクID、1以上の整数）
* `title`: `VARCHAR(100) NOT NULL` （タスク名、前後の空白を除いて1〜100文字）
* `completed`: `BOOLEAN DEFAULT FALSE NOT NULL` （完了状態）
* `created_at`: `TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP` （作成日時）



---

## 4. APIの共通ルール

* **エンドポイントのベース**: `/api/todo/v1/todos`
* **共通成功レスポンス（データ取得・作成・更新時）**:
* JSON形式で `data` プロパティに結果を包んで返却する。
* 例: `{ "data": { ... } }` または `{ "data": [ ... ] }`


* **共通エラーレスポンス**:
* エラー時は `error` プロパティ（またはメッセージ）を含めたJSONを返却する。
* ステータスコード:
* `400`: 不正なリクエスト / パラメータエラー
* `422`: バリデーションエラー（タイトルの文字数オーバーなど）
* `404`: 対象データが存在しない場合
* `500`: サーバー内部エラー
* `503`: データベース接続エラー等





---

## 5. 各APIエンドポイントの設計

### 5.1 REQ-01 TODO一覧を取得する

* **Method / Path**: `GET /api/todo/v1/todos`
* **Controller処理**: `todoModel` から全件取得（作成日時の降順など）し、`200 OK` で `{ data: [...] }` を返却する。0件の場合は `{ data: [] }`。

### 5.2 REQ-02 TODOを登録する

* **Method / Path**: `POST /api/todo/v1/todos`
* **Request Body**: `{ "title": "string" }`
* **Controller処理**: `title` の前後の空白をトリムし、1〜100文字の範囲内かバリデーションを行う。違反時は `422`。正常時はモデルを通じてDBに登録し、`201 Created` と共に登録データを返却する。

### 5.3 REQ-03 完了状態を変える

* **Method / Path**: `PATCH /api/todo/v1/todos/:id`
* **Request Body**: `{ "completed": boolean }`
* **URL Params**: `id` (1以上の整数)
* **Controller処理**: IDの形式チェックおよび `completed` の型チェックを行う。該当IDのレコードが存在しない場合は `404` を返却する。更新成功時は `200 OK` と更新後データを返却する。

### 5.4 REQ-04 TODOを削除する

* **Method / Path**: `DELETE /api/todo/v1/todos/:id`
* **URL Params**: `id` (1以上の整数)
* **Controller処理**: 該当IDのレコードが存在するか確認し、存在しない場合は `404`。削除成功時は本文なしの `204 No Content` を返却する。

---

## 6. 画面とTodoApiの設計

* **画面 (`views/todo/index.ejs`)**:
* シンプルなTODO入力フォーム、タスク一覧リスト、完了チェックボックス、削除ボタンを配置する。


* **クライアントサイドスクリプト (`public/todo/todo.js`)**:
* `TodoApi` クラスを定義し、各APIリクエスト（GET, POST, PATCH, DELETE）のカプセル化を行う。
* 画面のイベントリスナーから `TodoApi` を呼び出し、DOMを動的に更新する。



---

## 7. エラーはどこで処理するか

* **コントローラー層 / ミドルウェア**:
* 非同期処理（データベースアクセス等）で発生した例外は `try-catch` で捕捉し、共通のエラーハンドリング機構またはレスポンス生成関数へ受け渡す。
* データベース接続エラー（503）や予期せぬエラー（500）を適切にキャッチし、クライアントへ適切なHTTPステータスコードとエラーメッセージを返す。



---

## 8. 実装への注意事項

* **`index.ts` の追記は最低限にする**:
* ルーティングの設定は極力 `todoRouter.ts` にカプセル化し、`index.ts` ではルーターのインポートと `app.use('/api/todo/v1', todoRouter)` のような最低限のマウントのみを行うこと。


* **既存機能の維持**:
* 既存の `/` および `/api/db-check` の挙動に影響を与えないこと。


* **環境変数の利用**:
* データベース接続には必ず既存の `DATABASE_URL` を利用し、コード内に機密情報をハードコーディングしないこと。