# WI-0024: パフォーマンス最適化 - 全モジュール統合最適化

**作業ID**: WI-0024
**作業名**: Performance Optimization - All Modules Comprehensive Optimization
**作成日**: 2025-11-17
**対象Phase**: Phase 3（Week 5）
**優先度**: 高
**推定作業時間**: 2日

---

## 1. 作業目的

Phase 1で実装された全23モジュール（WI-0001 ~ WI-0023）に対して包括的なパフォーマンス最適化を実施する。Quest 2環境で60fps維持、VRChat Performance Rank「Good」以上の達成を目指す。

### 最適化の必要性
- **Quest制約**: Quest 2は限られたGPU/CPUリソースで動作するため、最適化が必須
- **マルチプレイヤー**: 10人以上の同時接続時もフレームレート維持が必要
- **VRChat制限**: ネットワーク帯域（5KB/秒目標）、メモリ（500MB以下）の制約
- **ユーザー体験**: フレームドロップはVR酔いの原因となり、滞在時間に直結

---

## 2. 対象

### 対象システム
- システム名: 森のきこりキャンプ (Woodcutter Camp)
- プラットフォーム: VRChat (PC / Quest対応)

### 対象モジュール
**全16モジュール（WI-0001 ~ WI-0023で実装）**:

**Core Layer**:
- M-01: GameManager (WI-0001)
- M-02: NetworkSync (WI-0002, WI-0004)
- M-03: PersistenceManager (WI-0003)

**Progression Layer**:
- M-04: SkillManager (WI-0005)
- M-05: SkillEffects (WI-0006)

**Economy Layer**:
- M-06: CoinManager (WI-0013, WI-0014)
- M-07: ShopManager (WI-0019)

**UI Layer**:
- M-08: HUDManager (WI-0017, WI-0018)
- M-09: LeaderboardUI (WI-0022)

**Gameplay Layer**:
- M-10: TreeController (WI-0007, WI-0008)
- M-11: AxeInteraction (WI-0009, WI-0010, WI-0011)
- M-12: LogSpawner (WI-0012)
- M-13: LogPickup (WI-0015, WI-0016)
- M-14: CuttingTable (WI-0023)
- M-15: CooperativeTracker (WI-0020)
- M-16: RankingTracker (WI-0021)

### 対象ファイルパス
```
Assets/WoodcutterCamp/
├── Scripts/
│   ├── Core/
│   │   ├── GameManager.cs
│   │   ├── NetworkSync.cs
│   │   └── PersistenceManager.cs
│   ├── Progression/
│   │   ├── SkillManager.cs
│   │   └── SkillEffects.cs
│   ├── Economy/
│   │   ├── CoinManager.cs
│   │   └── ShopManager.cs
│   ├── UI/
│   │   ├── HUDManager.cs
│   │   └── LeaderboardUI.cs
│   └── Gameplay/
│       ├── TreeController.cs
│       ├── AxeInteraction.cs
│       ├── LogSpawner.cs
│       ├── LogPickup.cs
│       ├── CuttingTable.cs
│       ├── CooperativeTracker.cs
│       └── RankingTracker.cs
├── Prefabs/
├── Materials/
├── Textures/
└── Audio/
```

---

## 3. 前提条件

### 環境
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0
- UdonSharp 1.1.9
- ClientSim 1.2.7
- **WI-0001 ~ WI-0023の実装が完了していること**

### ブランチ
- 作業ブランチ: `feature/wi-0024-performance-optimization`
- ベースブランチ: `main`（全WI実装済み）

### 初期状態
- 全モジュールが機能的に動作している
- 単体テスト・統合テストが完了している
- 既知のバグが修正されている

### 必要なツール
- Unity Profiler（CPU/GPU/Memory/Rendering）
- VRChat ClientSim
- VRChat Stats Panel（実機テスト用）
- Frame Debugger
- VRWorld Toolkit 3.3.2（診断機能）

---

## 4. パフォーマンス目標値

### 4.1 フレームレート目標

| プラットフォーム | 条件 | 目標FPS | 最小FPS |
|---------------|------|---------|---------|
| **Quest 2** | 10人接続、全員伐採中 | 60fps | 55fps |
| **Quest 3** | 16人接続、全員伐採中 | 72fps | 65fps |
| **PC VR** | 20人接続、全員伐採中 | 90fps | 80fps |
| **Desktop** | 20人接続、全員伐採中 | 120fps | 100fps |

### 4.2 パフォーマンス指標

#### CPU関連
- **フレーム時間**: <16.6ms/frame（60fps維持）
- **Update()実行回数**: 全スクリプト合計<20回/frame
- **GetComponent()呼び出し**: 0回/frame（キャッシング必須）
- **GC Alloc**: <1KB/frame

#### GPU関連
- **Draw Calls**: <50回/frame
- **三角ポリゴン数**: <100,000/frame
- **テクスチャメモリ**: <100MB

#### ネットワーク関連
- **帯域使用量**: <5KB/秒（プレイヤー1人あたり）
- **同期頻度**: 重要データのみ5秒間隔でバッチング
- **データサイズ**: Manual Sync 1回あたり<10KB

#### メモリ関連
- **総メモリ使用量**: <500MB
- **テクスチャメモリ**: <100MB
- **オーディオメモリ**: <50MB
- **スクリプトメモリ**: <20MB

### 4.3 VRChat Performance Rank目標

- **目標Rank**: Good（緑色）
- **最低許容Rank**: Medium（黄色）
- **NG Rank**: Poor（赤色）

**Performance Rank確認方法**:
```
VRChatメニュー > Performance & Safety > Show Avatar Performance Rank
※ワールドの場合はStats Panelで確認
```

---

## 5. 最適化チェックリスト

### 5.1 CPU最適化チェックリスト

