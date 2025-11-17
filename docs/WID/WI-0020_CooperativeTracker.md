# WI-0020: CooperativeTracker Implementation

**作業指示書バージョン**: 1.0
**最終更新日**: 2025-11-17
**対象フェーズ**: Phase 2（Week 4）
**優先度**: 高
**推定作業時間**: 1.5日

---

## 1. 作業概要

### 1.1 作業ID
**WI-0020**

### 1.2 作業名
協力伐採トラッカーシステムの実装（CooperativeTracker Implementation）

### 1.3 作業目的
複数プレイヤーが3秒以内に同じ木を攻撃した際に協力伐採を検出し、ダメージ・XPに+20%ボーナスを適用するシステムを実装する。これにより、プレイヤー間の自然な協力プレイを促進し、会話のきっかけを創出する。

### 1.4 対象モジュール
- **Module ID**: M-15 (CooperativeTracker)
- **Layer**: Gameplay Layer
- **関連機能**: F-50, F-51, F-52, F-53

---

## 2. 前提条件

### 2.1 依存する作業指示書
以下の作業が完了していることを前提とする：
- **WI-0007**: TreeController State Machine（木の状態管理）
- **WI-0009**: AxeInteraction Detection（斧のスイング判定）
- **WI-0010**: Damage Calculation System（ダメージ計算）
- **WI-0002**: NetworkSync Batching（ネットワーク同期基盤）

### 2.2 環境条件
- **Unity**: 2022.3.22f1 LTS
- **VRChat SDK**: 3.9.0
- **UdonSharp**: 1.1.9
- **開発ブランチ**: `feature/cooperative-tracker`

### 2.3 初期状態
- GameManagerが初期化済み
- TreeControllerが30本の木を管理中
- NetworkSyncが稼働中
- 複数プレイヤーがインスタンスに存在する（テストは最低2人）

---

## 3. 機能仕様

### 3.1 協力伐採の定義
**協力伐採（Cooperative Chopping）**とは、以下の条件を満たす状態を指す：
- 同じ木（treeID）に対して
- 2人以上の異なるプレイヤーが
- 3秒以内に攻撃を行った場合

### 3.2 実装する機能

#### F-50: 攻撃タイムスタンプの記録（Track Hit Timestamps）
- 各プレイヤーが木を攻撃した際、以下を記録する：
  - 木のID（treeID: 0〜29）
  - プレイヤーID（VRCPlayerApi.playerId）
  - 攻撃時刻（Time.time）
- データ構造は並列配列方式を採用（UdonSharpの制約のため）
- 最大容量：20本の木 × 10プレイヤー × 5回分の履歴 = 1000エントリ

#### F-51: 協力伐採の判定（Detect Cooperative Chopping）
- 木への攻撃発生時、以下をチェック：
  1. 対象の木（treeID）への過去3秒以内の攻撃履歴を取得
  2. 重複を除外したプレイヤー数をカウント
  3. 2人以上なら協力伐採と判定
- 判定結果をAxeControllerに返却（bool値）

#### F-52: ボーナス計算（Calculate Cooperative Bonus）
- 協力伐採判定がtrueの場合、以下のボーナスを適用：
  - **ダメージ**: +20%（1.2x倍率）
  - **XP**: +20%（1.2x倍率）
  - **コイン**: +2コイン（固定加算）
- ボーナス計算はDamageCalculation（WI-0010）に統合

#### F-53: 視覚フィードバック（Visual Feedback）
- 協力伐採発動時、以下の演出を実行：
  - **パーティクルエフェクト**: 金色の輝き（木の周囲）
  - **画面オーバーレイ**: 「協力伐採！」テキストを1秒表示
  - **サウンドエフェクト**: 「キラーン」という爽快音
  - **アイコン**: 握手マーク（🤝）を木の上部に3秒表示

---

## 4. データ構造設計

### 4.1 タイムスタンプ管理用データ構造

UdonSharpはDictionary<TKey, TValue>やListが使用できないため、**並列配列方式**で実装する。

```csharp
// 最大容量定数
private const int MAX_TREES = 30;
private const int MAX_PLAYERS = 20;
private const int MAX_HISTORY_PER_PLAYER = 5; // 1プレイヤーあたり過去5回分
private const int TOTAL_ENTRIES = MAX_TREES * MAX_PLAYERS * MAX_HISTORY_PER_PLAYER; // 3000

// タイムスタンプ記録用配列（事前割り当て）
private int[] entryTreeIDs = new int[TOTAL_ENTRIES];       // 木ID
private int[] entryPlayerIDs = new int[TOTAL_ENTRIES];     // プレイヤーID
private float[] entryTimestamps = new float[TOTAL_ENTRIES]; // Time.time
private int currentEntryIndex = 0; // 次に書き込むインデックス（リングバッファ）

// ネットワーク同期用（Manual Sync）
[UdonSynced] private int syncedEntryCount = 0;
[UdonSynced] private int[] syncedTreeIDs = new int[100];    // 直近100エントリのみ同期
[UdonSynced] private int[] syncedPlayerIDs = new int[100];
[UdonSynced] private float[] syncedTimestamps = new float[100];
```

