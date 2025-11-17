# 作業指示書：WI-0005 - SkillManager実装

## 1. 作業ID
**WI-0005**

## 2. 作業名
**SkillManager（スキル管理システム）の実装**

## 3. 作業目的
Woodcuttingスキルのレベル・XP管理機能を実装し、プレイヤーの成長体験の基盤を構築する。XP加算、レベルアップ判定、PlayerDataへの永続化を実現し、他モジュール（SkillEffects、HUDManager）との連携インターフェースを提供する。

## 4. 対象

* **対象システム**: 森のきこりキャンプ - VRChatワールド Phase 1
* **対象モジュール**: M-04 SkillManager（Progression Layer）
* **対象ファイルパス**: `/Assets/WoodcutterCamp/Scripts/Progression/SkillManager.cs`
* **依存モジュール**: M-03 PersistenceManager（データ永続化）

## 5. 前提条件

### 環境要件
* Unity 2022.3.22f1 LTS
* VRChat SDK 3.9.0
* UdonSharp 1.1.9
* 開発ブランチ: `feature/phase0-foundation` または `feature/phase1-core`

### 実装済み依存モジュール
* **WI-0001**: GameManager Singleton実装済み
* **WI-0003**: PersistenceManager実装済み（PlayerData I/O機能）

### 初期状態
* プレイヤーは初回入場時、Woodcutting Lv1、XP=0の状態でスタート
* PersistenceManager経由でPlayerDataへの読み書きが可能な状態

## 6. 入力

### SkillManager.AddXP()への入力
```csharp
// 木伐採完了時の呼び出し例（AxeInteractionから）
public void _OnTreeCutComplete(int treeType)
{
    SkillManager skillManager = GameManager.Instance.GetSkillManager();

    // XP量を決定
    int xpAmount = 0;
    switch (treeType)
    {
        case 0: xpAmount = 10; break;  // Pine (小さい木)
        case 1: xpAmount = 20; break;  // Oak (中くらいの木)
        case 2: xpAmount = 40; break;  // Ancient (大きな木)
    }

    // 協力伐採ボーナス（+20%）
    if (isCooperative)
    {
        xpAmount = Mathf.CeilToInt(xpAmount * 1.2f);
    }

    // XP加算
    skillManager._AddXP(xpAmount);
}
```

### PlayerData読み込み時の入力
```csharp
// ワールド入場時（GameManager.Start()から）
int savedLevel = PersistenceManager.GetInt("Woodcutting_Level", 1);
int savedXP = PersistenceManager.GetInt("Woodcutting_XP", 0);
```

## 7. 出力

### レベルアップイベント発火
```csharp
// HUDManager、SkillEffectsへの通知
public void _OnSkillLevelUp(int newLevel)
{
    // HUDManagerに通知
    HUDManager hudManager = GameManager.Instance.GetHUDManager();
    hudManager._ShowLevelUpNotification(newLevel);

    // SkillEffectsに通知（ダメージボーナス再計算）
    SkillEffects skillEffects = GameManager.Instance.GetSkillEffects();
    skillEffects._RecalculateBonuses(newLevel);
}
```

### PlayerDataへの保存
```csharp
// XP獲得時またはレベルアップ時
PersistenceManager.SetInt("Woodcutting_Level", currentLevel);
PersistenceManager.SetInt("Woodcutting_XP", currentXP);
```

### 他モジュールからの参照
```csharp
// SkillEffects、HUDManagerからの参照例
int currentLevel = skillManager.GetSkillLevel();
int currentXP = skillManager.GetCurrentXP();
int requiredXP = skillManager.GetRequiredXP(currentLevel);
float progressPercent = skillManager.GetLevelProgress();
```

## 8. 作業手順

### ステップ1: SkillManager.csの作成とクラス構造定義

1. **ファイル作成**
   ```bash
   # Unityプロジェクト内で以下のパスにファイルを作成
   /Assets/WoodcutterCamp/Scripts/Progression/SkillManager.cs
   ```

