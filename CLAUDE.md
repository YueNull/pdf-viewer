# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 概要

単一 HTML ファイルの PDF ビューワー。外部依存は PDF.js CDN のみ。  
動作ブラウザ: **Chrome / Edge のみ**（`showDirectoryPicker` API 使用）

## ファイル構成

| ファイル | 役割 |
|---------|------|
| `pdf-viewer.html` | アプリ本体（HTML + CSS + JS、全 1 ファイル） |
| [Function.md](Function.md) | 機能仕様一覧・未実装機能の Python 実装方針 |
| [tool.md](tool.md) | 使用ツール・ライブラリ詳細（現行 + Python 化時） |

## アーキテクチャ要点

- フォルダを開くと `scanDirRecursiveAll` で全 PDF を再帰スキャン → `loadFolderView` で右ペインに全ページ連続表示
- サムネイルは `IntersectionObserver` で遅延レンダリング（`renderThumb`）
- 全画面表示: `openModal` → `renderModalPage`（PDF.js canvas レンダリング）
- 注釈座標は 0〜1 の比率で localStorage 保存（キー: `pdf_annotations`）
- 前回フォルダハンドルは IndexedDB `pdf-viewer-db` に保存