**設計理由**:
- UdonSharpの制約により、Dictionary/Listは使用不可
- 配列は事前に固定サイズで確保し、リングバッファとして運用
- ネットワーク同期は直近100エントリのみに制限（帯域節約）

### 4.2 データフロー図

```
[AxeController]
   ↓ (OnTreeHit イベント)
[CooperativeTracker.RecordHit(treeID, playerID)]
   ↓
[並列配列に記録]
   ↓
[IsCooperativeHit(treeID, playerID) を呼び出し]
   ↓ (過去3秒の履歴をスキャン)
[2人以上なら true を返却]
   ↓
[AxeController] → DamageCalculation（ボーナス適用）
   ↓
[CooperativeTracker.ShowCooperativeEffect(treeID)]
   ↓
[パーティクル・UI・サウンド再生]
```

---

## 5. 実装詳細

### 5.1 ファイル構成

#### 作成するファイル
1. **CooperativeTracker.cs** (UdonSharpスクリプト)
   - パス: `/Assets/WoodcutterCamp/Scripts/Gameplay/CooperativeTracker.cs`
   - 行数: 約550〜650行

2. **CooperativeEffects.prefab** (プレハブ)
   - パス: `/Assets/WoodcutterCamp/Prefabs/Effects/CooperativeEffects.prefab`
   - 内容: パーティクルシステム、UIオーバーレイ、AudioSource

#### 編集するファイル
- `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeInteraction.cs` (連携コード追加)
- `/Assets/WoodcutterCamp/Scripts/Gameplay/DamageCalculation.cs` (ボーナス計算追加)

### 5.2 CooperativeTracker.cs 完全実装コード

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

