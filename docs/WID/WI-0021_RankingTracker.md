# WI-0021: RankingTracker Implementation

**作成日**: 2025-11-17
**モジュール**: M-16 RankingTracker
**優先度**: 中
**推定工数**: 1日

---

## 1. 作業概要

### 1.1 作業目的

VRChatワールド「森のきこりキャンプ」のランキングシステムを実装する。プレイヤーの伐採統計(伐採数、総ダメージ、獲得コイン)をリアルタイムで追跡し、インスタンス内のTOP3ランキングを計算・同期する。

### 1.2 達成目標

- プレイヤー統計の正確な記録(F-54)
- 効率的なランキング計算・ソート(F-55)
- リアルタイム同期とLate Joiner対応(F-56)
- Quest 2環境で60fps維持(80人分のデータ処理)

---

## 2. 対象システム情報

### 2.1 対象

- **対象システム**: 森のきこりキャンプ - Phase 1
- **対象モジュール**: M-16 RankingTracker
- **実装機能**: F-54, F-55, F-56
- **対象ファイルパス**: `Assets/WoodcutterCamp/Scripts/Gameplay/RankingTracker.cs`

### 2.2 技術スタック

- **Unity**: 2022.3.22f1 LTS
- **VRChat SDK**: 3.9.0
- **UdonSharp**: 1.1.9 (安定版)
- **同期モード**: Manual Sync (BehaviourSyncMode.Manual)

---

## 3. 前提条件

### 3.1 依存作業指示書

以下のWork Instructionが完了していること:

- **WI-0001**: GameManager実装済み (モジュール参照取得のため)
- **WI-0002**: NetworkSync実装済み (Manual Sync管理のため)
- **WI-0003**: PersistenceManager実装済み (プレイヤー自身の統計保存のため)
- **WI-0007**: TreeController実装済み (木の伐採完了イベント)
- **WI-0010**: DamageCalculation実装済み (ダメージ記録)
- **WI-0013**: CoinManager実装済み (コイン獲得記録)

### 3.2 環境条件

- Unity 2022.3.22f1がインストール済み
- VRChat SDK 3.9.0がインポート済み
- UdonSharp 1.1.9がインポート済み
- ClientSim 1.2.7が動作可能

### 3.3 初期状態

- GameManagerシングルトンが初期化済み
- ワールドシーン内にRankingTrackerオブジェクト配置済み(空のGameObject)

---

## 4. 機能仕様

### 4.1 統計データ構造

#### 4.1.1 プレイヤーごとのデータ (並列配列方式)

VRChatのインスタンス上限は80人だが、本プロジェクトは20人想定。拡張性のため80人分確保。

```csharp
// プレイヤー識別情報
[UdonSynced] private int[] playerIDs;        // VRCPlayerApi.playerId (最大80)
[UdonSynced] private string[] displayNames;  // VRCPlayerApi.displayName

// 統計データ
[UdonSynced] private int[] treesCut;         // 伐採完了数
[UdonSynced] private int[] totalDamage;      // 累計与ダメージ
[UdonSynced] private int[] coinsEarned;      // 獲得コイン数

// タイムスタンプ (同率時のソート用)
[UdonSynced] private float[] lastUpdateTime; // 最終更新時刻 (Time.time)
```

#### 4.1.2 ランキングデータ (TOP3のみ)

```csharp
// TOP3の情報 (LeaderboardUIで使用)
[UdonSynced] private string[] top3Names;     // 長さ3
[UdonSynced] private int[] top3Damage;       // 長さ3
[UdonSynced] private int[] top3TreesCut;     // 長さ3
```

### 4.2 ソート基準

**ランキング順位の決定ルール**:

1. **第一基準**: Total Damage (降順) - ダメージ量が多い方が上位
2. **第二基準**: Trees Cut (降順) - 伐採数が多い方が上位
3. **第三基準**: lastUpdateTime (昇順) - 早く達成した方が上位

### 4.3 統計の更新タイミング

#### F-54: 統計記録の呼び出し元

| 統計種別 | 呼び出し元モジュール | 呼び出しメソッド | タイミング |
|---------|-------------------|----------------|----------|
| **TreesCut** | TreeController (WI-0007) | `_OnTreeFallen()` | 木のHP=0になった瞬間 |
| **TotalDamage** | DamageCalculation (WI-0010) | `_OnAxeHit()` | 斧が木にヒットした瞬間 |
| **CoinsEarned** | CoinManager (WI-0013) | `AddCoins()` | コイン獲得時 |

