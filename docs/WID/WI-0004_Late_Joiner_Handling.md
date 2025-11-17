# 作業指示書: WI-0004 - Setup Late Joiner Handling

**作成日**: 2025-11-17
**対象フェーズ**: Phase 0 - 基盤実装
**優先度**: 高
**見積工数**: 0.5日

---

## 1. 作業ID

**WI-0004**

---

## 2. 作業名

Late Joiner 同期対応の実装（OnDeserialization統合と初期化遅延）

---

## 3. 作業目的

VRChatワールドに途中参加したプレイヤー（Late Joiner）が、既存のネットワーク同期データを正しく受信できるようにする。Late Joiner特有の既知の問題（OnDeserialization()が発火しない、初期化タイミングのずれ）を回避し、全プレイヤーが同じゲーム状態を共有できる基盤を構築する。

---

## 4. 対象

### 対象システム
- **システム名**: 森のきこりキャンプ (Woodcutter Camp)
- **モジュール**: M-02 NetworkSync

### 対象ファイルパス
- `/mnt/d/Dropbox/Dev/CLI/WCF/Assets/Scripts/Core/NetworkSync.cs` （新規作成）
- `/mnt/d/Dropbox/Dev/CLI/WCF/Assets/Scripts/Core/GameManager.cs` （WI-0001で作成済み、本WIで修正）

### 関連技術資料
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/TDD.md` - Module M-02定義（Section 5.2）
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/VRChat_TechResearch_2025.md` - Late Joiner既知の問題（Section 5.5）
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/TECHNICAL_DETAILS.md` - Late Joiner対応パターン（Section H.1）

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0 以上
- UdonSharp 1.1.9 (stable)
- VCC 2.4.0 以上

### 作業前提
- **WI-0001**: GameManager Singletonが実装済み
- **WI-0002**: NetworkSync Batching Systemが実装済み
- 開発ブランチ: `feature/late-joiner-handling`
- GameManager.cs に以下のメソッドが存在していること:
  - `_Initialize()` - システム初期化メソッド
  - `_DelayedInitialization()` - 1秒遅延後の初期化メソッド（本WIで追加）

### 初期状態
- NetworkSync.cs は未作成
- GameManager.cs の初期化処理は即座に実行されている（Late Joiner対応なし）

---

## 6. 入力

### Late Joiner参加時のシナリオ
1. **既存プレイヤー**: ワールドに3人がすでに参加済み、ゲームプレイ中
2. **Late Joiner**: 4人目としてワールドに参加
3. **同期データ**: 以下のデータが既にネットワーク上に存在
   - `[UdonSynced]` 変数（木の状態、プレイヤースコア、協力伐採タイムスタンプなど）
   - PlayerData（スキルレベル、コイン残高など）

### 既知の問題（技術制約）
- **問題1**: OnDeserialization()がLate Joinerで発火しないケースがある（VRChat_TechResearch_2025.md L477-480参照）
  - 発生条件: UdonSynced変数の大量データ時
  - 進捗: サーバ側修正検討中（2025年7月時点）
- **問題2**: Start()時点でPlayerDataが未準備（TECHNICAL_DETAILS.md Section H.1参照）
  - OnPlayerRestored()イベントで初めて安全にアクセス可能

---

## 7. 出力

### 期待する動作
1. **Late Joinerが参加したとき**:
   - GameManagerの初期化が1秒遅延する
   - 遅延後、OnDeserializationが確実に発火し、UdonSyncedデータを受信
   - PlayerDataの読み込みが完了している
   - 既存プレイヤーと同じゲーム状態で開始できる

2. **既存プレイヤーへの影響**:
   - 初期化が1秒遅延するが、ゲームプレイには影響しない
   - ネットワーク同期の安定性が向上

### ログ出力例
```
[GameManager] Late Joiner detected. Delaying initialization for 1 second...
[GameManager] Delayed initialization started
[NetworkSync] OnDeserialization called - synced data received
[NetworkSync] Late Joiner sync completed successfully
```

---

## 8. 作業手順

### 手順1: NetworkSync.cs の作成

#### 1.1 ファイル作成
1. Unityエディタを起動
2. Project ウィンドウで右クリック
3. `Create > Folder` で `Assets/Scripts/Core` ディレクトリを作成（未作成の場合）
4. `Assets/Scripts/Core` を右クリック
5. `Create > C# Script` を選択
6. ファイル名を `NetworkSync` に変更

