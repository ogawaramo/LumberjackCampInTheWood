# 作業指示書

**1. 作業ID**

WI-0014

**2. 作業名**

Implement Login Bonus System（ログインボーナスシステムの実装）

**3. 作業目的**

新規プレイヤーの離脱防止と継続プレイヤーのモチベーション向上を目的として、初回ログインボーナス（50コイン）とデイリーボーナス（20コイン）を実装する。これにより、ユーザーは初回訪問時にショップ体験が可能となり、継続的なログインへの動機付けが強化される。

**4. 対象**

* 対象システム：森のきこりキャンプ（Woodcutter Camp）
* 対象モジュール/パッケージ：M-06 CoinManager
* 対象ファイルパス：`Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`

**5. 前提条件**

* WI-0013（CoinManager Core Logic）が完了していること
* PersistenceManager（M-03）が実装済みでPlayerDataの読み書きが可能であること
* Unity 2022.3.22f1、VRChat SDK 3.9.0、UdonSharp 1.1.9の環境が整っていること
* 初期化は GameManager によって制御され、Late Joiner対応のため1秒遅延されること

**6. 入力**

* **初回訪問判定**:
  - PlayerData["FirstVisit"] が存在しない（null または空）
  - プレイヤーがワールドに初めて入場した状態

* **デイリーボーナス判定**:
  - PlayerData["LastVisit"] が存在する
  - 現在時刻と LastVisit の差が 24時間（86400秒）以上

**7. 出力**

* **初回ログインボーナス付与時**:
  - 50コイン付与
  - PlayerData["FirstVisit"] に現在のUnix時刻（UTC）を記録
  - PlayerData["LastVisit"] に現在のUnix時刻（UTC）を記録
  - 画面中央に「ようこそ！初回ログインボーナス 50コイン」を5秒間表示
  - 金色のパーティクルエフェクト再生

* **デイリーボーナス付与時**:
  - 20コイン付与
  - PlayerData["LastVisit"] を現在のUnix時刻（UTC）で更新
  - 画面中央に「デイリーボーナス 20コイン」を3秒間表示
  - 金色のパーティクルエフェクト再生

* **ボーナス対象外時**:
  - PlayerData["LastVisit"] を現在時刻で更新のみ
  - 通知表示なし

**8. 作業手順**

## 8.1 PlayerDataキーの追加定義

1. `CoinManager.cs` の定数セクションに以下を追加:
```csharp
// PlayerData Keys
private const string KEY_FIRST_VISIT = "FirstVisit";
private const string KEY_LAST_VISIT = "LastVisit";

// ログインボーナス定数
private const int FIRST_LOGIN_BONUS = 50;
private const int DAILY_BONUS = 20;
private const long DAILY_INTERVAL_SECONDS = 86400; // 24時間
```

## 8.2 CheckLoginBonus メソッドの実装

2. 以下のメソッドを `CoinManager.cs` に追加:

```csharp
/// <summary>
/// ログインボーナスをチェックし、該当する場合はコインを付与する
/// GameManager の初期化シーケンスから呼び出される
/// </summary>
public void _CheckLoginBonus()
{
    if (persistenceManager == null)
    {
        Debug.LogError("[CoinManager] PersistenceManager reference is null. Cannot check login bonus.");
        return;
    }

    long currentTime = GetCurrentUnixTime();

    // 初回訪問チェック
    long firstVisit = persistenceManager.GetLong(KEY_FIRST_VISIT, -1);

    if (firstVisit == -1)
    {
        // 初回ログインボーナス
        GrantFirstLoginBonus(currentTime);
        return;
    }

    // デイリーボーナスチェック
    long lastVisit = persistenceManager.GetLong(KEY_LAST_VISIT, -1);

    if (lastVisit == -1)
    {
        // LastVisit が未設定の場合（データ移行対応）
        persistenceManager.SetLong(KEY_LAST_VISIT, currentTime);
        return;
    }

    long timeDifference = currentTime - lastVisit;

    if (timeDifference >= DAILY_INTERVAL_SECONDS)
    {
        // デイリーボーナス
        GrantDailyBonus(currentTime);
    }
    else
    {
        // ボーナスなし、LastVisit のみ更新
        persistenceManager.SetLong(KEY_LAST_VISIT, currentTime);
    }
}
```