2. **基本クラス構造の実装**
   ```csharp
   using UdonSharp;
   using UnityEngine;
   using VRC.SDKBase;
   using VRC.Udon;

   namespace WoodcutterCamp.Progression
   {
       /// <summary>
       /// Woodcuttingスキルのレベル・XP管理を担当するモジュール
       /// SOLID原則: 単一責任 - スキル管理のみに専念
       /// </summary>
       [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
       public class SkillManager : UdonSharpBehaviour
       {
           // === 定数定義 ===
           private const int MAX_LEVEL = 10;
           private const int MIN_LEVEL = 1;
           private const int MAX_XP = 10500;

           // === プライベート変数 ===
           private int currentLevel = 1;
           private int currentXP = 0;

           // レベルテーブル（必要XP配列）
           private int[] levelRequirements;

           // 参照キャッシュ
           private PersistenceManager persistenceManager;
           private bool isInitialized = false;

           // === パブリックゲッター ===
           public int GetSkillLevel() => currentLevel;
           public int GetCurrentXP() => currentXP;
           public int GetRequiredXP(int level) => GetRequiredXPForLevel(level);
           public float GetLevelProgress() => CalculateLevelProgress();

           // === 初期化 ===
           void Start()
           {
               InitializeSkillManager();
           }

           // 以下、各メソッドを実装（次ステップ）
       }
   }
   ```

### ステップ2: レベルテーブルの実装

3. **必要XP配列の定義**
   ```csharp
   /// <summary>
   /// レベルテーブルを初期化する
   /// レベル1〜10の必要XP: 100, 250, 450, 700, 1000, 1350, 1750, 2200, 2700
   /// </summary>
   private void InitializeLevelTable()
   {
       // levelRequirements[0] = Lv1→Lv2に必要なXP
       // levelRequirements[1] = Lv2→Lv3に必要なXP
       // ...
       // levelRequirements[9] = Lv9→Lv10に必要なXP
       levelRequirements = new int[MAX_LEVEL]
       {
           100,  // Lv1→Lv2
           250,  // Lv2→Lv3
           450,  // Lv3→Lv4
           700,  // Lv4→Lv5
           1000, // Lv5→Lv6
           1350, // Lv6→Lv7
           1750, // Lv7→Lv8
           2200, // Lv8→Lv9
           2700, // Lv9→Lv10
           0     // Lv10（最大レベル、これ以上上がらない）
       };

       Debug.Log("[SkillManager] Level table initialized");
   }

   /// <summary>
   /// 指定レベルの次レベルまでの必要XPを取得
   /// </summary>
   private int GetRequiredXPForLevel(int level)
   {
       if (level < MIN_LEVEL || level > MAX_LEVEL)
       {
           Debug.LogError($"[SkillManager] Invalid level: {level}. Must be between {MIN_LEVEL} and {MAX_LEVEL}");
           return 0;
       }

       // レベル10は最大レベルなので0を返す
       if (level == MAX_LEVEL)
       {
           return 0;
       }

       // levelRequirements配列は0始まりなのでlevel-1でアクセス
       return levelRequirements[level - 1];
   }
   ```

### ステップ3: 初期化処理の実装

4. **InitializeSkillManager()の実装**
   ```csharp
   /// <summary>
   /// SkillManagerの初期化処理
   /// GameManager経由で呼び出される
   /// </summary>
   public void _Initialize()
   {
       if (isInitialized)
       {
           Debug.LogWarning("[SkillManager] Already initialized");
           return;
       }

       // レベルテーブル初期化
       InitializeLevelTable();

       // PersistenceManager参照取得
       GameObject gameManagerObj = GameObject.Find("GameManager");
       if (gameManagerObj == null)
       {
           Debug.LogError("[SkillManager] GameManager not found in scene");
           return;
       }

       persistenceManager = gameManagerObj.GetComponent<PersistenceManager>();
       if (persistenceManager == null)
       {
           Debug.LogError("[SkillManager] PersistenceManager not found on GameManager");
           return;
       }

       // PlayerDataからスキルデータ読み込み
       LoadSkillData();

       isInitialized = true;
       Debug.Log($"[SkillManager] Initialized: Level {currentLevel}, XP {currentXP}");
   }

   /// <summary>
   /// Start()から呼び出される初期化（念のため）
   /// 通常はGameManagerから_Initialize()が呼ばれるが、
   /// エラーハンドリングのため個別でも初期化可能にする
   /// </summary>
   private void InitializeSkillManager()
   {
       if (!isInitialized)
       {
           _Initialize();
       }
   }
   ```