#### F-56: ランキング更新頻度

- **バッチング**: 統計変更は即座にローカル反映、5秒ごとにネットワーク同期
- **理由**: 帯域節約(VRChat推奨5KB/秒以下を維持)

---

## 5. 実装手順

### 5.1 RankingTracker.csスクリプト作成

#### ステップ1: クラス定義とフィールド宣言

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class RankingTracker : UdonSharpBehaviour
{
    // 定数
    private const int MAX_PLAYERS = 80;
    private const float SYNC_INTERVAL = 5.0f;

    // 統計データ (UdonSynced)
    [UdonSynced] private int[] playerIDs;
    [UdonSynced] private string[] displayNames;
    [UdonSynced] private int[] treesCut;
    [UdonSynced] private int[] totalDamage;
    [UdonSynced] private int[] coinsEarned;
    [UdonSynced] private float[] lastUpdateTime;

    // TOP3データ (UdonSynced)
    [UdonSynced] private string[] top3Names;
    [UdonSynced] private int[] top3Damage;
    [UdonSynced] private int[] top3TreesCut;

    // ローカル変数
    private bool isInitialized = false;
    private float nextSyncTime = 0f;
    private bool isDirty = false; // 同期が必要か

    // 依存モジュール (GameManagerから取得)
    private PersistenceManager persistenceManager;
}
```

#### ステップ2: 初期化メソッド

```csharp
void Start()
{
    // Late Joiner対応のため1秒遅延
    SendCustomEventDelayedSeconds(nameof(_Initialize), 1.0f);
}

