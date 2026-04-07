# PDF Viewer — 機能仕様

## 概要

ローカルの PDF フォルダを閲覧するビューワーアプリ。  
現行実装: 単一 HTML ファイル（`pdf-viewer.html`）。  
将来: Python（PyWebView / Tkinter + CEF 等）でデスクトップアプリ化予定。  
使用ツール詳細 → [tool.md](tool.md)

### C. PDF 束ねる（PDFMerge）

ブラウザでは複数PDFバイナリを結合するAPIがないため不可。

**Python 実装方針：**
```python
from pypdf import PdfWriter

writer = PdfWriter()
for pdf_path in pdf_paths:
    writer.append(pdf_path)
with open("merged.pdf", "wb") as f:
    writer.write(f)
```

### D. PDF ばらす（PDFSplit）

**Python 実装方針：**
```python
from pypdf import PdfReader, PdfWriter

reader = PdfReader("input.pdf")
for i, page in enumerate(reader.pages):
    writer = PdfWriter()
    writer.add_page(page)
    with open(f"page_{i+1}.pdf", "wb") as f:
        writer.write(f)
```

### E. 実ファイルの名前変更・削除・移動

ブラウザの File System Access API は読み取り専用のため不可（`readwrite` モードでも rename/delete は未対応）。

**Python 実装方針：**
```python
import os, shutil
os.rename(src, dst)       # 名前変更・移動
os.remove(path)           # 削除
shutil.copy2(src, dst)    # 複製
```

### F. プリンタ詳細設定ダイアログ

**Python 実装方針：**
```python
import win32print, win32ui, win32con
hprinter = win32print.OpenPrinter(printer_name)
devmode = win32print.GetPrinter(hprinter, 8)['pDevMode']
win32ui.CreatePrintDialog().DoModal()  # OS標準ダイアログ
```

---

## UI レイアウト

```
┌─────────────────────────────────────────────────────┐
│ メニューバー（ファイル / 表示 / ヘルプ）              │
├─────────────────────────────────────────────────────┤
│ ツールバー（フォルダを開く | PDFを追加 | サムネサイズ）│
├─────────────────────────────────────────────────────┤
│ [復元バナー] 前回フォルダを再度開くか確認（任意表示）  │
├──────────────┬──┬──────────────────────────────────┤
│              │  │ ツールバー（ファイル名 | ページ数   │
│  左サイドバー │リ│           | ページジャンプ入力）   │
│  ファイルツリー│サ│  右コンテンツ（サムネイルグリッド） │
│              │イ│  ファイル区切りヘッダー + 全ページ  │
│              │ズ│                                    │
└──────────────┴──┴────────────────────────────────────┘
           ↓ サムネイルクリックで全画面モーダル
┌─────────────────────────────────────────────────────┐
│ モーダルバー（タイトル | ページナビ | ズーム | 閉じる）│
├─────────────────────────────────────────────────────┤
│ 注釈ツールバー（選択/ハイライト色/コメント/消しゴム/消去）│
├─────────────────────────────────────────────────────┤
│              PDF ページ表示（canvas）                 │
│              注釈オーバーレイ canvas                  │
└─────────────────────────────────────────────────────┘
```

---

## 機能一覧

### 1. フォルダ管理

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| フォルダを開く | `showDirectoryPicker` でフォルダ選択（Chrome/Edge のみ） | `openFolderHandle` |
| サブフォルダ再帰スキャン | フォルダ開封時に全サブフォルダを一括スキャン | `scanDirRecursiveAll` |
| フォルダパス表示 | サイドバー上部に現在フォルダ名を表示 | — |
| 前回フォルダ復元 | IndexedDB にハンドルを保存、次回起動時に復元バナー表示 | `saveLastHandle`, `loadLastHandle` |
| 前回フォルダ再オープン | バナーの「再度開く」で権限を再要求して即復元 | `restoreFolder` |

