# 作業指示書：WI-0013 - CoinManager実装

## 1. 作業ID
**WI-0013**

## 2. 作業名
**CoinManager（コイン管理システム）の実装**

## 3. 作業目的
きこりコインの加算・消費管理機能を実装し、経済システムの基盤を構築する。コイン獲得時の上限チェック（99,999コイン）、消費時の残高チェック、PlayerDataへの永続化を実現し、HUDManagerやShopManagerへの通知機能を提供する。

## 4. 対象

* **対象システム**: 森のきこりキャンプ - VRChatワールド Phase 1
* **対象モジュール**: M-06 CoinManager（Economy Layer）
* **対象ファイルパス**: `/Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`
* **依存モジュール**: M-03 PersistenceManager（データ永続化）

## 5. 前提条件

### 環境要件
* Unity 2022.3.22f1 LTS
* VRChat SDK 3.9.0
* UdonSharp 1.1.9
* 開発ブランチ: `feature/phase1-core`

### 実装済み依存モジュール
* **WI-0001**: GameManager Singleton実装済み
* **WI-0003**: PersistenceManager実装済み（PlayerData I/O機能）

### 初期状態
* プレイヤーは初回入場時、コイン残高0の状態でスタート
* 初回訪問ボーナス（50コイン）は別モジュール（WI-0014: Login Bonus）で実装予定
* PersistenceManager経由でPlayerDataへの読み書きが可能な状態

## 6. 入力

### CoinManager.AddCoins()への入力（伐採報酬）
```csharp
// 木伐採完了時の呼び出し例（AxeInteractionから）
public void _OnTreeCutComplete(int treeType, bool isCooperative)
{
    CoinManager coinManager = GameManager.Instance.GetCoinManager();

    // コイン量を決定
    int coinAmount = 0;
    switch (treeType)
    {
        case 0: coinAmount = 3; break;   // Pine (小さい木)
        case 1: coinAmount = 5; break;   // Oak (中くらいの木)
        case 2: coinAmount = 10; break;  // Ancient (大きな木)
    }

    // 協力伐採ボーナス（+2コイン）
    if (isCooperative)
    {
        coinAmount += 2;
    }

    // コイン加算
    coinManager._AddCoins(coinAmount, "tree_cutting");
}
```

### CoinManager.AddCoins()への入力（運搬報酬）
```csharp
// 丸太運搬完了時の呼び出し例（LogPickupから）
public void _OnLogDelivered(float carriedDistance)
{
    CoinManager coinManager = GameManager.Instance.GetCoinManager();

    // コイン量を計算（10m = 3コイン、最大100m = 30コイン）
    int coinAmount = Mathf.Min(Mathf.FloorToInt(carriedDistance / 10f) * 3, 30);

    // コイン加算
    coinManager._AddCoins(coinAmount, "log_delivery");
}
```

### CoinManager.DeductCoins()への入力（ショップ購入）
```csharp
// ショップアイテム購入時の呼び出し例（ShopManagerから）
public bool _PurchaseItem(int itemID, int price)
{
    CoinManager coinManager = GameManager.Instance.GetCoinManager();

    // コイン消費試行
    bool success = coinManager._DeductCoins(price, "shop_purchase");

    if (success)
    {
        Debug.Log($"[ShopManager] Purchase successful. Item {itemID} bought for {price} coins");
        return true;
    }
    else
    {
        Debug.LogWarning($"[ShopManager] Purchase failed. Insufficient coins");
        return false;
    }
}
```

### PlayerData読み込み時の入力
```csharp
// ワールド入場時（GameManager.Start()から）
int savedCoins = PersistenceManager.GetInt("WoodcutterCoins", 0);
int savedTotalEarned = PersistenceManager.GetInt("TotalCoinsEarned", 0);
```

## 7. 出力