5. **PlayerDataからの読み込み実装**
   ```csharp
   /// <summary>
   /// PlayerDataからスキルレベル・XPを読み込む
   /// データが存在しない場合はデフォルト値（Lv1、XP=0）
   /// </summary>
   private void LoadSkillData()
   {
       if (persistenceManager == null)
       {
           Debug.LogError("[SkillManager] PersistenceManager is null, cannot load data");
           currentLevel = MIN_LEVEL;
           currentXP = 0;
           return;
       }

       // レベル読み込み
       int loadedLevel = persistenceManager._GetInt("Woodcutting_Level", MIN_LEVEL);

       // バリデーション: 1〜10の範囲内であること
       if (loadedLevel < MIN_LEVEL || loadedLevel > MAX_LEVEL)
       {
           Debug.LogWarning($"[SkillManager] Invalid loaded level: {loadedLevel}. Resetting to {MIN_LEVEL}");
           loadedLevel = MIN_LEVEL;
       }

       // XP読み込み
       int loadedXP = persistenceManager._GetInt("Woodcutting_XP", 0);

       // バリデーション: 0〜10500の範囲内であること
       if (loadedXP < 0 || loadedXP > MAX_XP)
       {
           Debug.LogWarning($"[SkillManager] Invalid loaded XP: {loadedXP}. Resetting to 0");
           loadedXP = 0;
       }

       // データ整合性チェック: レベルとXPの矛盾を検出
       if (!ValidateLevelAndXP(loadedLevel, loadedXP))
       {
           Debug.LogError("[SkillManager] Level and XP data inconsistency detected. Resetting to defaults");
           loadedLevel = MIN_LEVEL;
           loadedXP = 0;
       }

       currentLevel = loadedLevel;
       currentXP = loadedXP;

       Debug.Log($"[SkillManager] Skill data loaded: Level {currentLevel}, XP {currentXP}");
   }

   /// <summary>
   /// レベルとXPの整合性をチェック
   /// 例: Lv5なのにXP=50は異常（Lv5に到達するには最低1500XP必要）
   /// </summary>
   private bool ValidateLevelAndXP(int level, int xp)
   {
       // 累計XP計算
       int minRequiredXP = 0;
       for (int i = 0; i < level - 1; i++)
       {
           minRequiredXP += levelRequirements[i];
       }

       // 現在XPが累計必要XPより小さい場合は異常
       if (xp < minRequiredXP)
       {
           Debug.LogWarning($"[SkillManager] Validation failed: Level {level} requires at least {minRequiredXP} XP, but current XP is {xp}");
           return false;
       }

       return true;
   }
   ```

### ステップ4: XP加算とレベルアップ判定の実装（F-10）

6. **AddXP()メソッドの実装**
   ```csharp
   /// <summary>
   /// XPを加算し、レベルアップ判定を実行する
   /// Udon CustomEvent用にpublicで"_"プレフィックス
   /// </summary>
   public void _AddXP(int amount)
   {
       if (!isInitialized)
       {
           Debug.LogError("[SkillManager] Not initialized, cannot add XP");
           return;
       }

       // バリデーション: 0以上の整数
       if (amount <= 0)
       {
           Debug.LogWarning($"[SkillManager] Invalid XP amount: {amount}. Must be positive");
           return;
       }

       // バリデーション: 1000以上は異常値として拒否
       if (amount > 1000)
       {
           Debug.LogError($"[SkillManager] Suspicious XP amount: {amount}. Rejecting to prevent cheating");
           return;
       }

       // 最大レベル到達済みの場合
       if (currentLevel >= MAX_LEVEL)
       {
           Debug.Log("[SkillManager] Already at max level (Lv10). XP will not be added");
           // UI通知: "最大レベルに到達しています"
           SendMaxLevelNotification();
           return;
       }

       // XP加算
       int previousXP = currentXP;
       currentXP += amount;

       // 上限チェック
       if (currentXP > MAX_XP)
       {
           currentXP = MAX_XP;
       }

       Debug.Log($"[SkillManager] Added {amount} XP. Total: {previousXP} → {currentXP}");

       // PlayerDataに保存
       SaveSkillData();

       // レベルアップ判定
       CheckLevelUp();
   }
   ```