### 2. ファイル管理

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| PDF 個別追加 | `<input type="file">` で複数ファイルを仮想ツリーに追加 | — |
| ドラッグ＆ドロップ追加 | PDF をウィンドウにドロップしてツリーに追加 | — |
| ファイル件数表示 | サイドバーヘッダーに PDF 合計件数表示 | `countFiles` |
| ファイルソート | 名前順 (A→Z/Z→A)・更新日時順・ファイルサイズ順 | `sortTreeChildren` |
| ファイル種別アイコン | 拡張子に応じた絵文字アイコン表示（PDF/Excel/Word/画像/動画 など） | `getFileIcon` |
| ファイルプロパティ | 右クリック → 名前・種類・サイズ・更新日時・ページ数を表示 | `showFileProperties` |
| 非PDFファイルを開く | ダブルクリックでブラウザ新タブに開く（blob URL経由）。右クリック「プログラムで開く」でも同様 | `openFileWithDefault` |
| ファイル種別アイコン拡充 | 弥生会計(.kaikei/.kdk/.mfm)・弥生給与(.kyuyo)・DB系(.dat/.db)等を追加 | `getFileIcon` |

### 3. 左サイドバー（ファイルツリー）

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| ツリービュー | フォルダ / PDF ファイルをネスト表示 | `renderTreeList`, `renderTreeNodes` |
| フォルダ展開・折りたたみ | クリックで展開（初回スキャン実行）/ 折りたたみ | `scanDir` |
| ファイルクリック | フォルダモード: そのファイル先頭へスクロール | `selectFile` |
| アクティブ強調 | 選択中ファイルを青背景でハイライト | — |
| ページ数表示 | 読み込み完了後にファイル名横にページ数表示 | — |
| サイドバーリサイズ | 左右ドラッグで幅変更（最小 140px、最大 400px） | resize event handlers |
| フォルダカラー | 右クリック → 23色からフォルダ色を選択・localStorage永続化 | `showFolderColorPicker`, `folderColorStore` |
| フォルダタグ | 右クリック → タグを追加・削除、タグバーでフィルタリング（同一階層のみ）・localStorage永続化 | `addFolderTag`, `updateTagFilterBar` |
| インラインリネーム | ファイル名をダブルクリックで直接編集 | tree label dblclick handler |

### 4. サムネイルグリッド — フォルダ結合ビュー（右コンテンツ）

フォルダを開くと全 PDF を連続表示。ファイル間に区切りヘッダーを挿入。

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| 全 PDF 連続表示 | フォルダ内の全 PDF を再帰スキャン後に一括読み込み | `loadFolderView` |
| ファイル区切りヘッダー | 各 PDF の先頭に `📄 ファイル名` 行を挿入 | `createFileDivider` |
| 合計ページ数表示 | ツールバーに「合計 XXX ページ」を表示 | `updateFolderPageCount` |
| グローバルページジャンプ | ツールバー右の入力欄でページ番号入力 → Enter でスクロール | `jumpToFolderPage` |
| 遅延レンダリング | `IntersectionObserver` でビューポート内のみレンダリング | `renderThumb` |
| サムネイルサイズ変更 | 小(80) / 中(120) / 大(160) / 特大(200) px | — |
| ページ番号ラベル | フォルダモード: グローバルページ番号、単一ファイル: ローカルページ番号 | `createThumbCard` |
| 全画面モーダルを開く | サムネイルクリックで対応ページの全画面表示 | `openModal` |

### 5. 全画面モーダル（PDF ビューワー）

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| ページ全画面表示 | PDF.js でページを canvas にレンダリング | `renderModalPage` |
| ページナビゲーション | 最初へ / 前へ / 次へ / 最後へ ボタン | `modalGoTo` |
| ページジャンプ | 入力フィールドにページ番号入力 + Enter | `modalGoTo` |
| ズームイン・アウト | 0.5x / 0.75x / 1x / 1.25x / 1.5x / 2x / 3x のステップ切替 | `renderModalPage` |
| ズームラベル表示 | 現在のズーム倍率をパーセント表示 | — |
| モーダルを閉じる | 「✕ 閉じる」ボタン / オーバーレイクリック / Escape キー | — |

### 6. 注釈機能（全画面モーダル内）

