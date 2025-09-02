# GitHub Releases ガイド — Yakubun Kobo

## 目的
このドキュメントは、Yakubun Kobo の GitHub Releases を「正式に整備」するための手順、署名手順、リリースノートの書式、Assets の配置基準、及び CI 自動化の例をまとめます。開発者・リリース担当が安全かつ再現可能にリリースできることを目的とします。

---

## 前提（必須準備）
- Windows 用コード署名証明書（PFX ファイル／EV 推奨）を入手し、安全に保管する。
  - 証明書は企業配布や SmartScreen 対策のために重要です。
- GitHub リポジトリに以下のシークレットを登録（Settings > Secrets）することを推奨：
  - SIGNING_PFX_BASE64: PFX を base64 エンコードした文字列（CI で復元して署名に使う）
  - SIGNING_PFX_PASSWORD: PFX のパスワード
  - GPG_PRIVATE_KEY: GPG キー（必要な場合）
  - GITHUB_TOKEN: CI 自動で Release を作る際に利用（自動的に GH Actions が用意）

---

## 手動署名手順（リリース担当者向け）
1. インストーラーをビルド（例: YakubunKobo_Setup-1.12.0.exe）
2. signtool を使って署名（Windows の Developer Kit に含まれる）：

```
# 管理者 PowerShell 例
signtool sign /f "path\to\certificate.pfx" /p "<PFX_PASSWORD>" /tr "http://timestamp.digicert.com" /td sha256 /fd sha256 "YakubunKobo_Setup-1.12.0.exe"
```

- /tr にタイムスタンプ URL を指定することで、署名の有効期限切れを回避できます。
- 署名が成功したら、署名情報を確認:

```
signtool verify /pa /v "YakubunKobo_Setup-1.12.0.exe"
```

3. ハッシュ（SHA256）を生成して checksums.txt に記載:

```
# PowerShell
Get-FileHash -Algorithm SHA256 "YakubunKobo_Setup-1.12.0.exe" | ForEach-Object { "{0}  {1}" -f $_.Hash, (Split-Path $_.Path -Leaf) } > checksums.txt
```

4. checksums.txt に GPG 署名を付与（オプションだが推奨）：

```
# GPG が使える環境で
gpg --armor --detach-sign checksums.txt
# これにより checksums.txt.asc が生成される
```

5. Release ページへアップロード:署名済インストーラー、checksums.txt、checksums.txt.asc（あれば）、スクリーンショット、デモ動画リンク（YouTube）等を Assets に追加する。

---

## リリースノートの書式（テンプレート）
リリースノートは日本語と英語の両方を用意することを推奨します。以下は推奨テンプレートです。

タイトル: Version {MAJOR}.{MINOR}.{PATCH} — リリース日（YYYY-MM-DD）

要約（1行）
- 例: 「トルコ語・インドネシア語・エスペラント語に対応。原文コンテキスト機能を追加。」

ハイライト
- 新機能
  - ・項目1（短く）
  - ・項目2
- 改善
  - ・項目
- 重大な修正（Security / Breaking changes）
  - ・項目（あれば）

インストール／更新手順
- ダウンロード: GitHub Releases の Assets から YakubunKobo_Setup-{version}.exe を取得
- システム要件: Windows（.NET 8 Desktop Runtime（x64）・Microsoft Edge WebView2 Runtime）
- API キー: 初回起動時に Google AI Studio の API キーを設定

署名と検証
- 本リリースのインストーラーはコード署名済みです。
- SHA256 ハッシュ: checksums.txt を参照。GPG による署名ファイル(checksums.txt.asc)を同梱しています。

既知の問題
- ・問題1（回避策）

問い合わせ
- GitHub: https://github.com/suzuryuquark/Yakubun_Kobo_Project

---

### 英語テンプレート（短縮）
Title: Version {version} — {YYYY-MM-DD}

Summary (1-line)
- e.g. "Added Turkish, Indonesian, and Esperanto support; introduced Source Context feature."

Highlights
- New features
- Improvements
- Security / Breaking changes

Installation
- Download the signed installer from Releases
- Requirements: .NET 8 Desktop Runtime (x64), Edge WebView2
- API: Set Google AI Studio API key on first run

Verification
- SHA256 in checksums.txt; checksums.txt.asc is GPG-signed.

Contact
- https://github.com/suzuryuquark/Yakubun_Kobo_Project

---

## Assets の配置ルール（推奨）
Assets はユーザーと自動化スクリプトが使いやすいように次の命名・配置規約に従うこと。

必須（各リリースに必ず含める）
- YakubunKobo_Setup-{version}.exe  (署名済)
- checksums.txt  (SHA256 を含む)
- checksums.txt.asc  (GPG 署名) — 可能なら