7. **CheckLevelUp()メソッドの実装**
   ```csharp
   /// <summary>
   /// レベルアップ条件を満たしているかチェックし、
   /// 満たしていれば連続してレベルアップ処理を実行
   /// </summary>
   private void CheckLevelUp()
   {
       bool leveledUp = false;
       int levelsGained = 0;

       // 連続レベルアップ対応（一度に複数レベル上がる可能性）
       while (currentLevel < MAX_LEVEL)
       {
           int requiredXP = GetRequiredXPForLevel(currentLevel);

           // 次レベルの必要XPに到達していない場合は終了
           if (currentXP < requiredXP)
           {
               break;
           }

           // レベルアップ
           int previousLevel = currentLevel;
           currentLevel++;
           levelsGained++;
           leveledUp = true;

           // 使用したXPを減算
           currentXP -= requiredXP;

           Debug.Log($"[SkillManager] LEVEL UP! {previousLevel} → {currentLevel}");

           // レベルアップイベント発火
           OnLevelUp(currentLevel);
       }

       // レベルアップした場合、最終データを保存
       if (leveledUp)
       {
           SaveSkillData();
           Debug.Log($"[SkillManager] Total levels gained: {levelsGained}");
       }
   }
   ```

### ステップ5: レベルアップイベント処理の実装（F-11）

8. **OnLevelUp()メソッドの実装**
   ```csharp
   /// <summary>
   /// レベルアップ時のイベント処理
   /// HUDManager、SkillEffectsへの通知を行う
   /// </summary>
   private void OnLevelUp(int newLevel)
   {
       Debug.Log($"[SkillManager] OnLevelUp triggered: Level {newLevel}");

       // HUDManagerに通知（レベルアップ演出）
       NotifyHUDManager(newLevel);

       // SkillEffectsに通知（ダメージボーナス再計算）
       NotifySkillEffects(newLevel);

       // 特典解禁チェック
       CheckUnlockRewards(newLevel);
   }

   /// <summary>
   /// HUDManagerへのレベルアップ通知
   /// </summary>
   private void NotifyHUDManager(int newLevel)
   {
       GameObject gameManagerObj = GameObject.Find("GameManager");
       if (gameManagerObj == null)
       {
           Debug.LogWarning("[SkillManager] GameManager not found, cannot notify HUDManager");
           return;
       }

       // HUDManagerコンポーネント取得（将来的に実装される）
       // 現時点では参照が存在しない可能性があるためnullチェック
       UdonSharpBehaviour hudManager = (UdonSharpBehaviour)gameManagerObj.GetComponent(typeof(UdonSharpBehaviour));
       if (hudManager != null && hudManager.GetType().Name == "HUDManager")
       {
           // Udon CustomEventで通知
           hudManager.SendCustomEvent("_ShowLevelUpNotification");
           Debug.Log($"[SkillManager] Notified HUDManager: Level {newLevel}");
       }
       else
       {
           Debug.LogWarning("[SkillManager] HUDManager not found, skipping notification");
       }
   }

   /// <summary>
   /// SkillEffectsへのレベルアップ通知
   /// </summary>
   private void NotifySkillEffects(int newLevel)
   {
       GameObject gameManagerObj = GameObject.Find("GameManager");
       if (gameManagerObj == null)
       {
           Debug.LogWarning("[SkillManager] GameManager not found, cannot notify SkillEffects");
           return;
       }

       // SkillEffectsコンポーネント取得
       UdonSharpBehaviour skillEffects = (UdonSharpBehaviour)gameManagerObj.GetComponent(typeof(UdonSharpBehaviour));
       if (skillEffects != null && skillEffects.GetType().Name == "SkillEffects")
       {
           // Udon CustomEventで通知
           skillEffects.SendCustomEvent("_RecalculateBonuses");
           Debug.Log($"[SkillManager] Notified SkillEffects: Level {newLevel}");
       }
       else
       {
           Debug.LogWarning("[SkillManager] SkillEffects not found, skipping notification");
       }
   }

   /// <summary>
   /// 特定レベルでの特典解禁チェック
   /// Lv3/6/9: 丸太生成数ボーナス
   /// Lv5/8: クリティカル率
   /// Lv10: マスター称号
   /// </summary>
   private void CheckUnlockRewards(int newLevel)
   {
       string rewardMessage = "";

       switch (newLevel)
       {
           case 3:
               rewardMessage = "丸太生成数+1（10%確率）解禁！";
               break;
           case 5:
               rewardMessage = "クリティカル率5%解禁！";
               break;
           case 6:
               rewardMessage = "丸太生成数+1（20%確率）解禁！";
               break;
           case 8:
               rewardMessage = "クリティカル率10%解禁！";
               break;
           case 9:
               rewardMessage = "丸太生成数+1（30%確率）解禁！";
               break;
           case 10:
               rewardMessage = "マスター称号取得！特殊エフェクト解禁！";
               break;
       }

       if (!string.IsNullOrEmpty(rewardMessage))
       {
           Debug.Log($"[SkillManager] Reward unlocked at Level {newLevel}: {rewardMessage}");
           // 将来的にHUDManagerに通知して画面表示
       }
   }
   ```