### コイン加算成功時の出力
```csharp
// HUDManagerへの通知
public void _OnCoinsAdded(int amount, int newBalance)
{
    HUDManager hudManager = GameManager.Instance.GetHUDManager();
    hudManager._UpdateCoinDisplay(newBalance);
    hudManager._ShowCoinNotification($"+{amount} コイン", true);
}
```

### コイン上限到達時の出力
```csharp
// 上限到達時のログとUI通知
Debug.LogWarning("[CoinManager] Coin cap reached (99,999). Cannot add more coins.");

HUDManager hudManager = GameManager.Instance.GetHUDManager();
hudManager._ShowNotification("コイン所持上限に達しています", 2f);
```

### コイン消費成功時の出力
```csharp
// コイン消費後の残高更新
public void _OnCoinsDeducted(int amount, int newBalance)
{
    HUDManager hudManager = GameManager.Instance.GetHUDManager();
    hudManager._UpdateCoinDisplay(newBalance);
}
```

### コイン不足時の出力
```csharp
// 残高不足時のエラー通知
Debug.LogWarning($"[CoinManager] Insufficient coins. Required: {amount}, Current: {currentCoins}");

HUDManager hudManager = GameManager.Instance.GetHUDManager();
hudManager._ShowNotification($"コイン不足です（{currentCoins}/{amount}）", 2f);
```

### PlayerDataへの保存
```csharp
// コイン加算・消費時
PersistenceManager.SetInt("WoodcutterCoins", currentCoins);
PersistenceManager.SetInt("TotalCoinsEarned", totalCoinsEarned);
```

### 他モジュールからの参照
```csharp
// ShopManager、HUDManagerからの参照例
int currentBalance = coinManager.GetCurrentCoins();
int totalEarned = coinManager.GetTotalCoinsEarned();
bool canAfford = coinManager.CanAfford(100);  // 100コイン支払い可能かチェック
```

## 8. 作業手順

### ステップ1: CoinManager.csの作成とクラス構造定義

1. **ディレクトリ作成**
   - Unityエディタを起動
   - Projectビューで右クリック → Create → Folder
   - パス: `Assets/WoodcutterCamp/Scripts/Economy/`
   - Economyフォルダが存在しない場合は作成

2. **スクリプトファイル作成**
   - Economyフォルダ内で右クリック → Create → C# Script
   - ファイル名: `CoinManager`
   - 作成されたファイルをダブルクリックしてエディタで開く

3. **基本クラス構造の実装**
   ```csharp
   using UdonSharp;
   using UnityEngine;
   using VRC.SDKBase;
   using VRC.Udon;

   namespace WoodcutterCamp.Economy
   {
       /// <summary>
       /// きこりコインの加算・消費管理を担当するモジュール
       /// SOLID原則: 単一責任 - コイン管理のみに専念
       /// </summary>
       [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
       public class CoinManager : UdonSharpBehaviour
       {
           // === 定数定義 ===
           /// <summary>コイン所持上限</summary>
           private const int MAX_COINS = 99999;

           /// <summary>初期コイン数</summary>
           private const int INITIAL_COINS = 0;

           // === プライベート変数 ===
           /// <summary>現在の所持コイン数</summary>
           private int currentCoins = INITIAL_COINS;

           /// <summary>累計獲得コイン数（統計用）</summary>
           private int totalCoinsEarned = 0;

           /// <summary>累計消費コイン数（統計用）</summary>
           private int totalCoinsSpent = 0;

           // === 参照キャッシュ ===
           private PersistenceManager persistenceManager;
           private HUDManager hudManager;
           private bool isInitialized = false;

           // === パブリックゲッター ===
           /// <summary>現在のコイン残高を取得</summary>
           public int GetCurrentCoins() => currentCoins;

           /// <summary>累計獲得コイン数を取得</summary>
           public int GetTotalCoinsEarned() => totalCoinsEarned;

           /// <summary>累計消費コイン数を取得</summary>
           public int GetTotalCoinsSpent() => totalCoinsSpent;

           /// <summary>指定金額を支払い可能かチェック</summary>
           public bool CanAfford(int amount) => currentCoins >= amount;

           // === 初期化 ===
           void Start()
           {
               InitializeCoinManager();
           }

           // 以下、各メソッドを実装（次ステップ）
       }
   }
   ```