/// <summary>
/// 協力伐採システム
/// 複数プレイヤーが3秒以内に同じ木を攻撃した際に協力判定を行い、ボーナスを適用する
/// </summary>
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class CooperativeTracker : UdonSharpBehaviour
{
    #region Constants
    private const int MAX_TREES = 30;
    private const int MAX_PLAYERS = 20;
    private const int MAX_HISTORY_PER_PLAYER = 5;
    private const int TOTAL_ENTRIES = MAX_TREES * MAX_PLAYERS * MAX_HISTORY_PER_PLAYER;
    private const float COOPERATIVE_TIME_WINDOW = 3.0f; // 協力判定の時間窓（秒）
    private const float CLEANUP_INTERVAL = 5.0f; // タイムスタンプクリーンアップ間隔（秒）
    private const float COOPERATIVE_DAMAGE_BONUS = 1.2f; // ダメージボーナス倍率
    private const int COOPERATIVE_COIN_BONUS = 2; // コインボーナス（固定値）
    #endregion

    #region Serialized Fields
    [Header("Dependencies")]
    [SerializeField] private GameManager gameManager;
    [SerializeField] private NetworkSync networkSync;

    [Header("Visual Effects")]
    [SerializeField] private GameObject cooperativeEffectPrefab; // パーティクル用プレハブ
    [SerializeField] private Canvas cooperativeUICanvas; // 「協力伐採!」表示用Canvas
    [SerializeField] private UnityEngine.UI.Text cooperativeText; // UIテキスト
    [SerializeField] private GameObject handshakeIconPrefab; // 握手アイコン（🤝）

    [Header("Audio")]
    [SerializeField] private AudioSource audioSource;
    [SerializeField] private AudioClip cooperativeSound; // キラーン音
    #endregion

    #region Private Fields - Timestamp Storage
    // ローカルタイムスタンプ記録（リングバッファ）
    private int[] entryTreeIDs;
    private int[] entryPlayerIDs;
    private float[] entryTimestamps;
    private int currentEntryIndex = 0;

    // ネットワーク同期用（直近100エントリのみ）
    [UdonSynced] private int syncedEntryCount = 0;
    [UdonSynced] private int[] syncedTreeIDs;
    [UdonSynced] private int[] syncedPlayerIDs;
    [UdonSynced] private float[] syncedTimestamps;

    // クリーンアップタイマー
    private float nextCleanupTime = 0f;

    // 協力伐採発動中の木ID追跡（重複通知防止）
    private bool[] isCooperativeActive;
    private float[] cooperativeActivationTimes;

    // エフェクトオブジェクトプール
    private GameObject[] effectPool;
    private int effectPoolSize = 5;
    private int nextEffectIndex = 0;

    // キャッシュされた参照
    private VRCPlayerApi localPlayer;
    #endregion

    #region Unity Lifecycle
    void Start()
    {
        _Initialize();
    }

    void Update()
    {
        // 定期的な古いタイムスタンプのクリーンアップ
        if (Time.time >= nextCleanupTime)
        {
            _CleanupOldTimestamps();
            nextCleanupTime = Time.time + CLEANUP_INTERVAL;
        }
    }
    #endregion

    #region Public API - Udon Custom Events
    /// <summary>
    /// 初期化処理（GameManagerから呼ばれる）
    /// </summary>
    public void _Initialize()
    {
        Debug.Log("[CooperativeTracker] Initializing...");

        // 配列の事前確保
        entryTreeIDs = new int[TOTAL_ENTRIES];
        entryPlayerIDs = new int[TOTAL_ENTRIES];
        entryTimestamps = new float[TOTAL_ENTRIES];

        syncedTreeIDs = new int[100];
        syncedPlayerIDs = new int[100];
        syncedTimestamps = new float[100];

        isCooperativeActive = new bool[MAX_TREES];
        cooperativeActivationTimes = new float[MAX_TREES];

        // 配列の初期化（-1 = 未使用エントリ）
        for (int i = 0; i < TOTAL_ENTRIES; i++)
        {
            entryTreeIDs[i] = -1;
            entryPlayerIDs[i] = -1;
            entryTimestamps[i] = 0f;
        }

        // エフェクトオブジェクトプールの作成
        InitializeEffectPool();

        // ローカルプレイヤー参照をキャッシュ
        localPlayer = Networking.LocalPlayer;

        // UIの初期非表示
        if (cooperativeUICanvas != null)
        {
            cooperativeUICanvas.gameObject.SetActive(false);
        }

        Debug.Log("[CooperativeTracker] Initialization completed");
    }

    /// <summary>
    /// 木への攻撃を記録する（AxeInteractionから呼ばれる）
    /// </summary>
    /// <param name="treeID">木のID（0〜29）</param>
    /// <param name="playerID">プレイヤーID</param>
    public void _RecordHit(int treeID, int playerID)
    {
        if (!ValidateTreeID(treeID))
        {
            Debug.LogWarning($"[CooperativeTracker] Invalid treeID: {treeID}");
            return;
        }

        if (!ValidatePlayerID(playerID))
        {
            Debug.LogWarning($"[CooperativeTracker] Invalid playerID: {playerID}");
            return;
        }

        // タイムスタンプを記録
        float hitTime = Time.time;

        // リングバッファに追加
        entryTreeIDs[currentEntryIndex] = treeID;
        entryPlayerIDs[currentEntryIndex] = playerID;
        entryTimestamps[currentEntryIndex] = hitTime;

        currentEntryIndex = (currentEntryIndex + 1) % TOTAL_ENTRIES;

        // ネットワーク同期（Master Clientのみ）
        if (Networking.IsMaster)
        {
            SyncTimestampToNetwork(treeID, playerID, hitTime);
        }

        Debug.Log($"[CooperativeTracker] Recorded hit: Tree={treeID}, Player={playerID}, Time={hitTime}");
    }

    /// <summary>
    /// 協力伐採判定を行う（AxeInteractionから呼ばれる）
    /// </summary>
    /// <param name="treeID">木のID</param>
    /// <param name="playerID">攻撃したプレイヤーID</param>
    /// <returns>協力伐採が成立したか</returns>
    public bool _IsCooperativeHit(int treeID, int playerID)
    {
        if (!ValidateTreeID(treeID) || !ValidatePlayerID(playerID))
        {
            return false;
        }

        float currentTime = Time.time;
        float windowStart = currentTime - COOPERATIVE_TIME_WINDOW;

        // 過去3秒以内に攻撃したユニークプレイヤーをカウント
        bool[] playerHitFlags = new bool[MAX_PLAYERS];
        int uniquePlayerCount = 0;

        for (int i = 0; i < TOTAL_ENTRIES; i++)
        {
            // 対象の木かつ有効なタイムスタンプかチェック
            if (entryTreeIDs[i] == treeID &&
                entryTimestamps[i] >= windowStart &&
                entryTimestamps[i] <= currentTime)
            {
                int pid = entryPlayerIDs[i];
                if (pid >= 0 && pid < MAX_PLAYERS && !playerHitFlags[pid])
                {
                    playerHitFlags[pid] = true;
                    uniquePlayerCount++;
                }
            }
        }

        bool isCooperative = uniquePlayerCount >= 2;

        if (isCooperative)
        {
            Debug.Log($"[CooperativeTracker] Cooperative hit detected! Tree={treeID}, Players={uniquePlayerCount}");

            // 視覚フィードバックの表示（重複防止チェック）
            if (!isCooperativeActive[treeID] ||
                (currentTime - cooperativeActivationTimes[treeID] > 1.0f))
            {
                _ShowCooperativeEffect(treeID);
                isCooperativeActive[treeID] = true;
                cooperativeActivationTimes[treeID] = currentTime;
            }
        }

        return isCooperative;
    }

    /// <summary>
    /// 協力ボーナス倍率を取得（DamageCalculationから呼ばれる）
    /// </summary>
    /// <returns>ダメージボーナス倍率（協力時1.2、通常時1.0）</returns>
    public float _GetCooperativeDamageBonus()
    {
        return COOPERATIVE_DAMAGE_BONUS;
    }

    /// <summary>
    /// 協力ボーナスコインを取得
    /// </summary>
    /// <returns>追加コイン数</returns>
    public int _GetCooperativeCoinBonus()
    {
        return COOPERATIVE_COIN_BONUS;
    }

    /// <summary>
    /// 協力伐採エフェクトを表示
    /// </summary>
    /// <param name="treeID">木のID</param>
    public void _ShowCooperativeEffect(int treeID)
    {
        // 木の位置を取得（TreeControllerから）
        GameObject treeObject = GetTreeObject(treeID);
        if (treeObject == null)
        {
            Debug.LogWarning($"[CooperativeTracker] Tree object not found: {treeID}");
            return;
        }

        Vector3 treePosition = treeObject.transform.position;

        // パーティクルエフェクトを再生
        PlayCooperativeParticle(treePosition);

        // サウンドエフェクトを再生
        PlayCooperativeSound();

        // UIオーバーレイを表示（ローカルプレイヤーのみ）
        ShowCooperativeUI();

        // 握手アイコンを木の上に表示
        ShowHandshakeIcon(treePosition);
    }
    #endregion

    #region Private Methods - Timestamp Management
    /// <summary>
    /// 古いタイムスタンプをクリーンアップ
    /// </summary>
    private void _CleanupOldTimestamps()
    {
        float currentTime = Time.time;
        float threshold = currentTime - (COOPERATIVE_TIME_WINDOW + 2.0f); // 余裕を持って5秒前

        int cleanedCount = 0;

        for (int i = 0; i < TOTAL_ENTRIES; i++)
        {
            if (entryTimestamps[i] > 0f && entryTimestamps[i] < threshold)
            {
                entryTreeIDs[i] = -1;
                entryPlayerIDs[i] = -1;
                entryTimestamps[i] = 0f;
                cleanedCount++;
            }
        }

        if (cleanedCount > 0)
        {
            Debug.Log($"[CooperativeTracker] Cleaned up {cleanedCount} old timestamps");
        }
    }

    /// <summary>
    /// タイムスタンプをネットワーク同期
    /// </summary>
    private void SyncTimestampToNetwork(int treeID, int playerID, float timestamp)
    {
        // 直近100エントリのみ同期（リングバッファ）
        int syncIndex = syncedEntryCount % 100;

        syncedTreeIDs[syncIndex] = treeID;
        syncedPlayerIDs[syncIndex] = playerID;
        syncedTimestamps[syncIndex] = timestamp;

        syncedEntryCount++;

        // NetworkSyncにリクエスト
        if (networkSync != null)
        {
            RequestSerialization();
        }
    }
    #endregion

    #region Private Methods - Visual Effects
    /// <summary>
    /// エフェクトオブジェクトプールを初期化
    /// </summary>
    private void InitializeEffectPool()
    {
        if (cooperativeEffectPrefab == null)
        {
            Debug.LogWarning("[CooperativeTracker] cooperativeEffectPrefab is not assigned");
            return;
        }

        effectPool = new GameObject[effectPoolSize];

        for (int i = 0; i < effectPoolSize; i++)
        {
            GameObject effect = Instantiate(cooperativeEffectPrefab);
            effect.SetActive(false);
            effect.transform.SetParent(transform);
            effectPool[i] = effect;
        }
    }

    /// <summary>
    /// 協力パーティクルを再生
    /// </summary>
    private void PlayCooperativeParticle(Vector3 position)
    {
        if (effectPool == null || effectPool.Length == 0)
        {
            return;
        }

        // オブジェクトプールから取得
        GameObject effect = effectPool[nextEffectIndex];
        nextEffectIndex = (nextEffectIndex + 1) % effectPoolSize;

        effect.transform.position = position + Vector3.up * 2.0f; // 木の2m上
        effect.SetActive(true);

        // パーティクルシステムを再生
        ParticleSystem ps = effect.GetComponent<ParticleSystem>();
        if (ps != null)
        {
            ps.Play();
        }

        // 3秒後に非アクティブ化
        SendCustomEventDelayedSeconds(nameof(_DeactivateEffect), 3.0f);
    }

    public void _DeactivateEffect()
    {
        // 最も古いエフェクトを非アクティブ化
        int deactivateIndex = (nextEffectIndex - 1 + effectPoolSize) % effectPoolSize;
        if (effectPool != null && deactivateIndex < effectPool.Length)
        {
            effectPool[deactivateIndex].SetActive(false);
        }
    }

    /// <summary>
    /// 協力サウンドを再生
    /// </summary>
    private void PlayCooperativeSound()
    {
        if (audioSource != null && cooperativeSound != null)
        {
            audioSource.PlayOneShot(cooperativeSound);
        }
    }

    /// <summary>
    /// 協力UIオーバーレイを表示
    /// </summary>
    private void ShowCooperativeUI()
    {
        if (cooperativeUICanvas == null || cooperativeText == null)
        {
            return;
        }

        cooperativeUICanvas.gameObject.SetActive(true);
        cooperativeText.text = "協力伐採！";

        // 1秒後にフェードアウト
        SendCustomEventDelayedSeconds(nameof(_HideCooperativeUI), 1.0f);
    }

    public void _HideCooperativeUI()
    {
        if (cooperativeUICanvas != null)
        {
            cooperativeUICanvas.gameObject.SetActive(false);
        }
    }

    /// <summary>
    /// 握手アイコンを表示
    /// </summary>
    private void ShowHandshakeIcon(Vector3 position)
    {
        if (handshakeIconPrefab == null)
        {
            return;
        }

        GameObject icon = Instantiate(handshakeIconPrefab);
        icon.transform.position = position + Vector3.up * 3.0f; // 木の3m上

        // 3秒後に削除
        Destroy(icon, 3.0f);
    }
    #endregion

    #region Private Methods - Utility
    /// <summary>
    /// 木IDの妥当性チェック
    /// </summary>
    private bool ValidateTreeID(int treeID)
    {
        return treeID >= 0 && treeID < MAX_TREES;
    }

    /// <summary>
    /// プレイヤーIDの妥当性チェック
    /// </summary>
    private bool ValidatePlayerID(int playerID)
    {
        return playerID >= 0 && playerID < MAX_PLAYERS;
    }

    /// <summary>
    /// 木オブジェクトを取得
    /// </summary>
    private GameObject GetTreeObject(int treeID)
    {
        // GameManagerからTreeController経由で取得
        if (gameManager != null)
        {
            // Note: 実際の実装ではGameManager.GetTreeController().GetTreeObject(treeID)
            // ここでは簡略化のため、GameObject.Findを使用（実装時は要修正）
            GameObject[] trees = GameObject.FindGameObjectsWithTag("Tree");
            if (treeID < trees.Length)
            {
                return trees[treeID];
            }
        }
        return null;
    }
    #endregion

    #region Network Sync - Udon Events
    public override void OnDeserialization()
    {
        // Late Joiner がインスタンスに参加した際、同期データをローカルに反映
        Debug.Log($"[CooperativeTracker] OnDeserialization: Synced {syncedEntryCount} entries");

        // 同期データをローカル配列にマージ
        for (int i = 0; i < Mathf.Min(syncedEntryCount, 100); i++)
        {
            int treeID = syncedTreeIDs[i];
            int playerID = syncedPlayerIDs[i];
            float timestamp = syncedTimestamps[i];

            // ローカル配列に追加（重複チェックは省略）
            entryTreeIDs[currentEntryIndex] = treeID;
            entryPlayerIDs[currentEntryIndex] = playerID;
            entryTimestamps[currentEntryIndex] = timestamp;

            currentEntryIndex = (currentEntryIndex + 1) % TOTAL_ENTRIES;
        }
    }
    #endregion
}
```

---

## 6. Unity Editor セットアップ手順

### 6.1 Scene Hierarchy 構成

1. **CooperativeTrackerオブジェクトの作成**
   - Hierarchy右クリック → Create Empty
   - 名前を「CooperativeTracker」に変更
   - Position: (0, 0, 0)

2. **CooperativeTracker.csをアタッチ**
   - CooperativeTrackerオブジェクトを選択
   - Add Component → CooperativeTracker
   - UdonBehaviourが自動的に追加される

3. **Dependencies設定**
   - `Game Manager`: Hierarchy内のGameManagerをドラッグ&ドロップ
   - `Network Sync`: Hierarchy内のNetworkSyncをドラッグ&ドロップ

### 6.2 パーティクルエフェクトの作成

1. **Particle Systemの作成**
   - Hierarchy右クリック → Effects → Particle System
   - 名前を「CooperativeParticle」に変更

2. **パーティクル設定**
   ```
   Main:
     - Start Lifetime: 1.5
     - Start Speed: 3
     - Start Size: 0.3
     - Start Color: Gold (255, 215, 0)
     - Simulation Space: World

   Emission:
     - Rate over Time: 50

   Shape:
     - Shape: Sphere
     - Radius: 1.5

   Color over Lifetime:
     - グラデーション: Gold → Transparent

   Size over Lifetime:
     - カーブ: 0.3 → 0.1
   ```

3. **Prefab化**
   - CooperativeParticleをProject Viewの`Assets/WoodcutterCamp/Prefabs/Effects/`にドラッグ
   - Hierarchyからは削除

4. **CooperativeTrackerに設定**
   - CooperativeTrackerオブジェクトを選択
   - `Cooperative Effect Prefab`にCooperativeParticle Prefabをドラッグ

### 6.3 UIオーバーレイの作成

1. **Canvas作成**
   - Hierarchy右クリック → UI → Canvas
   - 名前を「CooperativeUICanvas」に変更
   - Render Mode: Screen Space - Overlay
   - Canvas Scalerを追加（UI Scale Mode: Scale With Screen Size）

2. **Textオブジェクト作成**
   - CooperativeUICanvas右クリック → UI → Text
   - 名前を「CooperativeText」に変更
   - 設定：
     ```
     Rect Transform:
       - Anchor: Middle Center
       - Pos X: 0, Pos Y: 200
       - Width: 400, Height: 100

     Text:
       - Font Size: 60
       - Alignment: Center
       - Color: Gold (255, 215, 0)
       - Font: Bold
       - Text: "協力伐採！"

     Outline:
       - Effect Color: Black
       - Effect Distance: (3, -3)
     ```

3. **CooperativeTrackerに設定**
   - `Cooperative UI Canvas`: CooperativeUICanvasをドラッグ
   - `Cooperative Text`: CooperativeTextをドラッグ

4. **初期状態を非表示に**
   - CooperativeUICanvasのInspectorで、ゲームオブジェクトのチェックを外す

### 6.4 握手アイコンの作成

1. **3D Textオブジェクト作成**
   - Hierarchy右クリック → 3D Object → 3D Text
   - 名前を「HandshakeIcon」に変更

2. **設定**
   ```
   TextMesh:
     - Text: "🤝"
     - Font Size: 100
     - Anchor: Middle Center
     - Alignment: Center
     - Color: White

   Transform:
     - Scale: (0.1, 0.1, 0.1)
   ```

3. **ビルボードスクリプトを追加**
   - HandshakeIconに新規スクリプト「Billboard.cs」を追加（後述）

4. **Prefab化**
   - HandshakeIconを`Assets/WoodcutterCamp/Prefabs/UI/`にドラッグ
   - CooperativeTrackerの`Handshake Icon Prefab`に設定

### 6.5 AudioSourceの設定

1. **AudioSource追加**
   - CooperativeTrackerオブジェクトを選択
   - Add Component → Audio Source
   - 設定：
     ```
     - Play On Awake: OFF
     - Spatial Blend: 0 (2D)
     - Volume: 0.7
     ```

2. **サウンドファイルの準備**
   - 協力伐採音（キラーン音）を`Assets/WoodcutterCamp/Audio/`に配置
   - ファイル名: `cooperative_chime.wav`

3. **CooperativeTrackerに設定**
   - `Audio Source`: 自動的に検出される
   - `Cooperative Sound`: cooperative_chime.wavをドラッグ

### 6.6 Billboard.cs（補助スクリプト）

```csharp
using UdonSharp;
using UnityEngine;