public void _Initialize()
{
    if (isInitialized) return;

    // 配列初期化
    playerIDs = new int[MAX_PLAYERS];
    displayNames = new string[MAX_PLAYERS];
    treesCut = new int[MAX_PLAYERS];
    totalDamage = new int[MAX_PLAYERS];
    coinsEarned = new int[MAX_PLAYERS];
    lastUpdateTime = new float[MAX_PLAYERS];

    // 配列初期値設定
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        playerIDs[i] = -1; // 未使用スロット
        displayNames[i] = "";
        treesCut[i] = 0;
        totalDamage[i] = 0;
        coinsEarned[i] = 0;
        lastUpdateTime[i] = 0f;
    }

    // TOP3初期化
    top3Names = new string[3];
    top3Damage = new int[3];
    top3TreesCut = new int[3];

    // 依存モジュール取得
    GameObject gameManagerObj = GameObject.Find("GameManager");
    if (gameManagerObj == null)
    {
        Debug.LogError("[RankingTracker] GameManager not found");
        return;
    }

    // PersistenceManager取得 (自分の統計保存用)
    persistenceManager = gameManagerObj.GetComponent<PersistenceManager>();
    if (persistenceManager == null)
    {
        Debug.LogWarning("[RankingTracker] PersistenceManager not found, persistence disabled");
    }

    // ローカルプレイヤーのスロット確保
    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    if (localPlayer != null)
    {
        _RegisterPlayer(localPlayer);

        // 自分の統計をPlayerDataから読み込み
        _LoadLocalPlayerStats();
    }

    isInitialized = true;
    Debug.Log("[RankingTracker] Initialized");
}
```

#### ステップ3: プレイヤー登録メソッド

```csharp
/// <summary>
/// プレイヤーをランキングシステムに登録
/// </summary>
private int _RegisterPlayer(VRCPlayerApi player)
{
    if (player == null) return -1;

    int playerId = player.playerId;

    // 既存スロット検索
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == playerId)
        {
            // 既に登録済み
            return i;
        }
    }

    // 空きスロット検索
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == -1)
        {
            playerIDs[i] = playerId;
            displayNames[i] = player.displayName;
            treesCut[i] = 0;
            totalDamage[i] = 0;
            coinsEarned[i] = 0;
            lastUpdateTime[i] = Time.time;

            isDirty = true;
            Debug.Log($"[RankingTracker] Registered player: {player.displayName} at slot {i}");
            return i;
        }
    }

    Debug.LogWarning($"[RankingTracker] No available slot for player {player.displayName}");
    return -1;
}
```

#### ステップ4: 統計追加メソッド (F-54)

```csharp
/// <summary>
/// プレイヤーの統計を追加 (外部モジュールから呼び出し)
/// </summary>
/// <param name="playerId">VRCPlayerApi.playerId</param>
/// <param name="statType">0=TreesCut, 1=TotalDamage, 2=CoinsEarned</param>
/// <param name="value">追加する値</param>
public void AddStatistic(int playerId, int statType, int value)
{
    if (!isInitialized)
    {
        Debug.LogWarning("[RankingTracker] Not initialized yet");
        return;
    }

    // バリデーション
    if (value < 0 || value > 10000)
    {
        Debug.LogWarning($"[RankingTracker] Invalid value: {value}. Clamping to 0-10000");
        value = Mathf.Clamp(value, 0, 10000);
    }

    // プレイヤースロット検索
    int slotIndex = -1;
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == playerId)
        {
            slotIndex = i;
            break;
        }
    }

    if (slotIndex == -1)
    {
        Debug.LogWarning($"[RankingTracker] Player {playerId} not registered");
        return;
    }

    // 統計更新
    switch (statType)
    {
        case 0: // TreesCut
            treesCut[slotIndex] += value;
            break;
        case 1: // TotalDamage
            totalDamage[slotIndex] += value;
            break;
        case 2: // CoinsEarned
            coinsEarned[slotIndex] += value;
            break;
        default:
            Debug.LogError($"[RankingTracker] Invalid statType: {statType}");
            return;
    }

    lastUpdateTime[slotIndex] = Time.time;
    isDirty = true;

    // ローカルプレイヤーの場合、PlayerDataに保存
    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    if (localPlayer != null && localPlayer.playerId == playerId)
    {
        _SaveLocalPlayerStats();
    }
}
```

#### ステップ5: ランキング計算メソッド (F-55)

```csharp
/// <summary>
/// TOP3ランキングを計算 (Master Clientのみ実行)
/// </summary>
private void _CalculateTop3()
{
    if (!Networking.IsMaster) return;

    // ソート用の一時配列 (インデックスと統計のペア)
    int[] indices = new int[MAX_PLAYERS];
    int validPlayerCount = 0;

    // 有効なプレイヤーのインデックスを収集
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] != -1)
        {
            indices[validPlayerCount] = i;
            validPlayerCount++;
        }
    }

    // Insertion Sort (小規模データに効率的)
    // ソート基準: totalDamage降順 → treesCut降順 → lastUpdateTime昇順
    for (int i = 1; i < validPlayerCount; i++)
    {
        int key = indices[i];
        int j = i - 1;

        while (j >= 0 && _CompareRank(indices[j], key) > 0)
        {
            indices[j + 1] = indices[j];
            j--;
        }
        indices[j + 1] = key;
    }

    // TOP3を抽出
    for (int i = 0; i < 3; i++)
    {
        if (i < validPlayerCount)
        {
            int idx = indices[i];
            top3Names[i] = displayNames[idx];
            top3Damage[i] = totalDamage[idx];
            top3TreesCut[i] = treesCut[idx];
        }
        else
        {
            // プレイヤー不足の場合は空欄
            top3Names[i] = "";
            top3Damage[i] = 0;
            top3TreesCut[i] = 0;
        }
    }

    Debug.Log($"[RankingTracker] TOP3 calculated: 1st={top3Names[0]} ({top3Damage[0]} dmg)");
}

/// <summary>
/// ランキング比較関数
/// </summary>
/// <returns>a > b なら正、a < b なら負、a == b なら0</returns>
private int _CompareRank(int indexA, int indexB)
{
    // 第一基準: TotalDamage (降順)
    if (totalDamage[indexA] != totalDamage[indexB])
    {
        return totalDamage[indexB] - totalDamage[indexA];
    }

    // 第二基準: TreesCut (降順)
    if (treesCut[indexA] != treesCut[indexB])
    {
        return treesCut[indexB] - treesCut[indexA];
    }

    // 第三基準: LastUpdateTime (昇順 - 早い方が上位)
    if (lastUpdateTime[indexA] < lastUpdateTime[indexB])
        return -1;
    else if (lastUpdateTime[indexA] > lastUpdateTime[indexB])
        return 1;
    else
        return 0;
}
```

#### ステップ6: ネットワーク同期メソッド (F-56)

```csharp
void Update()
{
    if (!isInitialized) return;

    // Master Clientのみ定期同期
    if (Networking.IsMaster && Time.time >= nextSyncTime)
    {
        if (isDirty)
        {
            // ランキング再計算
            _CalculateTop3();

            // 同期リクエスト
            RequestSerialization();

            isDirty = false;
            Debug.Log("[RankingTracker] Synced ranking data");
        }

        nextSyncTime = Time.time + SYNC_INTERVAL;
    }
}