## 8.3 Unix時刻取得メソッドの実装

3. UTC時刻をUnix時刻（Epoch）に変換するメソッドを追加:

```csharp
/// <summary>
/// 現在のUTC時刻をUnix時刻（秒）で取得
/// UdonSharpではDateTimeを直接使用できないため、FileTimeUtcを経由
/// </summary>
private long GetCurrentUnixTime()
{
    // FileTimeUtc（100ナノ秒単位）を取得
    long fileTime = System.DateTime.UtcNow.ToFileTimeUtc();

    // Unix Epoch (1970-01-01 00:00:00 UTC) のFileTimeUtc
    const long EPOCH_FILE_TIME = 116444736000000000L;

    // 秒に変換
    long unixTime = (fileTime - EPOCH_FILE_TIME) / 10000000L;

    return unixTime;
}
```

**重要**: UdonSharpでは `DateTime` の一部機能が制限されているため、`ToFileTimeUtc()` を使用してUnix時刻を計算します。

## 8.4 初回ログインボーナス付与メソッド

4. 初回ログインボーナスの処理を実装:

```csharp
/// <summary>
/// 初回ログインボーナスを付与
/// </summary>
private void GrantFirstLoginBonus(long currentTime)
{
    // コイン付与
    _AddCoins(FIRST_LOGIN_BONUS);

    // 訪問時刻の記録
    persistenceManager.SetLong(KEY_FIRST_VISIT, currentTime);
    persistenceManager.SetLong(KEY_LAST_VISIT, currentTime);

    // 通知表示（5秒間）
    if (hudManager != null)
    {
        hudManager._ShowNotification("ようこそ！初回ログインボーナス 50コイン", 5.0f);
    }

    // パーティクルエフェクト
    PlayGoldenParticles();

    Debug.Log("[CoinManager] First login bonus granted: 50 coins");
}
```

## 8.5 デイリーボーナス付与メソッド

5. デイリーボーナスの処理を実装:

```csharp
/// <summary>
/// デイリーボーナスを付与
/// </summary>
private void GrantDailyBonus(long currentTime)
{
    // コイン付与
    _AddCoins(DAILY_BONUS);

    // LastVisit更新
    persistenceManager.SetLong(KEY_LAST_VISIT, currentTime);

    // 通知表示（3秒間）
    if (hudManager != null)
    {
        hudManager._ShowNotification("デイリーボーナス 20コイン", 3.0f);
    }

    // パーティクルエフェクト
    PlayGoldenParticles();

    Debug.Log("[CoinManager] Daily bonus granted: 20 coins");
}
```

## 8.6 パーティクルエフェクトの実装

6. 金色のパーティクルエフェクトを再生するメソッドを追加:

```csharp
[SerializeField] private ParticleSystem goldenParticles; // Inspectorで設定

/// <summary>
/// 金色のパーティクルエフェクトを再生
/// </summary>
private void PlayGoldenParticles()
{
    if (goldenParticles != null)
    {
        goldenParticles.Play();
    }
    else
    {
        Debug.LogWarning("[CoinManager] Golden particles reference is not set in Inspector.");
    }
}
```

## 8.7 PersistenceManager の拡張

7. `PersistenceManager.cs` に `long` 型の読み書きメソッドが存在しない場合、以下を追加:

```csharp
/// <summary>
/// Long値をPlayerDataに保存（上位32bit/下位32bitに分割）
/// </summary>
public void SetLong(string key, long value)
{
    int high = (int)(value >> 32);
    int low = (int)(value & 0xFFFFFFFF);

    PlayerData.SetInt(key + "_High", high);
    PlayerData.SetInt(key + "_Low", low);
}

/// <summary>
/// Long値をPlayerDataから読み込み
/// </summary>
public long GetLong(string key, long defaultValue)
{
    if (!PlayerData.HasKey(key + "_High"))
    {
        return defaultValue;
    }

    int high = PlayerData.GetInt(key + "_High");
    int low = PlayerData.GetInt(key + "_Low");

    long value = ((long)high << 32) | ((long)low & 0xFFFFFFFFL);

    return value;
}
```