| 機能 | 詳細 | 主な関数 |
|------|------|---------|
| ハイライト描画 | ドラッグで矩形ハイライト（黄/緑/青/ピンク/赤の 5 色・Edge準拠カラー） | anno canvas mousedown/move/up |
| ハイライト色選択 | カラースウォッチをクリックして描画色を切替（自動でハイライトモードに移行） | — |
| コメントピン | クリック位置に色付き丸アイコンを配置してテキスト入力・保存 | `showCommentPopup`, `updateCommentPins` |
| コメント色選択 | コメント用カラースウォッチ（オレンジ/緑/青/赤）で色を選択 | — |
| コメント表示 | ピンをクリックするとコメントテキストをポップアップ表示、再クリックまたは外側クリックで閉じる | `updateCommentPins`, `closeCommentViewPopup` |
| 付箋（スティッキーノート） | クリック位置にドラッグ移動・リサイズ可能な付箋を配置、テキスト入力可能 | `placeStickyNote`, `renderStickyNotes`, `createStickyNoteEl` |
| 付箋色選択 | 付箋用カラースウォッチ（黄/緑/ピンク/青）で色を選択 | — |
| 消しゴム | ハイライト矩形内・コメントピン近傍・付箋領域内をクリックで個別削除 | `eraseAt` |
| ページ消去 | 「このページを消去」で当該ページの全注釈を一括削除 | — |
| 選択モード | 注釈を操作しない通常閲覧モード | `setAnnoTool` |
| 注釈の永続化 | localStorage に JSON 形式で保存 | `saveAnnoStore`, `loadAnnoStore` |
| ズーム連動 | ズーム倍率に合わせて注釈の位置・サイズを再スケーリング | `redrawAnnoOverlay` |

### 7. キーボードショートカット（モーダル内のみ有効）

| キー | 動作 |
|------|------|
| ← / ↑ | 前のページへ |
| → / ↓ | 次のページへ |
| Escape | コメントポップアップを閉じる / モーダルを閉じる |
| Ctrl + + | ズームイン |
| Ctrl + - | ズームアウト |
| h | ハイライトツール |
| c | コメントツール |
| e | 消しゴムツール |
| s | 選択モード |
| n | 付箋ツール |

---

## データ永続化

| 対象 | 保存先 | キー |
|------|--------|------|
| 前回フォルダハンドル | IndexedDB (`pdf-viewer-db`, store: `handles`) | `lastDir` |
| 注釈データ | localStorage | `pdf_annotations` |

**注釈座標系：** x, y, w, h はすべて 0.0〜1.0 の比率で保存。レンダリング時にキャンバスサイズを乗算して画素座標に変換。これにより、ズームや表示サイズが変わっても再計算不要。

注釈データ構造：
```json
{
  "filename.pdf": {
    "1": [
      { "type": "highlight", "x": 0.1, "y": 0.2, "w": 0.3, "h": 0.05, "color": "#fef08a" },
      { "type": "comment",   "x": 0.5, "y": 0.4, "text": "メモ", "color": "#f59e0b" },
      { "type": "sticky",    "x": 0.6, "y": 0.1, "w": 0.22, "h": 0.15, "color": "#fef08a", "text": "付箋メモ" }
    ]
  }
}
```

---

## メニューバー機能（v1.1 追加）

### ファイルメニュー

| メニュー項目 | 実装状況 | 動作 |
|------------|---------|------|
| 新規作成 | ✅ HTML | 現在のビューをクリアして初期状態に戻す |
| ファイルの取り込み | ✅ HTML | ファイル選択ダイアログを開く |
| フォルダの取り込み | ✅ HTML | フォルダ選択ダイアログを開く（ワークスペースに登録） |
| 名前を付けて保存 | ✅ HTML | `showSaveFilePicker` / `<a download>` でPDFをダウンロード保存 |
| 名前の変更 | ✅ HTML | ツリー内の表示名を変更（実ファイルは変更不可） |
| プロパティ | ✅ HTML | ファイル情報ダイアログを表示 |
| スキャナの選択 | ❌ アプリ版 | ブラウザにTWAIN APIなし |
| スキャン開始 | ❌ アプリ版 | ブラウザにTWAIN APIなし |
| プリンタの設定 | ⚠️ 部分対応 | `window.print()` のみ。詳細設定はアプリ版 |
| 印刷 | ✅ HTML | 全画面モーダル経由で印刷ダイアログを開く |

### 編集メニュー

| メニュー項目 | 実装状況 | 動作 |
|------------|---------|------|
| 元に戻す / やり直し | ✅ HTML | 注釈操作のundo/redo（スタック30件） |
| 切り取り / コピー | ⚠️ 部分対応 | アプリ内クリップボードに記録（実ファイル操作は不可） |
| 貼り付け | ⚠️ 部分対応 | クリップボード確認のみ。実際のページ合体はアプリ版 |
| 削除 | ✅ HTML | ツリーから除去（実ファイルは削除不可） |
| 複製 | ✅ HTML | ツリー内に同名＋コピーとして追加 |
| 回転 | ✅ HTML | PDF.jsの`rotation`パラメータで90°/180°/270°回転。`localStorage`に永続化 |
| 束ねる | ❌ アプリ版 | PDF結合はブラウザAPIなし |
| ばらす | ❌ アプリ版 | PDF分割はブラウザAPIなし |
| すべてを選択 | ✅ HTML | サムネイルカードを全選択（視覚ハイライト） |
| 検索 | ✅ HTML | ツリー内のファイル名・フォルダ名で絞り込み検索（タブ切替）、フォルダ右クリックからスコープ指定検索も可 |

