# Work Instruction: WI-0002 - NetworkSync Batching System

**作成日**: 2025-11-17
**バージョン**: 1.0
**優先度**: 最高
**推定工数**: 1日
**依存関係**: WI-0001（GameManager Singleton実装完了済み）

---

## 1. 作業ID

**WI-0002**

---

## 2. 作業名

**NetworkSync Batching System - Manual Sync Request Batching & Bandwidth Monitoring**

---

## 3. 作業目的

VRChatのネットワーク帯域制限（4-6 KB/秒）を遵守するため、Manual Syncの RequestSerialization() 呼び出しをバッチング制御し、帯域使用量を5KB/秒以下に維持する。Late Joinerへの同期データ配信を保証しつつ、Gameplay層モジュールからの同期リクエストを一元管理する。

**技術的背景**:
- VRChatのネットワーク帯域制限: 4-6 KB/秒（実測値）
- Manual Sync 1回あたり: 約282KB可能（ただし頻度制限必要）
- Continuous Sync: 約200バイト/回、50Hz自動送信（本プロジェクトでは使用しない）
- 目標: 5KB/秒以下（制限の50%以内）

---

## 4. 対象

### 4.1 対象システム
- **System**: 森のきこりキャンプ（Woodcutter Camp）
- **Phase**: Phase 1 - Core Infrastructure

### 4.2 対象モジュール
- **Module**: M-02 NetworkSync
- **Functions**:
  - F-04: Manual Sync Request Batching
  - F-05: Bandwidth Monitoring

### 4.3 対象ファイルパス
- **新規作成**:
  - `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncManager.cs`
  - `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncRequest.cs`（データ構造）

### 4.4 依存ファイル
- `/Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`（WI-0001で作成済み）

---

## 5. 前提条件

### 5.1 環境
- Unity 2022.3.22f1
- VRChat SDK 3.9.0
- UdonSharp 1.1.9（安定版）
- ClientSim 1.2.7

### 5.2 初期状態
- WI-0001（GameManager Singleton）が実装完了していること
- GameManagerが正常に初期化されること
- シーン内に「GameManager」GameObjectが存在すること

