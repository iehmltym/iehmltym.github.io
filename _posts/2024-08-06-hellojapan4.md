---
layout: post
title: "日本語学習(4)：システムの開発"
author: "iehmltym（張）"
date: 2024-08-06
header-style: text
catalog: true
tags:
    - ソフトウェアテスト
    - プロジェクト管理
    - システム開発
    - 品質管理
---

> この記事は、Youtubeで独学時のプロジェクト管理と品質保証の観点から重要なポイントを網羅しています。

## 1. テストケースの概要

今回のテストケースは、一覧の画面に関するものです。主な焦点は以下の点です：

- 複数ページがある場合の表示と遷移
- 検索機能の動作確認
- ページ間の遷移と情報の保持
- 作業完了プロセスの確認

テストケース番号は1番から10番までが新規追加分です。

## 2. 主要なテストケース（7番-10番）の詳細

### 7番：初期表示の確認
- 確認項目：
  - 契約検索ボタンが表示されること
  - ページリンクが表示されること
  - 1ページあたりの最大件数が画面に表示されること

### 8番：ページ遷移の確認
- 操作：初期画面でページリンクをクリック
- 確認項目：
  - 次のページに遷移できること
  - 遷移先の画面に正しい契約番号が表示されること

### 9番：前の画面への戻り確認
- 操作：前の画面に戻るボタンをクリック
- 確認項目：
  - 契約で初段検索の画面に戻れること
  - 契約番号が表示されていたページ番号に戻ること
  - ページリンクが表示されること
  - 1ページあたりの表示件数が正しいこと

### 51番：確認後の画面遷移確認
- 操作：契約登録確認画面で確認ボタンをクリック
- 確認項目：
  - 契約・席替えで契約検索画面に遷移できること
  - 契約番号が表示されていたページ番号に戻ること
  - ページリンクが表示されること
  - 1ページあたりの表示件数が正しいこと

## 3. テスト実施上の注意点

1. ページ番号の指定：
   - テストは1ページ以上のページ番号で実施すること

2. 操作の連続性：
   - 50番と51番のテストケースは連続した操作であることに注意

3. 画面遷移の明確化：
   - 各テストケースで、開始画面と遷移先画面を明確に記述すること

4. 既存テストケースとの整合性：
   - 9番から13番のテストケースは既存のものであり、内容の重複に注意

5. 新規テストケースの確認：
   - 14番以降の新規テストケースの内容を確認し、必要に応じて調整

## 4. テスト仕様書作成のポイント

1. 明確な指示：
   - 各テストケースの操作手順を詳細に記述すること

2. 前提条件の明記：
   - テスト開始時の画面状態や必要なデータを明確に記述

3. 確認項目の具体化：
   - 何を確認するのか、具体的な項目を列挙すること

4. 連続操作の明示：
   - 複数の画面を跨ぐテストケースでは、操作の流れを明確に示すこと

5. エビデンスの取得方法：
   - 各確認項目に対するエビデンスの取得方法を明記

## 5. プロジェクト管理上の留意点

1. テストケースの追加と修正：
   - 新規追加されたテストケース（47番-60番）の意図を明確に理解すること
   - 既存のテストケースとの整合性を確認

2. チーム内のコミュニケーション：
   - テスト仕様書の記述方法について、チーム内で統一した理解を持つこと
   - 不明点があれば、即座に確認し解決すること

3. 品質管理：
   - 各テストケースが適切に設計され、システムの品質を確保できているか確認
   - テスト結果の分析と、必要に応じた改善提案

4. ドキュメンテーション：
   - テスト仕様書の記述を明確かつ詳細に行い、後の参照や修正が容易になるようにすること

## まとめ

ソフトウェアテストは、システム開発において極めて重要なプロセスです。今回の契約管理システムのテストケース設計と実施を通じて、以下の点が重要であることが再確認されました：

1. テストケースの詳細な記述
2. 操作手順と確認項目の明確化
3. テスト環境と前提条件の明記
4. チーム内でのコミュニケーションと理解の統一
5. 継続的な改善とフィードバックの重要性

これらの点に注意を払いながら、高品質なソフトウェア開発を目指していきましょう。テストプロセスの改善は、プロジェクト全体の成功に直結します。今後も、チーム全体で品質向上に取り組んでいきましょう。

---

著者：張（ちょう）- ソフトウェアテストエンジニア・プロジェクトマネージャー