#### Update()ループ削減
- [ ] **M-01 GameManager**: 不要なUpdate()削除、イベント駆動化
- [ ] **M-02 NetworkSync**: バッチング処理を5秒間隔にまとめる
- [ ] **M-04 SkillManager**: XP更新をイベントトリガーに変更
- [ ] **M-06 CoinManager**: コイン更新をイベントトリガーに変更
- [ ] **M-08 HUDManager**: UI更新を変更時のみに制限
- [ ] **M-09 LeaderboardUI**: 30秒間隔の定期更新を維持（既に最適化済み）
- [ ] **M-10 TreeController**: HP更新を攻撃時のみに制限
- [ ] **M-11 AxeInteraction**: スイング判定をイベント駆動に
- [ ] **M-13 LogPickup**: 運搬距離計測をDrop時のみに
- [ ] **M-15 CooperativeTracker**: 協力判定を攻撃時のみに
- [ ] **M-16 RankingTracker**: ソート処理を30秒間隔に制限

#### GetComponent()キャッシング
- [ ] **全モジュール**: Start()で全GetComponent()を実行、フィールドにキャッシュ
- [ ] **M-11 AxeInteraction**: Collider, Rigidbodyキャッシング
- [ ] **M-12 LogSpawner**: Transform, Rigidbodyキャッシング
- [ ] **M-13 LogPickup**: VRC_Pickupキャッシング

#### ガベージコレクション削減
- [ ] **M-02 NetworkSync**: 配列再利用、new回避
- [ ] **M-08 HUDManager**: 文字列連結をStringBuilder使用
- [ ] **M-09 LeaderboardUI**: UI更新時の一時配列削除
- [ ] **M-12 LogSpawner**: オブジェクトプール使用（実装済み確認）
- [ ] **M-16 RankingTracker**: ソート用一時配列の再利用

#### 実行ガード（Early Return）
```csharp
// 例: TreeController.cs
void Update()
{
    // 倒木状態なら処理スキップ
    if (isFallen) return;

    // リスポーン中のみタイマー更新
    if (isRespawning)
    {
        respawnTimer -= Time.deltaTime;
        if (respawnTimer <= 0)
        {
            Respawn();
        }
    }
}
```

### 5.2 GPU最適化チェックリスト

#### Draw Call削減
- [ ] **UI Canvas**: 1つのCanvasに統合（M-08, M-09）
- [ ] **木モデル**: Static Batching有効化
- [ ] **丸太モデル**: 同一マテリアル使用
- [ ] **環境オブジェクト**: Static Batching有効化
- [ ] **マテリアル統合**: 同じシェーダーのマテリアルを統合

#### ポリゴン数削減
- [ ] **木（小）**: <3,000三角ポリゴン
- [ ] **木（中）**: <5,000三角ポリゴン
- [ ] **木（大）**: <8,000三角ポリゴン
- [ ] **丸太**: <500三角ポリゴン
- [ ] **環境全体**: <50,000三角ポリゴン
- [ ] **LOD設定**: 距離に応じた3段階LOD（LOD0, LOD1, LOD2）

#### テクスチャ最適化
- [ ] **PC版テクスチャ**: 最大1024x1024
- [ ] **Quest版テクスチャ**: 512x512推奨
- [ ] **圧縮形式**: PC: DXT5, Quest: ASTC 6x6
- [ ] **ミップマップ**: 全テクスチャで有効化
- [ ] **テクスチャアトラス**: UIアイコンを1枚にまとめる

#### シェーダー最適化
- [ ] **標準シェーダー使用**: カスタムシェーダー回避
- [ ] **透明度**: 不要なアルファブレンディング削除
- [ ] **影**: Quest版は影オフ

### 5.3 Quest専用最適化チェックリスト

#### パーティクル最適化
- [ ] **M-11 Hit Effects**: パーティクル数を30個以下に制限
- [ ] **M-10 Tree Falling**: 木片パーティクルを20個以下に
- [ ] **影**: パーティクルの影を無効化
- [ ] **シェーダー**: Additive/Multiplyのみ使用（PBR不使用）

#### ライティング最適化
- [ ] **ベイクドライティング**: 全静的オブジェクトをLightmap Static化
- [ ] **リアルタイムライト**: 最大2個（太陽光+焚き火のみ）
- [ ] **影**: Quest版は全ライトの影をオフ
- [ ] **Light Probes**: 動的オブジェクト用に配置

#### ポストプロセス最適化
- [ ] **Quest版**: Bloom, SSAO, Motion Blur無効化
- [ ] **PC版**: 軽量設定のみ有効化

### 5.4 ネットワーク最適化チェックリスト

#### Manual Syncバッチング
- [ ] **M-02 NetworkSync**: 5秒間隔でバッチング実装
- [ ] **M-10 TreeController**: HP変更を即座に同期せず、バッチングに含める
- [ ] **M-15 CooperativeTracker**: 協力状態変更をバッチング
- [ ] **M-16 RankingTracker**: ランキング更新を30秒間隔に

#### データサイズ削減
- [ ] **M-03 PersistenceManager**: PlayerData圧縮（bool値のビットパッキング）
- [ ] **M-16 RankingTracker**: プレイヤー名を短縮（最大8文字）
- [ ] **M-02 NetworkSync**: 不要な同期変数削除

#### 同期優先度制御
- [ ] **高優先度**: 木のHP、倒木状態（即座に同期）
- [ ] **中優先度**: プレイヤー統計、協力状態（5秒間隔）
- [ ] **低優先度**: ランキング表示（30秒間隔）

### 5.5 メモリ最適化チェックリスト

#### 配列事前確保
- [ ] **M-16 RankingTracker**: プレイヤー配列を最大20個で固定
- [ ] **M-12 LogSpawner**: オブジェクトプール配列を30個で固定
- [ ] **M-02 NetworkSync**: 同期データ配列を固定サイズ化

