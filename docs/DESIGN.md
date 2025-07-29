# OAuth Training プロジェクト 設計書

## 1. プロジェクト概要

* Google OAuth を利用したユーザー認証付きの Web アプリケーションを構築
* ユーザー登録・ログイン機能、DB からのデータ取得・表示、画面からのデータ登録・編集・削除、ログアウト、アカウント削除

## 2. 技術スタック

* バックエンド: Go (Gin)
* フロントエンド: React + TypeScript (Chakra UI)
* データベース: PostgreSQL
* 認証: Google OAuth 2.0
* JWT を利用した API 認証 (Authorization ヘッダー)
* CI/CD: GitHub Actions

## 3. システムアーキテクチャ

```text
[Browser] ⇄ [React Frontend] ⇄ [Gin API Server] ⇄ [PostgreSQL]
```

* 認証状態管理: JWT (アクセストークンをクライアント側で保持し、API 呼び出し時に `Authorization: Bearer <token>` ヘッダーで送信)

## 4. 認証フロー

1. フロントの「Googleでログイン」ボタン押下
2. `/auth/google/login` エンドポイントにリダイレクト
3. サーバー側で OAuth 認証 URL 生成 → Google へ遷移
4. 認証後、`/auth/google/callback` でアクセストークン受け取り
5. ユーザー情報取得後、DB に登録／更新
6. JWT (アクセストークン) をレスポンス (`{ "token": "<jwt>" }`)
7. フロントで認証済み画面へリダイレクト

## 5. データベース設計

### ER 図

```mermaid
erDiagram
  USERS {
    id UUID PK
    email VARCHAR UNIQUE
    name VARCHAR
    google_id VARCHAR UNIQUE
    created_at TIMESTAMP
    updated_at TIMESTAMP
  }
  CONTENTS {
    id UUID PK
    user_id UUID FK
    content TEXT
    created_at TIMESTAMP
  }
  USERS ||--o{ CONTENTS : has
```

### テーブル定義例

```sql
-- users テーブル
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(100),
  google_id VARCHAR(100) UNIQUE,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);

-- contents テーブル
CREATE TABLE contents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  content TEXT NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now()
);
```
* `ON DELETE CASCADE` により、ユーザー削除時に紐づくコンテンツも自動削除
* 必要に応じてインデックスを追加: `CREATE INDEX idx_contents_user ON contents(user_id);`

## 6. API 設計

| メソッド   | パス                    | 説明                    |
| ------ | --------------------- | --------------------- |
| GET    | /auth/google/login    | Google OAuth 認証開始     |
| GET    | /auth/google/callback | Google OAuth コールバック処理 |
| POST   | /auth/logout          | ログアウト処理               |
| DELETE | /api/user             | アカウント削除 (ユーザー＋コンテンツ)  |
| GET    | /api/user             | 認証済みユーザー情報取得          |
| GET    | /api/contents         | コンテンツ一覧取得 (ユーザー単位)    |
| POST   | /api/contents         | コンテンツ新規登録             |
| PUT    | /api/contents/\:id    | コンテンツ更新               |
| DELETE | /api/contents/\:id    | コンテンツ削除               |

## 7. フロントエンド構成

```text
frontend/src/
├─ pages/
│   ├─ LoginPage.tsx
│   ├─ Dashboard.tsx
│   ├─ ContentListPage.tsx   ← /api/contents 取得 + 編集・削除
│   ├─ NewContentPage.tsx    ← テキスト入力 → POST /api/contents
│   ├─ EditContentPage.tsx   ← PUT /api/contents/:id
│   └─ SettingsPage.tsx      ← ログアウト・アカウント削除
├─ components/
│   ├─ Header.tsx
│   ├─ ContentList.tsx       ← 一覧レンダリング + 操作ボタン
│   ├─ ContentItem.tsx       ← 編集・削除ボタン
│   └─ EmptyState.tsx         ← データなし時の表示
└─ hooks/
    ├─ useAuth.ts
    └─ useContents.ts       ← fetch, create, update, delete
```

## 8. ディレクトリ構造

```text
oauth_training/
├─ backend/
│   ├─ Dockerfile           # Go アプリ用ビルド定義
│   ├─ main.go
│   ├─ handlers/
│   ├─ models/
│   └─ services/
├─ frontend/
│   ├─ Dockerfile           # React アプリ用ビルド定義
│   ├─ src/
│   └─ public/
├─ docker-compose.yml       # 各サービスの連携定義
├─ docs/
│   └─ daily/
├─ .gitignore
└─ README.md
```

## 9. 開発フロー

1. `feat/<機能>` ブランチ作成
2. 開発 → 単体テスト → PR 作成
3. コードレビュー → main へマージ
4. GitHub Actions で CI 実行

## 10. 画面遷移図

```mermaid
flowchart TD
  LoginPage["ログイン画面(LoginPage)"] -->|Googleでログインボタン押下| AuthCallback[("/auth/google/callback")]
  AuthCallback -->|認証成功| Dashboard["ダッシュボード(Dashboard)"]
  Dashboard -->|「コンテンツ一覧を見る」クリック| ContentListPage["コンテンツ一覧(ContentListPage)"]
  ContentListPage -->|「新規作成」ボタン押下| NewContentPage["コンテンツ新規作成(NewContentPage)"]
  ContentListPage -->|「編集」ボタン押下| EditContentPage["コンテンツ編集(EditContentPage)"]
  ContentListPage -->|「設定」アイコン押下| SettingsPage["設定(SettingsPage)"]
  NewContentPage -->|「保存」ボタン押下| ContentListPage
  EditContentPage -->|「更新」ボタン押下| ContentListPage
  SettingsPage -->|「ログアウト」ボタン押下| LoginPage
  SettingsPage -->|「アカウント削除」ボタン押下| LoginPage
```

## 11. UI / 機能実装概要. UI / 機能実装概要

### 11.1 コンテンツ表示（一覧）

* フロント: `GET /api/contents` → 認証ユーザーに紐づくレコード取得
* レスポンスが空の場合: `EmptyState` コンポーネントで **"データが存在しません"** を表示

### 11.2 コンテンツ登録

* フロント: `NewContentPage` のテキストボックス入力 → `POST /api/contents` に `{ content: string }` を送信
* バック: 受信した `content` と `user_id` で INSERT

### 11.3 コンテンツ編集

* フロント: `ContentItem` の「編集」ボタン → `EditContentPage` へ
* バック: `PUT /api/contents/:id` で指定 `id` の content を更新

### 11.4 コンテンツ削除

* フロント: `ContentItem` の「削除」ボタン → `DELETE /api/contents/:id`
* バック: 指定 `id` のレコードを削除

### 11.5 ユーザー作成時のコンテンツ初期化

* バック: `/auth/google/callback` 内の新規ユーザー作成後に `initContentsForUser(userID)` を呼び出し、初期レコードを INSERT

### 11.6 ログアウト

* フロント: `SettingsPage` の「ログアウト」クリック → `POST /auth/logout` → クライアント側で JWT を削除 → `LoginPage` へリダイレクト
* バック: `POST /auth/logout` エンドポイントでステータス 200 を返却

### 11.7 アカウント削除

* フロント: `SettingsPage` の「アカウント削除」クリック → `DELETE /api/user` → クライアント側で JWT を削除 → `LoginPage` へリダイレクト
* バック: `DELETE /api/user` エンドポイントで users テーブルから該当レコードを削除 (ON DELETE CASCADE により contents も自動削除)