### ステップ2: 初期化処理の実装

4. **InitializeCoinManager()メソッドの実装**
   ```csharp
   /// <summary>
   /// CoinManagerの初期化処理
   /// GameManager.Start()から1秒遅延後に呼ばれることを想定
   /// </summary>
   private void InitializeCoinManager()
   {
       if (isInitialized)
       {
           Debug.LogWarning("[CoinManager] Already initialized. Skipping...");
           return;
       }

       Debug.Log("[CoinManager] Initializing...");

       // GameManagerからPersistenceManager参照を取得
       persistenceManager = GameManager.Instance.GetPersistenceManager();
       if (persistenceManager == null)
       {
           Debug.LogError("[CoinManager] PersistenceManager reference is null. Cannot initialize.");
           return;
       }

       // HUDManager参照を取得（nullチェックのみ、初期化順によってはnullの可能性あり）
       hudManager = GameManager.Instance.GetHUDManager();

       // PlayerDataからコインデータを読み込み
       LoadCoinData();

       isInitialized = true;
       Debug.Log($"[CoinManager] Initialization completed. Current coins: {currentCoins}, Total earned: {totalCoinsEarned}");
   }
   ```

5. **LoadCoinData()メソッドの実装**
   ```csharp
   /// <summary>
   /// PlayerDataからコインデータを読み込む
   /// データが存在しない場合はデフォルト値（0）を使用
   /// </summary>
   private void LoadCoinData()
   {
       // PlayerDataからコイン残高を読み込み
       currentCoins = persistenceManager.GetInt("WoodcutterCoins", INITIAL_COINS);

       // PlayerDataから累計獲得コインを読み込み
       totalCoinsEarned = persistenceManager.GetInt("TotalCoinsEarned", 0);

       // PlayerDataから累計消費コインを読み込み
       totalCoinsSpent = persistenceManager.GetInt("TotalCoinsSpent", 0);

       // データ整合性チェック
       if (currentCoins < 0 || currentCoins > MAX_COINS)
       {
           Debug.LogWarning($"[CoinManager] Invalid coin data detected (currentCoins: {currentCoins}). Resetting to 0.");
           currentCoins = INITIAL_COINS;
       }

       if (totalCoinsEarned < 0)
       {
           Debug.LogWarning($"[CoinManager] Invalid totalCoinsEarned detected ({totalCoinsEarned}). Resetting to 0.");
           totalCoinsEarned = 0;
       }

       if (totalCoinsSpent < 0)
       {
           Debug.LogWarning($"[CoinManager] Invalid totalCoinsSpent detected ({totalCoinsSpent}). Resetting to 0.");
           totalCoinsSpent = 0;
       }

       Debug.Log($"[CoinManager] Coin data loaded. Balance: {currentCoins}, Total earned: {totalCoinsEarned}, Total spent: {totalCoinsSpent}");
   }
   ```

### ステップ3: コイン加算処理の実装

