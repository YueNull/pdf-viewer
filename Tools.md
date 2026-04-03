# PDF Viewer — ツール・ライブラリ詳細

## 1. 現在使用中のツール（HTML 実装）

### PDF.js 3.11.174

| 項目 | 値 |
|------|-----|
| CDN（本体） | `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js` |
| CDN（Worker） | `https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js` |
| 用途 | PDF ページを canvas にレンダリング |
| 主な API | `pdfjsLib.getDocument()`, `page.render()`, `page.getViewport()` |

Python 化の際はオフライン対応のためローカルファイルとして同梱し、CDN から切り替える。

---

### Browser API — File System Access API

| 項目 | 値 |
|------|-----|
| メソッド | `window.showDirectoryPicker({ mode: 'read' })` |
| 対応ブラウザ | Chrome 86+ / Edge 86+（**Safari・Firefox 非対応**） |
| 用途 | ローカルフォルダの選択・再帰スキャン |
| 代替（Python） | `tkinter.filedialog.askdirectory()` でフォルダ選択 → JS ブリッジ経由でパス渡し |

---

### Browser API — IndexedDB

| 項目 | 値 |
|------|-----|
| DB 名 | `pdf-viewer-db` |
| Store 名 | `handles` |
| 用途 | 前回フォルダの `FileSystemDirectoryHandle` を保存・復元 |
| 代替（Python） | アプリデータフォルダへの JSON/設定ファイル保存に切り替え |

---

### Browser API — localStorage

| 項目 | 値 |
|------|-----|
| キー | `pdf_annotations` |
| 用途 | ハイライト・コメント注釈データの永続化（JSON） |
| 代替（Python） | SQLite または JSON ファイルへの書き出しに移行を検討 |

---

### Browser API — IntersectionObserver

| 用途 | サムネイルの遅延レンダリング（ビューポート外はスキップ） |
|------|------|
| margin | `rootMargin: '200px'`（スクロール前に事前レンダリング） |
| 効果 | 大量ページ表示時のメモリ・CPU 節約 |

---

### Browser API — Canvas 2D API

| 用途 | PDF ページ表示 / 注釈オーバーレイ描画 |
|------|------|
| canvas 要素 | `#modal-canvas`（PDF 表示）, `#anno-canvas`（注釈オーバーレイ） |

---

## 2. Python 化時に必要なツール

### ウィンドウ・UI フレームワーク

| ツール | 用途 |
|--------|------|
| **PyWebView** | Python + HTML/CSS/JS をデスクトップウィンドウで動かす（推奨） |
| tkinter + CEF | 代替案。Chromium Embedded Framework を tkinter に組み込む |

PyWebView では `showDirectoryPicker` が動作しない可能性があるため、  
フォルダ選択は Python 側の `tkinter.filedialog.askdirectory()` で行い、  
結果をJS ブリッジ（`window.pywebview.api.xxx()`）経由でフロントエンドに渡す。

---

### PDF オフラインレンダリング

| ツール | 用途 |
|--------|------|
| **PDF.js（ローカル同梱）** | CDN 不要にする。`pdf.min.js` / `pdf.worker.min.js` をフォルダに同梱 |
| **pymupdf（fitz）** | Python 側で PDF → 画像変換が必要な場合（OCR 連携等） |

---

### プリンター連携（Brother HL-J7010CDW）

| ツール | 用途 | インストール |
|--------|------|------------|
| **pywin32** | `win32print` / `win32api` で印刷先プリンターを名前指定 | `pip install pywin32` |
| **SumatraPDF CLI** | コマンドラインから `-print-to` で指定プリンターに印刷（`pywin32` 代替） | 単体配布 exe |

```python
# pywin32 を使う場合
import win32print, win32api
win32print.SetDefaultPrinter("Brother HL-J7010CDW")
win32api.ShellExecute(0, "print", r"C:\path\to\file.pdf", None, ".", 0)

# SumatraPDF CLI を使う場合
import subprocess
subprocess.run(["SumatraPDF.exe", "-print-to", "Brother HL-J7010CDW", pdf_path])
```

---

### スキャナー連携（RICOH fi-7700 / fi-7700S + PaperStream IP TWAIN 3.40.1）

| ツール | 用途 | インストール |
|--------|------|------------|
| **twain** | Python から TWAIN ドライバー経由でスキャン実行 | `pip install twain` |
| **img2pdf** | スキャン画像を PDF に変換 | `pip install img2pdf` |
| **Pillow** | 画像後処理（トリミング・解像度変換等） | `pip install Pillow` |
| **fpdf2** | テキスト付き PDF 生成（代替案） | `pip install fpdf2` |

```python
import twain

sm = twain.SourceManager(0)
ss = sm.OpenSource("PaperStream IP fi-7700")  # ドライバー側の表示名に合わせる

# 解像度・カラーモード設定
ss.SetCapability(twain.ICAP_XRESOLUTION, twain.TWTY_FIX32, 300.0)
ss.SetCapability(twain.ICAP_YRESOLUTION, twain.TWTY_FIX32, 300.0)
ss.SetCapability(twain.ICAP_PIXELTYPE, twain.TWTY_UINT16, twain.TWPT_RGB)

ss.RequestAcquire(0, 0)
rv = ss.XferImageNatively()
if rv:
    handle, count = rv
    # handle から画像バイトを取得して保存
ss.destroy()
sm.destroy()
```

> **注意：** PaperStream IP (TWAIN) は Windows 専用。  
> TWAIN DSM の 32bit / 64bit がドライバーと一致する必要あり。  
> Python インタープリタのビット数を必ずドライバーに合わせること。

---

## 3. 環境要件まとめ

| 環境 | 要件 |
|------|------|
| **HTML 版（現行）** | Chrome 86+ または Edge 86+（File System Access API 必須） |
| **Python 版（将来）** | Python 3.10+（`str \| None` 型ヒント使用） |
| **プリンター** | Windows 専用。`pywin32` または SumatraPDF exe が必要 |
| **スキャナー** | Windows 専用。PaperStream IP TWAIN ドライバーと同ビット数の Python が必要 |
