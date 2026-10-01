# mitsumura-gbp-autopost

みつむら接骨院の Google ビジネスプロフィール（GBP）に、ブログ記事と連動した投稿を
**毎週 月曜・木曜の朝8時** に1本ずつ自動で出す。GitHub Actions で動くので Mac の電源は不要。

仕組みは名駅院の Threads 自動投稿（`kirin-threads-autopost`）と同じ。

## 何をするか

| ワークフロー | 動くタイミング | 内容 |
|---|---|---|
| `post.yml` | 月・木 08:00 JST | キューの次の1本を投稿し、投稿済みとして記録する |

- ストック: `posts/queue.csv`（422本 = 211記事 × 投稿A/B）。週2本で約4年分
- 並び順: 投稿A 211本 → 投稿B 211本。同じ記事のAとBは約2年空く。カテゴリは偏らないよう全期間に散らしてある
- **公開前の予約記事は飛ばす。** 記事公開日が来ていない、またはリンク先が表示できない（404など）行は後回しにする
- 画像: 3本に1本は `images/photos/` の院の写真（使用回数の少ない順）、それ以外は `images/cards/` の見出し入り画像
- 「詳細」ボタンに記事URLを付ける。種類は「最新情報」

## ファイル

```
posts/queue.csv          投稿の順番と本文（build_queue.py で xlsx から作る）
images/cards/{id}.jpg    見出し入りの画像（make_cards.py で作る。Mac のヒラギノを使う）
images/photos/           院の写真を入れる場所（jpg/png、横長 4:3 推奨、1枚 5MB 以下）
state/state.json         投稿済みIDと写真の使用回数。二重投稿を防ぐ
state/posted_log.csv     いつ何を投稿したか
scripts/post.py          投稿本体
scripts/setup_auth.py    初回だけ使う。管理アカウントで許可してトークンを取る
```

## 初回セットアップ（上から順に）

### 1. GBP API の利用申請（先生の操作。審査に数日〜数週間）

GBP の API は申請して承認されるまで使えない（割り当てが0のまま）。

1. GBP の管理アカウントで https://console.cloud.google.com/ を開き、プロジェクトを新規作成（名前の例: `mitsumura-gbp`）
2. プロジェクト番号を控える（ダッシュボードに表示される）
3. 申請フォーム https://support.google.com/business/contact/api_default を開き、「Application for Basic API Access」を選んで送信
   - 連絡先メールは GBP の管理アカウント
   - プロジェクト番号、ビジネスのサイト（https://www.058-327-7771.net/）を入力
   - 条件: GBP が **60日以上前にオーナー確認済み** で、ウェブサイトが登録されていること
4. 承認メールが届くまで待つ

### 2. API の有効化と OAuth クライアントの作成（承認後）

プロジェクトの「APIとサービス」→「ライブラリ」で次の3つを有効にする。

- Google My Business API
- My Business Account Management API
- My Business Business Information API

続いて:

1. 「OAuth 同意画面」を作成（種類: 外部、アプリ名: みつむらGBP投稿、サポートメール: 自分）
2. **公開ステータスを「本番環境」にする。**「テスト」のままだとトークンが7日で切れて止まる
   （未確認アプリの警告が出るが、自分のアカウントで使うだけなので「詳細」→「移動」で進めてよい）
3. 「認証情報」→「OAuth クライアント ID を作成」→ 種類「デスクトップアプリ」
4. 表示されたクライアントIDとシークレットを、このフォルダの `.env` に書く（git には入らない）

```
GBP_CLIENT_ID=...
GBP_CLIENT_SECRET=...
```

### 3. 許可してトークンを取る（先生の操作）

```bash
cd ~/dev/mitsumura-gbp-autopost && python3 scripts/setup_auth.py
```

ブラウザが開くので GBP の管理アカウントでログインして許可する。
`.env` にリフレッシュトークンが保存され、ターミナルにアカウントIDとビジネスIDが出る。

### 4. GitHub に登録する

リポジトリの Settings → Secrets and variables → Actions:

| 種類 | 名前 | 中身 |
|---|---|---|
| Secret | `GBP_CLIENT_ID` | `.env` の同名の値 |
| Secret | `GBP_CLIENT_SECRET` | 同上 |
| Secret | `GBP_REFRESH_TOKEN` | 同上 |
| Secret | `GBP_ACCOUNT_ID` | setup_auth.py が表示したアカウントID |
| Secret | `GBP_LOCATION_ID` | 同じくビジネスID（みつむら接骨院の行） |

画像は Google が URL から取りに行くので、誰でも開ける URL が必要。画像の URL（`IMAGE_BASE_URL`）は post.yml に既定値として書いてあるので登録不要。**このリポジトリは公開にして、画像をここから配信する**
（投稿文・画像・プログラムは公開される。API の鍵は Secrets に入れるので公開されない。`.env` は git に入らない）。

### 5. テスト投稿

Actions → GBP投稿 → Run workflow

1. `dry_run` にチェックを入れたまま実行 → ログで次に出る投稿を確認
2. `dry_run` を外して1本だけ実行 → GBP とGoogleマップで表示を確認

問題なければ、以後は月・木の朝8時に自動で出る。

## 運用

- **失敗したとき**: GitHub から失敗通知メールが届く。ログの `投稿に失敗` の行を見る
  - `invalid_grant` → トークン切れ。手順3をやり直し、`GBP_REFRESH_TOKEN` を更新
  - `429` / `PERMISSION_DENIED` → API 承認前か、API の有効化漏れ
- **院の写真を足す**: `images/photos/` に入れてコミットするだけ
- **ストックを足す**: xlsx を更新して `build_queue.py` → `make_cards.py` を実行してコミット。投稿済みの行は id で判定するので二重には出ない
- **一時停止**: Actions → GBP投稿 → 右上「…」→ Disable workflow

## 投稿内容の約束（生成時に検証済み）

保険への言及なし／体験談・患者エピソードなし／「完治」「必ず治る」等の断定なし／
危険なサインのときは受診するよう全投稿に明記／本文に電話番号を書かない。
詳細は xlsx の「使い方・設計方針」シート。