6. **_AddCoins()メソッドの実装（Udon CustomEvent用のパブリックメソッド）**
   ```csharp
   /// <summary>
   /// コインを加算する（Udon CustomEvent用）
   /// </summary>
   /// <param name="amount">加算するコイン数</param>
   /// <param name="source">コイン獲得源（ログ用、例: "tree_cutting", "log_delivery"）</param>
   public void _AddCoins(int amount, string source)
   {
       // 初期化チェック
       if (!isInitialized)
       {
           Debug.LogError("[CoinManager] Not initialized. Cannot add coins.");
           return;
       }

       // 入力バリデーション: 負値チェック
       if (amount <= 0)
       {
           Debug.LogWarning($"[CoinManager] Invalid coin amount: {amount}. Must be positive.");
           return;
       }

       // 上限チェック
       if (currentCoins >= MAX_COINS)
       {
           Debug.LogWarning("[CoinManager] Coin cap reached (99,999). Cannot add more coins.");

           // HUDManager通知
           if (hudManager != null)
           {
               hudManager._ShowNotification("コイン所持上限に達しています", 2f);
           }

           return;
       }

       // 上限を超えないように加算量を調整
       int actualAmount = amount;
       if (currentCoins + amount > MAX_COINS)
       {
           actualAmount = MAX_COINS - currentCoins;
           Debug.LogWarning($"[CoinManager] Adding {amount} coins would exceed cap. Adjusted to {actualAmount}.");
       }

       // コイン加算
       int previousCoins = currentCoins;
       currentCoins += actualAmount;
       totalCoinsEarned += actualAmount;

       Debug.Log($"[CoinManager] Added {actualAmount} coins from [{source}]. Previous: {previousCoins}, New: {currentCoins}");

       // PlayerDataに保存
       SaveCoinData();

       // HUDManager通知
       NotifyHUDCoinAdded(actualAmount, currentCoins);
   }
   ```

7. **SaveCoinData()メソッドの実装**
   ```csharp
   /// <summary>
   /// コインデータをPlayerDataに保存する
   /// </summary>
   private void SaveCoinData()
   {
       if (persistenceManager == null)
       {
           Debug.LogError("[CoinManager] PersistenceManager is null. Cannot save coin data.");
           return;
       }

       persistenceManager.SetInt("WoodcutterCoins", currentCoins);
       persistenceManager.SetInt("TotalCoinsEarned", totalCoinsEarned);
       persistenceManager.SetInt("TotalCoinsSpent", totalCoinsSpent);

       Debug.Log($"[CoinManager] Coin data saved. Balance: {currentCoins}");
   }
   ```

8. **NotifyHUDCoinAdded()メソッドの実装**
   ```csharp
   /// <summary>
   /// コイン加算をHUDManagerに通知する
   /// </summary>
   private void NotifyHUDCoinAdded(int amount, int newBalance)
   {
       if (hudManager == null)
       {
           // HUDManagerがまだ初期化されていない可能性がある（Phase 1初期実装では許容）
           Debug.LogWarning("[CoinManager] HUDManager is null. Cannot notify coin addition.");
           return;
       }

       // 残高表示更新
       hudManager._UpdateCoinDisplay(newBalance);

       // 獲得通知表示（"+5 コイン"など）
       hudManager._ShowCoinNotification($"+{amount} コイン", true);
   }
   ```

### ステップ4: コイン消費処理の実装

9. **_DeductCoins()メソッドの実装（Udon CustomEvent用のパブリックメソッド）**
   ```csharp
   /// <summary>
   /// コインを消費する（Udon CustomEvent用）
   /// </summary>
   /// <param name="amount">消費するコイン数</param>
   /// <param name="purpose">消費目的（ログ用、例: "shop_purchase"）</param>
   /// <returns>消費成功ならtrue、残高不足でfalse</returns>
   public bool _DeductCoins(int amount, string purpose)
   {
       // 初期化チェック
       if (!isInitialized)
       {
           Debug.LogError("[CoinManager] Not initialized. Cannot deduct coins.");
           return false;
       }

       // 入力バリデーション: 負値チェック
       if (amount <= 0)
       {
           Debug.LogWarning($"[CoinManager] Invalid deduction amount: {amount}. Must be positive.");
           return false;
       }

       // 残高チェック
       if (currentCoins < amount)
       {
           Debug.LogWarning($"[CoinManager] Insufficient coins for [{purpose}]. Required: {amount}, Current: {currentCoins}");

           // HUDManager通知
           if (hudManager != null)
           {
               hudManager._ShowNotification($"コイン不足です（{currentCoins}/{amount}）", 2f);
           }

           return false;
       }

       // コイン消費
       int previousCoins = currentCoins;
       currentCoins -= amount;
       totalCoinsSpent += amount;

       Debug.Log($"[CoinManager] Deducted {amount} coins for [{purpose}]. Previous: {previousCoins}, New: {currentCoins}");

       // PlayerDataに保存
       SaveCoinData();

       // HUDManager通知
       NotifyHUDCoinDeducted(amount, currentCoins);

       return true;
   }
   ```