**技術ポイント**: VRChat PlayerData は `int` 型のみサポートするため、`long` 型を上位32bit/下位32bitに分割して保存します。

## 8.8 Unity シーン設定

8. Unity Editor での設定手順:

   a. **パーティクルシステムの作成**
      - Hierarchy で `CoinManager` GameObject を選択
      - 子オブジェクトとして `Particle System` を追加
      - 名前を `GoldenParticles` に変更
      - Inspector で以下を設定:
        * Start Color: 金色（R: 255, G: 215, B: 0, A: 255）
        * Start Lifetime: 2秒
        * Start Speed: 2
        * Start Size: 0.2
        * Emission Rate: 50
        * Shape: Sphere, Radius: 1.5
        * Play On Awake: OFF（重要）
        * Looping: OFF

   b. **CoinManager Inspectorの設定**
      - `Golden Particles` フィールドに作成した `GoldenParticles` を割り当て

## 8.9 GameManager からの呼び出し

9. `GameManager.cs` の初期化シーケンスに追加:

```csharp
void Start()
{
    SendCustomEventDelayedSeconds(nameof(_InitializeAllModules), 1.0f);
}

public void _InitializeAllModules()
{
    // ... 他のモジュール初期化 ...

    // CoinManager初期化後にログインボーナスチェック
    if (coinManager != null)
    {
        coinManager._CheckLoginBonus();
    }

    Debug.Log("[GameManager] All modules initialized.");
}
```

## 8.10 テストケースの実装

10. 以下のテストケースを手動で実施:

**テストケース1: 初回ログイン**
- 手順:
  1. PlayerData を完全に削除（VRChat Settings > Saved Data > Clear All）
  2. ワールドに入場
- 期待結果:
  - 50コイン付与
  - 「ようこそ！初回ログインボーナス 50コイン」が5秒間表示
  - 金色のパーティクル再生
  - PlayerData["FirstVisit"] と PlayerData["LastVisit"] が記録される

**テストケース2: 24時間以内の再ログイン**
- 手順:
  1. 初回ログイン後、ワールドを退出
  2. 即座に再入場
- 期待結果:
  - ボーナスなし
  - 通知表示なし
  - PlayerData["LastVisit"] のみ更新

**テストケース3: 24時間後のデイリーボーナス**
- 手順:
  1. PlayerData["LastVisit"] を手動で24時間前の値に変更
     （現在時刻 - 86400秒）
  2. ワールドに再入場
- 期待結果:
  - 20コイン付与
  - 「デイリーボーナス 20コイン」が3秒間表示
  - 金色のパーティクル再生
  - PlayerData["LastVisit"] が現在時刻で更新

**テストケース4: データ移行（LastVisit未設定）**
- 手順:
  1. PlayerData["FirstVisit"] は存在するが、PlayerData["LastVisit"] が存在しない状態を作る
  2. ワールドに入場
- 期待結果:
  - ボーナスなし
  - PlayerData["LastVisit"] が現在時刻で初期化される

**テストケース5: 連続ログイン（3日連続）**
- 手順:
  1. 1日目：初回ログイン（50コイン）
  2. 2日目：24時間後にログイン（20コイン）
  3. 3日目：さらに24時間後にログイン（20コイン）
- 期待結果:
  - 各日で正しいボーナスが付与される
  - 累計コイン: 50 + 20 + 20 = 90コイン

**9. 完了条件（Doneの条件）**

* `CoinManager.cs` に `_CheckLoginBonus()` メソッドが実装されている
* 初回ログインボーナス（50コイン）が正しく付与される
* デイリーボーナス（20コイン）が24時間ぶりのログイン時に付与される
* 24時間以内の再ログイン時にボーナスが付与されない
* Unix時刻（UTC）でPlayerDataに日時が正しく記録される
* 通知メッセージが正しい表示時間で表示される（初回5秒、デイリー3秒）
* 金色のパーティクルエフェクトが再生される
* PersistenceManager の `GetLong()` / `SetLong()` メソッドが実装されている
* Unity Inspector でパーティクルシステムが正しく設定されている
* 上記のテストケース1〜5がすべてパスする
* エラーログが出力されない（Null参照、型変換エラーなど）

