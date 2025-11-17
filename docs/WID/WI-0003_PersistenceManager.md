# 作業指示書: WI-0003 - Implement PersistenceManager with Retry Logic

**作成日**: 2025-11-17
**Phase**: Phase 0（基盤実装・Week 1）
**Priority**: 最高
**推定工数**: 2日

---

## 1. 作業ID

**WI-0003**

---

## 2. 作業名

PersistenceManager実装 - PlayerData Load/Save抽象化とリトライロジック

---

## 3. 作業目的

VRChat PlayerData APIの抽象化レイヤーを提供し、以下を実現する：

- PlayerDataの読み込み/保存処理を一元管理
- 読み込み失敗時の自動リトライ機能（最大3回、1秒遅延）
- データサイズの監視（70KB制限の遵守）
- データバージョン管理によるマイグレーション対応
- OnPlayerRestoredイベントとの正しい統合

これにより、スキルレベル・コイン・購入済みアイテムなどのプレイヤーデータが確実に永続化され、ワールド再訪問時に復元される。

---

## 4. 対象

### 対象システム
- 森のきこりキャンプ - VRChatワールド

### 対象モジュール
- **M-03: PersistenceManager** (Core Layer)

### 担当機能
- **F-07**: PlayerData Load with Retry
- **F-08**: PlayerData Save with Size Check
- **F-09**: Data Versioning & Migration

### 対象ファイルパス
- `/Assets/UdonSharp/Scripts/Core/PersistenceManager.cs` (新規作成)
- `/Assets/UdonSharp/Scripts/Core/GameManager.cs` (参照呼び出し)

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0以上
- UdonSharp 1.1.9 (stable)

### 依存作業
- **WI-0001**: GameManager Singleton実装が完了していること
  - GameManager.Instanceでアクセス可能
  - 初期化順序が制御されていること

### ブランチ
- `feature/phase0-core-implementation`

### 初期状態
- PlayerData APIが利用可能なVRChat環境
- GameManagerが正常に初期化されている

---

## 6. 入力

### 保存処理の入力（SetInt/SetFloat/SetString）

```csharp
// 例1: スキルレベル保存
PersistenceManager.Instance.SetInt("Woodcutting_Level", 5);

// 例2: コイン残高保存
PersistenceManager.Instance.SetInt("WoodcutterCoins", 245);

// 例3: 文字列保存
PersistenceManager.Instance.SetString("LastVisit", "2025-11-17T10:00:00Z");
```

### 読み込み処理の入力（GetInt/GetFloat/GetString）

```csharp
// 例1: スキルレベル読み込み（デフォルト値1）
int level = PersistenceManager.Instance.GetInt("Woodcutting_Level", 1);

// 例2: コイン残高読み込み（デフォルト値0）
int coins = PersistenceManager.Instance.GetInt("WoodcutterCoins", 0);
```

---

## 7. 出力

### 保存成功時
```
[PersistenceManager] Saved: Woodcutting_Level = 5
[PersistenceManager] Current data size: 58 bytes / 70000 bytes (0.08%)
```

### 読み込み成功時
```
[PersistenceManager] Loaded: Woodcutting_Level = 5
[PersistenceManager] Loaded: WoodcutterCoins = 245
```

### リトライ発生時
```
[PersistenceManager] Load failed, retrying in 1 second... (Attempt 1/3)
[PersistenceManager] Load succeeded on retry (Attempt 2/3)
```

### データサイズ超過警告
```
[PersistenceManager] WARNING: Data size approaching limit: 65000 / 70000 bytes (92.8%)
```

### データバージョン不一致時
```
[PersistenceManager] Data version mismatch: stored=1.0.0, current=1.1.0
[PersistenceManager] Running migration from 1.0.0 to 1.1.0...
[PersistenceManager] Migration completed successfully
```

---

## 8. 作業手順

### Step 1: PersistenceManagerクラスの基本構造作成

**目的**: Singletonパターンで一元的にPlayerDataにアクセスできるようにする

**手順**:
1. Unityで新規UdonSharpスクリプトを作成
   - メニュー: `Assets > Create > UdonSharp > UdonSharpScript`
   - ファイル名: `PersistenceManager.cs`
   - 配置場所: `/Assets/UdonSharp/Scripts/Core/`

2. 以下の基本構造を実装:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