### ステップ6: データ永続化処理の実装（F-12）

9. **SaveSkillData()メソッドの実装**
   ```csharp
   /// <summary>
   /// 現在のスキルレベル・XPをPlayerDataに保存する
   /// XP獲得時、レベルアップ時に自動的に呼び出される
   /// </summary>
   private void SaveSkillData()
   {
       if (persistenceManager == null)
       {
           Debug.LogError("[SkillManager] PersistenceManager is null, cannot save data");
           return;
       }

       // レベル保存
       persistenceManager._SetInt("Woodcutting_Level", currentLevel);

       // XP保存
       persistenceManager._SetInt("Woodcutting_XP", currentXP);

       Debug.Log($"[SkillManager] Skill data saved: Level {currentLevel}, XP {currentXP}");
   }

   /// <summary>
   /// 手動保存メソッド（デバッグ用、通常は自動保存される）
   /// </summary>
   public void _SaveSkillDataManual()
   {
       SaveSkillData();
       Debug.Log("[SkillManager] Manual save executed");
   }
   ```

### ステップ7: 補助メソッドの実装

10. **進捗計算メソッド**
    ```csharp
    /// <summary>
    /// 現在のレベル内での進捗率を計算（0.0〜1.0）
    /// HUDManagerのプログレスバー表示用
    /// </summary>
    private float CalculateLevelProgress()
    {
        if (currentLevel >= MAX_LEVEL)
        {
            return 1.0f; // 最大レベル到達済み
        }

        int requiredXP = GetRequiredXPForLevel(currentLevel);

        if (requiredXP == 0)
        {
            return 1.0f;
        }

        float progress = (float)currentXP / (float)requiredXP;
        return Mathf.Clamp01(progress);
    }

    /// <summary>
    /// 累計XPを計算（Lv1から現在レベルまでの合計）
    /// </summary>
    private int GetCumulativeXP()
    {
        int cumulativeXP = 0;
        for (int i = 0; i < currentLevel - 1; i++)
        {
            cumulativeXP += levelRequirements[i];
        }
        cumulativeXP += currentXP;
        return cumulativeXP;
    }

    /// <summary>
    /// 最大レベル到達時の通知メソッド
    /// </summary>
    private void SendMaxLevelNotification()
    {
        Debug.Log("[SkillManager] Sending max level notification to HUDManager");
        // 将来的にHUDManagerに通知して画面表示
        // "最大レベルに到達しています"メッセージ
    }
    ```

11. **デバッグ用メソッド**
    ```csharp
    /// <summary>
    /// デバッグ用：強制的にレベルを設定
    /// リリースビルドでは削除またはコメントアウト推奨
    /// </summary>
    public void _DEBUG_SetLevel(int level)
    {
        #if UNITY_EDITOR || DEVELOPMENT_BUILD
        if (level < MIN_LEVEL || level > MAX_LEVEL)
        {
            Debug.LogError($"[SkillManager] DEBUG: Invalid level {level}");
            return;
        }

        currentLevel = level;
        currentXP = 0;
        SaveSkillData();
        Debug.Log($"[SkillManager] DEBUG: Level set to {level}");
        #else
        Debug.LogWarning("[SkillManager] DEBUG methods are disabled in release builds");
        #endif
    }

    /// <summary>
    /// デバッグ用：現在の状態をログ出力
    /// </summary>
    public void _DEBUG_PrintStatus()
    {
        Debug.Log("=== SkillManager Status ===");
        Debug.Log($"Level: {currentLevel} / {MAX_LEVEL}");
        Debug.Log($"XP: {currentXP} / {GetRequiredXPForLevel(currentLevel)}");
        Debug.Log($"Progress: {CalculateLevelProgress() * 100f}%");
        Debug.Log($"Cumulative XP: {GetCumulativeXP()}");
        Debug.Log("==========================");
    }
    ```

### ステップ8: Unity Inspectorでの設定

12. **Unityエディタでの設定手順**
    * Unity Editorを開く
    * Hierarchyで「GameManager」オブジェクトを選択
    * 「Add Component」→「Scripts」→「Skill Manager」を追加
    * Inspector上で設定項目は特になし（すべてコード内で初期化）

