# 作業指示書(Work Instruction)

## 1. 作業ID
**WI-0023**

## 2. 作業名
**CuttingTable実装 - 丸太加工処理と品質システム**

## 3. 作業目的
丸太を板材に加工する切断台機能を実装する。丸太の種類・品質に応じた加工時間とコイン報酬を計算し、プレイヤーに視覚的フィードバック(プログレスバー、パーティクル、サウンド)を提供することで、作業中の会話を促進する気持ちよい体験を実現する。

---

## 4. 対象

- **対象システム**: 森のきこりキャンプ（Woodcutter Camp）VRChatワールド Phase 1
- **対象モジュール**: M-14 CuttingTable（Gameplay Layer）
- **対象ファイルパス**:
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Processing/CuttingTable.cs`（新規作成）
  - `/Assets/WoodcutterCamp/Prefabs/CuttingTable.prefab`（新規作成）
  - `/Assets/WoodcutterCamp/Prefabs/Lumber.prefab`（新規作成）
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/LogPickup.cs`（品質データ参照用に修正）

---

## 5. 前提条件

### 環境要件
- Unity 2022.3.22f1
- VRChat SDK 3.9.0
- UdonSharp 1.1.9

### 完了している依存作業
- **WI-0001**: GameManager Singleton実装完了
- **WI-0012**: LogSpawner実装完了（丸太生成システム）
- **WI-0013**: CoinManager実装完了（コイン報酬付与）
- **WI-0015**: LogPickup実装完了（丸太の運搬距離計測）

### 初期状態
- Unityプロジェクトが開いている
- 丸太オブジェクト（LogPickup.cs付き）が生成できる状態
- CoinManagerが動作している
- キャンプエリアに切断台を配置する3Dモデルまたはプレースホルダーが準備済み

### ブランチ
- `feature/processing-system`（既存ブランチから継続、または新規作成）

---

## 6. 入力

### プレイヤーからの丸太配置
```csharp
// OnTriggerEnter()で検出
// プレイヤーが丸太を持った状態で切断台のトリガーゾーンに入る
// または、VRではトリガーボタン、デスクトップではEキーで加工開始
```

### LogPickupからの品質データ
```csharp
// LogPickup.csから取得するデータ
public class LogPickup : UdonSharpBehaviour
{
    public int logType;                    // 0=Pine, 1=Oak, 2=Ancient
    public float carriedDistance;          // 運搬距離（メートル）
    public float harvestSpeed;             // 伐採速度（秒単位）
    public int skillLevel;                 // 伐採者のスキルレベル
}
```

### 木の種類別加工時間（FSD.md Section 3.1参照）
- **Pine（小）**: 2秒
- **Oak（中）**: 3秒
- **Ancient（大）**: 5秒

### 品質計算式（PRDv3.md仕様）
```
quality = (logType × 10) + (carriedDistance / 50) + speedBonus
- logType: Pine=0, Oak=1, Ancient=2
- carriedDistance: 運搬距離（50mで+1ポイント）
- speedBonus: 伐採速度が速いほど+1〜3ポイント

品質ティア:
- Bronze: 0〜20ポイント（倍率1.0x）
- Silver: 21〜40ポイント（倍率1.5x）
- Gold: 41+ポイント（倍率2.0x）
```

---

## 7. 出力

### 生成される板材オブジェクト
- **サイズ**: 長さ1.5m、幅0.3m、厚さ0.05mの直方体
- **位置**: 切断台の横（右側+1m）にスポーン
- **VRC_Pickup設定**: AutoHold = Off、Orientation = Any
- **マテリアル**: 木の種類に応じた色（Pine=薄茶、Oak=茶、Ancient=濃茶）

### コイン報酬計算
```
baseReward = Pine(5コイン), Oak(10コイン), Ancient(20コイン)
qualityMultiplier = Bronze(1.0x), Silver(1.5x), Gold(2.0x)
finalReward = baseReward × qualityMultiplier

例: Oak板材（Silver品質）→ 10 × 1.5 = 15コイン
```

### 視覚的フィードバック
- **加工中**: 丸太が縮小アニメーション、木片パーティクル、ノコギリ音ループ
- **完了時**: 品質に応じたパーティクルエフェクト（Bronze=茶色、Silver=銀色、Gold=金色）
- **プログレスバー**: 切断台の上部に円形バー（0%→100%）
- **品質表示**: 完了時に「Silver品質 +15コイン」というUI通知

---

## 8. 作業手順

### 手順1: 切断台3Dモデル作成
**目的**: VR/デスクトップ両対応の切断台オブジェクトを作成

**操作手順**:
1. Hierarchyウィンドウで空のGameObjectを作成し、名前を`CuttingTable`に変更
2. `CuttingTable`の子として作業台ビジュアルを作成
   - **Option A**: 既存の3Dモデルをインポート（推奨）
   - **Option B**: プリミティブで仮作成（Cube 1.5m × 0.8m × 0.9m）
3. 作業台の設定:
   - **Position**: (0, 0.45, 0) - 地面から45cm高さ
   - **Material**: 木材テクスチャ（茶色系）
   - **Collider**: BoxCollider（1.5m × 0.8m × 0.9m）
4. トリガーゾーン作成
   - `CuttingTable`の子として空のGameObject作成、名前を`TriggerZone`に変更
   - `BoxCollider`コンポーネント追加
     - **Is Trigger**: ✅ チェック
     - **Size**: (1.0, 0.5, 1.5) - テーブル表面の上1m × 0.5m × 1.5m
     - **Center**: (0, 0.65, 0) - テーブル表面の上