public class PersistenceManager : UdonSharpBehaviour
{
    // Singleton Instance
    public static PersistenceManager Instance { get; private set; }

    // データバージョン管理
    [Header("Data Version")]
    public string currentDataVersion = "1.0.0";
    private const string VERSION_KEY = "DataVersion";

    // データサイズ制限
    [Header("Size Limits")]
    public int maxDataSizeBytes = 70000; // 70KB（100KBの70%）
    private int currentDataSize = 0;

    // リトライ設定
    [Header("Retry Settings")]
    public int maxRetryAttempts = 3;
    public float retryDelaySeconds = 1.0f;

    // 初期化フラグ
    private bool isInitialized = false;
    private bool isDataRestored = false;

    void Start()
    {
        // Singleton確立
        if (Instance == null)
        {
            Instance = this;
            Debug.Log("[PersistenceManager] Instance created");
        }
        else
        {
            Debug.LogError("[PersistenceManager] Duplicate instance detected! Destroying...");
            Destroy(gameObject);
            return;
        }
    }
}
```

3. Unityシーンに配置:
   - Hierarchy: `GameManager` の子オブジェクトとして `PersistenceManager` GameObject作成
   - PersistenceManagerスクリプトをアタッチ

**完了条件**:
- PersistenceManager.Instanceが他スクリプトからアクセス可能
- 重複インスタンスが検出されてDestroyされる

---

### Step 2: OnPlayerRestoredイベント実装

**目的**: VRChat PlayerData APIが安全にアクセス可能になるタイミングで初期化を行う

**重要**: PlayerDataは `Start()` では利用不可、`OnPlayerRestored()` で初めて安全にアクセス可能

**手順**:
1. 以下のコードを追加:

```csharp
public override void OnPlayerRestored(VRCPlayerApi player)
{
    if (!player.isLocal)
        return; // ローカルプレイヤーのみ処理

    Debug.Log("[PersistenceManager] OnPlayerRestored called - PlayerData is now safe to access");

    // データバージョンチェック
    CheckDataVersion();

    // データサイズ推定（初回読み込み時）
    EstimateCurrentDataSize();

    isDataRestored = true;
    Debug.Log("[PersistenceManager] Initialization complete");
}

private void CheckDataVersion()
{
    string storedVersion = PlayerData.GetString(VERSION_KEY);

    if (string.IsNullOrEmpty(storedVersion))
    {
        // 初回訪問
        Debug.Log("[PersistenceManager] First visit detected, setting data version to " + currentDataVersion);
        PlayerData.SetString(VERSION_KEY, currentDataVersion);
    }
    else if (storedVersion != currentDataVersion)
    {
        // バージョン不一致（マイグレーション必要）
        Debug.LogWarning($"[PersistenceManager] Data version mismatch: stored={storedVersion}, current={currentDataVersion}");
        MigrateData(storedVersion, currentDataVersion);
        PlayerData.SetString(VERSION_KEY, currentDataVersion);
    }
    else
    {
        Debug.Log($"[PersistenceManager] Data version OK: {currentDataVersion}");
    }
}

private void EstimateCurrentDataSize()
{
    // 概算計算（実際のバイト数はVRChat内部でのみ取得可能）
    // SDK 3.9.1-beta以降: PlayerData.GetStorageUsed() が利用可能
    // ここでは保存キー数から推定
    currentDataSize = 0;

    // 将来的にキー一覧を保持してサイズ計算
    Debug.Log($"[PersistenceManager] Estimated data size: {currentDataSize} / {maxDataSizeBytes} bytes");
}
```

**参照ドキュメント**: `VRChat_TechResearch_2025.md` Section 4.3, `TECHNICAL_DETAILS.md` Section A1

**完了条件**:
- OnPlayerRestoredが呼ばれるまでPlayerDataアクセスを待機
- データバージョンが正しくチェックされる
- ログに「Initialization complete」が表示される

---

### Step 3: Load処理（リトライロジック実装）

**目的**: データ読み込み失敗時に自動で最大3回リトライし、信頼性を向上させる

**手順**:
1. 以下のメソッドを実装:

```csharp
// Int型読み込み（リトライ付き）
public int GetInt(string key, int defaultValue = 0)
{
    if (!isDataRestored)
    {
        Debug.LogWarning("[PersistenceManager] PlayerData not yet restored! Returning default value.");
        return defaultValue;
    }

    return GetIntWithRetry(key, defaultValue, 0);
}