#### 1.2 基本コード実装
以下のコードを NetworkSync.cs に記述してください:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

namespace WoodcutterCamp.Core
{
    /// <summary>
    /// ネットワーク同期の抽象化レイヤー
    /// Late Joiner対応とManual Sync Batching制御を提供
    /// </summary>
    public class NetworkSync : UdonSharpBehaviour
    {
        // Late Joiner 検出フラグ
        private bool isLateJoiner = false;
        private bool hasReceivedInitialSync = false;

        // 初期化完了フラグ
        private bool isInitialized = false;

        // GameManager参照
        private GameManager gameManager;

        #region Initialization

        /// <summary>
        /// システム初期化（GameManagerから呼ばれる）
        /// </summary>
        public void _Initialize()
        {
            Debug.Log("[NetworkSync] Initializing...");

            // GameManager参照取得
            gameManager = GameObject.Find("GameManager").GetComponent<GameManager>();

            if (gameManager == null)
            {
                Debug.LogError("[NetworkSync] GameManager not found!");
                return;
            }

            // Late Joiner判定
            DetectLateJoiner();

            isInitialized = true;
            Debug.Log("[NetworkSync] Initialization completed");
        }

        /// <summary>
        /// Late Joiner判定
        /// </summary>
        private void DetectLateJoiner()
        {
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer == null)
            {
                Debug.LogWarning("[NetworkSync] LocalPlayer is null");
                return;
            }

            // プレイヤーがMasterでない かつ 他にプレイヤーがいる場合はLate Joiner
            VRCPlayerApi[] allPlayers = new VRCPlayerApi[VRCPlayerApi.GetPlayerCount()];
            VRCPlayerApi.GetPlayers(allPlayers);

            if (!localPlayer.isMaster && allPlayers.Length > 1)
            {
                isLateJoiner = true;
                Debug.Log("[NetworkSync] Late Joiner detected");
            }
            else
            {
                isLateJoiner = false;
                Debug.Log("[NetworkSync] Not a Late Joiner (Master or first player)");
            }
        }

        #endregion

        #region Late Joiner Sync Handling

        /// <summary>
        /// OnDeserialization - ネットワークデータ受信時
        /// Late Joinerの初回同期をここで検出
        /// </summary>
        public override void OnDeserialization()
        {
            if (!isInitialized) return;

            if (isLateJoiner && !hasReceivedInitialSync)
            {
                Debug.Log("[NetworkSync] OnDeserialization called - Late Joiner initial sync received");
                hasReceivedInitialSync = true;

                // Late Joiner同期完了を通知
                OnLateJoinerSyncCompleted();
            }
        }

        /// <summary>
        /// Late Joiner同期完了時のコールバック
        /// </summary>
        private void OnLateJoinerSyncCompleted()
        {
            Debug.Log("[NetworkSync] Late Joiner sync completed successfully");

            // 将来的に他のモジュールに通知するイベントをここで発火
            // 例: gameManager._OnNetworkSyncReady();
        }

        #endregion

        #region Utility Methods

        /// <summary>
        /// Late Joinerかどうかを取得
        /// </summary>
        public bool IsLateJoiner()
        {
            return isLateJoiner;
        }

        /// <summary>
        /// 初期同期が完了しているかを取得
        /// </summary>
        public bool HasReceivedInitialSync()
        {
            return hasReceivedInitialSync;
        }