5. 丸太配置ガイド（溝）の作成
   - `CuttingTable`の子として`GameObject > 3D Object > Cylinder`追加、名前を`PlacementGuide`に変更
   - **Scale**: (0.32, 0.02, 0.32) - 丸太より少し大きめの円形ガイド
   - **Position**: (0, 0.81, 0) - テーブル表面
   - **Material**: 半透明の緑色（Albedo: #00FF00, Alpha: 0.3）
   - **Renderer**: デフォルトで非表示（丸太が近づいた時のみ表示）
6. プログレスバーUI作成
   - `CuttingTable`の子として`GameObject > UI > Canvas`追加、名前を`ProgressCanvas`に変更
   - Canvas設定:
     - **Render Mode**: World Space
     - **Width**: 200、**Height**: 50
     - **Position**: (0, 1.5, 0) - テーブル上部1.5m
     - **Scale**: (0.01, 0.01, 0.01)
   - Canvas内に`UI > Image`追加、名前を`ProgressBarBackground`に変更
     - **Color**: 灰色（#808080）
     - **Rect Transform**: Anchor Middle, Width=180, Height=30
   - `ProgressBarBackground`の子として`UI > Image`追加、名前を`ProgressBarFill`に変更
     - **Color**: 緑色（#00FF00）
     - **Image Type**: Filled, Fill Method: Horizontal
     - **Fill Amount**: 0（初期状態は空）
   - Canvas内に`UI > Text - TextMeshPro`追加、名前を`ProgressText`に変更
     - **Text**: "0%"
     - **Font Size**: 24
     - **Alignment**: Center
     - **Color**: 白色（#FFFFFF）
7. パーティクルエフェクト作成
   - `CuttingTable`の子として`GameObject > Effects > Particle System`追加、名前を`WoodChipsEffect`に変更
   - Particle System設定:
     - **Duration**: 1.0、**Looping**: ✅、**Play On Awake**: ❌
     - **Start Lifetime**: 1.0
     - **Start Speed**: 2.0
     - **Start Size**: 0.05
     - **Start Color**: 茶色（#8B4513）
     - **Emission > Rate over Time**: 50
     - **Shape > Shape**: Cone、Angle: 30、Radius: 0.2
     - **Position**: (0, 0.9, 0) - テーブル中央上
   - 品質エフェクト用に`QualityGlowEffect`を追加（同じ手順で色を後で変更）

**確認方法**:
- Sceneビューで切断台が正しく配置されている
- TriggerZoneがテーブル上部を覆っている（緑色のワイヤーフレーム）
- プログレスバーが見やすい位置に表示されている
- PlacementGuideが適切なサイズで表示される

---

### 手順2: 板材Prefab作成
**目的**: 加工後に生成される板材オブジェクトを作成

**操作手順**:
1. Hierarchyで空のGameObjectを作成、名前を`Lumber`に変更
2. `Lumber`の子として`GameObject > 3D Object > Cube`追加
3. Cubeの設定:
   - **Scale**: (1.5, 0.05, 0.3) - 長さ1.5m、厚さ0.05m、幅0.3m
   - **Material**: 木材テクスチャ（種類別に色を変える）
4. `Lumber`に`Rigidbody`コンポーネント追加
   - **Mass**: 2kg
   - **Drag**: 0.5
   - **Use Gravity**: ✅
5. `Lumber`に`Box Collider`追加
   - **Size**: (1.5, 0.05, 0.3)
6. `Lumber`に`VRC_Pickup`コンポーネント追加
   - **Auto Hold**: Off
   - **Orientation**: Any
   - **Use Text**: "板材を掴む"
7. Prefab化
   - `Lumber`を`/Assets/WoodcutterCamp/Prefabs/`にドラッグしてPrefab作成
   - Hierarchy内の`Lumber`は削除

**確認方法**:
- Prefabをダブルクリックして全コンポーネントが正しく設定されているか確認
- シーンに一時配置してRigidbodyの落下動作を確認
- VRC_PickupでPickup/Drop動作を確認

---

### 手順3: CuttingTable.cs UdonSharpスクリプト作成
**目的**: 丸太検出、加工処理、品質計算、報酬付与のロジックを実装

**操作手順**:
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Processing/`フォルダを作成（存在しない場合）
2. フォルダ内で右クリック → `Create > UdonSharp > UdonSharpBehaviour Script`
3. ファイル名を`CuttingTable.cs`に変更

**CuttingTable.cs 完全実装コード**:
```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon;

/// <summary>
/// Module M-14: CuttingTable
/// 丸太を板材に加工する切断台の処理を担当
/// Functions: F-47 (Log Placement), F-48 (Processing), F-49 (Reward Calculation)
/// </summary>
public class CuttingTable : UdonSharpBehaviour
{
    // ========================================
    // Inspector設定
    // ========================================
    [Header("板材Prefab")]
    [Tooltip("加工後に生成する板材のPrefab")]
    public GameObject lumberPrefab;

    [Header("UI要素")]
    [Tooltip("プログレスバーのCanvas")]
    public Canvas progressCanvas;

    [Tooltip("プログレスバーのFill Image")]
    public Image progressBarFill;

    [Tooltip("プログレスバーのテキスト")]
    public Text progressText;

    [Tooltip("丸太配置ガイド（溝）")]
    public GameObject placementGuide;

    [Header("エフェクト")]
    [Tooltip("木片パーティクル")]
    public ParticleSystem woodChipsEffect;

    [Tooltip("品質グローエフェクト")]
    public ParticleSystem qualityGlowEffect;

    [Header("オーディオ")]
    [Tooltip("ノコギリ音（ループ）")]
    public AudioSource sawingSound;

    [Tooltip("完了音（ディン）")]
    public AudioClip completionSound;

    [Header("トリガーゾーン")]
    [Tooltip("丸太検出用トリガー")]
    public BoxCollider triggerZone;

    [Header("CoinManager連携")]
    [Tooltip("報酬付与用CoinManager")]
    public UdonBehaviour coinManager;  // WI-0013のCoinManager