#### テクスチャメモリ管理
- [ ] **総テクスチャサイズ**: <100MB
- [ ] **アトラス化**: UIアイコンを1枚のテクスチャにまとめる
- [ ] **不要テクスチャ削除**: 未使用テクスチャをビルドから除外

#### オーディオメモリ管理
- [ ] **長尺音源**: Streaming設定（>5秒）
- [ ] **短尺音源**: Compressed In Memory設定（<5秒）
- [ ] **圧縮形式**: Vorbis推奨

---

## 6. 作業手順

### Phase 1: パフォーマンス監査（0.5日）

#### 6.1 Unity Profilerセットアップ

**手順**:
1. Unity Editorでプロジェクトを開く
2. Window > Analysis > Profiler を開く
3. Profilerウィンドウで以下を有効化:
   - CPU Usage
   - GPU Usage
   - Memory
   - Rendering

**Profiler設定**:
```
Record: ON
Deep Profile: OFF（重いため）
Call Stacks: OFF
Editor Only: OFF（Standaloneビルドで測定）
```

#### 6.2 ベースライン測定

**測定シナリオ**:
```
1. ClientSimで10人プレイヤーをシミュレート
2. 全員が木を伐採中の状態を再現
3. 60秒間のプロファイリングデータ取得
```

**記録する指標**:
- CPU Frame Time: ___ms/frame
- GPU Frame Time: ___ms/frame
- Draw Calls: ___回/frame
- Triangles: ___個/frame
- SetPass Calls: ___回/frame
- Total Memory: ___MB
- GC Alloc: ___KB/frame

**記録フォーマット**:
```markdown
## ベースライン測定結果（最適化前）

測定日時: 2025-11-__
測定環境: Unity Editor / ClientSim

| 指標 | 測定値 | 目標値 | 判定 |
|------|--------|--------|------|
| CPU Frame Time | __ms | <16.6ms | __ |
| GPU Frame Time | __ms | <16.6ms | __ |
| Draw Calls | __ | <50 | __ |
| Triangles | __ | <100,000 | __ |
| Total Memory | __MB | <500MB | __ |
| GC Alloc | __KB/frame | <1KB | __ |
```

#### 6.3 ボトルネック特定

**CPU Profilerで確認**:
```
1. Profilerウィンドウ > CPU Usage
2. Timeline Viewで最も時間のかかる関数を特定
3. Hierarchy Viewで呼び出し階層を確認
```

**よくあるボトルネック**:
- Update()の過剰実行
- GetComponent()の頻繁な呼び出し
- 文字列連結によるGC Alloc
- ネストしたループ処理

**記録フォーマット**:
```markdown
## ボトルネック一覧

### CPU
1. [モジュール名] - [関数名]: __ms/frame
   - 原因: __
   - 対策: __

### GPU
1. [描画処理]: __ms/frame
   - 原因: __
   - 対策: __
```

### Phase 2: モジュール別最適化（1日）

#### 6.4 Core Layer最適化

##### M-01: GameManager
**最適化項目**:
```csharp
// ❌ 削除: 不要なUpdate()
void Update()
{
    // 何も処理していない場合は削除
}

// ✅ 必要な場合のみイベント駆動化
public void _OnSystemStateChanged()
{
    // 状態変化時のみ処理
}
```

##### M-02: NetworkSync
**最適化項目**:
```csharp
// ✅ バッチング処理の実装
private float syncInterval = 5.0f; // 5秒間隔
private float nextSyncTime = 0;
private bool pendingSync = false;

void Update()
{
    if (pendingSync && Time.time >= nextSyncTime)
    {
        RequestSerialization();
        pendingSync = false;
        nextSyncTime = Time.time + syncInterval;
    }
}

public void RequestSync()
{
    pendingSync = true;
}
```

##### M-03: PersistenceManager
**最適化項目**:
```csharp
// ✅ PlayerDataの圧縮（bool値のビットパッキング）
// 8つのboolを1つのintに圧縮
private int PackBools(bool b0, bool b1, bool b2, bool b3, bool b4, bool b5, bool b6, bool b7)
{
    int result = 0;
    if (b0) result |= 1 << 0;
    if (b1) result |= 1 << 1;
    if (b2) result |= 1 << 2;
    if (b3) result |= 1 << 3;
    if (b4) result |= 1 << 4;
    if (b5) result |= 1 << 5;
    if (b6) result |= 1 << 6;
    if (b7) result |= 1 << 7;
    return result;
}

private bool UnpackBool(int packed, int index)
{
    return (packed & (1 << index)) != 0;
}
```

#### 6.5 Progression Layer最適化

##### M-04: SkillManager
**最適化項目**:
```csharp
// ❌ 削除: Update()での不要なチェック
// void Update()
// {
//     CheckLevelUp(); // イベント駆動に変更
// }

// ✅ XP加算時のみレベルアップチェック
public void AddXP(int amount)
{
    currentXP += amount;

    // レベルアップ判定
    if (currentXP >= GetRequiredXP(skillLevel + 1))
    {
        LevelUp();
    }

    // UI更新イベント発火
    SendCustomEvent("_OnXPChanged");
}
```

##### M-05: SkillEffects
**最適化項目**:
```csharp
// ✅ 計算結果のキャッシング
private int cachedLevel = -1;
private float cachedDamageBonus = 0;
private float cachedCriticalRate = 0;

public float GetDamageBonus(int level)
{
    if (level != cachedLevel)
    {
        cachedLevel = level;
        cachedDamageBonus = 1.0f + (level * 0.05f);
        cachedCriticalRate = CalculateCriticalRate(level);
    }
    return cachedDamageBonus;
}
```