        #endregion
    }
}
```

#### 1.3 コンポーネント説明
- **DetectLateJoiner()**: プレイヤーがMasterでなく、既に他のプレイヤーがいる場合にLate Joinerと判定
- **OnDeserialization()**: UdonSyncedデータ受信時に自動的に呼ばれる。Late Joinerの初回受信を検出
- **OnLateJoinerSyncCompleted()**: 同期完了後に他モジュールへ通知するためのコールバック（拡張ポイント）

---

### 手順2: GameManager.cs の修正（初期化遅延の実装）

#### 2.1 GameManager.cs を開く
1. Unityエディタで `Assets/Scripts/Core/GameManager.cs` を開く

#### 2.2 初期化遅延メソッドの追加
GameManager.cs の以下の箇所を修正してください:

**変更前（WI-0001で実装済みの状態）**:
```csharp
void Start()
{
    _Initialize();
}

public void _Initialize()
{
    Debug.Log("[GameManager] Initializing...");

    // モジュール初期化
    InitializeModules();

    Debug.Log("[GameManager] Initialization completed");
}
```

**変更後**:
```csharp
void Start()
{
    // Late Joiner対応: 初期化を1秒遅延
    VRCPlayerApi localPlayer = Networking.LocalPlayer;
    if (localPlayer != null && !localPlayer.isMaster)
    {
        VRCPlayerApi[] allPlayers = new VRCPlayerApi[VRCPlayerApi.GetPlayerCount()];
        VRCPlayerApi.GetPlayers(allPlayers);

        if (allPlayers.Length > 1)
        {
            Debug.Log("[GameManager] Late Joiner detected. Delaying initialization for 1 second...");
            SendCustomEventDelayedSeconds(nameof(_DelayedInitialization), 1.0f);
            return;
        }
    }

    // 通常の初期化（Masterまたは最初のプレイヤー）
    _Initialize();
}

/// <summary>
/// 遅延初期化（Late Joiner用）
/// SendCustomEventDelayedSecondsで1秒後に呼ばれる
/// </summary>
public void _DelayedInitialization()
{
    Debug.Log("[GameManager] Delayed initialization started");
    _Initialize();
}