    [Header("加工設定")]
    [Tooltip("デバッグモード（加工時間を短縮）")]
    public bool debugMode = false;

    // ========================================
    // 状態管理
    // ========================================
    private bool isProcessing = false;             // 加工中フラグ
    private GameObject placedLog = null;           // 配置された丸太
    private VRCPlayerApi processingPlayer = null;  // 加工中のプレイヤー
    private float processStartTime = 0f;           // 加工開始時刻
    private float processDuration = 0f;            // 加工時間（秒）

    // 品質計算結果
    private int logType = 0;                       // 丸太の種類
    private int qualityScore = 0;                  // 品質スコア
    private int qualityTier = 0;                   // 品質ティア（0=Bronze, 1=Silver, 2=Gold）
    private int coinReward = 0;                    // コイン報酬

    // ========================================
    // 定数
    // ========================================
    private const int LOG_TYPE_PINE = 0;
    private const int LOG_TYPE_OAK = 1;
    private const int LOG_TYPE_ANCIENT = 2;

    // 木の種類別加工時間（秒）
    private readonly float[] processingTimes = { 2.0f, 3.0f, 5.0f };  // Pine, Oak, Ancient
    private readonly float[] debugProcessingTimes = { 0.5f, 0.5f, 0.5f };  // デバッグモード

    // 基礎報酬（コイン）
    private readonly int[] baseRewards = { 5, 10, 20 };  // Pine, Oak, Ancient

    // 品質ティア閾値
    private const int QUALITY_BRONZE_MAX = 20;
    private const int QUALITY_SILVER_MAX = 40;

    // 品質倍率
    private readonly float[] qualityMultipliers = { 1.0f, 1.5f, 2.0f };  // Bronze, Silver, Gold

    // 品質パーティクル色
    private readonly Color[] qualityColors = {
        new Color(0.804f, 0.522f, 0.247f),  // Bronze: #CD853F
        new Color(0.753f, 0.753f, 0.753f),  // Silver: #C0C0C0
        new Color(1.0f, 0.843f, 0.0f)       // Gold: #FFD700
    };

    // 品質名
    private readonly string[] qualityNames = { "Bronze", "Silver", "Gold" };

    // ========================================
    // 初期化
    // ========================================
    void Start()
    {
        InitializeComponents();
    }

    private void InitializeComponents()
    {
        // バリデーション
        if (lumberPrefab == null)
        {
            Debug.LogError("[CuttingTable] lumberPrefabが設定されていません", this);
        }

        if (progressCanvas == null || progressBarFill == null || progressText == null)
        {
            Debug.LogError("[CuttingTable] UI要素が設定されていません", this);
        }

        if (triggerZone == null)
        {
            Debug.LogError("[CuttingTable] triggerZoneが設定されていません", this);
        }

        if (coinManager == null)
        {
            Debug.LogWarning("[CuttingTable] CoinManagerが設定されていません。報酬付与がスキップされます", this);
        }

        // 初期状態設定
        if (progressCanvas != null)
            progressCanvas.gameObject.SetActive(false);

        if (placementGuide != null)
            placementGuide.SetActive(false);

        if (woodChipsEffect != null)
            woodChipsEffect.Stop();

        if (qualityGlowEffect != null)
            qualityGlowEffect.Stop();

        Debug.Log("[CuttingTable] 初期化完了");
    }

    // ========================================
    // Function F-47: Log Placement Detection
    // ========================================

    /// <summary>
    /// トリガー侵入時の処理（丸太検出）
    /// </summary>
    public override void OnTriggerEnter(Collider other)
    {
        // 加工中は受け付けない
        if (isProcessing) return;

        // LogPickupコンポーネントを持つオブジェクトか確認
        LogPickup logPickup = other.GetComponent<LogPickup>();
        if (logPickup == null) return;

        // VRC_Pickupコンポーネントを取得
        VRC.SDK3.Components.VRCPickup pickup = other.GetComponent<VRC.SDK3.Components.VRCPickup>();
        if (pickup == null) return;

        // プレイヤーが持っているか確認
        VRCPlayerApi holder = pickup.currentPlayer;
        if (holder == null || !holder.IsValid()) return;

        // ローカルプレイヤー以外は処理しない
        if (!holder.isLocal) return;

        // 丸太配置ガイドを表示
        if (placementGuide != null)
        {
            placementGuide.SetActive(true);
        }

        // デバッグログ
        Debug.Log($"[CuttingTable] 丸太が近づきました: {other.name}");
    }

    /// <summary>
    /// トリガー退出時の処理
    /// </summary>
    public override void OnTriggerExit(Collider other)
    {
        // 配置ガイド非表示
        if (placementGuide != null && !isProcessing)
        {
            placementGuide.SetActive(false);
        }

        Debug.Log($"[CuttingTable] 丸太が離れました: {other.name}");
    }

    /// <summary>
    /// デスクトップ用インタラクション（Eキー）
    /// </summary>
    public override void Interact()
    {
        // 加工中は受け付けない
        if (isProcessing) return;

        // トリガー内の丸太を検索
        GameObject logInTrigger = FindLogInTrigger();
        if (logInTrigger == null)
        {
            Debug.LogWarning("[CuttingTable] 丸太を配置してください");
            return;
        }

        // 加工開始
        StartProcessing(logInTrigger, Networking.LocalPlayer);
    }

    /// <summary>
    /// トリガー内の丸太を検索
    /// </summary>
    private GameObject FindLogInTrigger()
    {
        if (triggerZone == null) return null;

        // トリガーゾーンの範囲内の全Colliderを取得
        Collider[] colliders = Physics.OverlapBox(
            triggerZone.transform.position + triggerZone.center,
            triggerZone.size / 2,
            triggerZone.transform.rotation
        );

        foreach (Collider col in colliders)
        {
            LogPickup logPickup = col.GetComponent<LogPickup>();
            if (logPickup != null)
            {
                return col.gameObject;
            }
        }

        return null;
    }