10. **NotifyHUDCoinDeducted()メソッドの実装**
    ```csharp
    /// <summary>
    /// コイン消費をHUDManagerに通知する
    /// </summary>
    private void NotifyHUDCoinDeducted(int amount, int newBalance)
    {
        if (hudManager == null)
        {
            Debug.LogWarning("[CoinManager] HUDManager is null. Cannot notify coin deduction.");
            return;
        }

        // 残高表示更新
        hudManager._UpdateCoinDisplay(newBalance);
    }
    ```

### ステップ5: デバッグ機能の実装

11. **GetCoinDataSummary()メソッドの実装（デバッグ用）**
    ```csharp
    /// <summary>
    /// コインデータのサマリーを取得する（デバッグ用）
    /// </summary>
    public string GetCoinDataSummary()
    {
        return $"[CoinManager] Balance: {currentCoins}/{MAX_COINS}, Total Earned: {totalCoinsEarned}, Total Spent: {totalCoinsSpent}";
    }
    ```

12. **OnValidate()メソッドの実装（Unity Inspectorでの検証用）**
    ```csharp
    #if UNITY_EDITOR
    private void OnValidate()
    {
        // Inspectorでの値変更時に範囲チェック
        if (currentCoins < 0)
        {
            currentCoins = 0;
            Debug.LogWarning("[CoinManager] currentCoins cannot be negative. Reset to 0.");
        }

        if (currentCoins > MAX_COINS)
        {
            currentCoins = MAX_COINS;
            Debug.LogWarning($"[CoinManager] currentCoins exceeds cap. Clamped to {MAX_COINS}.");
        }
    }
    #endif
    ```

### ステップ6: GameManagerへの登録

13. **GameManagerスクリプトの編集**
    - `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs` を開く
    - CoinManager参照用のフィールドを追加
    ```csharp
    [Header("Economy System")]
    [SerializeField] private CoinManager coinManager;
    ```

14. **CoinManager参照用ゲッターの追加**
    ```csharp
    /// <summary>
    /// CoinManager参照を取得する（依存性注入）
    /// </summary>
    public CoinManager GetCoinManager()
    {
        if (coinManager == null)
        {
            Debug.LogError("[GameManager] CoinManager reference is null");
        }
        return coinManager;
    }
    ```

15. **GameManagerのInitialize処理に追加**
    ```csharp
    private void InitializeAllModules()
    {
        Debug.Log("[GameManager] Initializing all modules...");

        // ... 既存の初期化処理 ...

        // CoinManager初期化（PersistenceManager初期化後に実行）
        if (coinManager != null)
        {
            // CoinManager.Start()が自動実行されるため、明示的な初期化は不要
            Debug.Log("[GameManager] CoinManager registered");
        }
        else
        {
            Debug.LogError("[GameManager] CoinManager is not assigned");
        }
    }
    ```

### ステップ7: Unity Sceneへの配置

16. **GameManagerオブジェクトへのアタッチ**
    - Hierarchyビューで `GameManager` オブジェクトを選択
    - Inspectorビューで `Add Component` をクリック
    - 「CoinManager」と入力して、作成したCoinManagerスクリプトを追加

17. **GameManagerのCoinManager参照設定**
    - GameManagerオブジェクトを選択
    - InspectorのGameManagerコンポーネント内の「Coin Manager」フィールドに、同じオブジェクトにアタッチされているCoinManagerコンポーネントをドラッグ&ドロップ

18. **シーン保存**
    - `File` → `Save` でシーンを保存
    - パス: `Assets/WoodcutterCamp/Scenes/MainScene.unity`