#### 6.6 Economy Layer最適化

##### M-06: CoinManager
**最適化項目**:
```csharp
// ✅ コイン変更時のみUI更新
private int lastDisplayedCoins = -1;

public void AddCoins(int amount)
{
    currentCoins = Mathf.Min(currentCoins + amount, MAX_COINS);

    // 変更があった場合のみUI更新
    if (currentCoins != lastDisplayedCoins)
    {
        lastDisplayedCoins = currentCoins;
        SendCustomEvent("_UpdateCoinDisplay");
    }
}
```

#### 6.7 UI Layer最適化

##### M-08: HUDManager
**最適化項目**:
```csharp
// ✅ StringBuilder使用で文字列連結のGC削減
using System.Text;

private StringBuilder stringBuilder = new StringBuilder(256);

private void UpdateSkillText(int level, int currentXP, int requiredXP)
{
    stringBuilder.Clear();
    stringBuilder.Append("Woodcutting: Lv");
    stringBuilder.Append(level);
    stringBuilder.Append(" [");
    stringBuilder.Append(currentXP);
    stringBuilder.Append("/");
    stringBuilder.Append(requiredXP);
    stringBuilder.Append(" XP]");

    skillText.text = stringBuilder.ToString();
}
```

##### M-09: LeaderboardUI
**最適化項目**:
```csharp
// ✅ 固定配列の再利用
private string[] cachedRankingText = new string[3];

private void UpdateRankingDisplay()
{
    // 配列をその場で更新（new不要）
    for (int i = 0; i < 3; i++)
    {
        cachedRankingText[i] = FormatRankingLine(i);
    }

    // UI反映
    rankingText.text = string.Join("\n", cachedRankingText);
}
```

#### 6.8 Gameplay Layer最適化

##### M-10: TreeController
**最適化項目**:
```csharp
// ✅ Update()削減、リスポーン中のみタイマー更新
void Update()
{
    // 早期リターンでガード
    if (!isRespawning) return;

    respawnTimer -= Time.deltaTime;
    if (respawnTimer <= 0)
    {
        Respawn();
        isRespawning = false;
    }
}
```

##### M-11: AxeInteraction
**最適化項目**:
```csharp
// ✅ キャッシング
private Collider axeCollider;
private Rigidbody axeRigidbody;
private SkillEffects skillEffects;

void Start()
{
    axeCollider = GetComponent<Collider>();
    axeRigidbody = GetComponent<Rigidbody>();
    skillEffects = GameManager.Instance.GetSkillEffects();
}

// ✅ Update()削減、イベント駆動化
// Update()ではなく、OnTriggerEnter()でヒット判定
private void OnTriggerEnter(Collider other)
{
    TreeController tree = other.GetComponent<TreeController>();
    if (tree != null && CanAttack())
    {
        ProcessHit(tree);
    }
}
```

##### M-12: LogSpawner
**最適化項目**:
```csharp
// ✅ オブジェクトプール実装確認
// すでにWI-0012で実装されているはずだが、以下を再確認

private GameObject[] logPool = new GameObject[30];
private int poolIndex = 0;

private GameObject GetPooledLog()
{
    // 古いログから再利用
    GameObject log = logPool[poolIndex];
    poolIndex = (poolIndex + 1) % logPool.Length;

    if (log.activeSelf)
    {
        log.SetActive(false); // 古いログを非表示
    }

    return log;
}
```

##### M-13: LogPickup
**最適化項目**:
```csharp
// ✅ 運搬距離計測をDrop時のみに
private Vector3 pickupPosition;
private bool isCarrying = false;

public override void OnPickup()
{
    pickupPosition = transform.position;
    isCarrying = true;
}

public override void OnDrop()
{
    if (isCarrying)
    {
        float distance = Vector3.Distance(pickupPosition, transform.position);
        CalculateReward(distance);
        isCarrying = false;
    }
}

// ❌ 削除: Update()での距離計測
// void Update()
// {
//     if (isCarrying)
//     {
//         currentDistance = Vector3.Distance(pickupPosition, transform.position);
//     }
// }
```

##### M-15: CooperativeTracker
**最適化項目**:
```csharp
// ✅ タイムスタンプ記録をイベント駆動化
// Update()削減、攻撃イベント時のみ更新

public void OnTreeHit(VRCPlayerApi player, int treeID)
{
    int playerIndex = player.playerId;
    lastHitTimestamp[playerIndex] = Networking.GetServerTimeInMilliseconds();
    targetTreeID[playerIndex] = treeID;

    // 協力判定（3秒以内の攻撃をカウント）
    CheckCooperative(treeID);
}
```

##### M-16: RankingTracker
**最適化項目**:
```csharp
// ✅ ソート処理の最適化（Selection Sortで十分）
private void SortTop3()
{
    // TOP3のみソート（全体ソート不要）
    for (int i = 0; i < 3 && i < playerCount; i++)
    {
        int maxIndex = i;
        for (int j = i + 1; j < playerCount; j++)
        {
            if (playerCutCounts[j] > playerCutCounts[maxIndex])
            {
                maxIndex = j;
            }
        }

        if (maxIndex != i)
        {
            Swap(i, maxIndex);
        }
    }
}
```

### Phase 3: アセット最適化（0.5日）

#### 6.9 テクスチャ最適化

**手順**:
```
1. Project > Assets/Textures を開く
2. 各テクスチャを選択
3. Inspectorで以下を設定:

【PC版設定】
- Max Size: 1024
- Compression: DXT5 (Normal Quality)
- Generate Mip Maps: ON

【Quest版設定】
- Max Size: 512
- Compression: ASTC 6x6
- Generate Mip Maps: ON
- Override for Android: ON

4. Apply
```

