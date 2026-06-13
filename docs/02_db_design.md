# DB 設計書（LocalStorage JSON 構造）

**文書番号:** WLP-002  
**プロジェクト名:** Warehouse Layout Planner  
**バージョン:** 2.0（Phase 1〜6 対応）  
**作成日:** 2026-06-13  
**対象読者:** 開発担当者  

---

## 目次

1. [ストレージ概要](#1-ストレージ概要)
2. [スキーマバージョン管理](#2-スキーマバージョン管理)
3. [トップレベル構造](#3-トップレベル構造)
4. [エンティティ定義](#4-エンティティ定義)
5. [マスタデータ定義](#5-マスタデータ定義)
6. [サンプルデータ](#6-サンプルデータ)
7. [マイグレーション定義](#7-マイグレーション定義)
8. [ストレージ容量試算](#8-ストレージ容量試算)
9. [IndexedDB 移行方針（Phase 9）](#9-indexeddb-移行方針phase-9)

---

## 1. ストレージ概要

| 項目 | 内容 |
|------|------|
| ストレージ種別 | Browser LocalStorage |
| ストレージキー | `warehousePlannerData` |
| シリアライズ形式 | JSON（UTF-8） |
| 最大容量 | 5MB（ブラウザ制限） |
| 自動保存 | 変更から 500ms デバウンス後に自動保存 |
| 手動保存 | ツールバー「保存」ボタン |
| バックアップ | JSON エクスポート（ダウンロード） |

### 1.1 ストレージキー一覧

| キー | 内容 | 更新タイミング |
|------|------|--------------|
| `warehousePlannerData` | メインレイアウトデータ | 操作都度（デバウンス） |
| `warehousePlannerSettings` | ユーザー設定（表示設定等） | 設定変更時 |
| `warehousePlannerProposal` | 提案書データ（Phase 5） | 提案書保存時 |
| `warehousePlannerTemplates` | レイアウトテンプレート一覧（Phase 9） | テンプレート登録時 |

---

## 2. スキーマバージョン管理

```
バージョン履歴:
v1: 初期リリース（ラック・障害物・倉庫設定）
v2: Phase 1 - ラックパラメータ拡張・複数選択
v3: Phase 2 - レイヤー管理追加
v4: Phase 4 - ゾーン管理追加
v5: Phase 5 - 提案書データ追加
```

ロード時に `version` フィールドを確認し、古いバージョンのデータは自動マイグレーションを実行する。

---

## 3. トップレベル構造

```jsonc
{
  // === メタ情報 ===
  "version": 4,                    // スキーマバージョン (number)
  "savedAt": "2026-06-13T10:30:00Z", // 保存日時 ISO 8601 (string)
  "appVersion": "2.0.0",           // アプリバージョン (string)

  // === 倉庫設定 ===
  "warehouse": { /* → 4.1 参照 */ },

  // === 配置データ ===
  "items": [ /* → 4.2 参照 */ ],
  "obstacles": [ /* → 4.3 参照 */ ],
  "zones": [ /* → 4.4 参照 (Phase 4) */ ],
  "layers": [ /* → 4.5 参照 (Phase 2) */ ],

  // === UI 状態 ===
  "viewSettings": { /* → 4.6 参照 */ },

  // === ID 管理 ===
  "nextId": 1001                   // 次回採番 ID (number)
}
```

---

## 4. エンティティ定義

### 4.1 warehouse（倉庫設定）

```jsonc
"warehouse": {
  "name": "物流センター A棟",       // 倉庫名称 (string, max: 100文字)
  "width": 60000,                  // 倉庫幅 mm (number, 1000〜500000)
  "depth": 40000,                  // 倉庫奥行 mm (number, 1000〜500000)
  "height": 6000,                  // 天井高 mm (number, 2000〜30000)
  "floorLoadCapacity": 1500,       // 床荷重許容値 kg/m² (number) ← Phase 3 追加
  "temperatureZone": "ambient",    // 温度帯: ambient/chilled/frozen (string) ← Phase 6 追加
  "buildingType": "steel_frame"    // 建物種別: steel_frame/rc/wood (string) ← Phase 6 追加
}
```

| フィールド | 型 | 制約 | デフォルト | 説明 |
|-----------|---|-----|---------|------|
| name | string | max 100文字 | "新規倉庫" | 倉庫・センター名称 |
| width | number | 1,000〜500,000 | 30,000 | 倉庫幅（mm） |
| depth | number | 1,000〜500,000 | 20,000 | 倉庫奥行（mm） |
| height | number | 2,000〜30,000 | 6,000 | 天井高（mm） |
| floorLoadCapacity | number | 100〜20,000 | 1,500 | 床荷重許容値（kg/m²） |
| temperatureZone | string | enum | "ambient" | 主温度帯 |
| buildingType | string | enum | "steel_frame" | 建物種別 |

---

### 4.2 items（ラック配置オブジェクト）

```jsonc
"items": [
  {
    // === 識別子 ===
    "id": 101,                     // 一意 ID (number, auto increment)
    "type": "pallet",              // ラックタイプキー (string, → 5.1 参照)

    // === 位置・サイズ ===
    "x": 2000,                     // 左上 X 座標 mm (number, 0〜warehouse.width)
    "y": 3000,                     // 左上 Y 座標 mm (number, 0〜warehouse.depth)
    "width": 2700,                 // 幅 mm (number, 100〜50000)
    "depth": 1100,                 // 奥行 mm (number, 100〜50000)
    "rotation": 0,                 // 回転角度 deg (0/90/180/270)

    // === 収容量パラメータ（Phase 1 追加） ===
    "params": {
      "bays": 2,                   // 間口数（パレットラック専用）
      "levels": 4,                 // 棚段数（GL 含む）
      "shelves": 5,                // 棚板数（軽量棚専用）
      "divisionsPerShelf": 3,      // 棚段あたり区画数（軽量棚専用）
      "stackLevels": 4,            // 段積み数（ネスティング専用）
      "spotsPerUnit": 1,           // 1 連あたりスポット数（ネスティング専用）
      "palletWidth": 1100,         // 対象パレット幅 mm
      "palletDepth": 1100,         // 対象パレット奥行 mm
      "maxLoadPerPallet": 1000     // パレット最大積載 kg
    },

    // === レイヤー・ゾーン帰属（Phase 2/4 追加） ===
    "layerId": "layer_rack",       // 所属レイヤー ID (string)
    "zoneId": null,                // 所属ゾーン ID (number|null)

    // === 表示属性 ===
    "label": "",                   // カスタムラベル (string, max: 50文字)
    "color": null,                 // カスタム色 (string|null, CSS color)

    // === メタ情報 ===
    "count": 1,                    // 数量（集計用） (number, 1〜9999)
    "note": ""                     // メモ (string, max: 200文字)
  }
]
```

| フィールド | 型 | 必須 | 説明 |
|-----------|---|-----|------|
| id | number | ○ | 一意識別子（自動採番） |
| type | string | ○ | ラックタイプキー（masters のキーと一致） |
| x | number | ○ | 左上隅 X 座標（mm） |
| y | number | ○ | 左上隅 Y 座標（mm） |
| width | number | ○ | 幅（mm）（マスタから引き継ぎ、カスタム変更可） |
| depth | number | ○ | 奥行（mm）（同上） |
| rotation | number | ○ | 回転角度（0/90/180/270） |
| params | object | △ | 収容量詳細パラメータ（Phase 1） |
| layerId | string | △ | 所属レイヤー ID（Phase 2） |
| zoneId | number\|null | △ | 所属ゾーン ID（Phase 4） |
| label | string | - | カスタムラベル |
| color | string\|null | - | カスタム背景色 |
| count | number | ○ | 数量（デフォルト: 1） |
| note | string | - | 備考メモ |

---

### 4.3 obstacles（障害物配置オブジェクト）

```jsonc
"obstacles": [
  {
    // === 識別子 ===
    "id": 201,                     // 一意 ID (number)
    "type": "pillar",              // 障害物タイプキー (string, → 5.2 参照)

    // === 位置・サイズ ===
    "x": 5000,                     // 左上 X 座標 mm (number)
    "y": 5000,                     // 左上 Y 座標 mm (number)
    "width": 500,                  // 幅 mm (number)
    "depth": 500,                  // 奥行 mm (number)
    "rotation": 0,                 // 回転角度 deg (0/90/180/270)

    // === 保全設定（Phase 2 追加） ===
    "clearance": 0,                // 保全距離 mm（マスタのデフォルト値から上書き可）

    // === レイヤー帰属 ===
    "layerId": "layer_obstacle",   // 所属レイヤー ID (string)

    // === 表示属性 ===
    "label": "P-01",               // カスタムラベル (string)
    "note": ""                     // メモ (string)
  }
]
```

| フィールド | 型 | 必須 | 説明 |
|-----------|---|-----|------|
| id | number | ○ | 一意識別子 |
| type | string | ○ | 障害物タイプキー |
| x | number | ○ | 左上隅 X 座標（mm） |
| y | number | ○ | 左上隅 Y 座標（mm） |
| width | number | ○ | 幅（mm） |
| depth | number | ○ | 奥行（mm） |
| rotation | number | ○ | 回転角度 |
| clearance | number | - | 保全距離（個別上書き） |
| layerId | string | - | 所属レイヤー |
| label | string | - | カスタムラベル |
| note | string | - | 備考 |

---

### 4.4 zones（ゾーン管理）Phase 4 追加

```jsonc
"zones": [
  {
    // === 識別子 ===
    "id": 301,                     // 一意 ID (number)
    "type": "storage",             // ゾーンタイプキー (string, → 5.3 参照)
    "label": "A保管エリア",         // 表示名 (string, max: 50文字)

    // === 位置・サイズ ===
    "x": 1000,                     // 左上 X 座標 mm (number)
    "y": 1000,                     // 左上 Y 座標 mm (number)
    "width": 20000,                // 幅 mm (number)
    "depth": 15000,                // 奥行 mm (number)

    // === 表示設定 ===
    "color": "#2ecc71",            // 背景色 (string, CSS color)
    "opacity": 0.15,               // 不透明度 (number, 0.05〜0.5)
    "borderColor": "#27ae60",      // 枠色 (string)
    "borderStyle": "solid",        // 枠線スタイル: solid/dashed/dotted (string)

    // === 属性 ===
    "locked": false,               // 移動ロック (boolean)
    "temperatureZone": "ambient",  // 温度帯 (string) ← Phase 6
    "floorLoadLimit": 1500,        // 床荷重制限 kg/m² (number) ← Phase 3
    "note": ""                     // 備考 (string)
  }
]
```

| フィールド | 型 | 必須 | 説明 |
|-----------|---|-----|------|
| id | number | ○ | 一意識別子 |
| type | string | ○ | ゾーンタイプキー |
| label | string | ○ | 表示ラベル |
| x | number | ○ | 左上 X 座標（mm） |
| y | number | ○ | 左上 Y 座標（mm） |
| width | number | ○ | 幅（mm） |
| depth | number | ○ | 奥行（mm） |
| color | string | ○ | 背景色（CSS カラー） |
| opacity | number | ○ | 透明度（0.05〜0.5） |
| borderColor | string | - | 枠線色 |
| borderStyle | string | - | 枠線スタイル |
| locked | boolean | - | 操作ロック（デフォルト: false） |
| temperatureZone | string | - | 温度帯属性 |
| floorLoadLimit | number | - | 床荷重制限（kg/m²） |
| note | string | - | 備考 |

---

### 4.5 layers（レイヤー管理）Phase 2 追加

```jsonc
"layers": [
  {
    "id": "layer_zone",            // レイヤー ID (string, 固定)
    "name": "ゾーン",              // 表示名 (string)
    "visible": true,               // 表示フラグ (boolean)
    "locked": false,               // 操作ロック (boolean)
    "opacity": 1.0,                // 透明度 (number, 0.0〜1.0)
    "order": 0                     // 描画順（小さいほど下） (number)
  },
  {
    "id": "layer_obstacle",
    "name": "障害物",
    "visible": true,
    "locked": false,
    "opacity": 1.0,
    "order": 1
  },
  {
    "id": "layer_rack",
    "name": "ラック",
    "visible": true,
    "locked": false,
    "opacity": 1.0,
    "order": 2
  },
  {
    "id": "layer_annotation",
    "name": "寸法・注釈",
    "visible": true,
    "locked": false,
    "opacity": 1.0,
    "order": 3
  }
]
```

| フィールド | 型 | 必須 | 説明 |
|-----------|---|-----|------|
| id | string | ○ | レイヤー識別子（固定値） |
| name | string | ○ | UI 表示名 |
| visible | boolean | ○ | 表示/非表示 |
| locked | boolean | ○ | 操作ロック（ロック時はドラッグ不可） |
| opacity | number | ○ | 全体透明度（0.0〜1.0） |
| order | number | ○ | 描画 Z オーダー（小さいほど下層） |

---

### 4.6 viewSettings（表示設定）

```jsonc
"viewSettings": {
  "showGrid": true,               // グリッド表示 (boolean)
  "showDimensions": true,         // 寸法表示 (boolean)
  "showHeatmap": false,           // ヒートマップ表示 (boolean)
  "showValidation": false,        // バリデーション表示 (boolean)
  "showCorridors": false,         // 通路チェック表示 (boolean, Phase 2)
  "gridSize": 1000,               // グリッド間隔 mm (number)
  "snapSize": 100,                // スナップ間隔 mm (number)
  "snapSizeShift": 50,            // Shift+スナップ間隔 mm (number)
  "scale": 0.05,                  // 表示スケール (number)
  "offsetX": 0,                   // 表示オフセット X px (number)
  "offsetY": 0,                   // 表示オフセット Y px (number)
  "forkliftType": "reach"         // フォークリフト種別 (string)
}
```

---

## 5. マスタデータ定義

マスタデータは JavaScript ハードコードで管理する（LocalStorage には保存しない）。

### 5.1 ラックマスタ（APP.masters）

```jsonc
{
  "nesting": {
    "label": "ネスティングラック",
    "width": 1350,
    "depth": 1200,
    "height": 1700,
    "color": "#2980b9",
    "textColor": "#ffffff",
    "defaultParams": {
      "stackLevels": 4,
      "spotsPerUnit": 1,
      "palletWidth": 1100,
      "palletDepth": 1100,
      "maxLoadPerPallet": 1000
    },
    "calcFormula": "count * spotsPerUnit * stackLevels",
    "calcUnit": "パレット",
    "icon": "🔲"
  },
  "pallet": {
    "label": "パレットラック",
    "width": 2700,
    "depth": 1100,
    "height": 5000,
    "color": "#8e44ad",
    "textColor": "#ffffff",
    "defaultParams": {
      "bays": 2,
      "levels": 4,
      "palletWidth": 1100,
      "palletDepth": 1100,
      "beamHeight": 1200,
      "maxLoadPerPallet": 1000
    },
    "calcFormula": "count * bays * levels",
    "calcUnit": "ロケーション",
    "icon": "📦"
  },
  "shelf": {
    "label": "軽量棚",
    "width": 900,
    "depth": 450,
    "height": 1800,
    "color": "#27ae60",
    "textColor": "#ffffff",
    "defaultParams": {
      "shelves": 5,
      "divisionsPerShelf": 3
    },
    "calcFormula": "count * shelves * divisionsPerShelf",
    "calcUnit": "ロケーション",
    "icon": "🗂"
  },
  "flat": {
    "label": "平置きエリア",
    "width": 5000,
    "depth": 3000,
    "height": 0,
    "color": "#e67e22",
    "textColor": "#ffffff",
    "defaultParams": {
      "palletWidth": 1100,
      "palletDepth": 1100,
      "stackLevels": 1
    },
    "calcFormula": "floor((width / palletWidth) * (depth / palletDepth)) * stackLevels",
    "calcUnit": "パレット",
    "icon": "⬜"
  },
  "work": {
    "label": "作業エリア",
    "width": 3000,
    "depth": 3000,
    "height": 0,
    "color": "#16a085",
    "textColor": "#ffffff",
    "defaultParams": {},
    "calcFormula": null,
    "calcUnit": null,
    "icon": "🔧"
  }
}
```

### 5.2 障害物マスタ（APP.obstacleDefaults）

```jsonc
{
  "pillar": {
    "label": "柱",
    "width": 500,
    "depth": 500,
    "color": "#7f8c8d",
    "clearance": 0,
    "description": "構造柱"
  },
  "shutter": {
    "label": "シャッター",
    "width": 3000,
    "depth": 300,
    "color": "#95a5a6",
    "clearance": 3000,
    "description": "搬入出口シャッター"
  },
  "dock": {
    "label": "ドック",
    "width": 3500,
    "depth": 1000,
    "color": "#d35400",
    "clearance": 4000,
    "description": "トラックバースドック"
  },
  "fire_hydrant": {
    "label": "消火栓",
    "width": 400,
    "depth": 400,
    "color": "#e74c3c",
    "clearance": 1000,
    "description": "消火栓（保全距離 1,000mm 必須）"
  },
  "panel": {
    "label": "分電盤",
    "width": 600,
    "depth": 200,
    "color": "#f39c12",
    "clearance": 500,
    "description": "電気分電盤"
  },
  "office": {
    "label": "事務所",
    "width": 5000,
    "depth": 4000,
    "color": "#2c3e50",
    "clearance": 0,
    "description": "管理事務所スペース"
  },
  "toilet": {
    "label": "トイレ",
    "width": 2000,
    "depth": 2000,
    "color": "#1abc9c",
    "clearance": 0,
    "description": "トイレ・洗面"
  }
}
```

### 5.3 ゾーンマスタ（APP.zoneDefaults）Phase 4 追加

```jsonc
{
  "receiving": {
    "label": "入荷エリア",
    "color": "#3498db",
    "opacity": 0.15,
    "description": "荷受・検数・荷下ろしスペース"
  },
  "shipping": {
    "label": "出荷エリア",
    "color": "#e74c3c",
    "opacity": 0.15,
    "description": "出荷仕分け・荷揃えスペース"
  },
  "inspection": {
    "label": "検品エリア",
    "color": "#f39c12",
    "opacity": 0.15,
    "description": "品質検査・入出荷検品"
  },
  "storage": {
    "label": "保管エリア",
    "color": "#2ecc71",
    "opacity": 0.15,
    "description": "メイン格納・在庫保管"
  },
  "picking": {
    "label": "ピッキングエリア",
    "color": "#9b59b6",
    "opacity": 0.15,
    "description": "流動在庫・前出し棚・ピッキング"
  },
  "returns": {
    "label": "返品エリア",
    "color": "#1abc9c",
    "opacity": 0.15,
    "description": "返品品仮置き・検品待ち"
  },
  "charging": {
    "label": "充電エリア",
    "color": "#e67e22",
    "opacity": 0.15,
    "description": "フォークリフト充電・整備"
  }
}
```

### 5.4 フォークリフトマスタ（APP.forkliftTypes）

```jsonc
{
  "reach": {
    "label": "リーチ式",
    "minAisle": 2700,
    "turningRadius": 1600,
    "maxLoadKg": 1500,
    "maxHeight": 8000,
    "color": "#27ae60"
  },
  "counter": {
    "label": "カウンターバランス式",
    "minAisle": 3500,
    "turningRadius": 2500,
    "maxLoadKg": 3000,
    "maxHeight": 6000,
    "color": "#e74c3c"
  },
  "hand": {
    "label": "ハンドリフト",
    "minAisle": 1800,
    "turningRadius": 1200,
    "maxLoadKg": 2000,
    "maxHeight": 200,
    "color": "#3498db"
  },
  "order_picker": {
    "label": "オーダーピッカー",
    "minAisle": 2400,
    "turningRadius": 1800,
    "maxLoadKg": 1000,
    "maxHeight": 5500,
    "color": "#9b59b6"
  }
}
```

---

## 6. サンプルデータ

以下は中規模フルフィルメントセンター（6,000m²）の完全なサンプル JSON。

```json
{
  "version": 4,
  "savedAt": "2026-06-13T10:00:00Z",
  "appVersion": "2.0.0",
  "warehouse": {
    "name": "横浜フルフィルメントセンター A棟",
    "width": 100000,
    "depth": 60000,
    "height": 8000,
    "floorLoadCapacity": 1500,
    "temperatureZone": "ambient",
    "buildingType": "steel_frame"
  },
  "items": [
    {
      "id": 101,
      "type": "pallet",
      "x": 2000,
      "y": 5000,
      "width": 2700,
      "depth": 1100,
      "rotation": 0,
      "params": {
        "bays": 2,
        "levels": 4,
        "palletWidth": 1100,
        "palletDepth": 1100,
        "beamHeight": 1200,
        "maxLoadPerPallet": 1000
      },
      "layerId": "layer_rack",
      "zoneId": 301,
      "label": "PR-001",
      "color": null,
      "count": 1,
      "note": "主保管エリア パレットラック1号機"
    },
    {
      "id": 102,
      "type": "nesting",
      "x": 6000,
      "y": 5000,
      "width": 1350,
      "depth": 1200,
      "rotation": 0,
      "params": {
        "stackLevels": 4,
        "spotsPerUnit": 1,
        "palletWidth": 1100,
        "palletDepth": 1100,
        "maxLoadPerPallet": 1000
      },
      "layerId": "layer_rack",
      "zoneId": 302,
      "label": "NR-001",
      "color": null,
      "count": 1,
      "note": "入荷エリア ネスティングラック"
    },
    {
      "id": 103,
      "type": "shelf",
      "x": 2000,
      "y": 30000,
      "width": 900,
      "depth": 450,
      "rotation": 0,
      "params": {
        "shelves": 5,
        "divisionsPerShelf": 3
      },
      "layerId": "layer_rack",
      "zoneId": 303,
      "label": "SH-001",
      "color": null,
      "count": 1,
      "note": "ピッキングエリア 軽量棚"
    }
  ],
  "obstacles": [
    {
      "id": 201,
      "type": "pillar",
      "x": 10000,
      "y": 10000,
      "width": 500,
      "depth": 500,
      "rotation": 0,
      "clearance": 0,
      "layerId": "layer_obstacle",
      "label": "P-01",
      "note": "構造柱 グリッド A-1"
    },
    {
      "id": 202,
      "type": "dock",
      "x": 0,
      "y": 5000,
      "width": 3500,
      "depth": 1000,
      "rotation": 0,
      "clearance": 4000,
      "layerId": "layer_obstacle",
      "label": "D-01",
      "note": "メインバース 10t車対応"
    },
    {
      "id": 203,
      "type": "fire_hydrant",
      "x": 20000,
      "y": 0,
      "width": 400,
      "depth": 400,
      "rotation": 0,
      "clearance": 1000,
      "layerId": "layer_obstacle",
      "label": "FH-01",
      "note": ""
    }
  ],
  "zones": [
    {
      "id": 301,
      "type": "storage",
      "label": "主保管エリア（パレット）",
      "x": 1000,
      "y": 1000,
      "width": 60000,
      "depth": 35000,
      "color": "#2ecc71",
      "opacity": 0.12,
      "borderColor": "#27ae60",
      "borderStyle": "dashed",
      "locked": false,
      "temperatureZone": "ambient",
      "floorLoadLimit": 1500,
      "note": "パレットラック格納エリア"
    },
    {
      "id": 302,
      "type": "receiving",
      "label": "入荷エリア",
      "x": 1000,
      "y": 38000,
      "width": 20000,
      "depth": 10000,
      "color": "#3498db",
      "opacity": 0.12,
      "borderColor": "#2980b9",
      "borderStyle": "solid",
      "locked": false,
      "temperatureZone": "ambient",
      "floorLoadLimit": 2000,
      "note": "荷受・検数スペース"
    },
    {
      "id": 303,
      "type": "picking",
      "label": "ピッキングエリア",
      "x": 65000,
      "y": 1000,
      "width": 20000,
      "depth": 47000,
      "color": "#9b59b6",
      "opacity": 0.12,
      "borderColor": "#8e44ad",
      "borderStyle": "dashed",
      "locked": false,
      "temperatureZone": "ambient",
      "floorLoadLimit": 800,
      "note": "軽量棚・前出し棚"
    }
  ],
  "layers": [
    { "id": "layer_zone", "name": "ゾーン", "visible": true, "locked": false, "opacity": 1.0, "order": 0 },
    { "id": "layer_obstacle", "name": "障害物", "visible": true, "locked": false, "opacity": 1.0, "order": 1 },
    { "id": "layer_rack", "name": "ラック", "visible": true, "locked": false, "opacity": 1.0, "order": 2 },
    { "id": "layer_annotation", "name": "寸法・注釈", "visible": true, "locked": false, "opacity": 1.0, "order": 3 }
  ],
  "viewSettings": {
    "showGrid": true,
    "showDimensions": true,
    "showHeatmap": false,
    "showValidation": false,
    "showCorridors": false,
    "gridSize": 1000,
    "snapSize": 100,
    "snapSizeShift": 50,
    "scale": 0.012,
    "offsetX": 20,
    "offsetY": 20,
    "forkliftType": "reach"
  },
  "nextId": 1001
}
```

---

## 7. マイグレーション定義

```javascript
// マイグレーション関数（ロード時に自動実行）
const MIGRATIONS = {
  // v1 → v2: ラックパラメータ・複数選択対応
  2: (data) => {
    data.items = (data.items || []).map(item => {
      if (!item.params) {
        const defaults = APP.masters[item.type]?.defaultParams || {};
        item.params = { ...defaults };
      }
      if (!item.layerId) item.layerId = 'layer_rack';
      if (item.zoneId === undefined) item.zoneId = null;
      if (!item.label) item.label = '';
      if (!item.note) item.note = '';
      return item;
    });
    data.obstacles = (data.obstacles || []).map(obs => {
      if (!obs.layerId) obs.layerId = 'layer_obstacle';
      if (!obs.label) obs.label = '';
      if (!obs.note) obs.note = '';
      return obs;
    });
    return data;
  },

  // v2 → v3: レイヤー管理追加
  3: (data) => {
    if (!data.layers) {
      data.layers = [
        { id: 'layer_zone', name: 'ゾーン', visible: true, locked: false, opacity: 1.0, order: 0 },
        { id: 'layer_obstacle', name: '障害物', visible: true, locked: false, opacity: 1.0, order: 1 },
        { id: 'layer_rack', name: 'ラック', visible: true, locked: false, opacity: 1.0, order: 2 },
        { id: 'layer_annotation', name: '寸法・注釈', visible: true, locked: false, opacity: 1.0, order: 3 }
      ];
    }
    return data;
  },

  // v3 → v4: ゾーン管理追加
  4: (data) => {
    if (!data.zones) data.zones = [];
    if (!data.warehouse.floorLoadCapacity) data.warehouse.floorLoadCapacity = 1500;
    if (!data.warehouse.temperatureZone) data.warehouse.temperatureZone = 'ambient';
    if (!data.warehouse.buildingType) data.warehouse.buildingType = 'steel_frame';
    if (!data.viewSettings) data.viewSettings = {};
    if (data.viewSettings.showCorridors === undefined) data.viewSettings.showCorridors = false;
    return data;
  }
};

// マイグレーション実行関数
function runMigrations(data) {
  const currentVersion = 4;
  let version = data.version || 1;
  while (version < currentVersion) {
    version++;
    if (MIGRATIONS[version]) {
      data = MIGRATIONS[version](data);
      console.log(`Migration to v${version} complete`);
    }
  }
  data.version = currentVersion;
  return data;
}
```

---

## 8. ストレージ容量試算

LocalStorage の制限（5MB）に対する使用量の試算。

| シナリオ | ラック数 | 障害物数 | ゾーン数 | JSON サイズ（概算） |
|---------|---------|---------|---------|-----------------|
| 小規模（1,000m²） | 50 | 10 | 3 | ~15 KB |
| 中規模（3,000m²） | 200 | 30 | 6 | ~55 KB |
| 大規模（10,000m²） | 800 | 80 | 10 | ~210 KB |
| 超大規模（30,000m²） | 3,000 | 200 | 20 | ~780 KB |

5MB 上限には余裕があるが、ラック数 5,000 以上になる超大型センターでは IndexedDB への移行を推奨する。

### 8.1 容量超過時の処理

```javascript
function saveToLocalStorage(data) {
  try {
    const json = JSON.stringify(data);
    if (json.length > 4 * 1024 * 1024) { // 4MB 警告
      showWarning('データサイズが大きくなっています。JSONエクスポートによるバックアップを推奨します。');
    }
    localStorage.setItem('warehousePlannerData', json);
  } catch (e) {
    if (e.name === 'QuotaExceededError') {
      showError('保存容量の上限に達しました。不要なデータを削除するか、JSONエクスポートをご利用ください。');
    }
  }
}
```

---

## 9. IndexedDB 移行方針（Phase 9）

ラック数 5,000 超・複数レイアウト管理の実装時に IndexedDB に移行する。

### 9.1 DB 構造（IndexedDB）

```
Database: warehousePlanner
  ObjectStore: layouts
    keyPath: id (uuid)
    indexes:
      - name: savedAt
      - name: warehouseName
    value: { id, name, thumbnail, savedAt, data: {...} }

  ObjectStore: templates
    keyPath: id (uuid)
    value: { id, name, category, data: {...} }

  ObjectStore: proposalHistory
    keyPath: id (uuid)
    value: { id, clientName, createdAt, data: {...} }
```

### 9.2 LocalStorage → IndexedDB 移行スクリプト

```javascript
async function migrateToIndexedDB() {
  const legacyData = localStorage.getItem('warehousePlannerData');
  if (!legacyData) return;
  const data = JSON.parse(legacyData);
  const db = await openDB('warehousePlanner', 1);
  await db.put('layouts', {
    id: crypto.randomUUID(),
    name: data.warehouse.name,
    savedAt: new Date().toISOString(),
    data: data
  });
  localStorage.removeItem('warehousePlannerData');
  console.log('Migration to IndexedDB complete');
}
```

---

*文書終端*