private int GetIntWithRetry(string key, int defaultValue, int attemptCount)
{
    try
    {
        int value = PlayerData.GetInt(key);
        Debug.Log($"[PersistenceManager] Loaded: {key} = {value}");
        return value;
    }
    catch (System.Exception e)
    {
        if (attemptCount < maxRetryAttempts)
        {
            Debug.LogWarning($"[PersistenceManager] Load failed for '{key}', retrying in {retryDelaySeconds}s... (Attempt {attemptCount + 1}/{maxRetryAttempts})");
            Debug.LogWarning($"[PersistenceManager] Error: {e.Message}");

            // Udonでは直接のRetry待機ができないため、SendCustomEventDelayedSecondsを使用
            // ここでは簡易的に即座に再試行（実運用では遅延Eventを使用）
            return GetIntWithRetry(key, defaultValue, attemptCount + 1);
        }
        else
        {
            Debug.LogError($"[PersistenceManager] Load failed for '{key}' after {maxRetryAttempts} attempts. Returning default: {defaultValue}");
            Debug.LogError($"[PersistenceManager] Final error: {e.Message}");
            return defaultValue;
        }
    }
}

// Float型読み込み
public float GetFloat(string key, float defaultValue = 0f)
{
    if (!isDataRestored)
    {
        Debug.LogWarning("[PersistenceManager] PlayerData not yet restored! Returning default value.");
        return defaultValue;
    }

    try
    {
        float value = PlayerData.GetFloat(key);
        Debug.Log($"[PersistenceManager] Loaded: {key} = {value}");
        return value;
    }
    catch (System.Exception e)
    {
        Debug.LogError($"[PersistenceManager] Load failed for '{key}': {e.Message}. Returning default: {defaultValue}");
        return defaultValue;
    }
}

// String型読み込み
public string GetString(string key, string defaultValue = "")
{
    if (!isDataRestored)
    {
        Debug.LogWarning("[PersistenceManager] PlayerData not yet restored! Returning default value.");
        return defaultValue;
    }

    try
    {
        string value = PlayerData.GetString(key);
        Debug.Log($"[PersistenceManager] Loaded: {key} = {value}");
        return value;
    }
    catch (System.Exception e)
    {
        Debug.LogError($"[PersistenceManager] Load failed for '{key}': {e.Message}. Returning default: {defaultValue}");
        return defaultValue;
    }
}
```

**注意事項**:
- UdonSharpではThread.Sleep()が使えないため、リトライ遅延は `SendCustomEventDelayedSeconds()` を別途実装する必要がある（簡易版では即座にリトライ）
- PlayerData読み込みエラーは稀だが、Late Joiner時のネットワーク遅延で発生する可能性がある

**完了条件**:
- GetInt/GetFloat/GetStringが正常に動作
- エラー時に最大3回リトライされる
- デフォルト値が正しく返される

---

### Step 4: Save処理（サイズチェック実装）

**目的**: データ保存時にサイズ制限（70KB）を監視し、超過前に警告を出す

**手順**:
1. 以下のメソッドを実装:

```csharp
// Int型保存
public bool SetInt(string key, int value)
{
    if (!isDataRestored)
    {
        Debug.LogError("[PersistenceManager] Cannot save before PlayerData is restored!");
        return false;
    }

    // サイズチェック
    int estimatedSize = 4; // int = 4 bytes
    if (!CheckDataSizeLimit(key, estimatedSize))
    {
        return false;
    }

    try
    {
        PlayerData.SetInt(key, value);
        Debug.Log($"[PersistenceManager] Saved: {key} = {value}");
        UpdateDataSize(key, estimatedSize);
        return true;
    }
    catch (System.Exception e)
    {
        Debug.LogError($"[PersistenceManager] Save failed for '{key}': {e.Message}");
        return false;
    }
}

// Float型保存
public bool SetFloat(string key, float value)
{
    if (!isDataRestored)
    {
        Debug.LogError("[PersistenceManager] Cannot save before PlayerData is restored!");
        return false;
    }

    int estimatedSize = 4; // float = 4 bytes
    if (!CheckDataSizeLimit(key, estimatedSize))
    {
        return false;
    }

    try
    {
        PlayerData.SetFloat(key, value);
        Debug.Log($"[PersistenceManager] Saved: {key} = {value}");
        UpdateDataSize(key, estimatedSize);
        return true;
    }
    catch (System.Exception e)
    {
        Debug.LogError($"[PersistenceManager] Save failed for '{key}': {e.Message}");
        return false;
    }
}