### 表示メニュー

| メニュー項目 | 実装状況 | 動作 |
|------------|---------|------|
| サムネールで表示 | ✅ HTML | デフォルト表示モード |
| サムネールサイズ | ✅ HTML | 小(80) / 中(120) / 大(160) / 特大(200)px |
| 整列 | ✅ HTML | 名前昇順・降順でツリーをソート |
| 最新の情報に更新 | ✅ HTML | フォルダを再スキャン（F5） |

### ヘルプメニュー

| メニュー項目 | 実装状況 | 動作 |
|------------|---------|------|
| 操作マニュアル | ✅ HTML | `./manual.html` へリンク＋インライン簡易ガイド |
| GitHub: YueNull | ✅ HTML | `https://github.com/YueNull` を新タブで開く |
| バージョン情報 | ✅ HTML | バージョン・開発者・ライセンス情報ダイアログ |

### キーボードショートカット（グローバル）

| キー | 動作 |
|------|------|
| F2 | 名前の変更 |
| F3 | 検索を開く |
| F5 | 最新の情報に更新 |
| Ctrl+I | プロパティ |
| Ctrl+D | 複製 |
| Ctrl+Z | 元に戻す |
| Ctrl+Y | やり直し |
| Ctrl+P | 印刷 |
| Delete | 選択ファイルを削除 |

### 回転の永続化

```json
// localStorage key: "pdf_rotation"
// 構造: { [filename]: { [pageNum]: 0|90|180|270 } }
{
  "1法人税.pdf": {
    "1": 90,
    "3": 180
  }
}
```

---

## 未実装機能（HTML 不可 → Python 実装が必要）

### J. ネイティブアプリで開く（OS標準プログラムの選択・起動）

ブラウザ版の「プログラムで開く」は blob URL を新タブで開くため、ブラウザが対応しない形式（Excel, Word, 弥生会計 等）はダウンロード扱いになる。OS のデフォルトアプリで直接起動する機能はブラウザ API に存在しない。

**Python 実装方針：**
```python
import subprocess, os

# OS標準のデフォルトアプリで開く
os.startfile(file_path)  # Windows専用

# プログラムを選択して開く（「プログラムで開く」ダイアログ）
subprocess.run(['rundll32', 'shell32.dll,OpenAs_RunDLL', file_path])
```



### A. Brother HL-J7010CDW プリンター連携

`window.print()` はプリンター名を指定できないため HTML では不可。

**Python 実装方針：**
```python
import win32print, win32api
PRINTER_NAME = "Brother HL-J7010CDW"
win32print.SetDefaultPrinter(PRINTER_NAME)
win32api.ShellExecute(0, "print", pdf_path, None, ".", 0)
# または SumatraPDF CLI:
# subprocess.run(["SumatraPDF.exe", "-print-to", PRINTER_NAME, pdf_path])
```

### B. RICOH fi-7700 / fi-7700S スキャナー連携（PaperStream IP TWAIN 3.40.1）

ブラウザに TWAIN API は存在しないため HTML では完全不可。

**Python 実装方針：**
```python
import twain
sm = twain.SourceManager(0)
ss = sm.OpenSource("PaperStream IP fi-7700")
ss.SetCapability(twain.ICAP_XRESOLUTION, twain.TWTY_FIX32, 300.0)
ss.RequestAcquire(0, 0)
rv = ss.XferImageNatively()
```
詳細 → [tool.md](tool.md)

### G. PDF間ページドラッグ移動・結合・保存

ブラウザの File System Access API は PDF バイナリの書き換えをサポートしないため HTML では不可。

**要件：**
- 異なる PDF ファイル間でページを Drag & Drop して順序を変更
- 複数 PDF をページ単位で組み合わせて新しい PDF として保存
- 元の PDF ファイルへの上書き保存

**Python 実装方針：**
```python
from pypdf import PdfReader, PdfWriter

def merge_pages(page_list, output_path):
    """
    page_list: [(pdf_path, page_index), ...] の順序でページを結合
    """
    readers = {}
    writer = PdfWriter()
    for pdf_path, page_idx in page_list:
        if pdf_path not in readers:
            readers[pdf_path] = PdfReader(pdf_path)
        writer.add_page(readers[pdf_path].pages[page_idx])
    with open(output_path, 'wb') as f:
        writer.write(f)
```