13. **GameManager.csとの統合**
    * GameManager.csの初期化処理に以下を追加:
    ```csharp
    private SkillManager skillManager;

    void Start()
    {
        // ... 既存の初期化 ...

        // SkillManager初期化
        skillManager = GetComponent<SkillManager>();
        if (skillManager != null)
        {
            skillManager._Initialize();
        }
        else
        {
            Debug.LogError("[GameManager] SkillManager component not found");
        }
    }

    public SkillManager GetSkillManager()
    {
        return skillManager;
    }
    ```

### ステップ9: テストケース実装

14. **ユニットテスト用スクリプト作成**
    * `/Assets/WoodcutterCamp/Tests/Editor/SkillManagerTests.cs` を作成
    * 以下のテストケースを実装（6ケース以上）:

    ```csharp
    using NUnit.Framework;
    using UnityEngine;

    namespace WoodcutterCamp.Tests
    {
        [TestFixture]
        public class SkillManagerTests
        {
            // テストケース1: 初期状態の確認
            [Test]
            public void Test_InitialState_ShouldBeLv1XP0()
            {
                // Arrange & Act
                var skillManager = CreateTestSkillManager();

                // Assert
                Assert.AreEqual(1, skillManager.GetSkillLevel());
                Assert.AreEqual(0, skillManager.GetCurrentXP());
            }

            // テストケース2: XP加算処理
            [Test]
            public void Test_AddXP_ShouldIncreaseXP()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();

                // Act
                skillManager._AddXP(50);

                // Assert
                Assert.AreEqual(50, skillManager.GetCurrentXP());
            }

            // テストケース3: レベルアップ判定（Lv1→Lv2）
            [Test]
            public void Test_LevelUp_FromLv1ToLv2()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();

                // Act
                skillManager._AddXP(100); // Lv2に必要なXP

                // Assert
                Assert.AreEqual(2, skillManager.GetSkillLevel());
                Assert.AreEqual(0, skillManager.GetCurrentXP()); // XPはリセット
            }

            // テストケース4: 連続レベルアップ（大量XP付与）
            [Test]
            public void Test_MultiLevelUp_WithLargeXP()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();

                // Act
                skillManager._AddXP(1500); // Lv5到達に必要な累計XP

                // Assert
                Assert.AreEqual(5, skillManager.GetSkillLevel());
            }

            // テストケース5: 最大レベル到達後のXP加算
            [Test]
            public void Test_MaxLevel_ShouldNotExceedLv10()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();
                skillManager._DEBUG_SetLevel(10);

                // Act
                skillManager._AddXP(1000); // Lv10でXP加算試行

                // Assert
                Assert.AreEqual(10, skillManager.GetSkillLevel()); // Lv10のまま
            }

            // テストケース6: 異常値XPの拒否
            [Test]
            public void Test_InvalidXP_ShouldBeRejected()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();
                int initialXP = skillManager.GetCurrentXP();

                // Act
                skillManager._AddXP(-50);   // 負の値
                skillManager._AddXP(10000); // 異常に大きい値

                // Assert
                Assert.AreEqual(initialXP, skillManager.GetCurrentXP()); // 変化なし
            }

            // テストケース7: 進捗率計算
            [Test]
            public void Test_ProgressCalculation()
            {
                // Arrange
                var skillManager = CreateTestSkillManager();

                // Act
                skillManager._AddXP(50); // Lv2必要XP=100なので50%

                // Assert
                Assert.AreEqual(0.5f, skillManager.GetLevelProgress(), 0.01f);
            }

            // ヘルパーメソッド
            private SkillManager CreateTestSkillManager()
            {
                var gameObject = new GameObject("TestSkillManager");
                var skillManager = gameObject.AddComponent<SkillManager>();
                skillManager._Initialize();
                return skillManager;
            }
        }
    }
    ```

