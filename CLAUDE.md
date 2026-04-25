# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

CHAINOVA は `index.html` 単一ファイルで完結するブラウザ動作のパズルゲームです。ビルドツール・依存ライブラリ・サーバー不要で、ファイルをブラウザで直接開くだけで動作します。

## 起動方法

```
start index.html        # Windows（デフォルトブラウザで開く）
open index.html         # macOS
```

テスト・ビルド・lint コマンドは存在しません。動作確認はブラウザで直接行います。

## アーキテクチャ

`index.html` は CSS → HTML → JS の順で1ファイルに記述されており、外部ファイルへの依存は一切ありません。

### ゲームロジックの流れ

```
handleClick(r, c)          ← タイルクリック（選択1回目）
  └→ attemptSwap(r1,c1,r2,c2)  ← 隣接タイルクリック（選択2回目）
        └→ swapState()          ← グリッド状態を交換
        └→ findAllGroups()      ← 3個以上の連結グループを検索（BFS）
        └→ explodeWave()        ← ドミノ爆発アニメーション + スコア計算
              └→ applyGravityFill()  ← 重力落下 + 新タイル補充
              └→ findAllGroups()     ← 自動連鎖チェック（再帰）
```

### 状態管理

グローバルオブジェクト `G` が全ゲーム状態を保持します：
- `G.grid[r][c]` — タイルオブジェクト `{color, type, special}` または `null`
- `G.busy` — アニメーション中フラグ（入力ブロック用）
- `G.combo / G.maxCombo` — 現在・最大コンボインデックス（`CHAIN_M[]` 参照）
- `selR / selC` — 選択中タイル座標（`-1` = 未選択）

### 主要関数

| 関数 | 役割 |
|------|------|
| `findGroup(r,c)` | BFS で連結同色グループを返す（wild タイルは全色に接続） |
| `findAllGroups()` | ボード全体から有効グループ（3個以上）を列挙 |
| `hasValidMove()` | 全隣接ペアをスワップ試行して有効手があるか判定 |
| `collectCells(group)` | Bomb（半径2）・Rainbow（同色全消し）の爆発範囲を展開 |
| `explodeWave(group, depth, done)` | ドミノ爆発 → 自動連鎖ループの中心関数 |
| `stuckRatio()` | 消せないタイルの割合（警告・LAST CHANCE 判定に使用） |

### 定数

- `CHAIN_M = [1,3,8,20,50,100,200,400]` — 連鎖深度ごとのスコア倍率
- `FEVER_D = 6` — FEVER 発動の連鎖深度（0 始まりで 7 連鎖目）
- `BASE_DELAY = 60` — ドミノ爆発の1タイルあたり遅延（ms）、グループ数で動的に短縮
- `MAX_DEPTH = 18` — 連鎖の最大深度（無限ループ防止）

### 音声

Web Audio API で全音を生成（外部ファイルなし）。`initAudio()` は最初のユーザー操作時に呼び出し（ブラウザ制限対応）。

### データ永続化

`localStorage` のみ使用：
- `cn_best` — ハイスコア
- `cn_tut` — チュートリアル表示済みフラグ
