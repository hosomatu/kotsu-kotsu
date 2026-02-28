# Implementation Plan: こつこつ - 習慣トラッキングアプリ

## Overview

React Native + Expoプロジェクトを構築し、サービス層→カスタムフック→UIコンポーネントの順にボトムアップで実装する。各ステップでテストを組み込み、最後に全体を統合する。

## Tasks

- [ ] 1. プロジェクトセットアップとデータモデル定義
  - [ ] 1.1 Expoプロジェクトの初期化と依存関係のインストール
    - `npx create-expo-app`でプロジェクト作成
    - `uuid`, `@react-native-async-storage/async-storage`, `expo-sharing` をインストール
    - Jest, React Native Testing Library, fast-check をdev依存に追加
    - _Requirements: 全体_

  - [ ] 1.2 データモデルとバリデーション関数の作成
    - `src/types/habit.ts` に `Habit` インターフェースを定義（id, name, count）
    - `src/utils/validation.ts` にバリデーション関数を実装（空白チェック、件数上限チェック）
    - _Requirements: 1.3, 3.4, 4.4_

- [ ] 2. StorageServiceの実装
  - [ ] 2.1 StorageServiceの作成
    - `src/services/storageService.ts` を作成
    - `loadHabits`: AsyncStorageから `@kotsu-kotsu/habits` キーでデータ読み込み、JSON.parse
    - `saveHabits`: 習慣配列をJSON.stringifyしてAsyncStorageに保存
    - 読み込み失敗時は空配列を返す、保存失敗時はエラーをthrow
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5_

  - [ ]* 2.2 Property 1: ストレージラウンドトリップのプロパティテスト
    - **Property 1: ストレージラウンドトリップ**
    - fast-checkで有効なHabit配列を生成し、saveHabits→loadHabitsの等価性を検証
    - **Validates: Requirements 1.1, 6.1, 6.2**

- [ ] 3. ShareServiceの実装
  - [ ] 3.1 ShareServiceの作成
    - `src/services/shareService.ts` を作成
    - `formatShareText`: 習慣配列からテキスト形式の共有文字列を生成
    - `share`: `expo-sharing`を使用して共有シートを呼び出し
    - _Requirements: 5.2, 5.3_

  - [ ]* 3.2 Property 8: 共有テキストに全項目含有のプロパティテスト
    - **Property 8: 共有テキストに全項目含有**
    - fast-checkで任意のHabit配列を生成し、formatShareTextの出力に全項目の名前とカウントが含まれることを検証
    - **Validates: Requirements 5.2**

- [ ] 4. useHabitsフックの実装
  - [ ] 4.1 useHabitsカスタムフックの作成
    - `src/hooks/useHabits.ts` を作成
    - 初期化時にStorageServiceからデータ読み込み（loading, error状態管理）
    - `addHabit`: バリデーション→UUID生成→count:0で追加→保存
    - `incrementCount`: 対象項目のcount+1→保存
    - `updateHabit`: バリデーション→名前更新（count維持）→保存
    - `deleteHabit`: 対象項目削除→保存
    - `canAddHabit`: habits.length < 10
    - 各操作後にStorageServiceで永続化、失敗時はerror状態を更新
    - _Requirements: 1.1, 2.1, 2.2, 2.3, 3.3, 3.4, 3.5, 4.3, 4.4, 4.6, 6.3, 6.4, 6.5_

  - [ ]* 4.2 Property 3: カウント増加の正確性のプロパティテスト
    - **Property 3: カウント増加の正確性**
    - incrementCount後に対象のcountが+1、他項目は不変であることを検証
    - **Validates: Requirements 2.1, 2.2**

  - [ ]* 4.3 Property 4: 習慣追加でリスト成長のプロパティテスト
    - **Property 4: 習慣追加でリスト成長**
    - 10件未満のリストに有効な名前でaddHabit後、件数+1かつcount=0を検証
    - **Validates: Requirements 3.3**

  - [ ]* 4.4 Property 5: 空白名バリデーションのプロパティテスト
    - **Property 5: 空白名バリデーション**
    - 空白文字列でaddHabit/updateHabitが拒否され、リストが不変であることを検証
    - **Validates: Requirements 3.4, 4.4**

  - [ ]* 4.5 Property 6: 編集時カウント維持のプロパティテスト
    - **Property 6: 編集時カウント維持**
    - updateHabit後に名前が更新されcountが不変であることを検証
    - **Validates: Requirements 4.3**

  - [ ]* 4.6 Property 7: 削除後の除外のプロパティテスト
    - **Property 7: 削除後の除外**
    - deleteHabit後に対象がリストに含まれず件数が-1であることを検証
    - **Validates: Requirements 4.6**

  - [ ]* 4.7 Property 2: 最大10件制限の不変条件のプロパティテスト
    - **Property 2: 最大10件制限の不変条件**
    - 任意の操作列（追加・編集・削除・カウント更新）後にリスト件数が10以下であることを検証
    - **Validates: Requirements 1.3, 3.5**

  - [ ]* 4.8 Property 9: 操作後の永続化一貫性のプロパティテスト
    - **Property 9: 操作後の永続化一貫性**
    - 各操作後にAsyncStorageの保存データとメモリ上のリストが一致することを検証
    - **Validates: Requirements 2.3, 6.3**