**最適化対象**:
- [ ] 木テクスチャ（Bark, Leaves）
- [ ] 丸太テクスチャ
- [ ] 地面テクスチャ
- [ ] UIアイコン（アトラス化）

#### 6.10 メッシュ最適化

**手順**:
```
1. 3Dモデルを選択
2. Inspectorで以下を確認:
   - Read/Write Enabled: OFF
   - Optimize Mesh: ON
   - Normals/Tangents: Calculate（必要な場合のみ）

3. LOD設定:
   Window > LOD Group
   - LOD0 (0-30m): フル品質
   - LOD1 (30-60m): 50%削減
   - LOD2 (60m-): 75%削減
```

**最適化対象**:
- [ ] 木モデル（小・中・大）
- [ ] 丸太モデル
- [ ] 建物モデル

#### 6.11 オーディオ最適化

**手順**:
```
1. Project > Assets/Audio を開く
2. 各オーディオクリップを選択
3. Inspectorで以下を設定:

【長尺音源（>5秒）】
- Load Type: Streaming
- Compression Format: Vorbis
- Quality: 70%

【短尺音源（<5秒）】
- Load Type: Compressed In Memory
- Compression Format: Vorbis
- Quality: 80%

4. Apply
```

**最適化対象**:
- [ ] BGM（ループ）
- [ ] 効果音（斧ヒット、倒木など）
- [ ] 環境音（鳥、風）

### Phase 4: Quest専用ビルド設定（0.5日）

#### 6.12 Quest Build Settings

**手順**:
```
1. File > Build Settings
2. Platform: Android
3. Switch Platform
```

**Player Settings**:
```
Edit > Project Settings > Player > Android

【Graphics】
- Graphics API: OpenGLES3, Vulkan
- Color Space: Linear
- Multithreaded Rendering: ON
- Auto Graphics API: OFF

【Quality】
- Texture Quality: Low
- Shadow Distance: 0（影オフ）
- Anti Aliasing: 4x MSAA（固定）
- VSync Count: Don't Sync

【Other Settings】
- Scripting Backend: IL2CPP
- Target Architectures: ARM64
- Strip Engine Code: ON
```

#### 6.13 Quest専用最適化

**パーティクル設定**:
```
1. 全Particle Systemを選択
2. Inspector設定:
   - Max Particles: 30以下
   - Cast Shadows: OFF
   - Receive Shadows: OFF
   - Prewarm: OFF
```

**ライト設定**:
```
1. 全Lightを選択
2. Inspector設定:
   - Mode: Baked（静的オブジェクト用）
   - Shadows: None（Quest版）
   - Render Mode: Important → Not Important（リアルタイムライト以外）
```

**ポストプロセス無効化**:
```
1. Post Process Volumeを選択
2. Inspector:
   - Bloom: OFF
   - Ambient Occlusion: OFF
   - Motion Blur: OFF
   - Depth of Field: OFF
```

---

## 7. プロファイリング手順

### 7.1 Unity Profilerでの計測

**CPU Profiler**:
```
1. Window > Analysis > Profiler
2. CPU Usage タブを開く
3. Record ON、Deep Profile OFF
4. Playモード開始
5. 60秒間計測
6. Timeline Viewで重い関数を特定
7. Hierarchy Viewで呼び出し階層を確認
```

**記録項目**:
- Update() 実行時間
- GetComponent() 呼び出し回数
- GC Alloc サイズ

**GPU Profiler**:
```
1. Rendering タブを開く
2. 記録項目:
   - Draw Calls
   - SetPass Calls
   - Triangles
   - Vertices
```

**Memory Profiler**:
```
1. Memory タブを開く
2. Take Sample Editorボタンをクリック
3. 記録項目:
   - Total Memory
   - GC Allocated
   - Texture Memory
   - Mesh Memory
```

### 7.2 VRChat Stats Panelでの計測

**ClientSim計測**:
```
1. Unity Editor Playモード開始
2. Escキーでメニュー表示
3. Stats Panelを確認

記録項目:
- FPS
- Frame Time (ms)
- Network Send/Recv (KB/s)
```

**実機計測（VRChat Client）**:
```
1. ワールドをアップロード
2. VRChatで入室
3. メニュー > Performance & Safety > Show Performance Stats

記録項目:
- FPS
- Ping (ms)
- Network Send/Recv (KB/s)
- Performance Rank（Good/Medium/Poor）
```

### 7.3 Frame Debuggerでの解析

**Draw Call解析**:
```
1. Window > Analysis > Frame Debugger
2. Enable ボタンをクリック
3. 各Draw Callを選択
4. 以下を確認:
   - どのオブジェクトが描画されているか
   - シェーダー・マテリアル
   - バッチング状況
```

**バッチング確認**:
```
Batch Reason列を確認:
- "Static Batching": 成功
- "Not Batched": 要調査
- "Different Materials": マテリアル統合検討
```

---

## 8. 最適化レポートテンプレート

### 8.1 レポート構成

最適化完了後、以下のレポートを作成する。

**ファイル名**: `docs/Performance_Optimization_Report_WI-0024.md`