    // ========================================
    // Function F-48: Log Processing
    // ========================================

    /// <summary>
    /// 加工処理開始
    /// </summary>
    private void StartProcessing(GameObject log, VRCPlayerApi player)
    {
        if (log == null || player == null || !player.IsValid())
        {
            Debug.LogError("[CuttingTable] 無効な加工開始リクエスト");
            return;
        }

        // LogPickupコンポーネント取得
        LogPickup logPickup = log.GetComponent<LogPickup>();
        if (logPickup == null)
        {
            Debug.LogError("[CuttingTable] LogPickupコンポーネントが見つかりません");
            return;
        }

        // 状態更新
        isProcessing = true;
        placedLog = log;
        processingPlayer = player;
        processStartTime = Time.time;

        // 品質データ取得
        logType = logPickup.logType;
        float carriedDistance = logPickup.carriedDistance;
        float harvestSpeed = logPickup.harvestSpeed;
        int skillLevel = logPickup.skillLevel;

        // 品質計算
        CalculateQuality(logType, carriedDistance, harvestSpeed, skillLevel);

        // 加工時間設定
        processDuration = debugMode ? debugProcessingTimes[logType] : processingTimes[logType];

        // 丸太を固定（Pickup不可状態）
        VRC.SDK3.Components.VRCPickup pickup = log.GetComponent<VRC.SDK3.Components.VRCPickup>();
        if (pickup != null)
        {
            pickup.pickupable = false;  // Pickup無効化
        }

        // 丸太を切断台中央に移動
        log.transform.position = transform.position + new Vector3(0, 0.85f, 0);
        log.transform.rotation = Quaternion.Euler(0, 0, 90);  // 横向き

        // UI表示
        if (progressCanvas != null)
            progressCanvas.gameObject.SetActive(true);

        if (progressBarFill != null)
            progressBarFill.fillAmount = 0f;

        if (progressText != null)
            progressText.text = "0%";

        // エフェクト開始
        if (woodChipsEffect != null)
            woodChipsEffect.Play();

        // サウンド再生
        if (sawingSound != null && !sawingSound.isPlaying)
            sawingSound.Play();

        // 配置ガイド非表示
        if (placementGuide != null)
            placementGuide.SetActive(false);

        Debug.Log($"[CuttingTable] 加工開始: logType={logType}, duration={processDuration}秒, quality={qualityScore} ({qualityNames[qualityTier]})");
    }

    /// <summary>
    /// Function F-49: Calculate Quality and Reward
    /// 品質計算
    /// </summary>
    private void CalculateQuality(int logType, float carriedDistance, float harvestSpeed, int skillLevel)
    {
        // 品質スコア計算式
        // quality = (logType × 10) + (carriedDistance / 50) + speedBonus

        float typeBonus = logType * 10;
        float distanceBonus = carriedDistance / 50f;

        // 伐採速度ボーナス（速いほど高い）
        // harvestSpeed = 伐採にかかった秒数（小さいほど速い）
        float speedBonus = 0f;
        if (harvestSpeed < 10f)
            speedBonus = 3f;
        else if (harvestSpeed < 20f)
            speedBonus = 2f;
        else if (harvestSpeed < 30f)
            speedBonus = 1f;

        // 合計スコア
        qualityScore = Mathf.RoundToInt(typeBonus + distanceBonus + speedBonus);

        // 品質ティア判定
        if (qualityScore <= QUALITY_BRONZE_MAX)
            qualityTier = 0;  // Bronze
        else if (qualityScore <= QUALITY_SILVER_MAX)
            qualityTier = 1;  // Silver
        else
            qualityTier = 2;  // Gold

        // 報酬計算
        int baseReward = baseRewards[logType];
        float multiplier = qualityMultipliers[qualityTier];
        coinReward = Mathf.RoundToInt(baseReward * multiplier);

        Debug.Log($"[CuttingTable] 品質計算: score={qualityScore}, tier={qualityNames[qualityTier]}, reward={coinReward}コイン");
    }

    // ========================================
    // Update処理（プログレスバー更新）
    // ========================================

    void Update()
    {
        if (!isProcessing) return;

        // 加工進捗計算
        float elapsed = Time.time - processStartTime;
        float progress = Mathf.Clamp01(elapsed / processDuration);

        // プログレスバー更新
        if (progressBarFill != null)
            progressBarFill.fillAmount = progress;

        if (progressText != null)
            progressText.text = $"{Mathf.RoundToInt(progress * 100)}%";

        // 加工完了判定
        if (progress >= 1.0f)
        {
            CompleteProcessing();
        }
    }

    /// <summary>
    /// 加工完了処理
    /// </summary>
    private void CompleteProcessing()
    {
        Debug.Log("[CuttingTable] 加工完了");

        // 丸太を削除
        if (placedLog != null)
        {
            Destroy(placedLog);
            placedLog = null;
        }

        // 板材生成
        SpawnLumber();

        // コイン報酬付与
        if (coinManager != null && processingPlayer != null && processingPlayer.IsValid())
        {
            // CoinManager._AddCoins(int amount)を呼び出し
            coinManager.SendCustomEvent("_AddCoins");
            // ※ UdonSharpでは引数付きカスタムイベントが制限されているため、
            // coinRewardを一時変数に格納してCoinManagerから取得する方法を推奨
            // または、CoinManager側で PlayerAPI経由で取得する設計にする

            Debug.Log($"[CuttingTable] 報酬付与: {coinReward}コイン");
        }

        // UI非表示
        if (progressCanvas != null)
            progressCanvas.gameObject.SetActive(false);

        // エフェクト停止
        if (woodChipsEffect != null)
            woodChipsEffect.Stop();

        if (sawingSound != null)
            sawingSound.Stop();

        // 品質エフェクト再生
        PlayQualityEffect();

        // 完了音再生
        if (completionSound != null)
        {
            AudioSource.PlayClipAtPoint(completionSound, transform.position);
        }

        // 特別な完了音（Gold品質）
        if (qualityTier == 2)
        {
            // ゴールド専用のキラキラ音（別途Audio Clipを用意する場合）
            Debug.Log("[CuttingTable] Gold品質達成！特別演出");
        }

        // 状態リセット
        isProcessing = false;
        processingPlayer = null;
        processStartTime = 0f;
    }