/// <summary>
/// 同期データ受信時 (Late Joinerも含む)
/// </summary>
public override void OnDeserialization()
{
    Debug.Log("[RankingTracker] Received ranking data update");

    // TOP3が更新された場合、LeaderboardUIに通知 (WI-0022で実装)
    // GameObject leaderboardUI = GameObject.Find("LeaderboardUI");
    // if (leaderboardUI != null)
    // {
    //     leaderboardUI.SendCustomEvent("_UpdateDisplay");
    // }
}
```

#### ステップ7: PlayerData永続化メソッド

```csharp
/// <summary>
/// ローカルプレイヤーの統計をPlayerDataから読み込み
/// </summary>
private void _LoadLocalPlayerStats()
{
    if (persistenceManager == null) return;

    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    if (localPlayer == null) return;

    // PlayerDataから読み込み (別セッションの累計)
    // 注: インスタンス内統計は別管理、これは全期間累計
    int totalTreesCutAllTime = persistenceManager.GetInt("TotalTreesCut", 0);
    int totalDamageAllTime = persistenceManager.GetInt("TotalDamageDealt", 0);

    Debug.Log($"[RankingTracker] Loaded all-time stats: Trees={totalTreesCutAllTime}, Damage={totalDamageAllTime}");
}

/// <summary>
/// ローカルプレイヤーの統計をPlayerDataに保存
/// </summary>
private void _SaveLocalPlayerStats()
{
    if (persistenceManager == null) return;

    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    if (localPlayer == null) return;

    int playerId = localPlayer.playerId;
    int slotIndex = -1;

    // スロット検索
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == playerId)
        {
            slotIndex = i;
            break;
        }
    }

    if (slotIndex == -1) return;

    // 現在のインスタンス統計を取得
    int currentTrees = treesCut[slotIndex];
    int currentDamage = totalDamage[slotIndex];

    // PlayerDataから全期間累計を読み込み
    int previousTrees = persistenceManager.GetInt("TotalTreesCut", 0);
    int previousDamage = persistenceManager.GetInt("TotalDamageDealt", 0);

    // 全期間累計に今回分を加算して保存
    persistenceManager.SetInt("TotalTreesCut", previousTrees + currentTrees);
    persistenceManager.SetInt("TotalDamageDealt", previousDamage + currentDamage);
}
```

#### ステップ8: プレイヤー参加/退出イベント

```csharp
/// <summary>
/// プレイヤー参加時
/// </summary>
public override void OnPlayerJoined(VRCPlayerApi player)
{
    if (!isInitialized) return;

    Debug.Log($"[RankingTracker] Player joined: {player.displayName}");

    // Master Clientのみプレイヤー登録
    if (Networking.IsMaster)
    {
        _RegisterPlayer(player);
    }
}

/// <summary>
/// プレイヤー退出時
/// </summary>
public override void OnPlayerLeft(VRCPlayerApi player)
{
    if (!isInitialized) return;

    Debug.Log($"[RankingTracker] Player left: {player.displayName}");

    // Master Clientのみスロット削除
    if (Networking.IsMaster)
    {
        int playerId = player.playerId;

        for (int i = 0; i < MAX_PLAYERS; i++)
        {
            if (playerIDs[i] == playerId)
            {
                // スロットを空に戻す
                playerIDs[i] = -1;
                displayNames[i] = "";
                treesCut[i] = 0;
                totalDamage[i] = 0;
                coinsEarned[i] = 0;
                lastUpdateTime[i] = 0f;

                isDirty = true;
                Debug.Log($"[RankingTracker] Removed player {player.displayName} from slot {i}");
                break;
            }
        }
    }
}
```

#### ステップ9: ランキング取得メソッド (UI用)

```csharp
/// <summary>
/// 指定プレイヤーの順位を取得
/// </summary>
/// <param name="playerId">VRCPlayerApi.playerId</param>
/// <returns>順位 (1-80)、未登録の場合は-1</returns>
public int GetRanking(int playerId)
{
    if (!isInitialized) return -1;

    // プレイヤースロット検索
    int targetSlot = -1;
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == playerId)
        {
            targetSlot = i;
            break;
        }
    }

    if (targetSlot == -1) return -1;

    // 順位計算
    int rank = 1;
    for (int i = 0; i < MAX_PLAYERS; i++)
    {
        if (playerIDs[i] == -1) continue;
        if (i == targetSlot) continue;

        // 自分より上位のプレイヤー数をカウント
        if (_CompareRank(i, targetSlot) < 0)
        {
            rank++;
        }
    }

    return rank;
}