**テンプレート**:
```markdown
# パフォーマンス最適化レポート WI-0024

**作成日**: 2025-11-__
**作成者**: [担当者名]
**対象バージョン**: Phase 1 完成版

---

## 1. エグゼクティブサマリー

### 最適化結果概要
- 最適化前FPS: __fps → 最適化後FPS: __fps（向上率: __%）
- Draw Calls削減: __回 → __回（削減率: __%）
- メモリ削減: __MB → __MB（削減率: __%）
- VRChat Performance Rank: __ → __

### 主要な変更点
1. __
2. __
3. __

---

## 2. パフォーマンス指標の比較

### 2.1 フレームレート

| プラットフォーム | 最適化前 | 最適化後 | 目標値 | 達成 |
|---------------|---------|---------|--------|------|
| Quest 2 | __fps | __fps | 60fps | ○/× |
| Quest 3 | __fps | __fps | 72fps | ○/× |
| PC VR | __fps | __fps | 90fps | ○/× |
| Desktop | __fps | __fps | 120fps | ○/× |

### 2.2 CPU指標

| 指標 | 最適化前 | 最適化後 | 目標値 | 達成 |
|------|---------|---------|--------|------|
| Frame Time | __ms | __ms | <16.6ms | ○/× |
| Update()回数 | __回/frame | __回/frame | <20回 | ○/× |
| GC Alloc | __KB/frame | __KB/frame | <1KB | ○/× |

### 2.3 GPU指標

| 指標 | 最適化前 | 最適化後 | 目標値 | 達成 |
|------|---------|---------|--------|------|
| Draw Calls | __回 | __回 | <50回 | ○/× |
| Triangles | __個 | __個 | <100,000 | ○/× |
| SetPass Calls | __回 | __回 | <30回 | ○/× |

### 2.4 メモリ指標

| 指標 | 最適化前 | 最適化後 | 目標値 | 達成 |
|------|---------|---------|--------|------|
| Total Memory | __MB | __MB | <500MB | ○/× |
| Texture Memory | __MB | __MB | <100MB | ○/× |
| Audio Memory | __MB | __MB | <50MB | ○/× |

### 2.5 ネットワーク指標

| 指標 | 最適化前 | 最適化後 | 目標値 | 達成 |
|------|---------|---------|--------|------|
| 帯域使用量 | __KB/s | __KB/s | <5KB/s | ○/× |
| 同期頻度 | __回/分 | __回/分 | 12回/分 | ○/× |

---

## 3. モジュール別最適化詳細

### M-01: GameManager
**変更内容**:
- __

**効果**:
- CPU削減: __ms/frame

### M-02: NetworkSync
**変更内容**:
- __

**効果**:
- 帯域削減: __KB/s

（以下、全モジュールについて記載）

---

## 4. アセット最適化詳細

### テクスチャ最適化
**変更内容**:
- __個のテクスチャをリサイズ
- 圧縮形式変更: __

**効果**:
- テクスチャメモリ削減: __MB → __MB

### メッシュ最適化
**変更内容**:
- __個のメッシュにLOD設定
- ポリゴン削減: __%

**効果**:
- 三角ポリゴン削減: __個 → __個

### オーディオ最適化
**変更内容**:
- __個のオーディオクリップを圧縮
- Streaming設定: __個

**効果**:
- オーディオメモリ削減: __MB → __MB

---

## 5. Quest専用最適化

### パーティクル削減
- Hit Effectsパーティクル: __個 → __個
- Tree Fallingパーティクル: __個 → __個

### ライティング設定
- リアルタイムライト: __個 → __個
- ベイクドライト: __個追加

### ポストプロセス
- 無効化した効果: __

---

## 6. プロファイリング結果

### Unity Profiler
（スクリーンショットまたは詳細データ添付）

### VRChat Stats Panel
（スクリーンショットまたは詳細データ添付）

---

## 7. 既知の問題・トレードオフ

### 問題1: __
**内容**: __
**影響**: __
**対策**: __

### 問題2: __
**内容**: __
**影響**: __
**対策**: __

---

## 8. 今後の最適化案

### Phase 2での検討事項
1. __
2. __
3. __

---

## 9. 添付資料

- Unity Profilerスクリーンショット
- Frame Debuggerキャプチャ
- VRChat Stats Panelスクリーンショット
- 最適化前後のビルドサイズ比較
```

---

## 9. テストケース

### 9.1 パフォーマンステスト

#### TC-OPT-001: Quest 2フレームレートテスト

**前提条件**:
- Quest 2実機
- ワールドにアップロード済み
- 10人のテストアカウント準備

**手順**:
1. Quest 2でワールドに入室
2. 10人のアカウントを同時参加させる
3. 全員が木を伐採する動作を60秒間継続
4. VRChat Stats Panelを開く
5. FPSを記録

**期待結果**:
- FPS: 60fps以上を維持
- Frame Time: 16.6ms以下
- Performance Rank: Good（緑色）

**実際の結果**:
- FPS: __fps
- Frame Time: __ms
- Performance Rank: __
- 判定: ○合格 / ×不合格

---

#### TC-OPT-002: CPU負荷テスト

**前提条件**:
- Unity Editor
- ClientSim有効
- Profiler起動

**手順**:
1. Unity Editorで再生開始
2. Unity Profiler > CPU Usageタブを開く
3. 60秒間計測
4. Timeline ViewでUpdate()実行時間を確認
5. GC Allocを記録

**期待結果**:
- 全Update()合計: <16.6ms/frame
- GC Alloc: <1KB/frame
- GetComponent()呼び出し: 0回/frame

**実際の結果**:
- Update()合計: __ms/frame
- GC Alloc: __KB/frame
- GetComponent(): __回/frame
- 判定: ○合格 / ×不合格

---

#### TC-OPT-003: GPU負荷テスト

**前提条件**:
- Unity Editor
- Frame Debugger有効

**手順**:
1. Unity EditorでFrame Debuggerを開く
2. 再生開始
3. Enable ボタンをクリック
4. Draw Calls数を確認
5. 三角ポリゴン数を確認

**期待結果**:
- Draw Calls: <50回/frame
- Triangles: <100,000個/frame
- SetPass Calls: <30回/frame

**実際の結果**:
- Draw Calls: __回/frame
- Triangles: __個/frame
- SetPass Calls: __回/frame
- 判定: ○合格 / ×不合格

---

#### TC-OPT-004: ネットワーク帯域テスト

**前提条件**:
- VRChat実機
- 10人のテストアカウント