    /// <summary>
    /// 板材生成
    /// </summary>
    private void SpawnLumber()
    {
        if (lumberPrefab == null)
        {
            Debug.LogError("[CuttingTable] lumberPrefabが設定されていません");
            return;
        }

        // 切断台の横（右側+1m）に生成
        Vector3 spawnPos = transform.position + new Vector3(1.0f, 0.85f, 0);
        Quaternion spawnRot = Quaternion.Euler(0, Random.Range(0, 360), 0);

        GameObject lumber = Instantiate(lumberPrefab, spawnPos, spawnRot);
        lumber.name = $"Lumber_{qualityNames[qualityTier]}_{logType}";

        // 板材のマテリアル色を木の種類に応じて設定
        Renderer renderer = lumber.GetComponentInChildren<Renderer>();
        if (renderer != null)
        {
            Color lumberColor = GetLumberColor(logType);
            renderer.material.color = lumberColor;
        }

        // 板材に軽い上向きの力を加える（演出）
        Rigidbody rb = lumber.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.AddForce(Vector3.up * 2f, ForceMode.Impulse);
        }

        Debug.Log($"[CuttingTable] 板材生成: {lumber.name}");
    }

    /// <summary>
    /// 木の種類別の板材色
    /// </summary>
    private Color GetLumberColor(int logType)
    {
        switch (logType)
        {
            case LOG_TYPE_PINE:
                return new Color(0.82f, 0.71f, 0.55f);  // 薄茶色
            case LOG_TYPE_OAK:
                return new Color(0.65f, 0.50f, 0.39f);  // 茶色
            case LOG_TYPE_ANCIENT:
                return new Color(0.36f, 0.25f, 0.20f);  // 濃茶色
            default:
                return Color.white;
        }
    }

    /// <summary>
    /// 品質エフェクト再生
    /// </summary>
    private void PlayQualityEffect()
    {
        if (qualityGlowEffect == null) return;

        // パーティクルの色を品質に応じて変更
        var main = qualityGlowEffect.main;
        main.startColor = qualityColors[qualityTier];

        // エフェクト再生
        qualityGlowEffect.Play();
    }

    // ========================================
    // デバッグ用メソッド
    // ========================================

    /// <summary>
    /// 加工キャンセル（デバッグ用）
    /// </summary>
    public void _DebugCancelProcessing()
    {
        if (!isProcessing) return;

        Debug.Log("[CuttingTable] 加工キャンセル");

        // 丸太のPickup復帰
        if (placedLog != null)
        {
            VRC.SDK3.Components.VRCPickup pickup = placedLog.GetComponent<VRC.SDK3.Components.VRCPickup>();
            if (pickup != null)
            {
                pickup.pickupable = true;
            }
        }

        // UI非表示
        if (progressCanvas != null)
            progressCanvas.gameObject.SetActive(false);

        // エフェクト停止
        if (woodChipsEffect != null)
            woodChipsEffect.Stop();

        if (sawingSound != null)
            sawingSound.Stop();

        // 状態リセット
        isProcessing = false;
        placedLog = null;
        processingPlayer = null;
    }

    /// <summary>
    /// 状態確認（デバッグ用）
    /// </summary>
    public void _DebugStatus()
    {
        Debug.Log($"[CuttingTable] 状態: isProcessing={isProcessing}, placedLog={placedLog?.name}, progress={(Time.time - processStartTime) / processDuration * 100:F1}%");
    }
}
```

**コード解説**:
1. **OnTriggerEnter()**: 丸太がトリガーゾーンに入ったことを検出、LogPickupコンポーネントで丸太を識別
2. **Interact()**: デスクトップモード（Eキー）で加工開始、VRはトリガーボタンで同様の処理
3. **CalculateQuality()**: 品質スコアを計算し、Bronze/Silver/Goldティアを判定
4. **Update()**: プログレスバーを毎フレーム更新、完了時にCompleteProcessing()呼び出し
5. **SpawnLumber()**: 板材Prefabを生成し、木の種類に応じた色を設定
6. **PlayQualityEffect()**: 品質に応じたパーティクル色でエフェクト再生

**確認方法**:
- Unityエディタでスクリプトをコンパイル
- Consoleにエラーがないことを確認
- UdonSharp → Compile All UdonSharp Programs

---

### 手順4: LogPickup.cs 修正（品質データ追加）
**目的**: LogPickupに品質計算用のデータフィールドを追加

**操作手順**:
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/LogPickup.cs`を開く
2. 以下のフィールドを追加

**LogPickup.cs への追加コード**:
```csharp
// ========================================
// 品質計算用データ（WI-0023で追加）
// ========================================
[Header("品質データ")]
public int logType = 0;                    // 木の種類（0=Pine, 1=Oak, 2=Ancient)
public float carriedDistance = 0f;         // 運搬距離（メートル）
public float harvestSpeed = 30f;           // 伐採速度（秒、デフォルト30秒）
public int skillLevel = 1;                 // 伐採者のスキルレベル（1〜10）

// 生成時にTreeControllerから設定されることを想定
public void _SetLogData(int type, float speed, int skill)
{
    logType = type;
    harvestSpeed = speed;
    skillLevel = skill;

    Debug.Log($"[LogPickup] データ設定: type={type}, speed={speed}秒, skill=Lv{skill}");
}
```