推奨（あると信頼性・運用性が上がる）
- YakubunKobo_Portable-{version}.zip  (サイレント展開用、署名またはハッシュ付)
- YakubunKobo_Setup-{version}.exe.sig  (別途署名ファイル形式がある場合)
- release-notes-{version}.md  (機械読み取りしやすいリリース本文)
- screenshots/*.png  (3〜5枚、機能別に分けてアップロード)
- demo-{version}.mp4 もしくは YouTube リンクを記載

ネーミングルール
- 小文字・ハイフン区切りで統一
- バージョンはタグと一致（例: v1.12.0 ⇒ ファイル名に 1.12.0 を含める）

---

## CI 自動化（GitHub Actions） — 概念的なフローとサンプル
目的: ビルド→署名→ハッシュ作成→Release 作成→Assets アップロード を自動化

ワークフローの主要ステップ
1. checkout
2. ビルド（Windows ランナーで Inno Setup 等を実行して .exe を生成）
3. 署名（PFX を secrets から復元して signtool を実行）
4. SHA256 を生成して checksums.txt を作成
5. GPG 署名を作成（オプション）
6. release を作成して assets をアップロード

簡易例（ワークフローの抜粋、実装時は secrets の取り扱いに注意）：

```yaml
name: Build and Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-sign-release:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '8.0.x'

      - name: Build
        run: |
          # ビルド手順（例: dotnet publish / InnoSetup など）
          echo "Build here"

      - name: Restore PFX
        env:
          SIGNING_PFX_BASE64: ${{ secrets.SIGNING_PFX_BASE64 }}
        run: |
          echo "$env:SIGNING_PFX_BASE64" | Out-File -Encoding ascii signing.pfx

      - name: Sign installer
        run: |
          signtool sign /f signing.pfx /p "${{ secrets.SIGNING_PFX_PASSWORD }}" /tr "http://timestamp.digicert.com" /td sha256 /fd sha256 "dist\YakubunKobo_Setup-${{ github.ref_name }}.exe"

      - name: Create checksums
        run: |
          Get-FileHash -Algorithm SHA256 "dist\YakubunKobo_Setup-${{ github.ref_name }}.exe" | ForEach-Object { "{0}  {1}" -f $_.Hash, (Split-Path $_.Path -Leaf) } > checksums.txt

      - name: Create Release and upload assets
        uses: softprops/action-gh-release@v1
        with:
          tag_name: ${{ github.ref_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Upload release assets
        uses: actions/upload-release-asset@v1
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: dist\YakubunKobo_Setup-${{ github.ref_name }}.exe
          asset_name: YakubunKobo_Setup-${{ github.ref_name }}.exe
          asset_content_type: application/x-msdownload

      - name: Upload checksums
        uses: actions/upload-release-asset@v1
        with:
          upload_url: ${{ steps.create_release.outputs.upload_url }}
          asset_path: checksums.txt
          asset_name: checksums.txt
          asset_content_type: text/plain
```

注意点: 上記は簡易例です。実際には PFX の復元を PowerShell でバイナリとして復元する処理や、create_release の step 名（例: create_release）を正しく参照する必要があります。

---

## Release 公開時の手順（短縮チェックリスト）
- [ ] ビルドをローカル／CI で実施し署名済インストーラーを生成
- [ ] checksums.txt を生成し、可能なら GPG 署名を付与
- [ ] GitHub Release を作成（タグ作成は事前に行う）
- [ ] Release の本文に日本語・英語の要約とインストール手順を記載
- [ ] Assets に署名済インストーラー、checksums.txt、screenshots、デモ動画リンクを追加
- [ ] リリースを公開後、SNS とメールで告知

---

## 付録: リリース本文（例）
````markdown
name=release-notes-1.12.0.md
```markdown
# Version 1.12.0 (2025-08-25)

要約
- トルコ語、インドネシア語、エスペラント語に対応しました。
- 原文コンテキスト機能を追加し、翻訳精度を向上しました。

新機能
- トルコ語 / インドネシア語 / エスペラント語の翻訳対応
- 原文コンテキスト（タイトル・著者・出典の指定）

改善
- 利用規約の明確化（機密情報・生成物の免責）

インストール
- ダウンロード: YakubunKobo_Setup-1.12.0.exe（署名済）
- SHA256: checksums.txt を参照

既知の問題
- なし（現時点）

お問い合わせ
- https://github.com/suzuryuquark/Yakubun_Kobo_Project
```
````

---

## 次の推奨アクション（優先順）
1. このドキュメントを .github/RELEASE_GUIDE.md としてコミット（このコミットを実行しました）。
2. CI 実装: GitHub Actions を用いた自動ビルド→署名→リリースのパイプラインを作成。まずは test-run をタグ vTEST などで実行して動作確認。
3. 署名済インストーラーを過去の重要リリース（例: v1.12.0）にも適用可能なら、署名済アセットに差し替えを検討。
4. Winget / Chocolatey 用マニフェスト作成を並行して検討。

---

# End of file