### ステップ8: UdonSharpへのコンパイル確認

19. **UdonSharpコンパイル実行**
    - Unityメニューバー → `VRChat SDK` → `Utilities` → `Force Compile All UdonSharp Programs`
    - Consoleビューでエラーがないことを確認

20. **コンパイルエラーが出た場合の対処**
    - エラーメッセージを確認し、以下をチェック
      - `using` ステートメントの不足（UnityEngine、VRC.SDKBase等）
      - UdonSharpで使用できないC#機能の使用（LINQ、Dictionary等）
      - 変数名のタイポ
    - エラー修正後、再度Force Compileを実行

## 9. 完了条件（Doneの条件）

### 機能要件の充足
- [ ] CoinManager.csファイルが正しいパスに作成されている
- [ ] コイン加算機能（_AddCoins()）が実装され、上限99,999チェックが機能している
- [ ] コイン消費機能（_DeductCoins()）が実装され、残高チェックが機能している
- [ ] PlayerDataへのコイン保存・読み込みが正常に動作している
- [ ] 累計獲得コイン、累計消費コインの記録が正常に動作している
- [ ] GameManagerからCoinManager参照が取得できる

### コード品質
- [ ] 全メソッドにXMLドキュメントコメント（`/// <summary>`）が記載されている
- [ ] ログ出力が適切に実装されている（`[CoinManager]` プレフィックス付き）
- [ ] 入力バリデーション（負値チェック、上限チェック）が実装されている
- [ ] UdonSharpコンパイルエラーが0件である

### Unity設定
- [ ] CoinManagerがGameManagerオブジェクトにアタッチされている
- [ ] GameManagerのInspectorでCoinManager参照が設定されている
- [ ] シーンが保存されている

## 10. テスト観点／テストケース

### テスト種別
**ユニットテスト**（Unity Play Modeでの手動テスト）

### テストケース

#### 正常系テスト

**TC-001: コイン加算の基本動作**
1. **手順**:
   - Unity Play Modeで実行
   - Consoleビューでログ確認用に `Debug.Log("[Test] Starting TC-001")` を追加したテストスクリプトを実行
   - `coinManager._AddCoins(100, "test_add")` を呼び出し
2. **期待結果**:
   - `currentCoins` が0から100に増加
   - `totalCoinsEarned` が100に増加
   - Consoleに `[CoinManager] Added 100 coins from [test_add]. Previous: 0, New: 100` と表示
   - PlayerDataに保存される

**TC-002: 複数回のコイン加算**
1. **手順**:
   - 初期状態（currentCoins = 0）
   - `_AddCoins(50, "test1")` → `_AddCoins(30, "test2")` → `_AddCoins(20, "test3")` を順次実行
2. **期待結果**:
   - `currentCoins` が100になる（50+30+20）
   - `totalCoinsEarned` が100になる
   - 各加算時にログが出力される

**TC-003: コイン消費の基本動作**
1. **手順**:
   - 初期状態（currentCoins = 100）
   - `bool result = _DeductCoins(30, "test_purchase")` を実行
2. **期待結果**:
   - `result` が `true`
   - `currentCoins` が70に減少（100-30）
   - `totalCoinsSpent` が30に増加
   - Consoleに `[CoinManager] Deducted 30 coins for [test_purchase]. Previous: 100, New: 70` と表示

**TC-004: コイン上限到達時の加算**
1. **手順**:
   - 初期状態（currentCoins = 99,990）
   - `_AddCoins(20, "test_cap")` を実行
2. **期待結果**:
   - `currentCoins` が99,999になる（上限で停止）
   - 実際の加算量は9コイン（調整済み）
   - Consoleに警告ログ `[CoinManager] Adding 20 coins would exceed cap. Adjusted to 9.` が表示
   - `totalCoinsEarned` は9コイン分のみ増加

**TC-005: PlayerDataからのロード**
1. **手順**:
   - PlayerDataに `WoodcutterCoins = 500` を事前設定
   - Unity Play Modeで実行（CoinManager初期化）