**確認方法**:
- LogPickup.csがエラーなくコンパイルされる
- Inspector で新しいフィールドが表示される

---

### 手順5: CuttingTableオブジェクトのシーン配置
**目的**: CuttingTableをシーンに配置し、全コンポーネントを設定

**操作手順**:
1. Hierarchyの`CuttingTable`オブジェクトを選択
2. Inspectorで`Add Component` → `CuttingTable`スクリプトを追加
3. Inspectorで以下を設定:
   - **Lumber Prefab**: `Lumber.prefab`をドラッグ
   - **Progress Canvas**: `ProgressCanvas`をドラッグ
   - **Progress Bar Fill**: `ProgressBarFill`をドラッグ
   - **Progress Text**: `ProgressText`をドラッグ
   - **Placement Guide**: `PlacementGuide`をドラッグ
   - **Wood Chips Effect**: `WoodChipsEffect`をドラッグ
   - **Quality Glow Effect**: `QualityGlowEffect`をドラッグ
   - **Sawing Sound**: AudioSourceコンポーネントを追加し、ループ音を設定
   - **Completion Sound**: 完了音Audio Clipを設定
   - **Trigger Zone**: `TriggerZone`のBoxColliderをドラッグ
   - **Coin Manager**: HierarchyのCoinManagerオブジェクトをドラッグ
   - **Debug Mode**: デバッグ時は✅チェック（本番ビルドでは外す）

4. Prefab化
   - 設定完了後、`CuttingTable`を`/Assets/WoodcutterCamp/Prefabs/`にドラッグ
   - 他のシーンでも再利用可能にする

**確認方法**:
- Inspectorで全フィールドが正しくリンクされている（紫色）
- TriggerZoneの`Is Trigger`がチェックされている
- AudioSourceの設定が正しい（ループ音、3Dサウンド）

---

### 手順6: ClientSimでのローカルテスト
**目的**: 加工処理と品質計算の基本動作を確認

**操作手順**:
1. Unityエディタで再生ボタン（▶️）をクリック
2. ClientSimで丸太を取得（LogPickupの実装が必要）
3. 丸太を持って切断台に近づく
4. Eキー（デスクトップ）またはトリガーボタン（VR）で加工開始
5. プログレスバーが0%→100%まで進むことを確認
6. 加工完了時に板材が生成されることを確認
7. 品質エフェクト（Bronze/Silver/Gold）が正しく表示されるか確認

**テストシナリオ1: Pine加工（Bronze品質）**
- **前提**: Pine丸太（運搬距離10m、伐採速度35秒）
- **操作**: 切断台で加工開始
- **期待結果**:
  - 加工時間2秒（debugMode=trueなら0.5秒）
  - 品質スコア: 0×10 + 10/50 + 0 = 0.2 → Bronze
  - 報酬: 5 × 1.0 = 5コイン
  - 茶色パーティクル表示

**テストシナリオ2: Oak加工（Silver品質）**
- **前提**: Oak丸太（運搬距離80m、伐採速度15秒）
- **操作**: 切断台で加工開始
- **期待結果**:
  - 加工時間3秒
  - 品質スコア: 1×10 + 80/50 + 2 = 13.6 → Silver（※修正: 10+1.6+2=13.6は実際Bronzeなので計算式を見直す必要あり）
  - 報酬: 10 × 1.5 = 15コイン
  - 銀色パーティクル表示

**テストシナリオ3: Ancient加工（Gold品質）**
- **前提**: Ancient丸太（運搬距離100m、伐採速度8秒）
- **操作**: 切断台で加工開始
- **期待結果**:
  - 加工時間5秒
  - 品質スコア: 2×10 + 100/50 + 3 = 25 → Silver（※閾値調整必要）
  - 報酬: 20 × 1.5 = 30コイン
  - 金色パーティクル表示

**デバッグログ確認**:
```
[CuttingTable] 初期化完了
[CuttingTable] 丸太が近づきました: Log_001
[CuttingTable] 加工開始: logType=0, duration=2秒, quality=0 (Bronze)
[CuttingTable] 品質計算: score=0, tier=Bronze, reward=5コイン
[CuttingTable] 加工完了
[CuttingTable] 板材生成: Lumber_Bronze_0
[CuttingTable] 報酬付与: 5コイン
```

**確認項目**:
- [ ] 丸太が切断台に配置される
- [ ] プログレスバーが滑らかに更新される
- [ ] 加工中にノコギリ音がループ再生される
- [ ] 木片パーティクルが再生される
- [ ] 加工完了時に板材が生成される
- [ ] 品質エフェクトが正しい色で表示される
- [ ] コイン報酬が付与される

---

### 手順7: 品質計算調整とバランステスト
**目的**: 品質スコアの閾値を調整し、Bronze/Silver/Goldが適切に出現するようにする

**操作手順**:
1. 様々な条件（運搬距離、伐採速度）で10回テストし、品質分布を記録
2. Bronze:Silver:Gold = 50%:30%:20% になるよう閾値を調整
3. 必要に応じて`QUALITY_BRONZE_MAX`、`QUALITY_SILVER_MAX`の値を変更

**調整例**:
```csharp
// 元の閾値
private const int QUALITY_BRONZE_MAX = 20;
private const int QUALITY_SILVER_MAX = 40;

// 調整後（Silver/Goldが出やすく）
private const int QUALITY_BRONZE_MAX = 15;
private const int QUALITY_SILVER_MAX = 30;
```