- [ ] 5. チェックポイント - サービス層とフックの検証
  - Ensure all tests pass, ask the user if questions arise.

- [ ] 6. UIコンポーネントの実装
  - [ ] 6.1 Headerコンポーネントの作成
    - `src/components/Header.tsx` を作成
    - アプリタイトル「こつこつ」を表示
    - 追加ボタン（右上）と共有ボタンを配置
    - `canAddHabit`がfalseの時、追加ボタンを無効化
    - 習慣が0件の時、共有ボタンを無効化
    - _Requirements: 3.1, 3.5, 5.1, 5.4_

  - [ ] 6.2 EmptyStateコンポーネントの作成
    - `src/components/EmptyState.tsx` を作成
    - 習慣が0件の時に表示する空状態メッセージを実装
    - _Requirements: 1.4_

  - [ ] 6.3 HabitItemコンポーネントの作成
    - `src/components/HabitItem.tsx` を作成
    - 習慣名と累積カウントを表示
    - タップで`onTap`、長押しで`onLongPress`を呼び出し
    - _Requirements: 1.2, 2.1, 4.1_

  - [ ] 6.4 CreateModalコンポーネントの作成
    - `src/components/CreateModal.tsx` を作成
    - React Nativeの`Modal`コンポーネントを使用
    - 習慣名入力フィールド、保存ボタン、キャンセルボタン
    - 空白のみの入力時にインラインエラーメッセージを表示
    - _Requirements: 3.2, 3.3, 3.4_

  - [ ] 6.5 EditModalコンポーネントの作成
    - `src/components/EditModal.tsx` を作成
    - 現在の習慣名をプリセット表示
    - 名前編集、保存ボタン、削除ボタン、キャンセルボタン
    - 削除タップ時にAlert.alertで確認ダイアログを表示
    - 空白のみの入力時にインラインエラーメッセージを表示
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7_

  - [ ]* 6.6 UIコンポーネントのユニットテスト
    - EmptyStateの表示テスト（要件1.4）
    - Headerの追加ボタン・共有ボタンの有効/無効テスト（要件3.1, 3.5, 5.1, 5.4）
    - HabitItemのタップ・長押しテスト（要件2.1, 4.1）
    - CreateModalのバリデーションエラー表示テスト（要件3.4）
    - EditModalの名前プリセット・削除確認テスト（要件4.2, 4.5, 4.7）
    - _Requirements: 1.4, 2.1, 3.1, 3.4, 3.5, 4.1, 4.2, 4.5, 4.7, 5.1, 5.4_

- [ ] 7. 画面統合とApp.tsx
  - [ ] 7.1 HabitListScreenの作成
    - `src/screens/HabitListScreen.tsx` を作成
    - `useHabits`フックを使用してデータ取得・操作
    - Header, HabitItem（FlatList）, EmptyState, CreateModal, EditModalを統合
    - モーダルの表示/非表示状態管理（useState）
    - ローディング状態・エラー状態の表示
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 2.1, 2.2, 3.1, 3.2, 4.1, 5.1_

  - [ ] 7.2 App.tsxの更新
    - `App.tsx` で `HabitListScreen` をレンダリング
    - _Requirements: 全体_

  - [ ]* 7.3 統合テスト
    - アプリ起動時のデータ読み込みと表示テスト（要件1.1, 6.2）
    - 習慣追加→リスト表示→カウント増加の一連フローテスト（要件2.1, 3.3）
    - ストレージ読み込み失敗時の空リスト起動テスト（要件6.4）
    - ストレージ保存失敗時のエラー表示テスト（要件6.5）
    - _Requirements: 1.1, 2.1, 3.3, 6.2, 6.4, 6.5_

- [ ] 8. 最終チェックポイント - 全テスト実行と最終確認
  - Ensure all tests pass, ask the user if questions arise.

## Notes

- `*` マーク付きタスクはオプションで、MVP優先時はスキップ可能
- 各タスクは要件番号で追跡可能
- プロパティテストはfast-checkで最低100回イテレーション実行
- ユニットテストとプロパティテストは相補的に使用