**UI 方針（App化時）：**
- サムネイルカードを他 PDF のサムネイルグリッドへドラッグ
- ドロップ位置にページを挿入（プレビューラインで位置を示す）
- 「保存」ボタンで新規 PDF として書き出し or 元ファイルに上書き

---

### H. AI モデル連携（PDF 内容のチャット質問）

ブラウザ HTML 単体では API キー管理・CORS・セキュリティの観点から実用的な実装が困難。Python アプリ化推奨。

**対応モデル方針：**

| 種別 | サービス / モデル | 接続方法 |
|------|-----------------|---------|
| オンライン | **Claude** (Anthropic) | API キー → `anthropic` SDK |
| オンライン | **GPT-4o** (OpenAI) | API キー → `openai` SDK |
| オンライン | **Gemini** (Google) | API キー → `google-generativeai` SDK |
| ローカル | **Ollama** (Gemma, Qwen, DeepSeek, Llama, GLM 等) | `http://localhost:11434/api/chat` (OpenAI互換) |
| ローカル | **LM Studio** | `http://localhost:1234/v1/chat/completions` (OpenAI互換) |

**Python 実装方針：**

```python
# ── ローカルモデル（Ollama / LM Studio は共通 OpenAI 互換エンドポイント）
from openai import OpenAI

# Ollama の場合
client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
# LM Studio の場合
# client = OpenAI(base_url="http://localhost:1234/v1", api_key="lm-studio")

response = client.chat.completions.create(
    model="qwen3:latest",   # ollama pull qwen3 などでダウンロード
    messages=[
        {"role": "system", "content": "以下のPDF内容について質問に答えてください:\n" + pdf_text},
        {"role": "user",   "content": user_question},
    ]
)
print(response.choices[0].message.content)

# ── オンライン Claude（Anthropic SDK）
import anthropic
client = anthropic.Anthropic(api_key="sk-ant-...")
msg = client.messages.create(
    model="claude-opus-4-6",
    max_tokens=2048,
    messages=[{"role":"user","content": pdf_text + "\n\n" + user_question}]
)
print(msg.content[0].text)
```

**PDF テキスト抽出（Python）：**
```python
import fitz  # pymupdf
doc = fitz.open("file.pdf")
pdf_text = "\n".join(page.get_text() for page in doc)
```

**UI レイアウト（App化時）：**

```
┌─────────────────────────────────────────────────────────────┐
│  左サイドバー（ファイルツリー）│  中央（PDF表示）  │  右パネル（AIチャット）│
│                              │                 │                      │
│  📁 フォルダ                  │  [PDF ページ]    │ ┌──モデル選択──────┐  │
│   📄 a.pdf ←──── @引用 ──────┼─────────────────┼─│ Claude ▼  ⚙設定  │  │
│   📄 b.pdf                   │                 │ └──────────────────┘  │
│                              │                 │                      │
│                              │                 │ 📎 a.pdf（p.3 参照）   │
│                              │                 │                      │
│                              │                 │ AI: この页の内容は    │
│                              │                 │ 〇〇について...▋      │
│                              │                 │                      │
│                              │                 │ You: 要約して         │
│                              │                 │ ┌──────────────────┐  │
│                              │                 │ │ @ メッセージ入力  │  │
│                              │                 │ └──────────[送信]─┘  │
└──────────────────────────────┴─────────────────┴──────────────────────┘
```

**ファイル読み込み方式（4種）：**

| 方式 | トリガー | 説明 |
|------|---------|------|
| 現在ページ自動注入 | PDF を開いたとき | 表示中ページのテキストを自動でコンテキストに追加 |
| @ファイル参照 | チャット入力で `@filename.pdf` と入力 | ファイル全体または指定ページ範囲を読み込み |
| 右クリック送信 | サムネイル右クリック →「AI に送る」 | 選択ページ範囲をチャットコンテキストに追加 |
| 画像モード切替 | チャットパネルのトグル | テキスト抽出の代わりにページ画像を送信（スキャン PDF 向け） |

**チャット UI 仕様：**