2. **期待結果**:
   - `currentCoins` が500でロードされる
   - Consoleに `[CoinManager] Coin data loaded. Balance: 500...` と表示

#### 異常系テスト

**TC-006: 残高不足時のコイン消費**
1. **手順**:
   - 初期状態（currentCoins = 50）
   - `bool result = _DeductCoins(100, "test_insufficient")` を実行
2. **期待結果**:
   - `result` が `false`
   - `currentCoins` は50のまま（変化なし）
   - Consoleに警告ログ `[CoinManager] Insufficient coins for [test_insufficient]. Required: 100, Current: 50` が表示
   - HUDManagerに通知される（HUDManager実装後に確認）

**TC-007: 負値のコイン加算試行**
1. **手順**:
   - `_AddCoins(-50, "test_negative")` を実行
2. **期待結果**:
   - コイン加算が実行されない
   - `currentCoins` が変化しない
   - Consoleに警告ログ `[CoinManager] Invalid coin amount: -50. Must be positive.` が表示

**TC-008: 負値のコイン消費試行**
1. **手順**:
   - `bool result = _DeductCoins(-30, "test_negative_deduct")` を実行
2. **期待結果**:
   - `result` が `false`
   - `currentCoins` が変化しない
   - Consoleに警告ログ `[CoinManager] Invalid deduction amount: -30. Must be positive.` が表示

**TC-009: 上限超過状態でのコイン加算**
1. **手順**:
   - 初期状態（currentCoins = 99,999）
   - `_AddCoins(100, "test_already_max")` を実行
2. **期待結果**:
   - コイン加算が実行されない
   - `currentCoins` は99,999のまま
   - Consoleに警告ログ `[CoinManager] Coin cap reached (99,999). Cannot add more coins.` が表示

**TC-010: データ破損時のロード**
1. **手順**:
   - PlayerDataに異常値 `WoodcutterCoins = -500` を設定
   - Unity Play Modeで実行
2. **期待結果**:
   - `currentCoins` が0にリセットされる（デフォルト値）
   - Consoleに警告ログ `[CoinManager] Invalid coin data detected (currentCoins: -500). Resetting to 0.` が表示

**TC-011: 初期化前のメソッド呼び出し**
1. **手順**:
   - CoinManager.Start()を意図的にスキップした状態で `_AddCoins(50, "test_not_init")` を実行
2. **期待結果**:
   - コイン加算が実行されない
   - Consoleにエラーログ `[CoinManager] Not initialized. Cannot add coins.` が表示

### 統合テスト（Phase 1後期実装後）

**TC-INT-001: 木伐採からコイン加算まで**
1. **手順**:
   - AxeInteractionで木を伐採完了
   - AxeInteractionから `_AddCoins()` 呼び出し
2. **期待結果**:
   - 木の種類に応じたコイン（3, 5, 10）が加算される
   - HUDにコイン獲得通知が表示される

**TC-INT-002: 丸太運搬からコイン加算まで**
1. **手順**:
   - LogPickupで丸太を50m運搬して納品
   - LogPickupから `_AddCoins(15, "log_delivery")` 呼び出し
2. **期待結果**:
   - 15コイン（50m/10×3）が加算される
   - HUDにコイン獲得通知が表示される

**TC-INT-003: ショップでのアイテム購入**
1. **手順**:
   - 初期状態（currentCoins = 100）
   - ShopManagerから `_DeductCoins(50, "shop_purchase")` 呼び出し
   - アイテム購入処理
2. **期待結果**:
   - コイン消費が成功（`true` 返却）
   - `currentCoins` が50に減少
   - HUDのコイン表示が更新される

## 11. 成果物

### 変更ファイル
* **新規作成**: `/Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`
  - CoinManagerクラスの完全実装（約350行）

* **変更**: `/Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`
  - CoinManager参照用フィールド追加
  - GetCoinManager()メソッド追加
  - InitializeAllModules()にCoinManager登録処理追加