**手順**:
1. VRChatでワールドに入室
2. 10人が同時に伐採・運搬作業
3. VRChat Stats Panelでネットワーク使用量を確認
4. 60秒間計測

**期待結果**:
- Network Send: <5KB/秒（プレイヤー1人あたり）
- Network Recv: <50KB/秒（全体）
- Ping: <100ms

**実際の結果**:
- Network Send: __KB/秒
- Network Recv: __KB/秒
- Ping: __ms
- 判定: ○合格 / ×不合格

---

#### TC-OPT-005: メモリ使用量テスト

**前提条件**:
- Unity Editor
- Memory Profiler有効

**手順**:
1. Unity Editorで再生開始
2. Memory Profilerタブを開く
3. "Take Sample Editor" ボタンをクリック
4. 総メモリ使用量を確認
5. テクスチャメモリを確認

**期待結果**:
- Total Memory: <500MB
- Texture Memory: <100MB
- Audio Memory: <50MB
- Script Memory: <20MB

**実際の結果**:
- Total Memory: __MB
- Texture Memory: __MB
- Audio Memory: __MB
- Script Memory: __MB
- 判定: ○合格 / ×不合格

---

### 9.2 機能回帰テスト

最適化により既存機能が壊れていないか確認。

#### TC-OPT-101: 伐採システム動作確認

**手順**:
1. 木を斧で攻撃
2. HPが減少することを確認
3. HP=0で倒木アニメーション開始を確認
4. 丸太が正しく生成されることを確認

**期待結果**: 正常動作

**実際の結果**: ○正常 / ×異常（詳細: __）

---

#### TC-OPT-102: スキル成長動作確認

**手順**:
1. 木を伐採してXP獲得
2. XPバーが更新されることを確認
3. レベルアップ時にエフェクトが表示されることを確認

**期待結果**: 正常動作

**実際の結果**: ○正常 / ×異常（詳細: __）

---

#### TC-OPT-103: ネットワーク同期確認

**手順**:
1. 2つのクライアントで入室
2. 片方で木を伐採
3. もう片方でHP変化が同期されることを確認
4. 倒木状態が同期されることを確認

**期待結果**: 正常同期

**実際の結果**: ○正常 / ×異常（詳細: __）

---

## 10. 完了条件（Done Definition）

### 必須条件（すべて満たすこと）

- [ ] **CPU最適化完了**: 全Update()削減、GetComponent()キャッシング完了
- [ ] **GPU最適化完了**: Draw Calls <50回、Triangles <100,000個達成
- [ ] **メモリ最適化完了**: 総メモリ <500MB達成
- [ ] **ネットワーク最適化完了**: 帯域使用量 <5KB/秒達成
- [ ] **Quest 2目標達成**: 60fps以上を10人同時接続時に維持
- [ ] **VRChat Performance Rank**: Good（緑色）達成
- [ ] **全テストケース合格**: TC-OPT-001 ~ TC-OPT-105すべて合格
- [ ] **機能回帰テスト合格**: 既存機能に問題なし
- [ ] **最適化レポート作成**: Performance_Optimization_Report_WI-0024.md完成
- [ ] **コードレビュー完了**: 他開発者による最適化コードレビュー

### 推奨条件（可能な限り満たす）

- [ ] **Quest 3目標達成**: 72fps以上を維持
- [ ] **PC VR目標達成**: 90fps以上を維持
- [ ] **Desktop目標達成**: 120fps以上を維持
- [ ] **メモリ余裕**: 総メモリ <400MB（80%使用）
- [ ] **帯域余裕**: <4KB/秒（80%使用）

---

## 11. 成果物

### コード変更
**変更ファイル**:
- `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`
- `Assets/WoodcutterCamp/Scripts/Core/NetworkSync.cs`
- `Assets/WoodcutterCamp/Scripts/Core/PersistenceManager.cs`
- `Assets/WoodcutterCamp/Scripts/Progression/SkillManager.cs`
- `Assets/WoodcutterCamp/Scripts/Progression/SkillEffects.cs`
- `Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`
- `Assets/WoodcutterCamp/Scripts/Economy/ShopManager.cs`
- `Assets/WoodcutterCamp/Scripts/UI/HUDManager.cs`
- `Assets/WoodcutterCamp/Scripts/UI/LeaderboardUI.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/TreeController.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/AxeInteraction.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/LogSpawner.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/LogPickup.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/CuttingTable.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/CooperativeTracker.cs`
- `Assets/WoodcutterCamp/Scripts/Gameplay/RankingTracker.cs`

### アセット変更
**テクスチャ最適化**:
- `Assets/WoodcutterCamp/Textures/` 配下の全テクスチャ

**メッシュ最適化**:
- `Assets/WoodcutterCamp/Models/` 配下の全3Dモデル

**オーディオ最適化**:
- `Assets/WoodcutterCamp/Audio/` 配下の全音声ファイル

### ドキュメント
**最適化レポート**:
- `docs/Performance_Optimization_Report_WI-0024.md`

**Unity Profilerデータ**:
- `docs/WID/Profiler_Data_Before.png`
- `docs/WID/Profiler_Data_After.png`

**Frame Debuggerキャプチャ**:
- `docs/WID/Frame_Debugger_Before.png`
- `docs/WID/Frame_Debugger_After.png`

---

## 12. 備考

### 注意点

#### パフォーマンス測定の環境依存性
- Unity Editorでの測定値は実機より10-20%低いFPSになる傾向がある
- Quest 2実機での測定を最優先とする
- 複数のQuest 2デバイスで測定し、平均値を記録する

#### 最適化のトレードオフ
- バッチングにより同期遅延が最大5秒発生する（許容範囲内）
- テクスチャ圧縮により若干の画質劣化がある（Quest版のみ）
- LOD設定により遠距離の木の品質が下がる（視認性に影響なし）