/// <summary>
/// TOP3データ取得 (LeaderboardUIから呼び出し)
/// </summary>
public string[] GetTop3Names() { return top3Names; }
public int[] GetTop3Damage() { return top3Damage; }
public int[] GetTop3TreesCut() { return top3TreesCut; }
```

---

### 5.2 Unity Editorでの設定手順

#### ステップ1: GameObjectの作成

1. Unity Editorでワールドシーンを開く
2. Hierarchy右クリック → `Create Empty`
3. 名前を `RankingTracker` に変更
4. 位置を `(0, 0, 0)` に設定

#### ステップ2: コンポーネントの追加

1. `RankingTracker` GameObjectを選択
2. Inspector → `Add Component` → `RankingTracker` スクリプトを追加
3. UdonBehaviourコンポーネントが自動追加されることを確認

#### ステップ3: Udon Behaviourの設定

1. `Udon Behaviour` コンポーネントを確認
2. `Sync Method` が `Manual` になっていることを確認
3. `Serialization Timing` は `Default` のまま

#### ステップ4: GameManagerへの参照追加

1. `GameManager` GameObjectを選択
2. Inspector → `GameManager` スクリプトで参照フィールドを確認
3. `Ranking Tracker` フィールドに `RankingTracker` GameObjectをドラッグ&ドロップ

---

## 6. テストケース

### 6.1 正常系テスト

#### TC-01: 統計記録の正常動作

**前提条件**:
- RankingTrackerが初期化済み
- 1人以上のプレイヤーがインスタンスに存在

**手順**:
1. ClientSimでPlayモードに入る
2. 木を伐採する (TreesCut +1, TotalDamage +10〜50)
3. 丸太を納品する (CoinsEarned +3〜30)
4. Consoleでログ確認

**期待結果**:
- `[RankingTracker] Registered player: <プレイヤー名>` が表示される
- 統計が正しく記録される
- エラーログが出ない

#### TC-02: ランキング計算の正確性

**前提条件**:
- 3人以上のプレイヤーがインスタンスに存在 (テストデータを手動投入)

**手順**:
1. プレイヤーAの統計を設定: Damage=100, Trees=5
2. プレイヤーBの統計を設定: Damage=150, Trees=3
3. プレイヤーCの統計を設定: Damage=100, Trees=6
4. `_CalculateTop3()` を呼び出し
5. `top3Names`, `top3Damage` を確認

**期待結果**:
- 1位: プレイヤーB (Damage 150)
- 2位: プレイヤーC (Damage 100, Trees 6)
- 3位: プレイヤーA (Damage 100, Trees 5)

#### TC-03: ネットワーク同期

**前提条件**:
- VRChatテストワールドにアップロード済み
- 2台のVRChatクライアントで同じインスタンスに参加

**手順**:
1. クライアントAで木を伐採
2. 5秒待機 (SYNC_INTERVAL)
3. クライアントBでランキング表示を確認

**期待結果**:
- クライアントBでクライアントAの統計が反映される
- 5秒以内に同期される

### 6.2 異常系テスト

#### TC-04: 無効な値の処理

**前提条件**:
- RankingTrackerが初期化済み

**手順**:
1. `AddStatistic(playerId, 1, -100)` を呼び出し (負値)
2. `AddStatistic(playerId, 1, 999999)` を呼び出し (異常値)
3. Consoleでログ確認

**期待結果**:
- Warningログが出力される
- 値が0または10000にClampされる
- クラッシュしない

#### TC-05: プレイヤー数上限

**前提条件**:
- RankingTrackerが初期化済み

**手順**:
1. 80人分のダミープレイヤーを登録
2. 81人目のプレイヤーを登録試行
3. Consoleでログ確認

**期待結果**:
- Warningログ `No available slot` が出力される
- クラッシュしない

#### TC-06: Master Client退出

**前提条件**:
- VRChatテストワールドで2人以上のインスタンス

**手順**:
1. Master Clientが統計を記録
2. Master Clientがワールドから退出
3. 新しいMaster Clientでランキングを確認

**期待結果**:
- UdonSyncedデータが新Master Clientに引き継がれる
- ランキングが維持される

### 6.3 パフォーマンステスト

#### TC-07: ソート処理時間

**前提条件**:
- 80人分のダミーデータを投入

**手順**:
1. UnityのProfilerを起動
2. `_CalculateTop3()` を100回実行
3. 平均実行時間を測定

**期待結果**:
- 1回あたり10ms以内
- Quest 2環境でも60fps維持

#### TC-08: 同期データサイズ

**前提条件**:
- 80人分のデータを投入

**手順**:
1. VRChat Debugログで同期データサイズを確認
2. `RequestSerialization()` 呼び出し時のバイト数を記録

**期待結果**:
- 1回の同期が5KB以内
- 5秒間隔で1KB/秒以下の帯域使用

---

## 7. 完了条件 (Done Definition)

### 7.1 実装完了の定義

- [ ] RankingTracker.csスクリプトが作成され、すべてのメソッドが実装されている
- [ ] Unity Editorでコンパイルエラーが0件
- [ ] GameObjectが正しくシーンに配置され、UdonBehaviourが設定されている
- [ ] GameManagerへの参照が正しく設定されている

### 7.2 機能確認の定義

- [ ] TC-01〜TC-03の正常系テストがすべてパスする
- [ ] TC-04〜TC-06の異常系テストがすべてパスする
- [ ] TC-07〜TC-08のパフォーマンステストがすべてパスする

### 7.3 品質基準の定義

- [ ] Consoleに **Error** ログが0件
- [ ] Consoleに **Warning** ログが5件以下 (想定される警告のみ)
- [ ] ClientSimで10分以上動作してもメモリリークがない
- [ ] Quest 2実機テストで60fps維持 (VRChat Stats表示で確認)

### 7.4 ドキュメント完了の定義

- [ ] コード内にXMLドキュメントコメントが記述されている
- [ ] 主要メソッドに日本語コメントが記述されている
- [ ] README.mdに使用方法が記載されている (オプション)

---

## 8. 成果物

### 8.1 作成ファイル

| ファイルパス | 種類 | 説明 |
|------------|------|------|
| `Assets/WoodcutterCamp/Scripts/Gameplay/RankingTracker.cs` | スクリプト | RankingTrackerの実装 |

### 8.2 変更ファイル

| ファイルパス | 変更内容 |
|------------|---------|
| `Assets/WoodcutterCamp/Scenes/MainWorld.unity` | RankingTracker GameObjectの追加 |
| `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs` | RankingTrackerへの参照追加 (オプション) |

### 8.3 テストコード

- 本WIではUdonSharpの制約によりユニットテストは作成しない
- 手動テストケースのみ実施

---

## 9. 外部モジュールとの連携

### 9.1 呼び出し側モジュール (RankingTrackerを使用)

| モジュール | メソッド | 呼び出すタイミング | 渡すパラメータ |
|----------|---------|------------------|--------------|
| TreeController (WI-0007) | `AddStatistic()` | 木が倒れた時 | playerId, statType=0 (TreesCut), value=1 |
| DamageCalculation (WI-0010) | `AddStatistic()` | 斧がヒットした時 | playerId, statType=1 (TotalDamage), value=ダメージ量 |
| CoinManager (WI-0013) | `AddStatistic()` | コイン獲得時 | playerId, statType=2 (CoinsEarned), value=獲得コイン |

### 9.2 呼び出し例 (TreeControllerから)

```csharp
// TreeController.cs の _OnTreeFallen() メソッド内
public void _OnTreeFallen()
{
    // ... 既存の倒木処理 ...

    // RankingTrackerに伐採完了を通知
    GameObject rankingObj = GameObject.Find("RankingTracker");
    if (rankingObj != null)
    {
        RankingTracker rankingTracker = rankingObj.GetComponent<RankingTracker>();
        if (rankingTracker != null)
        {
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer != null)
            {
                // TreesCut +1
                rankingTracker.AddStatistic(localPlayer.playerId, 0, 1);
            }
        }
    }
}
```

---

## 10. パフォーマンス最適化ポイント

### 10.1 配列サイズの最適化

- MAX_PLAYERS=80としているが、実際の想定は20人
- 拡張性のため80確保、メモリ使用量は約10KB程度で許容範囲

### 10.2 ソートアルゴリズムの選択

- **Insertion Sort**を採用
  - 理由1: データ量が少ない (最大80件)
  - 理由2: UdonSharpでQuickSort実装は複雑
  - 理由3: 80件のInsertion Sortは約1ms以下で完了
  - 計算量: O(n²) だが n=80 なら6400回比較 (許容範囲)

### 10.3 同期頻度の調整

- **5秒間隔**でバッチング
  - 理由: 毎ヒットごとに同期すると帯域オーバー
  - 5秒間隔なら最大0.2KB/秒 (目標5KB/秒の4%のみ使用)

### 10.4 GetComponent呼び出しの削減

- Start()で依存モジュールをキャッシュ
- Update()内でのGameObject.Find()は避ける

---

## 11. トラブルシューティング

### 11.1 統計が同期されない

**症状**: 他プレイヤーの統計が更新されない

**原因**:
- Master Clientが存在しない
- `RequestSerialization()` が呼ばれていない

**対処法**:
1. Consoleで `[RankingTracker] Synced ranking data` ログを確認
2. `Networking.IsMaster` の値を確認
3. `isDirty` フラグが正しく設定されているか確認

### 11.2 ランキング順位がおかしい

**症状**: ダメージが多いのに順位が低い

**原因**:
- `_CompareRank()` の比較ロジックが逆

**対処法**:
1. `_CompareRank()` の戻り値を確認
2. テストデータで手動デバッグ

### 11.3 Late Joinerでデータが表示されない

**症状**: 途中参加したプレイヤーにランキングが表示されない

**原因**:
- `OnDeserialization()` が呼ばれていない
- UdonSynced変数が正しく宣言されていない

**対処法**:
1. UdonBehaviourのSync Modeが `Manual` か確認
2. `[UdonSynced]` 属性が正しく付いているか確認
3. VRChatテストワールドで実機確認

---

## 12. 参考資料

### 12.1 関連ドキュメント

- TDD.md - Section 5.16 (Module M-16 RankingTracker)
- FSD.md - Section 6 (FNC-006 協力・ソーシャル機能)
- PRD.md - Section 2.6 (デイリーランキングシステム)
- VRChat_Tools_API_Reference.md - Section 2.2 (Udon API)
- VRChat_Tools_API_Reference.md - Section 2.3 (UdonSharp)

### 12.2 VRChat公式ドキュメント

- [Udon Networking](https://creators.vrchat.com/worlds/udon/networking/)
- [PlayerData API](https://creators.vrchat.com/worlds/udon/players/player-data/)
- [UdonSharp Documentation](https://udonsharp.docs.vrchat.com/)

### 12.3 コード例

- 本WIのステップ1〜9のコードスニペット
- TreeController連携の呼び出し例 (Section 9.2)

---

## 13. 備考

### 13.1 注意事項

- **Master Client検証必須**: 統計更新とランキング計算は必ずMaster Clientで実行すること
- **Late Joiner対応**: 初期化を1秒遅延させることでLate Joinerのデータ同期を保証
- **PlayerData制限**: 本モジュールは100KB制限の約0.5%のみ使用 (他モジュールと共用)
- **Quest最適化**: 配列操作を最小限にし、Update()内の処理を軽量化すること

### 13.2 将来の拡張予定 (Phase 2)

- グローバルランキング (全インスタンス横断)
- ランキングティア (金・銀・銅バッジ)
- ランキング履歴 (日次・週次・月次)
- ランキング報酬 (上位者に特別アイテム)

### 13.3 既知の制限

- インスタンスごとにランキングがリセットされる (仕様)
- 同時接続80人が上限 (VRChat制限)
- UdonSyncedの同期遅延が最大5秒 (SYNC_INTERVAL設定による)

---

**作業指示書バージョン**: 1.0
**最終更新日**: 2025-11-17
**次回レビュー**: WI-0021実装完了後