15. **手動テスト手順書作成**
    * `/Assets/WoodcutterCamp/Tests/Manual/SkillManager_ManualTest.md` を作成:

    ```markdown
    # SkillManager 手動テスト手順書

    ## テスト1: 木伐採によるXP獲得
    1. ワールドに入場
    2. HUD左下のWoodcuttingスキル表示を確認（Lv1, 0 XP）
    3. 木を1本伐採（小さい木 = 10 XP）
    4. XPが10に増加していることを確認

    ## テスト2: レベルアップ演出
    1. 木を10本伐採（累計100 XP）
    2. Lv2にレベルアップすることを確認
    3. 「LEVEL UP!」演出が表示されることを確認
    4. HUDのレベル表示がLv2に更新されることを確認

    ## テスト3: データ永続化
    1. 木を5本伐採（50 XP獲得）
    2. ワールドから退出
    3. 再度ワールドに入場
    4. XPが50のまま維持されていることを確認

    ## テスト4: 協力伐採ボーナス
    1. 他プレイヤーと同じ木を3秒以内に攻撃
    2. 協力伐採ボーナス発動を確認
    3. XPが+20%されていることを確認（10 XP → 12 XP）

    ## テスト5: 最大レベル到達
    1. デバッグコマンドでLv10に設定
    2. 木を伐採してXP獲得試行
    3. 「最大レベルに到達しています」メッセージ表示を確認
    4. Lv10のまま変化しないことを確認
    ```

## 9. 完了条件（Doneの条件）

以下のすべての条件を満たすこと:

### コード実装完了
- [ ] SkillManager.csファイルが作成され、すべてのメソッドが実装されている
- [ ] レベルテーブル（必要XP配列）が正しく定義されている（Lv1〜10）
- [ ] AddXP()メソッドが正常に動作する（F-10実装完了）
- [ ] CheckLevelUp()メソッドが正常に動作する（F-11実装完了）
- [ ] SaveSkillData()メソッドが正常に動作する（F-12実装完了）
- [ ] PersistenceManager経由でPlayerDataへの読み書きが成功する
- [ ] HUDManager、SkillEffectsへの通知インターフェースが実装されている

### テスト完了
- [ ] 6つ以上のユニットテストが実装され、すべてパスする
- [ ] 手動テスト手順書に従ったテストをすべて実施し、異常なし
- [ ] エラーハンドリングが適切に動作する（異常値XP、null参照など）
- [ ] データ永続化が正常に動作する（ログアウト→再ログインでデータ維持）

### ドキュメント完了
- [ ] コード内に適切なコメント・XMLドキュメントコメントが記載されている
- [ ] デバッグログが適切に出力される（"[SkillManager]"プレフィックス）
- [ ] 手動テスト手順書が作成されている

### 統合確認
- [ ] GameManager.csに統合され、初期化処理が正常に動作する
- [ ] Unity Inspector上でコンポーネントが正しく設定されている
- [ ] ビルドエラー、コンパイルエラーがない
- [ ] UdonSharpコンパイルが成功する

## 10. テスト観点／テストケース

### ユニットテスト（自動テスト）

| テストID | テスト項目 | 期待結果 | 優先度 |
|---------|-----------|---------|--------|
| UT-001 | 初期状態確認 | Lv1、XP=0で開始 | 必須 |
| UT-002 | XP加算処理 | 指定XPが正しく加算される | 必須 |
| UT-003 | レベルアップ判定（Lv1→Lv2） | 100XP到達でLv2に昇格 | 必須 |
| UT-004 | 連続レベルアップ | 大量XP付与で複数レベル同時昇格 | 必須 |
| UT-005 | 最大レベル到達後のXP加算 | Lv10でXP加算されない | 必須 |
| UT-006 | 異常値XPの拒否 | 負の値・異常値が拒否される | 必須 |
| UT-007 | 進捗率計算 | 正しい進捗率（0.0〜1.0）が返される | 高 |
| UT-008 | データバリデーション | 不正なレベル・XPデータを検出 | 高 |

### 統合テスト（手動テスト）

| テストID | テスト項目 | 手順 | 期待結果 | 優先度 |
|---------|-----------|------|---------|--------|
| IT-001 | 木伐採によるXP獲得 | 木を1本伐採 | XPが正しく加算される | 必須 |
| IT-002 | レベルアップ演出 | 累計100XP獲得 | Lv2昇格、演出表示 | 必須 |
| IT-003 | データ永続化 | ワールド退出→再入場 | XP・レベルが維持される | 必須 |
| IT-004 | 協力伐採ボーナス | 2人で同じ木を攻撃 | XP+20%される | 高 |
| IT-005 | 最大レベル到達 | Lv10で木伐採 | メッセージ表示、Lv10維持 | 中 |
| IT-006 | PersistenceManager連携 | データ保存・読み込み | エラーなく動作 | 必須 |