/// <summary>
/// 常にカメラの方を向くビルボードスクリプト
/// </summary>
public class Billboard : UdonSharpBehaviour
{
    private Camera mainCamera;

    void Start()
    {
        mainCamera = Camera.main;
    }

    void LateUpdate()
    {
        if (mainCamera != null)
        {
            transform.LookAt(transform.position + mainCamera.transform.rotation * Vector3.forward,
                             mainCamera.transform.rotation * Vector3.up);
        }
    }
}
```

---

## 7. AxeInteractionとの連携実装

### 7.1 AxeInteraction.cs への追加コード

`AxeInteraction.cs` の該当部分に以下のコードを追加：

```csharp
// ===== CooperativeTracker連携用フィールド =====
[Header("Cooperative Tracking")]
[SerializeField] private CooperativeTracker cooperativeTracker;

// ===== OnTreeHit メソッド内に追加 =====
public void _OnTreeHit(int treeID, Vector3 hitPosition)
{
    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    int playerID = localPlayer.playerId;

    // 協力伐採トラッカーに記録
    if (cooperativeTracker != null)
    {
        cooperativeTracker._RecordHit(treeID, playerID);

        // 協力伐採判定
        bool isCooperative = cooperativeTracker._IsCooperativeHit(treeID, playerID);

        // ダメージ計算にボーナスを渡す
        float damageMultiplier = isCooperative ?
            cooperativeTracker._GetCooperativeDamageBonus() : 1.0f;

        // DamageCalculationを呼び出し
        int finalDamage = CalculateDamage(baseDamage, damageMultiplier);

        // TreeControllerにダメージを通知
        SendDamageToTree(treeID, finalDamage);

        // 協力ボーナスコインの付与
        if (isCooperative)
        {
            int bonusCoins = cooperativeTracker._GetCooperativeCoinBonus();
            coinManager._AddCoins(bonusCoins);
        }
    }
}
```

---

## 8. テスト仕様

### 8.1 単体テスト（Unit Test）

#### テストケース1: タイムスタンプ記録の正常動作
**目的**: `_RecordHit`が正しくタイムスタンプを記録するか確認

**手順**:
1. Unity Editorで再生ボタンを押す
2. Console Windowを開く
3. AxeInteractionから`cooperativeTracker._RecordHit(0, 1001)`を呼び出す
4. Consoleに`[CooperativeTracker] Recorded hit: Tree=0, Player=1001, Time=X.XXX`が表示される

**期待結果**:
- エラーなくログが出力される
- 2回目の呼び出しで異なるタイムスタンプが記録される

---

#### テストケース2: 協力伐採判定（2人）
**目的**: 3秒以内に2人が攻撃した際に協力判定がtrueになるか確認

**手順**:
1. Unity Editorで再生
2. `_RecordHit(0, 1001)`を呼び出し（プレイヤー1）
3. 0.5秒待機
4. `_RecordHit(0, 1002)`を呼び出し（プレイヤー2）
5. `_IsCooperativeHit(0, 1002)`を呼び出し

**期待結果**:
- `_IsCooperativeHit`が`true`を返す
- Consoleに`Cooperative hit detected! Tree=0, Players=2`が表示

---

#### テストケース3: 協力伐採判定失敗（3秒超過）
**目的**: 3秒を超えた場合に協力判定がfalseになるか確認

**手順**:
1. `_RecordHit(0, 1001)`を呼び出し
2. 4秒待機
3. `_RecordHit(0, 1002)`を呼び出し
4. `_IsCooperativeHit(0, 1002)`を呼び出し

**期待結果**:
- `_IsCooperativeHit`が`false`を返す

---

#### テストケース4: タイムスタンプクリーンアップ
**目的**: 古いタイムスタンプが正しく削除されるか確認

**手順**:
1. 10個のタイムスタンプを記録
2. 6秒待機（CLEANUP_INTERVALが5秒）
3. Consoleを確認

**期待結果**:
- `[CooperativeTracker] Cleaned up X old timestamps`が表示される

---

### 8.2 結合テスト（Integration Test）

#### テストケース5: AxeInteractionとの連携
**目的**: 実際の斧攻撃から協力判定までが正しく動作するか確認

**手順**:
1. VRChat ClientSimを起動（Tools → VRChat SDK → Test Client Sim）
2. 2人のプレイヤー（Player1, Player2）をシミュレート
3. Player1が木ID=0を攻撃
4. 1秒後、Player2が木ID=0を攻撃
5. パーティクル、UI、サウンドが再生されるか確認

**期待結果**:
- 金色のパーティクルが木の周囲に表示
- 「協力伐採！」テキストが画面中央に1秒表示
- キラーン音が再生
- 握手アイコン（🤝）が木の上に3秒表示

---

#### テストケース6: ネットワーク同期
**目的**: Master Clientが退出した際、新しいMaster Clientにデータが引き継がれるか確認

**手順**:
1. VRChat Testワールドをビルド＆パブリッシュ
2. VRChatクライアントAでインスタンス作成（Master Client）
3. クライアントAが木ID=0を攻撃
4. VRChatクライアントBが参加（Late Joiner）
5. クライアントAが退出
6. クライアントBが木ID=0を攻撃（新しいMaster Clientとして）
7. クライアントCが参加し、木ID=0を攻撃

**期待結果**:
- クライアントCが攻撃した際、協力判定が発動する（クライアントBの履歴が引き継がれている）

---

### 8.3 パフォーマンステスト

#### テストケース7: Quest 2環境でのFPS計測
**目的**: Quest 2環境で10人同時攻撃時に60fps維持できるか確認

**手順**:
1. Quest 2実機でワールドをテスト
2. VRChat Perfボタンでfps表示を有効化
3. 10人のプレイヤーが同時に木を攻撃（Client Simで模擬）
4. FPS値を記録

**期待結果**:
- FPS: 60fps以上維持
- パーティクル再生中も58fps以上

---

#### テストケース8: メモリ使用量
**目的**: CooperativeTrackerのメモリ使用量が許容範囲内か確認

**手順**:
1. Unity Profilerを起動（Window → Analysis → Profiler）
2. VRChat ClientSimで1時間プレイ
3. Memory使用量を記録

**期待結果**:
- Total Memory増加量: 5MB以下
- 配列の事前確保により、動的メモリ確保が発生しない

---

### 8.4 異常系テスト

#### テストケース9: 無効なtreeIDの処理
**目的**: 範囲外のtreeIDが渡された際にエラーハンドリングされるか確認

**手順**:
1. `_RecordHit(99, 1001)`を呼び出し（treeID範囲外）
2. Consoleを確認

**期待結果**:
- `[CooperativeTracker] Invalid treeID: 99`というWarningが表示される
- アプリケーションがクラッシュしない

---

#### テストケース10: 参照未設定時の動作
**目的**: `cooperativeEffectPrefab`がnullの際に安全に動作するか確認

**手順**:
1. CooperativeTrackerの`Cooperative Effect Prefab`をNoneに設定
2. 協力伐採を発動
3. Consoleを確認

**期待結果**:
- `[CooperativeTracker] cooperativeEffectPrefab is not assigned`というWarningが表示
- パーティクルは表示されないが、その他の機能（UI、サウンド）は正常動作

---

## 9. 完了条件（Done Definition）

以下の全ての条件を満たした場合、本作業は完了とする：

- [ ] **コード実装完了**
  - [ ] CooperativeTracker.cs（550〜650行）が完成
  - [ ] Billboard.cs（補助スクリプト）が完成
  - [ ] AxeInteraction.csへの連携コード追加完了

- [ ] **Unity Editor設定完了**
  - [ ] CooperativeTrackerオブジェクトがScene内に配置
  - [ ] Dependencies（GameManager, NetworkSync）が設定済み
  - [ ] パーティクルエフェクトPrefabが作成＆設定済み
  - [ ] UIオーバーレイ（Canvas, Text）が作成＆設定済み
  - [ ] 握手アイコンPrefabが作成＆設定済み
  - [ ] AudioSourceとサウンドファイルが設定済み

- [ ] **テスト実施完了**
  - [ ] 単体テスト（TC1〜4）が全てPass
  - [ ] 結合テスト（TC5〜6）が全てPass
  - [ ] パフォーマンステスト（TC7〜8）が基準値クリア
  - [ ] 異常系テスト（TC9〜10）が全てPass

- [ ] **ドキュメント更新**
  - [ ] TDD.mdのWI-0020ステータスを「完了」に更新
  - [ ] 本作業指示書にテスト結果を追記

- [ ] **コードレビュー承認**
  - [ ] シニアエンジニアによるコードレビュー完了
  - [ ] UdonSharpのベストプラクティスに準拠していることを確認
  - [ ] SOLID原則（特に単一責任の原則）が守られていることを確認

- [ ] **ビルド検証**
  - [ ] PC VR環境でのビルドが成功
  - [ ] Android（Quest 2）環境でのビルドが成功
  - [ ] VRChat SDKのValidationエラーが0件

---

## 10. 依存関係

### 10.1 前提条件となる作業
以下の作業が完了していること：
- **WI-0001**: GameManager Singleton（依存性注入）
- **WI-0002**: NetworkSync Batching（ネットワーク同期基盤）
- **WI-0007**: TreeController State Machine（木の状態管理）
- **WI-0009**: AxeInteraction Detection（斧の攻撃判定）
- **WI-0010**: Damage Calculation System（ダメージ計算）

### 10.2 本作業完了後に可能となる作業
- **WI-0021**: RankingTracker（協力伐採回数の統計収集）
- **WI-0024**: Performance Optimization（協力判定の最適化）

---

## 11. 技術的注意事項

### 11.1 UdonSharpの制約
- **Dictionaryが使えない**: 並列配列で代替
- **Listが使えない**: 固定サイズ配列で代替
- **LINQが使えない**: 手動ループで実装
- **asyncが使えない**: SendCustomEventDelayedSecondsで非同期処理

### 11.2 パフォーマンス最適化
- **GetComponent()の回避**: Start()で1回のみ取得しキャッシュ
- **Update()の最適化**: 5秒間隔でのタイマーベース実行
- **オブジェクトプール**: パーティクルは5個のプールで再利用
- **配列の事前確保**: 動的メモリ確保を避ける

### 11.3 ネットワーク同期の最適化
- **Manual Sync使用**: Continuous Syncは帯域過多のため不使用
- **同期データの削減**: 直近100エントリのみ同期（3000→100に削減）
- **バッチング**: NetworkSync経由で5秒間隔でまとめて同期

### 11.4 Quest最適化
- **パーティクル数制限**: 最大50個/秒（Rate over Time: 50）
- **テクスチャサイズ**: 512x512以下（ASTC圧縮）
- **ドローコール削減**: 同一マテリアルの使用

---

## 12. トラブルシューティング

### 問題1: 協力判定が発動しない
**症状**: 2人で攻撃しても協力判定がtrueにならない

**原因と対処**:
1. タイムスタンプが記録されていない
   - → AxeInteractionから`_RecordHit`が呼ばれているか確認
   - → CooperativeTrackerの参照が設定されているか確認

2. 時間窓が狭すぎる
   - → `COOPERATIVE_TIME_WINDOW`を3.0秒から5.0秒に増やして検証

3. ネットワーク同期が遅延している
   - → Master Clientのログで`RequestSerialization()`が呼ばれているか確認

---

### 問題2: パーティクルが表示されない
**症状**: 協力判定は成功するがパーティクルが見えない

**原因と対処**:
1. Prefabが未設定
   - → CooperativeTrackerの`Cooperative Effect Prefab`を確認

2. オブジェクトプールが初期化されていない
   - → `InitializeEffectPool()`がStart()で呼ばれているか確認

3. パーティクルの位置が間違っている
   - → 木の位置取得ロジック（`GetTreeObject`）を確認

---

### 問題3: Quest 2でFPSが低下する
**症状**: 協力伐採発動時にFPSが50以下に低下

**原因と対処**:
1. パーティクル数が多すぎる
   - → Rate over Timeを50→30に削減
   - → Max Particlesを100→50に削減

2. UIオーバーレイのレンダリング負荷
   - → Canvasを1つのみに統合
   - → 不要なOutlineコンポーネントを削除

3. 配列スキャンの最適化
   - → `_IsCooperativeHit`内のループを早期breakで最適化

---

## 13. 参考資料

### 13.1 公式ドキュメント
- [VRChat UdonSharp Documentation](https://udonsharp.docs.vrchat.com/)
- [VRChat SDK 3.9.0 Release Notes](https://creators.vrchat.com/releases/release-3-9-0/)
- [UdonSync Best Practices](https://creators.vrchat.com/worlds/udon/networking/)

### 13.2 プロジェクト内ドキュメント
- `/docs/TDD.md` - Module M-15の詳細仕様
- `/docs/FSD.md` - FNC-006（協力・ソーシャル機能）の機能仕様
- `/docs/PRDv3.md` - 協力伐採の背景と目的
- `/docs/VRChat_Tools_API_Reference.md` - VRChat API詳細

### 13.3 サンプルコード
- `/Assets/WoodcutterCamp/Examples/NetworkSyncExample.cs` - Manual Syncの実装例
- `/Assets/WoodcutterCamp/Examples/ObjectPoolExample.cs` - オブジェクトプールの実装例

---

## 14. 作業見積もり

| タスク | 推定時間 |
|--------|---------|
| CooperativeTracker.cs実装 | 4時間 |
| Unity Editorセットアップ（パーティクル、UI、サウンド） | 2時間 |
| AxeInteraction連携コード追加 | 1時間 |
| 単体テスト実施（TC1〜4） | 1時間 |
| 結合テスト実施（TC5〜6） | 1.5時間 |
| パフォーマンステスト（TC7〜8） | 1時間 |
| 異常系テスト（TC9〜10） | 0.5時間 |
| ドキュメント更新・コードレビュー | 1時間 |
| **合計** | **12時間（1.5日）** |

---

## 15. 承認

- [ ] **作業指示書レビュー**: シニアエンジニア
- [ ] **実装レビュー**: リードプログラマー
- [ ] **テスト結果承認**: QAエンジニア
- [ ] **最終承認**: プロジェクトマネージャー

---

**作業指示書作成日**: 2025-11-17
**作成者**: VRChat World Development Team
**バージョン**: 1.0