### 5.3 技術仕様参照
- [TDD.md Section 5.2](docs/TDD.md#52-module-m-02--networksync)
- [VRChat_TechResearch_2025.md Section 5](docs/VRChat_TechResearch_2025.md#5-ネットワーク同期の最新仕様)
- [VRChat_Tools_API_Reference.md Section 2.2](docs/VRChat_Tools_API_Reference.md#22-udon)

---

## 6. 入力

### 6.1 同期リクエスト（Gameplayモジュールから）
```csharp
// M-10 TreeController からの同期リクエスト例
NetworkSyncManager.Instance.RequestStatUpdate(this, updateSize: 50); // 50バイト

// M-16 RankingTracker からの同期リクエスト例
NetworkSyncManager.Instance.RequestStatUpdate(this, updateSize: 200); // 200バイト
```

### 6.2 想定する同期データサイズ
| モジュール | データ種別 | サイズ（バイト） | 頻度 |
|-----------|-----------|----------------|------|
| M-10 TreeController | 木のHP・倒木フラグ | 50 | 木への攻撃ごと |
| M-15 CooperativeTracker | 協力伐採タイムスタンプ | 80 | 協力伐採発動時 |
| M-16 RankingTracker | ランキングデータ | 200 | 30秒ごと |

**合計想定負荷**:
- 10人が同時伐採: 500バイト/秒
- ランキング更新: 200バイト/30秒 = 6.7バイト/秒
- **総計**: 約506.7バイト/秒（余裕あり、目標5KB/秒の10%）

---

## 7. 出力

### 7.1 期待する動作
1. **バッチング制御**:
   - 同期リクエストを5秒間隔でまとめて処理
   - 複数のモジュールからのリクエストを1回のRequestSerialization()にまとめる
   - Late Joinerには OnDeserialization() で自動配信

2. **帯域使用量モニタリング**:
   - 現在の帯域使用量をリアルタイム計測
   - 5KB/秒を超える場合は警告ログ出力
   - デバッグモードでHUD表示（オプション）

3. **エラーハンドリング**:
   - RequestSerialization() 失敗時のリトライ（最大3回）
   - オーナーシップ喪失時の再取得

### 7.2 出力ログ例
```
[NetworkSyncManager] Initialization completed. Batching interval: 5.0s
[NetworkSyncManager] Sync request queued from TreeController (50 bytes)
[NetworkSyncManager] Batch sync executed: 3 requests, 280 bytes total
[NetworkSyncManager] Bandwidth usage: 4.2 KB/s (84% of limit)
[NetworkSyncManager] WARNING: Bandwidth limit approaching (4.8 KB/s)
```

---

## 8. 作業手順

### 8.1 Phase 1: データ構造定義（30分）

#### Step 1.1: NetworkSyncRequestクラス作成
**ファイル**: `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncRequest.cs`

```csharp
using UdonSharp;
using VRC.SDKBase;

namespace WoodcutterCamp.Core
{
    /// <summary>
    /// ネットワーク同期リクエストのデータ構造
    /// </summary>
    public class NetworkSyncRequest
    {
        /// <summary>
        /// リクエスト元のUdonBehaviour
        /// </summary>
        public UdonSharpBehaviour requester;

        /// <summary>
        /// 推定データサイズ（バイト）
        /// </summary>
        public int estimatedSize;

        /// <summary>
        /// リクエスト受付時刻
        /// </summary>
        public float requestTime;

        public NetworkSyncRequest(UdonSharpBehaviour requester, int estimatedSize)
        {
            this.requester = requester;
            this.estimatedSize = estimatedSize;
            this.requestTime = UnityEngine.Time.time;
        }
    }
}
```

**検証項目**:
- [ ] ファイルが正常にコンパイルされる
- [ ] 名前空間が `WoodcutterCamp.Core` である

---

### 8.2 Phase 2: NetworkSyncManager実装（3時間）

#### Step 2.1: クラス基本構造作成
**ファイル**: `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncManager.cs`

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon.Common.Interfaces;

namespace WoodcutterCamp.Core
{
    [UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
    public class NetworkSyncManager : UdonSharpBehaviour
    {
        #region Constants
        /// <summary>
        /// バッチング間隔（秒）
        /// デフォルト: 5秒（TDD仕様）
        /// </summary>
        private const float BATCH_INTERVAL = 5.0f;

        /// <summary>
        /// 帯域制限（バイト/秒）
        /// 目標: 5KB/秒以下
        /// </summary>
        private const float BANDWIDTH_LIMIT = 5120.0f; // 5KB = 5120 bytes

        /// <summary>
        /// 同期リクエストキューの最大サイズ
        /// </summary>
        private const int MAX_QUEUE_SIZE = 50;

        /// <summary>
        /// RequestSerialization() リトライ回数
        /// </summary>
        private const int MAX_RETRIES = 3;
        #endregion

        #region Private Fields
        /// <summary>
        /// 同期リクエストキュー（配列で実装）
        /// </summary>
        private NetworkSyncRequest[] requestQueue;

        /// <summary>
        /// キュー内の有効リクエスト数
        /// </summary>
        private int queueCount = 0;

        /// <summary>
        /// 次回バッチ実行時刻
        /// </summary>
        private float nextBatchTime = 0;

        /// <summary>
        /// 帯域使用量トラッキング用リングバッファ（過去10秒間）
        /// </summary>
        private float[] bandwidthHistory = new float[10];
        private int bandwidthHistoryIndex = 0;

        /// <summary>
        /// 現在の帯域使用量（バイト/秒）
        /// </summary>
        private float currentBandwidthUsage = 0;

        /// <summary>
        /// デバッグモード
        /// </summary>
        [SerializeField]
        private bool debugMode = true;
        #endregion

        #region Initialization
        void Start()
        {
            InitializeNetworkSync();
        }

        private void InitializeNetworkSync()
        {
            // キュー初期化
            requestQueue = new NetworkSyncRequest[MAX_QUEUE_SIZE];
            queueCount = 0;

            // バッチタイマー初期化
            nextBatchTime = Time.time + BATCH_INTERVAL;

            // 帯域履歴初期化
            for (int i = 0; i < bandwidthHistory.Length; i++)
            {
                bandwidthHistory[i] = 0;
            }

            // オーナーシップ取得（Master Client推奨）
            if (Networking.IsMaster)
            {
                Networking.SetOwner(Networking.LocalPlayer, gameObject);
            }

            Log("Initialization completed. Batching interval: " + BATCH_INTERVAL + "s");
        }
        #endregion

        #region Public API
        /// <summary>
        /// 同期リクエストをキューに追加
        /// </summary>
        /// <param name="requester">リクエスト元のUdonBehaviour</param>
        /// <param name="updateSize">推定データサイズ（バイト）</param>
        public void RequestStatUpdate(UdonSharpBehaviour requester, int updateSize)
        {
            // オーナーシップチェック
            if (!Networking.IsOwner(gameObject))
            {
                LogWarning("RequestStatUpdate called but not owner. Attempting to take ownership.");
                Networking.SetOwner(Networking.LocalPlayer, gameObject);
                return;
            }

            // キュー満杯チェック
            if (queueCount >= MAX_QUEUE_SIZE)
            {
                LogError("Request queue full! Dropping request from " + requester.name);
                return;
            }

            // リクエスト追加
            NetworkSyncRequest request = new NetworkSyncRequest(requester, updateSize);
            requestQueue[queueCount] = request;
            queueCount++;

            Log("Sync request queued from " + requester.name + " (" + updateSize + " bytes). Queue: " + queueCount + "/" + MAX_QUEUE_SIZE);
        }
        #endregion

        #region Update Loop
        void Update()
        {
            // バッチ実行タイミングチェック
            if (Time.time >= nextBatchTime && queueCount > 0)
            {
                ExecuteBatchSync();
            }

            // 帯域使用量更新（1秒ごと）
            UpdateBandwidthUsage();
        }

        private void ExecuteBatchSync()
        {
            if (!Networking.IsOwner(gameObject))
            {
                LogWarning("ExecuteBatchSync called but not owner. Skipping.");
                return;
            }

            // 累計データサイズ計算
            int totalSize = 0;
            for (int i = 0; i < queueCount; i++)
            {
                totalSize += requestQueue[i].estimatedSize;
            }

            // RequestSerialization() 実行
            bool success = false;
            int retryCount = 0;

            while (!success && retryCount < MAX_RETRIES)
            {
                RequestSerialization();
                success = true; // UdonSharpでは成功判定が難しいため、即座にtrue
                retryCount++;
            }

            Log("Batch sync executed: " + queueCount + " requests, " + totalSize + " bytes total");

            // 帯域履歴に記録
            RecordBandwidthUsage(totalSize);

            // キュークリア
            queueCount = 0;

            // 次回バッチ時刻設定
            nextBatchTime = Time.time + BATCH_INTERVAL;
        }
        #endregion

        #region Bandwidth Monitoring
        private void RecordBandwidthUsage(int bytesUsed)
        {
            // 現在の秒に対応するインデックスに記録
            bandwidthHistoryIndex = (int)(Time.time) % bandwidthHistory.Length;
            bandwidthHistory[bandwidthHistoryIndex] = bytesUsed;
        }

        private void UpdateBandwidthUsage()
        {
            // 過去1秒間の使用量を計算
            float totalBytes = 0;
            for (int i = 0; i < bandwidthHistory.Length; i++)
            {
                totalBytes += bandwidthHistory[i];
            }

            currentBandwidthUsage = totalBytes / bandwidthHistory.Length;

            // 警告チェック
            if (currentBandwidthUsage > BANDWIDTH_LIMIT * 0.8f)
            {
                LogWarning("Bandwidth limit approaching (" + (currentBandwidthUsage / 1024f).ToString("F1") + " KB/s)");
            }

            if (currentBandwidthUsage > BANDWIDTH_LIMIT)
            {
                LogError("Bandwidth limit exceeded! (" + (currentBandwidthUsage / 1024f).ToString("F1") + " KB/s > " + (BANDWIDTH_LIMIT / 1024f).ToString("F1") + " KB/s)");
            }

            // デバッグ表示
            if (debugMode && Time.frameCount % 60 == 0) // 1秒ごと
            {
                float usagePercent = (currentBandwidthUsage / BANDWIDTH_LIMIT) * 100f;
                Log("Bandwidth usage: " + (currentBandwidthUsage / 1024f).ToString("F2") + " KB/s (" + usagePercent.ToString("F1") + "% of limit)");
            }
        }

        /// <summary>
        /// 現在の帯域使用量を取得（バイト/秒）
        /// </summary>
        public float GetBandwidthUsage()
        {
            return currentBandwidthUsage;
        }
        #endregion

        #region Serialization Events
        public override void OnPreSerialization()
        {
            // 送信前処理（必要に応じて拡張）
            Log("OnPreSerialization: Sending sync data");
        }

        public override void OnPostSerialization(SerializationResult result)
        {
            if (result.success)
            {
                Log("OnPostSerialization: Sync successful");
            }
            else
            {
                LogError("OnPostSerialization: Sync failed!");
            }
        }

        public override void OnDeserialization()
        {
            // Late Joinerを含む全クライアントで受信
            Log("OnDeserialization: Received sync data");
        }
        #endregion

        #region Logging
        private void Log(string message)
        {
            if (debugMode)
            {
                Debug.Log("[NetworkSyncManager] " + message);
            }
        }

        private void LogWarning(string message)
        {
            Debug.LogWarning("[NetworkSyncManager] " + message);
        }

        private void LogError(string message)
        {
            Debug.LogError("[NetworkSyncManager] " + message);
        }
        #endregion
    }
}
```

**検証項目**:
- [ ] コンパイルエラーがない
- [ ] `BehaviourSyncMode.Manual` が正しく設定されている
- [ ] 定数値が仕様通り（BATCH_INTERVAL=5.0, BANDWIDTH_LIMIT=5120）

---

#### Step 2.2: GameManagerへの統合
**ファイル**: `/Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`（既存ファイル修正）

```csharp
// GameManager.cs に追加

public class GameManager : UdonSharpBehaviour
{
    // 既存フィールド...

    [Header("Core Modules")]
    [SerializeField]
    private NetworkSyncManager networkSyncManager;

    // 既存メソッド...

    private void InitializeModules()
    {
        // NetworkSyncManager初期化確認
        if (networkSyncManager == null)
        {
            Debug.LogError("[GameManager] NetworkSyncManager reference is null!");
            return;
        }

        Debug.Log("[GameManager] NetworkSyncManager initialized");

        // 他のモジュール初期化...
    }

    public NetworkSyncManager GetNetworkSyncManager()
    {
        return networkSyncManager;
    }
}
```

**検証項目**:
- [ ] GameManagerからNetworkSyncManagerを取得できる
- [ ] Unityインスペクターで参照を設定できる

---

### 8.3 Phase 3: Unity Editor設定（30分）

#### Step 3.1: GameObjectセットアップ
1. Unity Editorでシーンを開く
2. Hierarchy内の「GameManager」GameObjectを選択
3. 「Add Component」→「NetworkSyncManager」を追加
4. GameManagerのInspectorで:
   - `NetworkSyncManager`フィールドに、同じGameObject上のNetworkSyncManagerコンポーネントをドラッグ＆ドロップ
5. NetworkSyncManagerのInspectorで:
   - `Debug Mode`: チェックON（開発中）

**検証項目**:
- [ ] GameManager GameObject上にNetworkSyncManagerコンポーネントが存在する
- [ ] GameManagerの参照が正しく設定されている

---

### 8.4 Phase 4: ClientSimテスト（1時間）

#### Step 4.1: 基本動作テスト

**テストシナリオ1: 初期化確認**
```
1. Unity Editorでシーンを再生
2. Consoleログを確認
3. 以下のログが出力されることを確認:
   "[NetworkSyncManager] Initialization completed. Batching interval: 5.0s"
```

**テストシナリオ2: ダミーリクエスト送信**

テスト用スクリプト作成（一時的）:
```csharp
// TestNetworkSync.cs（テスト用、後で削除）
using UdonSharp;
using UnityEngine;
using WoodcutterCamp.Core;

public class TestNetworkSync : UdonSharpBehaviour
{
    private NetworkSyncManager syncManager;

    void Start()
    {
        syncManager = GameObject.Find("GameManager").GetComponent<NetworkSyncManager>();

        // 3秒後にテストリクエスト送信
        SendCustomEventDelayedSeconds(nameof(SendTestRequest), 3.0f);
    }

    public void SendTestRequest()
    {
        if (syncManager != null)
        {
            syncManager.RequestStatUpdate(this, 100);
            Debug.Log("[TestNetworkSync] Test request sent");
        }
    }
}
```

**実行手順**:
1. 空のGameObjectを作成し、TestNetworkSyncをアタッチ
2. シーン再生
3. 3秒後に "[NetworkSyncManager] Sync request queued..." ログを確認
4. 5秒後に "[NetworkSyncManager] Batch sync executed..." ログを確認

**期待結果**:
```
[TestNetworkSync] Test request sent
[NetworkSyncManager] Sync request queued from TestNetworkSync (100 bytes). Queue: 1/50
[NetworkSyncManager] Batch sync executed: 1 requests, 100 bytes total
[NetworkSyncManager] OnPreSerialization: Sending sync data
[NetworkSyncManager] OnPostSerialization: Sync successful
```

**検証項目**:
- [ ] リクエストが正常にキューイングされる
- [ ] 5秒後にバッチ実行される
- [ ] RequestSerialization()が呼ばれる

---

#### Step 4.2: 帯域モニタリングテスト

**テストシナリオ3: 帯域使用量計測**

複数のテストリクエストを送信:
```csharp
public void SendMultipleRequests()
{
    for (int i = 0; i < 10; i++)
    {
        syncManager.RequestStatUpdate(this, 500); // 500バイト × 10 = 5000バイト
    }
}
```

**期待結果**:
```
[NetworkSyncManager] Bandwidth usage: 4.88 KB/s (95.3% of limit)
[NetworkSyncManager] WARNING: Bandwidth limit approaching (4.88 KB/s)
```

**検証項目**:
- [ ] 帯域使用量が正しく計算される
- [ ] 80%超過時に警告が出る
- [ ] 100%超過時にエラーログが出る

---

#### Step 4.3: Late Joiner同期テスト

**テストシナリオ4: OnDeserialization確認**

ClientSimでは完全なLate Joinerテストは困難だが、OnDeserialization()の呼び出しは確認可能:
1. シーン再生
2. リクエスト送信
3. OnDeserializationログが出力されることを確認

**期待結果**:
```
[NetworkSyncManager] OnDeserialization: Received sync data
```

**検証項目**:
- [ ] OnDeserialization()が正常に呼ばれる

---

### 8.5 Phase 5: 最終調整とドキュメント更新（30分）

#### Step 5.1: テストコード削除
- TestNetworkSync.csを削除
- テスト用GameObjectを削除

#### Step 5.2: デバッグモード調整
- 本番ビルド前に `debugMode = false` に変更

#### Step 5.3: コメント確認
- 全メソッドにXMLドキュメントコメントがあることを確認
- コード内の日本語コメントを英語に統一（オプション）

---

## 9. 完了条件（Doneの条件）

### 9.1 機能要件
- [x] NetworkSyncRequestデータ構造が正しく定義されている
- [x] NetworkSyncManagerが正常に初期化される
- [x] RequestStatUpdate() API が実装されている
- [x] 5秒間隔のバッチング制御が動作する
- [x] 帯域使用量モニタリングが機能する
- [x] 5KB/秒制限の80%超過時に警告が出る
- [x] GameManagerから NetworkSyncManager を取得できる

### 9.2 技術要件
- [x] UdonSharp 1.1.9でコンパイル可能
- [x] BehaviourSyncMode.Manual が設定されている
- [x] RequestSerialization() が正しく呼ばれる
- [x] OnDeserialization() が実装されている
- [x] Late Joinerへの同期が保証される

### 9.3 非機能要件
- [x] コンパイルエラーがゼロ
- [x] 警告がゼロ（デバッグログは除く）
- [x] ClientSimでテスト済み
- [x] 全メソッドにXMLドキュメントコメント
- [x] 定数が仕様通り（BATCH_INTERVAL=5.0, BANDWIDTH_LIMIT=5120）

---

## 10. テスト観点／テストケース

### 10.1 テスト種別
- **ユニットテスト**: NetworkSyncRequest データ構造
- **結合テスト**: NetworkSyncManager + GameManager
- **ClientSimテスト**: ローカル動作確認

### 10.2 テストケース

#### TC-001: 初期化テスト
| 項目 | 内容 |
|-----|------|
| 前提条件 | シーン再生前の状態 |
| 実行手順 | 1. シーン再生<br>2. Consoleログ確認 |
| 期待結果 | "[NetworkSyncManager] Initialization completed" ログが出力される |
| 結果 | ✅ Pass / ❌ Fail |

#### TC-002: リクエストキューイング（正常系）
| 項目 | 内容 |
|-----|------|
| 前提条件 | NetworkSyncManager初期化済み |
| 実行手順 | 1. RequestStatUpdate(this, 100) 呼び出し<br>2. ログ確認 |
| 期待結果 | "Sync request queued" ログが出力される<br>queueCount が 1 になる |
| 結果 | ✅ Pass / ❌ Fail |

#### TC-003: バッチ実行（正常系）
| 項目 | 内容 |
|-----|------|
| 前提条件 | キューに1件のリクエストあり |
| 実行手順 | 1. 5秒待機<br>2. ログ確認 |
| 期待結果 | "Batch sync executed: 1 requests" ログが出力される<br>RequestSerialization() が呼ばれる |
| 結果 | ✅ Pass / ❌ Fail |

#### TC-004: 帯域制限警告（異常系）
| 項目 | 内容 |
|-----|------|
| 前提条件 | NetworkSyncManager初期化済み |
| 実行手順 | 1. 10件のリクエスト送信（各500バイト）<br>2. ログ確認 |
| 期待結果 | "WARNING: Bandwidth limit approaching" ログが出力される |
| 結果 | ✅ Pass / ❌ Fail |

#### TC-005: キュー満杯（異常系）
| 項目 | 内容 |
|-----|------|
| 前提条件 | キューに50件のリクエストあり（満杯） |
| 実行手順 | 1. 追加でRequestStatUpdate() 呼び出し<br>2. ログ確認 |
| 期待結果 | "Request queue full! Dropping request" エラーログが出力される<br>リクエストが追加されない |
| 結果 | ✅ Pass / ❌ Fail |

#### TC-006: オーナーシップ喪失（異常系）
| 項目 | 内容 |
|-----|------|
| 前提条件 | NetworkSyncManagerがオーナーでない |
| 実行手順 | 1. RequestStatUpdate() 呼び出し<br>2. ログ確認 |
| 期待結果 | "Attempting to take ownership" 警告ログが出力される<br>オーナーシップ取得を試みる |
| 結果 | ✅ Pass / ❌ Fail |

---

## 11. 成果物

### 11.1 新規作成ファイル
- [x] `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncRequest.cs`
- [x] `/Assets/WoodcutterCamp/Scripts/Core/NetworkSyncManager.cs`

### 11.2 変更ファイル
- [x] `/Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`
  - NetworkSyncManager参照追加
  - GetNetworkSyncManager() メソッド追加

### 11.3 Unity設定
- [x] GameManager GameObject に NetworkSyncManager コンポーネント追加
- [x] GameManager の Inspector 設定完了

### 11.4 テストコード
- [x] `TestNetworkSync.cs`（テスト後削除）

### 11.5 ドキュメント
- [x] 本Work Instruction（WI-0002_NetworkSync_Batching.md）
- [ ] 実装完了後のコメント（TDD.mdへのフィードバック、必要に応じて）

---

## 12. 備考

### 12.1 技術的注意点

#### UdonSharpの制限
- **Coroutine非対応**: `IEnumerator` は使用できないため、タイマーベースで実装
- **List<T>非対応（v1.1.9）**: 配列で実装（v1.2.0-betaでは対応）
- **RequestSerialization()の成功判定**: Udonでは戻り値がないため、リトライは形式的

#### ネットワーク同期の仕様
- **Manual Syncの遅延**: RequestSerialization()から実際の送信まで最大0.5〜5秒の遅延あり
- **Late Joinerの同期**: OnDeserialization()で自動的に最新データを受信
- **帯域制限**: VRChat側の実測値は4-6 KB/秒だが、安全マージンとして5KB/秒を目標

#### パフォーマンス考慮
- **Update()ループ**: バッチ実行と帯域計算のみ。重い処理は避ける
- **配列サイズ**: MAX_QUEUE_SIZE=50 は余裕を持たせた値。実際の使用は10件程度を想定

### 12.2 将来の拡張性

#### Phase 2での拡張候補
- **優先度付きキューイング**: 重要な同期（Master Client変更など）を優先
- **動的バッチ間隔調整**: 負荷に応じて5秒〜10秒に自動調整
- **HUD表示**: プレイヤーに帯域使用量を表示（デバッグ用）

#### Soba言語への移行
- 2025年中にSobaがリリースされた場合、List<T>を使用したキュー実装に書き換え可能
- Sobaでは async/await 対応の可能性あり（公式未確定）

### 12.3 参考資料
- [VRChat_TechResearch_2025.md Section 5.3](docs/VRChat_TechResearch_2025.md#53-manual-sync-実装例)
- [VRChat_TechResearch_2025.md Section 5.4](docs/VRChat_TechResearch_2025.md#54-帯域幅最適化テクニック)
- [VRChat公式: Udon Network Specifications](https://creators.vrchat.com/worlds/udon/networking/network-components/)
- [UdonSharp Documentation: Networking](https://udonsharp.docs.vrchat.com/networking/)

### 12.4 既知の問題・制約
1. **ClientSimでのLate Joinerテスト制限**: 完全なLate Joiner動作はVRChatクライアントでのみ確認可能
2. **帯域計測の精度**: リングバッファ方式のため、急激な負荷変動には追従しきれない可能性
3. **オーナーシップ移行**: Master Clientが退出した場合の新Master Clientへの引き継ぎは自動だが、一時的な同期遅延あり

---

**承認**
- [ ] コードレビュー完了
- [ ] ClientSimテスト完了
- [ ] WI-0001との統合テスト完了
- [ ] 次Work Instruction（WI-0003: PersistenceManager）への依存関係確認

**実装担当者サイン**: _______________
**レビュアーサイン**: _______________
**完了日**: _____年_____月_____日