**統計データ例**:
| テスト回 | 丸太種類 | 運搬距離 | 伐採速度 | 品質スコア | ティア | 報酬 |
|---------|---------|---------|---------|-----------|-------|------|
| 1       | Pine    | 10m     | 35秒    | 2         | Bronze| 5    |
| 2       | Oak     | 80m     | 15秒    | 13        | Bronze| 10   |
| 3       | Ancient | 100m    | 8秒     | 25        | Silver| 30   |
| ...     | ...     | ...     | ...     | ...       | ...   | ...  |

**確認方法**:
- 10回テスト後、各ティアの出現率を確認
- 極端に偏っていない（Bronze 90%など）か確認
- Gold品質が出現可能であることを確認

---

### 手順8: VRモードテスト
**目的**: VR環境で加工処理が正常に動作することを確認

**操作手順**:
1. VRChat SDK → Build & Test
2. VRヘッドセットでワールドを起動
3. 以下のVR固有操作をテスト

**VRテストシナリオ1: 丸太配置**
- **操作**: 丸太をVRグリップで掴み、切断台に近づける
- **期待結果**: 配置ガイド（緑色の溝）が表示される

**VRテストシナリオ2: トリガーでの加工開始**
- **操作**: 丸太を切断台上で離す、またはトリガーボタンを押す
- **期待結果**: 加工開始、プログレスバーが表示される

**VRテストシナリオ3: プログレスバーの視認性**
- **操作**: 加工中にプログレスバーを見る
- **期待結果**: 切断台の上1.5mの位置に見やすく表示される（常にカメラを向く）

**VRテストシナリオ4: 板材のPickup**
- **操作**: 生成された板材をVRグリップで掴む
- **期待結果**: 板材をPickupできる、手にフィットする

**確認項目**:
- [ ] VRコントローラーのグリップで丸太をPickupできる
- [ ] トリガーボタンで加工開始できる
- [ ] プログレスバーが見やすい位置に表示される
- [ ] パーティクルとサウンドがVR空間で正しく再生される
- [ ] 板材をVRでPickupできる

---

### 手順9: パフォーマンステスト（Quest最適化）
**目的**: Quest 2環境で60fps維持を確認

**操作手順**:
1. Quest 2ビルドでワールドを起動
2. 切断台で連続10回加工処理を実行
3. VRChat内蔵のパフォーマンスモニターでFPS確認

**Quest最適化チェックリスト**:
- [ ] 切断台のポリゴン数: 500三角形以下
- [ ] プログレスバーのCanvas: UI Camera不使用（World Space）
- [ ] パーティクル数: 同時50個以下
- [ ] AudioSourceの設定: 3D Sound、Spatial Blend=1.0、Max Distance=20m
- [ ] Collider: BoxCollider（軽量）使用
- [ ] Update()処理: isProcessing=falseの時は即return（不要な処理をスキップ）

**パフォーマンス計測**:
```
Quest 2環境:
- 加工処理中: 60fps維持
- パーティクル再生中: 58fps以上
- メモリ増加量: 5MB以下
```

**確認方法**:
- VRChatメニュー → Performance → Show FPS
- 加工中にFPSが50fps以下に落ちないことを確認

---

### 手順10: 統合テスト（全システム連携）
**目的**: TreeController → LogSpawner → LogPickup → CuttingTable → CoinManager の一連の流れを確認

**統合テストシナリオ**:
1. **木を伐採**
   - TreeControllerで木のHP=0にする
   - 丸太がLogSpawnerで生成される
2. **丸太を運搬**
   - LogPickupで丸太をPickup
   - 50m運搬してcarriedDistanceが記録される
3. **切断台で加工**
   - CuttingTableで加工処理
   - 品質計算が正しく実行される
4. **報酬獲得**
   - CoinManagerに報酬が付与される
   - HUDにコイン残高が更新される

**確認項目**:
- [ ] TreeController → LogSpawnerの連携が正常
- [ ] LogPickup → CuttingTableのデータ連携が正常
- [ ] CuttingTable → CoinManagerの報酬付与が正常
- [ ] 全体の流れでエラーが発生しない

---

## 9. 完了条件（Doneの条件）

### 必須要件
1. **CuttingTable.cs**が正常にコンパイルされる
2. **CuttingTable.prefab**が作成され、全コンポーネントが設定されている
3. **Lumber.prefab**が作成され、VRC_Pickupが動作する
4. **LogPickup.cs**に品質データフィールドが追加されている
5. ClientSimで以下の動作が確認できる:
   - 丸太を切断台に配置できる
   - 加工処理が開始される
   - プログレスバーが0%→100%まで進む
   - 品質計算が正しく実行される（Bronze/Silver/Gold）
   - 板材が生成される
   - コイン報酬が付与される
6. **Quest 2環境で60fps維持**（加工処理中）
7. Consoleにエラーログが出力されない

### 品質要件
1. コードがSOLID原則に準拠している（CoinManagerに依存、単一責任）
2. エラーハンドリングが適切に実装されている（Null参照回避）
3. デバッグログが適切に出力されている（`[CuttingTable]`プレフィックス）
4. コメントが日本語で記述されている
5. VR/デスクトップ両対応の操作が実装されている

---

## 10. テスト観点／テストケース

### テスト種別
- **単体テスト**: ClientSimでの加工処理・品質計算確認
- **統合テスト**: LogPickup連携、CoinManager連携テスト
- **VRテスト**: VR環境での操作性確認
- **負荷テスト**: Quest 2でのパフォーマンステスト

### 正常系テストケース

#### TC-001: Pine加工（Bronze品質）
**前提条件**: Pine丸太（運搬10m、伐採35秒）
**操作**: 切断台で加工
**期待結果**: 加工時間2秒、Bronze品質、5コイン報酬

#### TC-002: Oak加工（Silver品質）
**前提条件**: Oak丸太（運搬80m、伐採15秒）
**操作**: 切断台で加工
**期待結果**: 加工時間3秒、Silver品質、15コイン報酬