// String型保存
public bool SetString(string key, string value)
{
    if (!isDataRestored)
    {
        Debug.LogError("[PersistenceManager] Cannot save before PlayerData is restored!");
        return false;
    }

    // 文字列サイズ = 文字数 × 2 bytes (UTF-16想定) + キー名サイズ
    int estimatedSize = (value.Length * 2) + (key.Length * 2);
    if (!CheckDataSizeLimit(key, estimatedSize))
    {
        return false;
    }

    try
    {
        PlayerData.SetString(key, value);
        Debug.Log($"[PersistenceManager] Saved: {key} = {value}");
        UpdateDataSize(key, estimatedSize);
        return true;
    }
    catch (System.Exception e)
    {
        Debug.LogError($"[PersistenceManager] Save failed for '{key}': {e.Message}");
        return false;
    }
}

private bool CheckDataSizeLimit(string key, int additionalSize)
{
    int projectedSize = currentDataSize + additionalSize;

    if (projectedSize > maxDataSizeBytes)
    {
        Debug.LogError($"[PersistenceManager] Data size limit exceeded! Current: {currentDataSize}, Adding: {additionalSize}, Limit: {maxDataSizeBytes}");
        return false;
    }

    float usagePercent = (float)projectedSize / maxDataSizeBytes * 100f;

    if (usagePercent >= 90f)
    {
        Debug.LogError($"[PersistenceManager] CRITICAL: Data size at {usagePercent:F1}% ({projectedSize} / {maxDataSizeBytes} bytes)!");
    }
    else if (usagePercent >= 70f)
    {
        Debug.LogWarning($"[PersistenceManager] WARNING: Data size approaching limit: {usagePercent:F1}% ({projectedSize} / {maxDataSizeBytes} bytes)");
    }

    return true;
}

private void UpdateDataSize(string key, int size)
{
    // 簡易的な累計サイズ管理（実際はキーごとの更新を追跡する必要あり）
    currentDataSize += size;

    float usagePercent = (float)currentDataSize / maxDataSizeBytes * 100f;
    Debug.Log($"[PersistenceManager] Current data size: {currentDataSize} bytes / {maxDataSizeBytes} bytes ({usagePercent:F1}%)");
}
```

**参照制約**: `VRChat_TechResearch_2025.md` Section 4.2 (100KB制限), `TDD.md` Section 5.3 (70KB目標)

**完了条件**:
- SetInt/SetFloat/SetStringが正常に動作
- データサイズが70KBを超える前に警告が表示される
- 90%以上でCRITICAL警告が表示される

---

### Step 5: データバージョン管理とマイグレーション

**目的**: データ構造変更時に古いデータを新しい形式に自動変換する

**手順**:
1. 以下のマイグレーションロジックを実装:

```csharp
private void MigrateData(string fromVersion, string toVersion)
{
    Debug.Log($"[PersistenceManager] Running migration from {fromVersion} to {toVersion}...");

    // バージョンごとのマイグレーション処理
    if (fromVersion == "1.0.0" && toVersion == "1.1.0")
    {
        Migrate_1_0_to_1_1();
    }
    else if (fromVersion == "1.1.0" && toVersion == "1.2.0")
    {
        Migrate_1_1_to_1_2();
    }
    else
    {
        Debug.LogWarning($"[PersistenceManager] No migration path defined from {fromVersion} to {toVersion}");
    }

    Debug.Log("[PersistenceManager] Migration completed successfully");
}

// 例: 1.0.0 → 1.1.0 マイグレーション
private void Migrate_1_0_to_1_1()
{
    Debug.Log("[PersistenceManager] Migrating data from 1.0.0 to 1.1.0...");

    // 例: 古いキー "WoodcuttingLevel" を "Woodcutting_Level" にリネーム
    int oldLevel = PlayerData.GetInt("WoodcuttingLevel", 0);
    if (oldLevel > 0)
    {
        PlayerData.SetInt("Woodcutting_Level", oldLevel);
        Debug.Log($"[PersistenceManager] Migrated WoodcuttingLevel ({oldLevel}) to Woodcutting_Level");
    }

    // 例: 新しいフィールド "TotalTreesCut" の初期化
    if (PlayerData.GetInt("TotalTreesCut", -1) == -1)
    {
        PlayerData.SetInt("TotalTreesCut", 0);
        Debug.Log("[PersistenceManager] Initialized TotalTreesCut to 0");
    }
}