### パフォーマンステスト

| テストID | テスト項目 | 測定項目 | 合格基準 | 優先度 |
|---------|-----------|---------|---------|--------|
| PT-001 | XP加算処理時間 | AddXP()実行時間 | 50ms以内 | 高 |
| PT-002 | レベルアップ判定時間 | CheckLevelUp()実行時間 | 50ms以内 | 高 |
| PT-003 | データ保存時間 | SaveSkillData()実行時間 | 100ms以内 | 中 |

### エラーハンドリングテスト

| テストID | テスト項目 | トリガー | 期待動作 | 優先度 |
|---------|-----------|---------|---------|--------|
| EH-001 | PersistenceManager未初期化 | persistenceManager == null | エラーログ出力、デフォルト値使用 | 必須 |
| EH-002 | 異常なレベルデータ | レベル=99で読み込み | Lv1にリセット、警告ログ | 必須 |
| EH-003 | 異常なXPデータ | XP=999999で読み込み | 0にリセット、警告ログ | 必須 |
| EH-004 | レベル・XP不整合 | Lv5なのにXP=50 | データリセット、エラーログ | 必須 |

## 11. 成果物

### ソースコード
* `/Assets/WoodcutterCamp/Scripts/Progression/SkillManager.cs`（約600行）

### テストコード
* `/Assets/WoodcutterCamp/Tests/Editor/SkillManagerTests.cs`（約200行）

### ドキュメント
* `/Assets/WoodcutterCamp/Tests/Manual/SkillManager_ManualTest.md`（手動テスト手順書）

### Unity設定
* GameManagerオブジェクトにSkillManagerコンポーネントが追加されている
* GameManager.csにSkillManager初期化コードが統合されている

## 12. 備考

### 注意点

1. **SOLID原則の厳守**
   * SkillManagerは「スキル管理」のみに専念し、UI表示やダメージ計算は他モジュールに委譲する
   * HUDManager、SkillEffectsとの連携はイベント通知のみで、直接的な依存を避ける

2. **パフォーマンス考慮**
   * Update()は使用せず、イベント駆動で実装する
   * GetComponent()はStart()で1回のみ実行し、変数にキャッシュする

3. **エラーハンドリング**
   * すべてのpublicメソッドで引数バリデーションを実施する
   * PlayerData読み込み失敗時はデフォルト値（Lv1、XP=0）で動作継続する
   * 異常値検出時はエラーログを出力し、データをリセットする

4. **デバッグ機能**
   * _DEBUG_SetLevel()などのデバッグメソッドはリリースビルドで無効化する
   * すべてのログに"[SkillManager]"プレフィックスを付ける

### 依存関係

* **WI-0003 (PersistenceManager)**: 必須（データ永続化）
* **WI-0006 (SkillEffects)**: 連携先（レベルアップ通知）
* **WI-0017 (HUDManager)**: 連携先（UI更新通知）

### 次の作業

* **WI-0006**: SkillEffectsの実装（ダメージボーナス計算）
* **WI-0017**: HUDManagerの実装（UI表示）

### 制約事項

* レベル上限: Lv10（Phase 1仕様）
* XP範囲: 0〜10500
* Woodcuttingスキルのみ実装（Carrying/Craftingスキルは Phase 2）

### 参考資料

* **TDD.md**: Module M-04の詳細設計
* **FSD.md**: FNC-004スキル成長システムの機能仕様
* **PRDv3.md**: レベルテーブル（Section 2.2.1）、XP獲得源（Section 2.2.2）

---

**作業開始前の最終確認**
- [ ] WI-0003 (PersistenceManager)が実装済みであることを確認
- [ ] Unity 2022.3.22f1、VRChat SDK 3.9.0、UdonSharp 1.1.9がインストール済みであることを確認
- [ ] `/Assets/WoodcutterCamp/Scripts/Progression/`ディレクトリが存在することを確認（なければ作成）

**作業完了後の提出物チェックリスト**
- [ ] SkillManager.csのコードレビュー完了
- [ ] 6つ以上のユニットテストがすべてパス
- [ ] 手動テスト5項目をすべて実施し、結果を記録
- [ ] GameManagerとの統合確認完了
- [ ] UdonSharpコンパイル成功、エラーなし

---

**作業指示書バージョン**: 1.0
**作成日**: 2025-11-17
**承認者**: （Phase 1実装開始前に承認必要）