public void _Initialize()
{
    Debug.Log("[GameManager] Initializing...");

    // モジュール初期化
    InitializeModules();

    Debug.Log("[GameManager] Initialization completed");
}
```

#### 2.3 変更内容の説明
- **Late Joiner判定**: Start()でプレイヤーがMasterでなく、他にプレイヤーがいる場合
- **SendCustomEventDelayedSeconds**: UdonSharpのタイマーメソッド。1秒後に`_DelayedInitialization()`を実行
- **アンダースコアプレフィックス**: `_DelayedInitialization()`はUdon CustomEventとして呼び出し可能にするため

---

### 手順3: Unity Scene への配置

#### 3.1 NetworkSync GameObjectの作成
1. Hierarchyウィンドウで右クリック
2. `Create Empty` を選択
3. 名前を `NetworkSync` に変更
4. `NetworkSync` オブジェクトを選択した状態で Inspector ウィンドウを確認

#### 3.2 NetworkSync.cs のアタッチ
1. `NetworkSync` オブジェクトを選択
2. Inspector ウィンドウで `Add Component` をクリック
3. `NetworkSync` (UdonSharpBehaviour) を検索して追加
4. UdonBehaviour コンポーネントが自動的に追加されることを確認

#### 3.3 GameManagerからの参照設定
1. `GameManager` オブジェクトを選択
2. GameManager.cs の `InitializeModules()` メソッドに以下を追加:

```csharp
private void InitializeModules()
{
    Debug.Log("[GameManager] Initializing modules...");

    // NetworkSync初期化
    GameObject networkSyncObj = GameObject.Find("NetworkSync");
    if (networkSyncObj != null)
    {
        NetworkSync networkSync = networkSyncObj.GetComponent<NetworkSync>();
        if (networkSync != null)
        {
            networkSync._Initialize();
        }
        else
        {
            Debug.LogError("[GameManager] NetworkSync component not found!");
        }
    }
    else
    {
        Debug.LogError("[GameManager] NetworkSync GameObject not found!");
    }

    // 将来的に他のモジュール初期化もここに追加
}
```

---

### 手順4: UdonSharp コンパイル

#### 4.1 コンパイル実行
1. Unity上部メニューから `UdonSharp > Compile All UdonSharp Programs` を選択
2. Consoleウィンドウでエラーがないことを確認
3. エラーがある場合は以下を確認:
   - UdonSharp 1.1.9がインストールされているか
   - namespace の記述が正しいか
   - VRChat SDK 3.9.0以上がインストールされているか

#### 4.2 コンパイルエラー対応
もしコンパイルエラーが発生した場合:

**エラー例1**: `'SendCustomEventDelayedSeconds' is not supported`
- **原因**: UdonSharpバージョンが古い
- **対応**: VCC から UdonSharp 1.1.9 以上に更新

**エラー例2**: `Type 'NetworkSync' could not be found`
- **原因**: namespace が未解決
- **対応**: GameManager.cs の先頭に `using WoodcutterCamp.Core;` を追加

---

### 手順5: ClientSim テスト（ローカル単体テスト）

#### 5.1 ClientSim 起動
1. Unity Scene View で Play ボタンをクリック
2. VRChat ClientSim が起動する（VCC でインストール済みの場合）
3. Console ウィンドウでログを確認

#### 5.2 期待されるログ出力
```
[GameManager] Initializing...
[GameManager] Initializing modules...
[NetworkSync] Initializing...
[NetworkSync] Not a Late Joiner (Master or first player)
[NetworkSync] Initialization completed
[GameManager] Initialization completed
```

#### 5.3 Late Joiner シミュレーション
ClientSimではLate Joinerを直接テストできないため、以下の方法で疑似テスト:

1. GameManager.cs の Start() に以下を一時的に追加:
```csharp
// テスト用: 強制的にLate Joiner扱い
Debug.Log("[GameManager] Late Joiner detected. Delaying initialization for 1 second...");
SendCustomEventDelayedSeconds(nameof(_DelayedInitialization), 1.0f);
return;
```

2. Play ボタンをクリックして実行
3. 1秒後に初期化が実行されることを確認
4. テスト完了後、上記のコードを削除

---

### 手順6: VRChat Build & Upload テスト（実機テスト）

#### 6.1 ビルド設定
1. Unity上部メニュー `VRChat SDK > Show Control Panel` を選択
2. `Builder` タブを開く
3. `Build & Test` ボタンをクリック
4. ビルドが完了するまで待機（約3-5分）

#### 6.2 Late Joiner 実機テスト手順
**必要人数**: 2人（テスターA: Master、テスターB: Late Joiner）

1. **テスターA（Master）**:
   - VRChatクライアントでワールドに参加
   - Console ログを確認（Unity Consoleではなく、VRChat内のDebug Console）
   - 以下のログが表示されることを確認:
     ```
     [NetworkSync] Not a Late Joiner (Master or first player)
     ```

2. **テスターB（Late Joiner）**:
   - テスターAが参加後、30秒待ってから同じワールドに参加
   - 以下のログが表示されることを確認:
     ```
     [GameManager] Late Joiner detected. Delaying initialization for 1 second...
     [GameManager] Delayed initialization started
     [NetworkSync] Late Joiner detected
     [NetworkSync] OnDeserialization called - Late Joiner initial sync received
     [NetworkSync] Late Joiner sync completed successfully
     ```

3. **Late Joiner同期確認**:
   - テスターBのゲーム状態がテスターAと同じであることを確認
   - （将来的な確認項目: 木の状態、スコア、UIが正しく表示される）

#### 6.3 VRChat Debug Console の有効化
Windows環境の場合、以下のコマンドでVRChatを起動すると詳細ログが確認できます:

```bash
"C:\Program Files (x86)\Steam\steamapps\common\VRChat\VRChat.exe" --enable-debug-gui
```

Debug GUIで `Show Debug Console` をクリックするとログが表示されます。

---

## 9. 完了条件（Doneの条件）

以下の全ての条件を満たすこと:

### コード実装
- [ ] NetworkSync.cs が作成され、以下のメソッドが実装されている:
  - [ ] `_Initialize()`
  - [ ] `DetectLateJoiner()`
  - [ ] `OnDeserialization()`
  - [ ] `IsLateJoiner()` / `HasReceivedInitialSync()`
- [ ] GameManager.cs に以下が実装されている:
  - [ ] `_DelayedInitialization()` メソッド
  - [ ] Start()でLate Joiner判定と1秒遅延処理
  - [ ] InitializeModules()でNetworkSync初期化

### Unity設定
- [ ] NetworkSync GameObjectがHierarchyに配置されている
- [ ] NetworkSync.cs がコンポーネントとして正しくアタッチされている
- [ ] UdonSharpコンパイルがエラーなく完了している

### テスト完了
- [ ] ClientSimでの単体テストが成功（初期化ログ確認）
- [ ] VRChat実機でのLate Joinerテストが成功（2人テスト）
- [ ] OnDeserialization()が正しく発火することを確認
- [ ] 初期化が1秒遅延することを確認（Late Joinerのみ）

### ドキュメント
- [ ] 本作業指示書のチェックリストが全て完了
- [ ] TDD.mdのModule M-02 Function F-06が実装完了とマーク

---

## 10. テスト観点／テストケース

### テスト種別
- **ユニットテスト**: ClientSimでのローカル動作確認
- **結合テスト**: VRChat実機での2人同時接続テスト

### テストケース

#### 正常系

| TC-ID | テストケース | 入力条件 | 期待結果 | 確認方法 |
|-------|------------|---------|---------|---------|
| TC-001 | Master初期化 | 最初のプレイヤーとしてワールド参加 | 即座に初期化、Late Joiner判定なし | Consoleログ確認 |
| TC-002 | Late Joiner初期化 | 2人目として参加 | 1秒遅延後に初期化、Late Joiner判定あり | Consoleログ確認 |
| TC-003 | OnDeserialization発火 | Late Joinerとして参加 | OnDeserialization()が呼ばれる | Consoleログ確認 |
| TC-004 | 同期データ受信 | Late Joinerとして参加 | hasReceivedInitialSync = true | IsLateJoiner()で確認 |

#### 異常系

| TC-ID | テストケース | 入力条件 | 期待結果 | 確認方法 |
|-------|------------|---------|---------|---------|
| TC-101 | GameManager未検出 | NetworkSync初期化時にGameManagerなし | エラーログ出力、処理中断なし | Consoleログ確認 |
| TC-102 | LocalPlayerがnull | Start()実行時にNetworking.LocalPlayerがnull | 警告ログ出力、初期化スキップ | Consoleログ確認 |
| TC-103 | OnDeserialization未発火 | 大量データ同期時 | タイムアウト後もゲーム続行可能 | （既知の問題、将来対応） |

#### パフォーマンステスト

| TC-ID | テストケース | 入力条件 | 期待結果 | 確認方法 |
|-------|------------|---------|---------|---------|
| TC-201 | 初期化遅延が1秒 | Late Joiner参加 | 1.0秒±0.1秒で初期化完了 | Time.realtimeSinceStartupで計測 |
| TC-202 | 既存プレイヤーへの影響なし | Late Joiner参加中 | 既存プレイヤーのフレームレート低下なし | VRChat Stats表示で確認 |

---

## 11. 成果物

### 変更ファイル
- `/mnt/d/Dropbox/Dev/CLI/WCF/Assets/Scripts/Core/NetworkSync.cs` （新規作成）
- `/mnt/d/Dropbox/Dev/CLI/WCF/Assets/Scripts/Core/GameManager.cs` （修正）

### 追加Unity Assets
- `/mnt/d/Dropbox/Dev/CLI/WCF/Assets/Scenes/MainScene.unity` （NetworkSync GameObject追加）

### 更新したドキュメント
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/TDD.md` - Module M-02 Function F-06を実装完了とマーク
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/WID/WI-0004_Late_Joiner_Handling.md` （本ドキュメント）

---

## 12. 備考

### 既知の問題と制限事項

#### 1. OnDeserialization()が発火しない問題（2025年7月時点）
- **詳細**: VRChat_TechResearch_2025.md L477-480参照
- **発生条件**: UdonSynced変数の大量データ時（詳細は不明）
- **進捗**: VRChatサーバ側で修正検討中
- **本WIでの対応**:
  - 1秒の初期化遅延により、OnDeserialization()が発火する確率を向上
  - 将来的にタイムアウト処理を追加する予定（WI-0005で対応）

#### 2. SendCustomEventDelayedSecondsの精度
- **詳細**: 1秒の遅延は厳密ではなく、±0.1秒程度の誤差がある
- **影響**: Late Joiner同期に大きな影響はない（許容範囲）

#### 3. 3人以上の同時Late Joiner
- **詳細**: 同時に複数人がLate Joinerとして参加した場合の挙動は未検証
- **対応**: Phase 1では2人テストのみ実施。Phase 2で10人テストを実施予定

### 技術的な注意点

#### UdonSharpの制約
- `async/await` は使用不可 → `SendCustomEventDelayedSeconds` で代用
- `OnDeserialization()` はoverrideキーワード必須
- CustomEventメソッドは `_` プレフィックス推奨（TDD.md Section 8.2参照）

#### VRChat SDKのバージョン依存
- SDK 3.9.0未満では動作しない可能性
- SDK 3.9.1-beta以降では追加のテストが必要（PlayerObjects対応）

### 次のWork Instructionへの引き継ぎ事項
- **WI-0005 (Bandwidth Monitoring)**: NetworkSyncにバッチング処理を追加
- **WI-0007 (TreeController State)**: NetworkSyncのRequestSerialization()を利用
- **WI-0020 (CooperativeTracker)**: Late Joiner用のタイムスタンプ補正が必要

### トラブルシューティング

#### 問題: UdonSharpコンパイルエラー
```
UdonSharpBehaviourEditor: Build failed
```
**対応**:
1. VCC で UdonSharp 1.1.9 がインストールされているか確認
2. `Assets/UdonSharp` ディレクトリが存在するか確認
3. Unity を再起動

#### 問題: Late Joiner判定が常にfalse
**対応**:
1. `Networking.LocalPlayer` が null でないか確認
2. `VRCPlayerApi.GetPlayerCount()` が正しい値を返すか確認
3. ClientSimではなくVRChat実機でテスト

#### 問題: OnDeserialization()が呼ばれない
**対応**:
1. NetworkSync GameObject に UdonBehaviour コンポーネントがアタッチされているか確認
2. UdonSharp Compiler が最新版か確認
3. VRChat Build & Upload でテスト（ClientSimでは動作しない場合がある）

---

**作業完了チェックリスト確認者**: ________________
**作業完了日**: ________________
**レビュー承認者**: ________________
**レビュー日**: ________________

---

**参照ドキュメント**:
- TDD.md - Module M-02 NetworkSync（Section 5.2）
- VRChat_TechResearch_2025.md - Late Joiner既知の問題（Section 5.5, L477-480）
- TECHNICAL_DETAILS.md - Late Joiner対応パターン（Section H.1, L529-545）
- PRDv3.md - Phase 1仕様書（ネットワーク同期要件）

**関連Work Instructions**:
- WI-0001: Implement GameManager Singleton and Initialization Order（依存）
- WI-0002: Implement NetworkSync Batching System（依存）
- WI-0005: Implement Bandwidth Monitoring（次タスク）