- **ストリーミング出力**：タイピングエフェクトでリアルタイム表示（`anthropic` SDK の `.stream()` 使用）
- **多ターン会話**：会話履歴を保持して文脈を維持（最大トークン超過時は古い履歴を自動削減）
- **Markdown レンダリング**：AI 回答内の表・コードブロック・リストを正しく表示
- **引用ジャンプ**：AI が「3ページ目」等に言及した場合、クリックでそのページへ移動
- **コンテキスト表示**：チャット上部に現在参照中のファイル名・ページ範囲をバッジ表示
- **会話リセット**：「新しい会話」ボタンで履歴クリア・コンテキスト解除

**モデル設定：**

- モデル選択ドロップダウン（オンライン / ローカル切替）
- API キーは設定画面で入力・アプリデータフォルダに暗号化保存
- Ollama / LM Studio はエンドポイント URL を設定画面でカスタマイズ可能

**Python 実装方針（ストリーミング）：**

```python
# pywebview JS ブリッジ経由でフロントエンドに逐次送信
class API:
    def ask_ai(self, question, file_paths, page_range=None):
        context = self._extract_context(file_paths, page_range)
        client = anthropic.Anthropic(api_key=self.api_key)
        with client.messages.stream(
            model="claude-opus-4-6",
            max_tokens=4096,
            messages=self.history + [{"role": "user", "content": context + "\n\n" + question}]
        ) as stream:
            for text in stream.text_stream:
                window.evaluate_js(f'appendChatChunk({json.dumps(text)})')
        self.history.append({"role": "user", "content": question})
        self.history.append({"role": "assistant", "content": stream.get_final_text()})

    def _extract_context(self, file_paths, page_range):
        import fitz
        parts = []
        for path in file_paths:
            doc = fitz.open(path)
            pages = range(*page_range) if page_range else range(len(doc))
            parts.append("\n".join(doc[i].get_text() for i in pages))
        return "\n\n---\n\n".join(parts)
```

```javascript
// フロントエンド：チャンク受信してリアルタイム追記
function appendChatChunk(text) {
    currentBubble.textContent += text;
    chatPanel.scrollTop = chatPanel.scrollHeight;
}
```

---

### I. OCR 連携（画像 PDF・スキャン文書のテキスト認識）

ブラウザでは Wasm ベースの Tesseract.js が使用可能だが、日本語精度が低いため Python + 専用 OCR エンジン推奨。

**日本語対応 OCR エンジン比較：**

| エンジン | 日本語精度 | ライセンス | 備考 |
|---------|-----------|-----------|------|
| **PaddleOCR** | ★★★★★ | Apache 2.0 | 縦書き対応・GPU加速・最推奨 |
| **EasyOCR** | ★★★★☆ | Apache 2.0 | 導入簡単・GPU省メモリ |
| **Tesseract 5** | ★★★☆☆ | Apache 2.0 | 軽量・横書き向き。`jpn` + `jpn_vert` データが必要 |
| **manga-ocr** | ★★★★★ | MIT | 日本語漫画・縦書き特化（学術論文にも有効） |
| **docTR** | ★★★☆☆ | Apache 2.0 | TensorFlow/PyTorch 両対応 |

**Python 実装方針（PaddleOCR）：**

```python
# pip install paddlepaddle paddleocr pymupdf Pillow
from paddleocr import PaddleOCR
import fitz, numpy as np
from PIL import Image

ocr = PaddleOCR(use_angle_cls=True, lang='japan')

doc = fitz.open("scanned.pdf")
results_all = []
for page in doc:
    mat = fitz.Matrix(2, 2)  # 2x 解像度
    pix = page.get_pixmap(matrix=mat)
    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
    result = ocr.ocr(np.array(img), cls=True)
    results_all.append(result)

# 認識テキスト取得
for page_result in results_all:
    for line in page_result:
        print(line[1][0])  # テキスト文字列
```

**Python 実装方針（manga-ocr / 縦書き文書向け）：**

```python
# pip install manga-ocr Pillow pymupdf
from manga_ocr import MangaOcr
import fitz
from PIL import Image

mocr = MangaOcr()
doc = fitz.open("vertical_text.pdf")
for page in doc:
    pix = page.get_pixmap(matrix=fitz.Matrix(2, 2))
    img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
    text = mocr(img)
    print(text)
```

**UI 方針（App化時）：**
- 「OCR 実行」ボタンでページ画像を選択したエンジンに渡す
- 認識結果をテキストレイヤーとして PDF に埋め込み（テキスト選択・コピー可能にする）
- OCR エンジンは設定画面でドロップダウン選択
