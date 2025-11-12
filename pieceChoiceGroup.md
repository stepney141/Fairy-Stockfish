# pieceChoiceGroup 設定構文リファレンス

## 概要

`pieceChoiceGroup`は、セットアップフェーズで使用できる駒の種類を制限し、プレイヤーに選択を強制するためのFairy-Stockfishの設定オプションです。66shogiなどの配置型バリアントで使用されます。

## 基本構文

```ini
pieceChoiceGroup = options=<駒文字列> limit=<整数> [required=<真偽値>] [lockUnused=<真偽値>]
pieceChoiceGroupWhite = options=<駒文字列> limit=<整数> [required=<真偽値>] [lockUnused=<真偽値>]
pieceChoiceGroupBlack = options=<駒文字列> limit=<整数> [required=<真偽値>] [lockUnused=<真偽値>]
```

- **pieceChoiceGroup**: 両プレイヤーに適用
- **pieceChoiceGroupWhite**: 白プレイヤーのみ
- **pieceChoiceGroupBlack**: 黒プレイヤーのみ

## パラメータ詳細

### 必須パラメータ

#### `options=<駒文字列>` **【必須】**
選択可能な駒の種類を指定します。

- **形式**: 駒タイプの1文字コードを連結した文字列
- **文字マッピング**: `pieceToChar`で定義された文字（大文字・小文字どちらでも可）
  - 標準的な例: `p`=Pawn, `n`=Knight, `b`=Bishop, `r`=Rook, `q`=Queen, `k`=King
  - 将棋: `g`=Gold, `s`=Silver, `l`=Lance, `n`=Knight
- **特殊文字**: `*` = すべての駒タイプ（ワイルドカード）
- **例**:
  - `options=rb` → RookとBishop
  - `options=nsg` → Knight、Silver、Gold
  - `options=*` → すべての駒

**検証**: 認識できない文字が含まれる場合、そのグループエントリは無効となり無視されます。

#### `limit=<整数>` **【必須】**
選択グループから使用できる駒の最大数を指定します。

- **形式**: 正の整数
- **制約**: `limit > 0` でなければならない（0以下の場合、グループは無効）
- **例**:
  - `limit=1` → グループから1個だけ選択可能
  - `limit=2` → グループから最大2個まで

### オプションパラメータ

#### `required=<真偽値>` 【オプション】
セットアップフェーズでこのグループの枠を満たす必要があるかを指定します。

- **別名**: `requiredForSetup`（どちらも使用可能）
- **形式**: `true`, `1` (真) / その他 (偽)
- **デフォルト**: `false`
- **動作**:
  - `true`の場合: セットアップフェーズ中、このグループの`limit`まで駒を配置しなければならない
  - 制約は`setupDropsActive`が有効な間のみ適用される

#### `lockUnused=<真偽値>` 【オプション】
枠を使い切った後、一度も使われなかった駒を永久に使用禁止にするかを指定します。

- **別名**: `lockUnusedOptions`, `banUnused`（すべて使用可能）
- **形式**: `true`, `1` (真) / その他 (偽)
- **デフォルト**: `false`
- **動作**:
  - `true`の場合: `limit`に達した後、一度も使われていない駒タイプは、取られて持ち駒に戻っても打てなくなる
  - `false`の場合: 枠の制限のみ（取られた駒は再利用可能）

