# あのね (Anone) — LP / 招待リンク着地ページ

iOSアプリ「あのね」（バンドルID: `com.zoniq.anone`）の紹介LPと、夫婦ペアリング招待リンクの着地ページ。Astro（TypeScript）製の静的サイトで、Cloudflare Pagesにデプロイする。

このリポジトリにはもともと [Claude Design](https://claude.ai/design) からのデザイン書き出し（`chats/`, `project/`）が入っている。`project/Anone LP.dc.html` がこのLPの元デザインで、実装の参照用に残している。

## 構成

```
src/
  layouts/Layout.astro     # 共通<head>（フォント、グローバルCSS読込）
  components/
    SiteHeader.astro        # 共通ヘッダー（ロゴ＋ナビ）
    SiteFooter.astro        # 共通フッター（運営会社/プライバシー/利用規約/お問い合わせリンク）
  pages/
    index.astro             # LP本体（Anone LP.dc.html を移植）
    invite/index.astro      # 招待リンクの着地ページ
    company/index.astro     # 運営会社（Company.dc.html を移植）
    privacy/index.astro     # プライバシーポリシー（Privacy.dc.html を移植）
    terms/index.astro       # 利用規約（Terms.dc.html を移植）
    contact/index.astro     # お問い合わせ（Contact.dc.html を移植）
  styles/global.css         # リセットCSS + キーフレームアニメーション
public/
  .well-known/apple-app-site-association   # iOSユニバーサルリンク用AASA
  _headers                 # AASAにContent-Type: application/jsonを付与
  _redirects               # /invite/* を invite/index.html に200リライト
```

## ローカル開発

```bash
npm install
npm run dev       # http://localhost:4321
npm run build     # dist/ に静的出力
npm run preview   # ビルド済みdist/をローカルで確認
```

## 招待リンクの仕組み

招待URLは `https://<サブドメイン>/invite/<コード>`（コードは8文字のCrockford base32）。静的サイトでは未知のコードを事前生成できないため、`public/_redirects` で `/invite/*` を `/invite/index.html` に200リライトし、`src/pages/invite/index.astro` 内のクライアントJSが `location.pathname` の末尾からコードを読み取って画面に表示する。

アプリがインストール済みの端末では、iOSのユニバーサルリンクがこの着地ページより先にアプリを起動する想定（OS側の挙動なので、このサイト側に分岐ロジックはない）。

## iOSユニバーサルリンク（AASA）

`public/.well-known/apple-app-site-association` に以下を設置済み：

```json
{
  "applinks": {
    "details": [
      {
        "appIDs": ["<TEAM_ID>.com.zoniq.anone"],
        "components": [
          { "/": "/invite/*", "comment": "ペアリング招待" }
        ]
      }
    ]
  }
}
```

**`<TEAM_ID>` はプレースホルダ。** Apple Developerの「Team ID」（10文字の英数字）が分かったら、このファイルの `<TEAM_ID>.com.zoniq.anone` を実際の値（例: `ABCDE12345.com.zoniq.anone`）に書き換える。AASAは厳密なJSONとして配信する必要があり、ファイル内にコメントを残せないため、この置き換え指示はこのREADMEに記載している。

`public/_headers` で配信時に `Content-Type: application/json` を強制している（拡張子が無いため、ホスティング側のデフォルト推測に頼らないようにするため）。

## デプロイ手順（Cloudflare Pages）

### 1. GitHubにpush

```bash
git push -u origin main
```

### 2. Cloudflare PagesでGitHub連携

1. Cloudflareダッシュボード → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
2. このリポジトリを選択
3. ビルド設定:
   - **Framework preset**: Astro
   - **Build command**: `npm run build`
   - **Build output directory**: `dist`
4. **Save and Deploy**

### 3. カスタムサブドメインの割り当て

1. Pagesプロジェクト → **Custom domains** → **Set up a custom domain**
2. 招待リンク用のサブドメイン（例: `invite.example.com`）を入力
3. Cloudflareがネームサーバーを管理しているドメインなら自動でDNSレコードが追加される。外部DNSの場合は、表示されたCNAMEターゲット（`<project>.pages.dev`）を自分のDNSに手動で追加:

   ```
   Type:  CNAME
   Name:  invite (またはサブドメイン名)
   Target: <project>.pages.dev
   ```

4. SSL証明書の発行が完了するまで数分待つ

### 4. AASA配信の検証

```bash
curl -sv https://<サブドメイン>/.well-known/apple-app-site-association
```

確認すること:
- `HTTP/2 200`（リダイレクト無し。`< location:` ヘッダが出ていないこと）
- `content-type: application/json`（`_headers` が効いているか）
- レスポンスボディが想定のJSON（`<TEAM_ID>` を実際の値に差し替えていればそれが反映されているか）

### 5. 招待リンクの動作確認

```bash
curl -sI https://<サブドメイン>/invite/2W7K9XHM
```

`_redirects` のリライトが効いていれば、リダイレクトなしで200と`invite/index.html`の内容が返る。ブラウザで開いて「招待コード: 2W7K9XHM」のように表示されることを確認する。