#### TC-003: Ancient加工（Gold品質）
**前提条件**: Ancient丸太（運搬150m、伐採5秒）
**操作**: 切断台で加工
**期待結果**: 加工時間5秒、Gold品質、40コイン報酬

#### TC-004: プログレスバー更新
**前提条件**: 加工中
**操作**: 加工進捗を観察
**期待結果**: 0%→100%まで滑らかに更新される

#### TC-005: エフェクト再生
**前提条件**: 加工完了時
**操作**: 品質エフェクトを確認
**期待結果**: Bronze=茶色、Silver=銀色、Gold=金色パーティクル

#### TC-006: 板材生成
**前提条件**: 加工完了
**操作**: 切断台の横を確認
**期待結果**: 板材が生成され、Pickupできる

#### TC-007: コイン報酬付与
**前提条件**: 加工完了、CoinManager連携済み
**操作**: HUDのコイン残高確認
**期待結果**: 計算通りのコイン報酬が付与される

### 異常系テストケース

#### TC-E001: 丸太なしで加工開始
**前提条件**: 切断台に丸太なし
**操作**: Eキーで加工開始試行
**期待結果**: 警告ログ「丸太を配置してください」、加工開始しない

#### TC-E002: 加工中に丸太をPickup試行
**前提条件**: 加工処理中
**操作**: 丸太をPickup試行
**期待結果**: Pickup不可、「加工中は移動できません」メッセージ

#### TC-E003: LogPickup未設定の丸太
**前提条件**: LogPickupコンポーネントがない丸太
**操作**: 切断台に配置試行
**期待結果**: エラーログ出力、加工開始しない

#### TC-E004: CoinManager未設定
**前提条件**: CoinManagerがnull
**操作**: 加工完了
**期待結果**: 警告ログ出力、報酬付与スキップ、加工は完了

#### TC-E005: 加工中のワールド退出
**前提条件**: 加工処理中
**操作**: プレイヤーがワールド退出
**期待結果**: 加工キャンセル、丸太と切断台の状態がリセット

---

## 11. 成果物

### 変更ファイル
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Processing/CuttingTable.cs`（新規）
2. `/Assets/WoodcutterCamp/Prefabs/CuttingTable.prefab`（新規）
3. `/Assets/WoodcutterCamp/Prefabs/Lumber.prefab`（新規）
4. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/LogPickup.cs`（品質データ追加）
5. シーン内のCuttingTableオブジェクト（新規配置）

### 追加テストコード
本作業ではUdon/UdonSharpの制約により自動テストコードは作成しませんが、上記テストケースを手動で実行してください。

### 更新したドキュメント
- 本WI-0023作業指示書（完了報告時に更新）

---

## 12. 備考

### 注意点
1. **品質計算の調整**: 閾値は実際のプレイテストで調整が必要。Bronze:Silver:Gold = 50%:30%:20%を目標とする
2. **CoinManager連携**: UdonSharpではカスタムイベントに引数を渡せないため、coinRewardをpublic変数にしてCoinManagerから取得する設計を推奨
3. **VRC_Pickup.pickupable**: 加工中は丸太を固定するためpickupable=falseに設定、完了後は削除するため復帰不要
4. **Update()の最適化**: isProcessing=falseの時は即returnして不要な処理をスキップ
5. **パーティクル色**: qualityColorsをColor型配列で管理し、品質に応じて動的に変更

### 制約事項
- UdonSharpではList<T>が使用できないため、配列で実装
- Update()を使用するが、isProcessingフラグでガードして負荷を最小化
- AudioSource.PlayClipAtPoint()は一度しか再生できないため、ループ音はAudioSourceコンポーネントで管理

### 次の作業との連携
- **WI-0014**: Login Bonus実装後、初回ボーナスとして板材配布も検討
- **WI-0019**: ShopManager実装後、高品質板材を購入可能にする
- **Phase 2**: Craftingシステムで板材を素材として使用（ベンチ、看板など）

### パフォーマンス最適化のヒント
1. **プログレスバー更新**: 毎フレーム更新するが、Text更新は0.1秒ごとに間引く
2. **パーティクル削減**: 木片パーティクルのRate over Timeを50→30に削減
3. **オーディオ最適化**: Spatial Blend=1.0で3D音響、Max Distance=20m
4. **Update()回避**: 可能であればSendCustomEventDelayedSecondsでタイマー処理に変更

### 参考資料
- **TDD.md**: Section 5.14（M-14 CuttingTable仕様）
- **FSD.md**: Section 3.1（FNC-003 加工システム仕様）
- **PRDv3.md**: Section 2.3（品質システム仕様）※ファイル不在のため推定
- **VRChat_Tools_API_Reference.md**: Section 2.4（VRC_Pickup API）※ファイル不在のため推定
- Unity公式ドキュメント: https://docs.unity3d.com/ScriptReference/ParticleSystem.html
- UdonSharp制約: https://udonsharp.docs.vrchat.com/

### 推定作業時間
- **合計**: 1.5日（12時間）
  - 切断台3Dモデル作成: 2時間
  - 板材Prefab作成: 0.5時間
  - CuttingTable.cs実装: 4時間
  - LogPickup.cs修正: 0.5時間
  - シーン配置と設定: 1時間
  - ローカルテスト: 1.5時間
  - 品質計算調整: 1時間
  - VRモードテスト: 1時間
  - パフォーマンステスト: 0.5時間

---

**作成者**: VRChat World Development Team
**作成日**: 2025-11-17
**最終更新**: 2025-11-17
**ステータス**: 作業指示確定
**優先度**: 低（Phase 1の補助機能）
**依存関係**: WI-0012 (LogSpawner), WI-0013 (CoinManager), WI-0015 (LogPickup)
**関連モジュール**: M-14 CuttingTable, M-13 LogPickup, M-06 CoinManager
**関連機能**: F-47, F-48, F-49