#### VRChat Performance Rankの確認方法
```
VRChatクライアント内で確認:
1. メニューを開く
2. Performance & Safety タブ
3. Show Performance Stats

表示される色:
- 緑（Good）: 60fps以上、最適
- 黄（Medium）: 30-60fps、許容範囲
- 赤（Poor）: 30fps未満、要改善
```

#### Unity Profilerの制限
- Deep Profileは非常に重いため、通常のProfileで測定
- Editor ProfileとStandalone Profileで結果が異なる場合あり
- Standalone Build + Profilerで正確な測定を行う

### トラブルシューティング

#### 問題: 最適化後もFPSが目標に達しない
**対策**:
1. Unity Profiler > Timeline Viewで最も重い処理を特定
2. 該当モジュールの最適化を追加実施
3. 必要に応じて機能の削減を検討（Phase 2へ延期）

#### 問題: 最適化後に機能が動作しない
**対策**:
1. 変更前のコードと比較（Git Diff）
2. Update()削減によるイベント駆動の実装ミスを確認
3. キャッシング処理のnullチェック追加

#### 問題: ネットワーク同期がずれる
**対策**:
1. バッチング間隔を5秒から3秒に短縮
2. 重要なイベント（倒木など）は即座に同期
3. OnDeserialization()でLate Joiner対応を確認

### 参考資料

**公式ドキュメント**:
- Unity Profiler Manual: https://docs.unity3d.com/Manual/Profiler.html
- VRChat Performance Rank: https://creators.vrchat.com/worlds/udon/networking/
- Quest最適化ガイド: https://developer.oculus.com/documentation/unity/

**プロジェクト内資料**:
- TDD.md Section 8.3「Other Constraints」（パフォーマンス制約）
- PRD.md Section 3.3「パフォーマンス最適化」
- REF.md Section 8「パフォーマンス最適化」
- VRChat_Tools_API_Reference.md（プロファイリングツール）

**技術調査資料**:
- docs/VRChat_TechResearch_2025.md（Quest最適化詳細）

### コーディング規約（最適化版）

**Update()削減パターン**:
```csharp
// ✅ 推奨: タイマーベース
private float updateInterval = 1.0f;
private float nextUpdateTime = 0;

void Update()
{
    if (Time.time >= nextUpdateTime)
    {
        DoPeriodicUpdate();
        nextUpdateTime = Time.time + updateInterval;
    }
}

// ✅ 推奨: イベント駆動
public void _OnDataChanged()
{
    UpdateDisplay();
}
```

**GetComponent()キャッシングパターン**:
```csharp
// ✅ 推奨: Start()でキャッシング
private Rigidbody rb;
private Renderer rend;

void Start()
{
    rb = GetComponent<Rigidbody>();
    rend = GetComponent<Renderer>();

    // nullチェック
    if (rb == null)
    {
        Debug.LogError("[ModuleName] Rigidbody not found");
    }
}

void Update()
{
    // キャッシュを使用
    rb.velocity = Vector3.forward;
}
```

**GC削減パターン**:
```csharp
// ✅ 推奨: StringBuilderで文字列構築
using System.Text;

private StringBuilder sb = new StringBuilder(256);

private void UpdateText()
{
    sb.Clear();
    sb.Append("Score: ");
    sb.Append(score);
    text.text = sb.ToString();
}

// ✅ 推奨: 配列再利用
private int[] tempArray = new int[20];

private void ProcessData()
{
    // 既存配列を再利用（new不要）
    for (int i = 0; i < tempArray.Length; i++)
    {
        tempArray[i] = 0;
    }
}
```

### Quest専用ビルドチェックリスト

最終ビルド前に以下を確認:

- [ ] **Platform**: Android設定
- [ ] **Graphics API**: OpenGLES3, Vulkan
- [ ] **Texture Compression**: ASTC有効
- [ ] **Shadow Distance**: 0
- [ ] **Lightmap Static**: 全静的オブジェクトに設定
- [ ] **Particle Max Count**: 全て30以下
- [ ] **Real-time Lights**: 2個以下
- [ ] **Post Processing**: 無効化
- [ ] **LOD Groups**: 全3Dモデルに設定

---

## 13. 最適化実施スケジュール

### Day 1（1日目）

**午前（4時間）**:
- Phase 1: パフォーマンス監査
  - Unity Profilerセットアップ（0.5h）
  - ベースライン測定（1h）
  - ボトルネック特定（2h）
  - 最適化計画レビュー（0.5h）

**午後（4時間）**:
- Phase 2: モジュール別最適化（前半）
  - Core Layer最適化（M-01, M-02, M-03）（2h）
  - Progression Layer最適化（M-04, M-05）（1h）
  - Economy Layer最適化（M-06, M-07）（1h）

### Day 2（2日目）

**午前（4時間）**:
- Phase 2: モジュール別最適化（後半）
  - UI Layer最適化（M-08, M-09）（1h）
  - Gameplay Layer最適化（M-10 ~ M-16）（3h）

**午後（4時間）**:
- Phase 3: アセット最適化（2h）
  - テクスチャ最適化（1h）
  - メッシュ・オーディオ最適化（1h）
- Phase 4: Quest専用ビルド設定（1h）
- 最終テスト・レポート作成（1h）

---

**作業指示書承認**:
- [ ] 技術リード承認
- [ ] アーキテクト承認
- [ ] 作業開始承認

**バージョン管理**:
- 初版作成: 2025-11-17
- 最終更新: 2025-11-17
- ステータス: レビュー待ち

---

**関連Work Instructions**:
- 依存: WI-0001 ~ WI-0023（全実装完了が前提）
- 次工程: WI-0025 Quest Build & Testing