**実装箇所**: [src/position.h:1685-1691](src/position.h#L1685-L1691)

## 複数グループの指定

セミコロン（`;`）で区切って複数のグループを定義できます：

```ini
pieceChoiceGroup = options=rb limit=1 required=true; options=nsg limit=2
```

**制限**: 各色につき最大8グループまで（`PieceChoiceGroupMax = 8`）

## 実例

### 66shogi の設定

```ini
pieceChoiceGroup = options=rb limit=1 required=true lockUnused=true
```

**解説**:
- RookとBishopから1つだけ選択
- セットアップフェーズで必ず選ばなければならない
- 選ばなかった方の駒は終局まで使用不可

### 仮想例：非対称な設定

```ini
pieceChoiceGroupWhite = options=qr limit=1 lockUnused=true
pieceChoiceGroupBlack = options=bn limit=1 lockUnused=true
```

**解説**:
- 白はQueenかRookを1つ選択、未使用の方は使用禁止
- 黒はBishopかKnightを1つ選択、未使用の方は使用禁止

### 複雑な例：多段階選択

```ini
pieceChoiceGroup = options=qr limit=1 required=true lockUnused=true; options=bn limit=2 required=true
```

**解説**:
- 第1グループ: QueenかRookから1つ、未選択の方は禁止
- 第2グループ: BishopとKnightから合計2つまで（両方選択可、ロック無し）

## 関連する設定オプション

pieceChoiceGroupを使用するには、セットアップフェーズ関連の設定も必要です：

```ini
setupDrops = true                    # セットアップドロップを有効化
setupMustDrop = true                 # パス不可（ドロップ強制）
setupDropRegionWhite = *1            # 白の配置可能領域
setupDropRegionBlack = *6            # 黒の配置可能領域
setupDropPieceTypes = kgsnl          # 配置可能な駒タイプ
pocketSize = 7                       # 持ち駒のサイズ
startFen = 6[KGSNLRBkgsnlrb]         # 初期配置（持ち駒含む）
```

## 動作メカニズム

### 使用状況の追跡

各グループについて、以下が追跡されます（[src/position.h:60-61](src/position.h#L60-L61)）：

- **choiceGroupUsage**: 使用された駒の数
- **choiceGroupUsedTypes**: 使用された駒タイプのビットセット

### 打つ手の検証

駒を打つ際、[src/position.h:1671-1694](src/position.h#L1671-L1694)の`choice_group_allows_drop()`で検証：

1. グループの`limit`に達しているか確認
2. `lockUnusedOptions=true`の場合:
   - その駒タイプが以前に使われたか確認
   - 未使用 + 枠が埋まっている → **打てない**

### セットアップ終了時のクリーンアップ

セットアップフェーズ終了時（[src/position.cpp:604-633](src/position.cpp#L604-L633)）：

- `required=true`かつ`limit`に達したグループについて
- グループの全オプション駒を持ち駒から物理的に削除
- これにより未使用駒の復活を防止

## エラー処理

### パーサーエラー（DoCheck=true時）

1. **無効な駒文字**: "Invalid pieceChoiceGroup entry"
2. **limit≤0またはoptions空**: 同上
3. **グループ数超過**: "Maximum number of piece choice groups exceeded"

### 実行時

- 無効な設定は静かに無視される
- 手生成は設定されたグループを尊重する

## デフォルト値

パラメータを省略した場合（[src/variant.h:38-43](src/variant.h#L38-L43)）：

- `options`: `NO_PIECE_SET`（空セット、**指定必須**）
- `limit`: `0`（**正の値が必須**）
- `requiredForSetup`: `false`
- `lockUnusedOptions`: `false`

## 技術仕様

- **実装ファイル**:
  - 構造体定義: [src/variant.h:36-43](src/variant.h#L36-L43)
  - パーサー: [src/parser.cpp:243-329](src/parser.cpp#L243-L329)
  - 検証ロジック: [src/position.h:1671-1694](src/position.h#L1671-L1694)
  - 使用追跡: [src/position.cpp:586-602](src/position.cpp#L586-L602)
- **データ型**: PieceSet (uint64_tビットマスク)
- **制限**: 各色8グループまで

## 検証結果

### 66shogiルールの適合性

pieceChoiceGroupの実装は、66shogiの正式ルール「選ばなかった方のrook/bishopは終局まで一切使用禁止」を正しく実装しています：

1. **主要防御**: `lockUnusedOptions`ロジックにより、未使用駒タイプは`limit`到達後永久に打てない
2. **副次防御**: セットアップ終了時に未使用駒を持ち駒から物理的に削除
3. **包括的統合**: すべての駒打ちパス（手生成、合法性チェック等）で検証を実施
4. **状態永続性**: `choiceGroupUsedTypes`が状態管理に組み込まれ、undo/redo操作でも整合性を維持

---

この構文リファレンスは、Fairy-Stockfishのソースコード分析に基づいて作成されました。現在、66shogiバリアントのみがこの機能を使用していますが、他の配置型バリアントにも適用可能な汎用的な設計になっています。