* **変更**: `/Assets/WoodcutterCamp/Scenes/MainScene.unity`
  - GameManagerオブジェクトにCoinManagerコンポーネント追加
  - GameManagerのCoinManager参照設定

### 追加テストコード
* **新規作成**: `/Assets/WoodcutterCamp/Tests/Editor/CoinManagerTests.cs`（オプション）
  - Unity Test Frameworkを使用した自動テストコード
  - 上記テストケース（TC-001〜TC-011）の自動化

### 更新したドキュメント
* **TDD.md**: Module M-06の実装完了ステータス更新
* **開発進捗ログ**: WI-0013完了記録

## 12. 備考

### 注意点

1. **HUDManager未実装時の対応**
   - Phase 1初期段階でHUDManager（WI-0017）がまだ実装されていない場合、`hudManager == null` チェックで安全に処理をスキップする設計となっています
   - HUDManager実装後、自動的に通知機能が有効化されます

2. **コイン獲得源の実装順序**
   - WI-0010（Damage Calculation）で伐採報酬のコイン加算を実装
   - WI-0016（Delivery Zone）で運搬報酬のコイン加算を実装
   - 本WI-0013では、これらから呼ばれる基盤機能のみを実装します

3. **ログインボーナス機能**
   - 初回ログインボーナス（50コイン）、デイリーボーナス（20コイン）は別WI（WI-0014: Login Bonus）で実装予定
   - 本WI-0013では、基本的なコイン加算・消費機能のみを実装します

4. **UdonSharpの制約**
   - C#の `??` null合体演算子、LINQ、Dictionaryは使用できません
   - 代わりに明示的なnullチェック（`if (obj == null)`）、配列、for/whileループを使用します

5. **PlayerDataのサイズ制約**
   - 本モジュールが使用するPlayerDataサイズ: 約12バイト
     - WoodcutterCoins (int): 4バイト
     - TotalCoinsEarned (int): 4バイト
     - TotalCoinsSpent (int): 4バイト
   - 全体の100KB制限に対して影響は微小です

### 次の作業

**依存する作業**:
- **WI-0014**: Login Bonus実装（初回ログインボーナス、デイリーボーナス）
- **WI-0017**: HUDManager実装（コイン表示UI、通知機能）
- **WI-0019**: ShopManager実装（コイン消費処理の統合）

**本WIに依存する作業**:
- **WI-0010**: Damage Calculation（木伐採時のコイン加算呼び出し）
- **WI-0016**: Delivery Zone（丸太納品時のコイン加算呼び出し）

### 参考資料

* **PRD.md**: Section 7「通貨・交換・コレクション」
  - きこりコインの獲得源と用途
  - コイン上限99,999の仕様

* **FSD.md**: FNC-005「経済システム」
  - コイン獲得の詳細フロー
  - ショップでの消費フロー
  - データエンティティ定義（CoinData）

* **TDD.md**: Module M-06「CoinManager」
  - Functions: F-16（Add Coins with Cap）、F-17（Deduct Coins）、F-18（Coin Persistence）
  - API定義: API-06（CoinManager.AddCoins()）

* **VRChat Persistence API**:
  - PlayerData.SetInt() / GetInt() の使用方法
  - データ永続化のベストプラクティス

### 開発者向けメモ

* **デバッグ方法**:
  - Unity Play Modeで実行中、Hierarchyの `GameManager` → `CoinManager` コンポーネントのInspectorで `currentCoins` の値をリアルタイム監視可能
  - Consoleビューで `[CoinManager]` でフィルタリングして、コイン関連ログのみ表示できます

* **将来の拡張性**:
  - Phase 2でキャンプ木材ポイント（拠点通貨）を追加する場合、CoinManagerを `CurrencyManager` にリファクタリングし、複数通貨対応に拡張することを推奨
  - 現在の設計（単一責任原則）により、拡張時の影響範囲は限定的です