**10. テスト観点/テストケース**

* **テスト種別**: ユニットテスト（手動）、統合テスト
* **テストケース**:
  * **正常系**:
    - 初回訪問時に50コイン付与
    - 24時間後の訪問時に20コイン付与
    - パーティクルエフェクトの再生確認
    - 通知メッセージの表示時間確認
    - PlayerDataへの日時記録確認

  * **異常系**:
    - PersistenceManager が null の場合のエラーハンドリング
    - PlayerData読み込み失敗時のフォールバック（デフォルト値-1）
    - 24時間以内の再ログイン時にボーナスが付与されないこと
    - Unix時刻計算のオーバーフロー対策（2038年問題は long 型で回避）
    - HUDManager が null の場合でもエラーにならないこと
    - パーティクルシステムが未設定でもエラーにならないこと

  * **境界値テスト**:
    - 86400秒ちょうどでデイリーボーナス付与
    - 86399秒ではボーナス付与なし
    - 複数回の連続デイリーボーナス（30日間など）

**11. 成果物**

* **変更ファイル**:
  - `Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`
  - `Assets/WoodcutterCamp/Scripts/Core/PersistenceManager.cs`
  - `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`

* **追加Prefab/Scene設定**:
  - `CoinManager` GameObject に `GoldenParticles` パーティクルシステムを追加

* **追加テストコード**:
  - 手動テストケーススクリプト（テスト用のデータ設定補助スクリプト）

* **更新したドキュメント**:
  - 本作業指示書（WI-0014）

**12. 備考**

### 技術的注意点

* **DateTime制約**: UdonSharpでは `DateTime` の一部機能が制限されているため、`ToFileTimeUtc()` を使用してUnix時刻を計算する必要があります。
* **Null/未設定値の判定**: PlayerData のキーが存在しない場合、`GetLong()` はデフォルト値 `-1` を返します。この値で初回訪問を判定します。
* **タイムゾーン非依存**: すべての時刻は UTC で管理するため、プレイヤーのタイムゾーンに依存しません。
* **データサイズ**: `long` 型は2つの `int` に分割されるため、1つの日時で PlayerData の2キーを消費します（FirstVisit: 2キー、LastVisit: 2キー = 合計4キー）。

### パフォーマンス考慮

* ログインボーナスチェックは1回のみ（ワールド入場時）実行されるため、パフォーマンスへの影響は最小限です。
* パーティクルエフェクトは短時間（2秒）のみ再生され、Quest環境でも影響は軽微です。

### Quest最適化

* パーティクルの `Emission Rate` は50に抑え、Quest 2でも60fps維持を確保します。
* パーティクルシステムは `Play On Awake: OFF` に設定し、必要時のみ再生します。

### 依存関係

* **WI-0013（CoinManager Core）**: `_AddCoins()` メソッドが実装されている必要があります。
* **M-03（PersistenceManager）**: `SetLong()` / `GetLong()` メソッドが必要です（本WIで追加）。
* **M-08（HUDManager）**: `_ShowNotification()` メソッドが実装されていることが望ましいですが、未実装でも動作します（null チェックあり）。

### セキュリティ考慮

* Unix時刻の計算はクライアント側で行われるため、システム時刻の改ざんにより不正なボーナス取得が可能です。Phase 2 以降で、Master Client による時刻検証を検討します。

### 実装優先度

* **Phase 1**: 本WIで実装（初回ボーナス、デイリーボーナス）
* **Phase 2**: 連続ログインボーナス（7日連続で追加報酬）、時刻検証の強化

### 関連ドキュメント

* TDD.md - Section 5.6（Module M-06 CoinManager）
* FSD.md - FNC-005（経済システム）
* PRDv3.md - Section 2.5.1（きこりコインの詳細）

---

**作成日**: 2025-11-17
**作成者**: VRChat World Development Team
**レビュー状態**: 未レビュー
**優先度**: 中
**推定工数**: 0.5日