// 例: 1.1.0 → 1.2.0 マイグレーション（将来用）
private void Migrate_1_1_to_1_2()
{
    Debug.Log("[PersistenceManager] Migrating data from 1.1.0 to 1.2.0...");

    // 将来の変更に備えた空実装
}
```

**完了条件**:
- バージョン不一致時にマイグレーションが実行される
- 古いキーが新しいキーに変換される
- 新しいフィールドが初期化される

---

### Step 6: GameManagerへの統合

**目的**: GameManagerから PersistenceManager を参照できるようにする

**手順**:
1. `GameManager.cs` を開く

2. 以下のコードを追加:

```csharp
public class GameManager : UdonSharpBehaviour
{
    // 既存コード...

    [Header("Core Managers")]
    public PersistenceManager persistenceManager;

    void Start()
    {
        // 初期化順序確認
        if (persistenceManager == null)
        {
            persistenceManager = PersistenceManager.Instance;
        }

        if (persistenceManager == null)
        {
            Debug.LogError("[GameManager] PersistenceManager not found!");
        }
        else
        {
            Debug.Log("[GameManager] PersistenceManager connected");
        }
    }

    // Getter for other modules
    public PersistenceManager GetPersistenceManager()
    {
        return persistenceManager;
    }
}
```

3. Unityエディタで:
   - GameManagerオブジェクトを選択
   - Inspector > GameManager > Persistence Manager フィールドに PersistenceManagerオブジェクトをドラッグ&ドロップ

**完了条件**:
- GameManager.GetPersistenceManager()で参照を取得可能
- Consoleに「PersistenceManager connected」と表示される

---

## 9. 完了条件（Doneの条件）

以下のすべてを満たすこと：

- [ ] PersistenceManagerクラスが実装され、Singletonとして動作する
- [ ] OnPlayerRestoredイベントが正しく実装され、PlayerData読み込みが安全に行われる
- [ ] GetInt/GetFloat/GetStringメソッドがリトライロジック付きで動作する
- [ ] SetInt/SetFloat/SetStringメソッドがデータサイズチェック付きで動作する
- [ ] データサイズが70KB制限の70%（49KB）を超えたら警告が表示される
- [ ] データバージョン管理が実装され、マイグレーションが動作する
- [ ] GameManagerから PersistenceManager が参照可能
- [ ] Unity Consoleにエラーが表示されない（Warning以下は許容）
- [ ] VRChat ClientSimでテストが成功する

---

## 10. テスト観点／テストケース

### テスト種別
- **ユニットテスト**: Unity EditorでのPlay Mode Test
- **統合テスト**: VRChat ClientSimでの動作確認

### テストケース

#### 正常系テスト

**TC-001: 初回訪問時のデータ初期化**
- 手順:
  1. PlayerDataが空の状態でワールド入場
  2. OnPlayerRestoredが呼ばれるまで待機
  3. GetInt("Woodcutting_Level", 1)を実行
- 期待結果: デフォルト値 `1` が返される
- 確認方法: Consoleに `Loaded: Woodcutting_Level = 1` が表示される

**TC-002: データ保存と再読み込み**
- 手順:
  1. SetInt("Woodcutting_Level", 5)を実行
  2. ワールドを退出
  3. 再度ワールドに入場
  4. GetInt("Woodcutting_Level", 1)を実行
- 期待結果: 保存した値 `5` が読み込まれる
- 確認方法: Consoleに `Loaded: Woodcutting_Level = 5` が表示される

**TC-003: データサイズ警告**
- 手順:
  1. 大量のデータを保存（約60KB相当）
  2. Consoleを確認
- 期待結果: `WARNING: Data size approaching limit` が表示される
- 確認方法: 警告メッセージに使用率が表示される

#### 異常系テスト

**TC-004: OnPlayerRestored前のアクセス試行**
- 手順:
  1. Start()内で GetInt("test", 0)を実行（OnPlayerRestored前）
  2. Consoleを確認
- 期待結果: `PlayerData not yet restored! Returning default value.` 警告が表示され、デフォルト値が返される

**TC-005: データサイズ超過時の保存拒否**
- 手順:
  1. currentDataSizeを手動で69,000に設定
  2. SetInt("test", 100)を実行（4bytes追加）
- 期待結果: `Data size limit exceeded!` エラーが表示され、保存が失敗する（戻り値 false）

**TC-006: バージョン不一致時のマイグレーション**
- 手順:
  1. PlayerDataに古いバージョン "1.0.0" を設定
  2. currentDataVersion = "1.1.0" でワールド入場
  3. Consoleを確認
- 期待結果: `Data version mismatch` 警告と `Migration completed successfully` が表示される

---

## 11. 成果物

### 変更ファイル
- `/Assets/UdonSharp/Scripts/Core/PersistenceManager.cs` (新規作成)
- `/Assets/UdonSharp/Scripts/Core/GameManager.cs` (参照追加)

### 追加テストコード
- `/Assets/UdonSharp/Tests/PersistenceManagerTests.cs` (オプション - ClientSimで手動テスト)

### 更新したドキュメント
- なし（本作業指示書が完了後のドキュメントとなる）

---

## 12. 備考

### 注意点

**VRChat PlayerData APIの制約**
- **容量制限**: 100KB/Player/World（本プロジェクトは70KB目標）
- **アクセスタイミング**: OnPlayerRestored以降のみ安全にアクセス可能
- **データ型**: int, long, float, double, Vector3, Quaternion, Color, string, byte[]のみ対応
- **Late Joiner**: 後から参加したプレイヤーも OnPlayerRestored で復元される

**UdonSharp 1.1.9の制約**
- try-catchは使えるが、async/awaitは不可
- Thread.Sleep()不可（SendCustomEventDelayedSecondsで代用）
- List/Dictionaryは非対応（1.2.0-betaで対応予定）

**パフォーマンス考慮**
- PlayerData.SetXX()は即座に保存されるため、頻繁な呼び出しは避ける
- SkillManager/CoinManagerなどで「変更があった時のみ」保存する設計とする
- 推奨保存タイミング: レベルアップ時、コイン100枚獲得ごと、ワールド退出時

**デバッグ方法**
- Unity EditorのPlay Modeでは PlayerData は動作しない
- VRChat ClientSim（SDK 3.9.0+）を使用してテストすること
- 本番テストはVRChatクライアントでの動作確認が必須

### セキュリティ
- PlayerDataは暗号化されて保存される（VRChat側で自動処理）
- クライアント側でのデータ改ざんは検出されない可能性があるため、サーバ側（Master Client）での検証が重要
- 異常値チェック（例: スキルレベルが10を超える、コインが99,999を超える）は各Managerで実装すること

### 将来の拡張
- SDK 3.9.1-beta以降: `PlayerData.GetStorageUsed()` でより正確なサイズ取得が可能
- UdonSharp 1.2.0: List/Dictionary対応により、複雑なデータ構造の保存が容易に
- Soba言語（2025年中予定）: より高速なコンパイルとC#互換性向上

### 参考ドキュメント
- `TDD.md` Section 5.3（Module M-03定義）
- `VRChat_TechResearch_2025.md` Section 4（PlayerData仕様）
- `TECHNICAL_DETAILS.md` Section A（Persistence APIリファレンス）
- 公式: https://creators.vrchat.com/worlds/udon/persistence/player-data/

---

**作業開始前チェックリスト**:
- [ ] WI-0001（GameManager）が完了している
- [ ] Unity 2022.3.22f1 LTS環境が準備されている
- [ ] VRChat SDK 3.9.0以上がインストールされている
- [ ] UdonSharp 1.1.9がインストールされている
- [ ] VRChat ClientSimがセットアップされている

**作業完了後チェックリスト**:
- [ ] すべてのテストケースが成功している
- [ ] Unity Consoleにエラーがない（Warning以下は許容）
- [ ] ClientSimでの動作確認が完了している
- [ ] GameManagerとの統合が確認されている
- [ ] 本作業指示書の「完了条件」がすべて満たされている

---

**承認**:
- [ ] 技術レビュー承認
- [ ] コードレビュー承認
- [ ] 動作テスト承認
- [ ] Phase 0実装への統合承認
