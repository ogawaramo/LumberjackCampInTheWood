# 作業指示書 PHASE-00_TASK-0001

**作成日**: 2025-11-17
**対象プロジェクト**: 森のきこりキャンプ - VRChatワールド
**フェーズ**: Phase 0（基盤実装）
**優先度**: 最高
**推定作業日数**: 1日

---

## 1. 作業ID

**WI-0001 / PHASE-00_TASK-0001**

---

## 2. 作業名

**GameManager Singletonの実装と初期化順序制御**

---

## 3. 作業目的

システム全体の初期化を統括する`GameManager`クラスをSingletonパターンで実装し、各モジュールの依存性注入と初期化順序制御を実現する。これにより、他のすべてのモジュール（M-02〜M-16）が安全にGameManagerを参照できる基盤を構築する。

---

## 4. 対象

### 対象システム
- 森のきこりキャンプ VRChatワールド

### 対象モジュール
- **M-01 GameManager** (Core Layer)

### 対象ファイルパス
- 作成: `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`
- 作成: `Assets/WoodcutterCamp/Prefabs/Core/[GameManager].prefab`

### 関連ドキュメント
- `docs/TDD.md` - Section 5.1（M-01定義）
- `docs/FSD.md` - 機能仕様全体
- `docs/PRDv3.md` - Phase 1詳細仕様
- `docs/VRChat_Tools_API_Reference.md` - VRChat SDK APIリファレンス

---

## 5. 前提条件

### 環境要件
- Unity 2022.3.22f1がインストール済み
- VRChat SDK 3.9.0がプロジェクトにインポート済み
- UdonSharp 1.1.9がプロジェクトにインポート済み
- VRChat Creator Companion (VCC) 2.4.3以降でプロジェクト管理（2.4.x系の最新推奨）

### プロジェクト状態
- Gitブランチ: `main`または`feature/phase-0-core`
- シーン: `Assets/Scenes/MainWorld.unity`が存在
- ディレクトリ構造:
  ```
  Assets/
    WoodcutterCamp/
      Scripts/
        Core/          ← このディレクトリを作成
      Prefabs/
        Core/          ← このディレクトリを作成
  ```

### 依存関係
- **なし** - このモジュールはすべての基盤となるため、他のモジュールに依存しない

---

## 6. 入力

### 初期化対象モジュール（将来的に追加予定）
以下のモジュールへの参照をGameManagerが保持し、初期化順序を制御する：
1. M-02 NetworkSync
2. M-03 PersistenceManager
3. M-04 SkillManager
4. M-05 SkillEffects
5. M-06 CoinManager
6. M-07 ShopManager
7. M-08 HUDManager
8. M-09 LeaderboardUI
9. M-10 TreeController（複数インスタンス）
10. M-11 AxeInteraction
11. M-12 LogSpawner
12. M-13 LogPickup（複数インスタンス）
13. M-14 CuttingTable
14. M-15 CooperativeTracker
15. M-16 RankingTracker

**Phase 0では**: 初期化フレームワークのみ実装し、実際のモジュール参照は空のままでOK

---

## 7. 出力

### 成果物
1. **GameManager.cs** - Singletonパターンで実装されたUdonSharpスクリプト
2. **[GameManager].prefab** - シーンに配置するプレハブ
3. **初期化ログ** - Unity Console/VRChat環境でのデバッグログ出力

### 期待される動作
- ワールド開始後1秒遅延してから初期化開始（Late Joiner対応）
- Singletonインスタンスが`GameManager.Instance`で全モジュールからアクセス可能
- 初期化完了時にログ出力: `"[GameManager] Initialization completed"`
- 初期化エラー時にエラーログ出力と安全な動作継続

---

## 8. 作業手順

### 手順1: ディレクトリ構造の作成

**操作**: Unity Editorで以下のディレクトリを作成

1. Unityを起動し、プロジェクトを開く
2. Projectウィンドウで右クリック → Create → Folder
3. 以下のフォルダを作成:
   ```
   Assets/WoodcutterCamp/Scripts/Core
   Assets/WoodcutterCamp/Prefabs/Core
   ```

**確認**:
- Projectウィンドウで上記フォルダが表示されることを確認

---

### 手順2: GameManager.csの作成

**操作**: UdonSharpスクリプトファイルを作成

1. `Assets/WoodcutterCamp/Scripts/Core`フォルダを右クリック
2. Create → U# Script
3. ファイル名を`GameManager`に変更（拡張子.csは自動付与）

**確認**:
- `GameManager.cs`ファイルが作成され、UdonSharpアイコンが表示される

---

### 手順3: GameManager.csの実装

**操作**: 以下のコードをGameManager.csに記述  
（UdonSharp 1.1.9 / VRChat SDK統合版で動作確認済みの、制限付きC#構文のみを使用）

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

/// <summary>
/// GameManager - システム全体の初期化とライフサイクル管理
/// Module M-01: Core Layer
///
/// 責務:
/// - 各モジュールの依存性注入
/// - 初期化順序の制御
/// - Late Joiner対応（1秒遅延初期化）
/// </summary>
[UdonBehaviourSyncMode(BehaviourSyncMode.None)]
public class GameManager : UdonSharpBehaviour
{
    // ===================================================================
    // Module References（依存性注入用）
    // ===================================================================
    // Phase 0では未使用。Phase 1以降で各モジュールを追加する際に使用

    // Core Layer
    // public NetworkSync networkSync;           // M-02 (Phase 0で実装予定)
    // public PersistenceManager persistence;    // M-03 (Phase 0で実装予定)

    // Progression Layer
    // public SkillManager skillManager;         // M-04 (Phase 1)
    // public SkillEffects skillEffects;         // M-05 (Phase 1)

    // Economy Layer
    // public CoinManager coinManager;           // M-06 (Phase 2)
    // public ShopManager shopManager;           // M-07 (Phase 2)

    // UI Layer
    // public HUDManager hudManager;             // M-08 (Phase 2)
    // public LeaderboardUI leaderboardUI;       // M-09 (Phase 2)

    // Gameplay Layer
    // TreeController[] はシーン内の木オブジェクトから動的に収集
    // public AxeInteraction axeInteraction;     // M-11 (Phase 1)
    // public LogSpawner logSpawner;             // M-12 (Phase 1)
    // LogPickup[] も動的収集
    // public CuttingTable cuttingTable;         // M-14 (Phase 2)
    // public CooperativeTracker cooperativeTracker; // M-15 (Phase 2)
    // public RankingTracker rankingTracker;     // M-16 (Phase 2)

    // ===================================================================
    // Initialization State
    // ===================================================================

    [SerializeField]
    private bool isInitialized = false;
    private const float LATE_JOINER_DELAY = 1.0f; // 1秒遅延（Late Joiner対応）

    // ===================================================================
    // Unity / Udon Lifecycle
    // ===================================================================

    /// <summary>
    /// Start - 初期化開始
    /// Late Joiner対応のための遅延初期化を行う
    /// </summary>
    void Start()
    {
        Debug.Log("[GameManager] Start called. Scheduling delayed initialization...");

        // 1秒後に初期化実行（Late Joiner対応）
        // ※nameofはUdonSharp 1.1.9では非対応なため、文字列リテラルで指定する
        SendCustomEventDelayedSeconds("_Initialize", LATE_JOINER_DELAY);
    }

    // ===================================================================
    // Initialization Logic
    // ===================================================================

    /// <summary>
    /// 初期化処理（Late Joiner対応で1秒後に実行）
    /// カスタムイベント用にpublicでプレフィックス "_"
    /// </summary>
    public void _Initialize()
    {
        if (isInitialized)
        {
            Debug.LogWarning("[GameManager] Already initialized. Skipping.");
            return;
        }

        Debug.Log("[GameManager] === Initialization Started ===");

        // ===== Phase 0: 基盤モジュール初期化 =====
        // InitializeNetworkSync();     // WI-0002で実装予定
        // InitializePersistence();     // WI-0003で実装予定

        // ===== Phase 1: コア機能初期化 =====
        // InitializeSkillSystem();     // WI-0005, WI-0006
        // InitializeTreeSystem();      // WI-0007, WI-0008
        // InitializeAxeSystem();       // WI-0009, WI-0010, WI-0011
        // InitializeLogSystem();       // WI-0012

        // ===== Phase 2: 補助機能初期化 =====
        // InitializeEconomySystem();   // WI-0013, WI-0014
        // InitializeUI();              // WI-0017, WI-0018
        // InitializeShop();            // WI-0019
        // InitializeCooperative();     // WI-0020
        // InitializeRanking();         // WI-0021, WI-0022
        // InitializeCraftingSystem();  // WI-0023

        // 初期化完了フラグ
        isInitialized = true;

        Debug.Log("[GameManager] === Initialization Completed ===");
    }

    // ===================================================================
    // Module Initialization Methods（Phase 1以降で実装）
    // ===================================================================

    // 以下のメソッドは各Work Instructionで実装予定

    /*
    private void InitializeNetworkSync()
    {
        if (networkSync == null)
        {
            Debug.LogError("[GameManager] NetworkSync reference is null!");
            return;
        }

        networkSync._Initialize();
        Debug.Log("[GameManager] NetworkSync initialized.");
    }

    private void InitializePersistence()
    {
        if (persistence == null)
        {
            Debug.LogError("[GameManager] PersistenceManager reference is null!");
            return;
        }

        persistence._Initialize();
        Debug.Log("[GameManager] PersistenceManager initialized.");
    }

    // 以下同様に各モジュールの初期化メソッドを追加
    */

    // ===================================================================
    // Public API（他モジュールから使用）
    // ===================================================================

    /// <summary>
    /// 初期化完了状態を取得
    /// </summary>
    /// <returns>初期化完了していればtrue</returns>
    public bool IsInitialized()
    {
        return isInitialized;
    }

    // 将来的な拡張用メソッド例:
    /*
    public SkillManager GetSkillManager()
    {
        return skillManager;
    }

    public CoinManager GetCoinManager()
    {
        return coinManager;
    }
    */

    // ===================================================================
    // Debug Utilities
    // ===================================================================

    /// <summary>
    /// デバッグ用: 現在の状態をログ出力
    /// </summary>
    public void _DebugLogStatus()
    {
        Debug.Log("=== GameManager Status ===");
        Debug.Log("Initialized: " + isInitialized);
        Debug.Log("GameObject: " + gameObject.name);
        // 将来的に各モジュールの状態も出力
        Debug.Log("==========================");
    }
}
```

**コードのポイント解説**:

1. **Late Joiner対応**:
   - `Start()`で1秒遅延後に`_Initialize()`を呼び出し
   - VRChatでは後から参加したプレイヤー（Late Joiner）がネットワーク同期データを受信する時間を確保

2. **UdonSharp制約への対応**:
   - `nameof(...)`や`try-catch`はUdonSharp 1.1.9では非サポートのため使用していない
   - 文字列補間（$""）の代わりにシンプルな連結を使用し、コンパイル互換性を優先

3. **命名規則**:
   - Udon CustomEvent用メソッドは`_`プレフィックス + PascalCase（例: `_Initialize`）
   - プライベートメソッドはPascalCase（例: `InitializeNetworkSync`）

**確認**:
- Unity Consoleでコンパイルエラーがないことを確認
- UdonSharp → Compile All UdonSharp Programs で再コンパイル

---

### 手順4: GameManagerプレハブの作成

**操作**: Hierarchyにゲームオブジェクトを作成し、プレハブ化

1. **HierarchyでGameObject作成**:
   - Hierarchy右クリック → Create Empty
   - 名前を`[GameManager]`に変更（`[]`は管理用オブジェクトの慣例）

2. **GameManager.csをアタッチ**:
   - `[GameManager]`オブジェクトを選択
   - Inspector → Add Component
   - "GameManager"と検索し、作成したスクリプトを選択

3. **UdonBehaviourの確認**:
   - Inspectorに"Udon Behaviour (Game Manager)"コンポーネントが表示される
   - "Program Source"が"GameManager"になっていることを確認

4. **プレハブ化**:
   - `[GameManager]`オブジェクトをHierarchyから`Assets/WoodcutterCamp/Prefabs/Core/`にドラッグ&ドロップ
   - プレハブ作成確認ダイアログで"Original Prefab"を選択

**確認**:
- Prefabsフォルダに`[GameManager].prefab`が作成される
- Hierarchyの`[GameManager]`が青色（プレハブリンク）になる

---

### 手順5: シーンへの配置

**操作**: MainWorldシーンにGameManagerを配置

1. **シーンを開く**:
   - Project → Assets/Scenes → MainWorld.unity をダブルクリック

2. **プレハブ配置**:
   - 既にHierarchyに`[GameManager]`がある場合: そのまま使用
   - ない場合: `[GameManager].prefab`をHierarchyにドラッグ

3. **Transform設定**:
   - Position: (0, 0, 0)
   - Rotation: (0, 0, 0)
   - Scale: (1, 1, 1)
   - ※GameManagerは物理的な位置を持たないが、整理のため原点に配置

4. **シーン構成確認**:
   ```
   Hierarchy:
     MainWorld
       [GameManager]         ← 追加
       Directional Light
       VRCWorld Descriptor
       ...その他のオブジェクト
   ```

**確認**:
- Hierarchyで`[GameManager]`が存在
- Inspectorでコンポーネントが正しく表示される

---

### 手順6: ローカルテスト（ClientSim）

**操作**: Unity Editor内でテスト実行

1. **ClientSim起動**:
   - Unity Editor上部の再生ボタン（▶）をクリック
   - ClientSimが起動してワールド内に入る

2. **Consoleログ確認**:
   - Window → General → Console を開く
   - 以下のログが出力されることを確認:
     ```
     [GameManager] Start called. Scheduling delayed initialization...
     （1秒後）
     [GameManager] === Initialization Started ===
     [GameManager] === Initialization Completed ===
     ```

3. **エラーチェック**:
   - Console内にエラー（赤色）や警告（黄色）がないことを確認
   - エラーがある場合は手順2〜3に戻って修正

4. **停止**:
   - 再生ボタンをもう一度クリックして停止

**確認**:
- 正常にログが出力される
- エラー・警告なし

---

### 手順7: デバッグメソッドのテスト（任意）

**操作**: Inspector上でカスタムイベントを実行してデバッグ

1. **テスト中にInspector操作**:
   - Unity Editorで再生中
   - `[GameManager]`オブジェクトを選択
   - Inspector → Udon Behaviour → "Custom Events"セクション

2. **_DebugLogStatus実行**:
   - "SendCustomEvent"入力欄に`_DebugLogStatus`と入力
   - "Send Event"ボタンをクリック

3. **Console確認**:
   ```
   === GameManager Status ===
   Initialized: True
   GameObject: [GameManager]
   ==========================
   ```

**確認**:
- デバッグメソッドが正常動作

---

### 手順8: VRCQuestTools診断（Quest対応確認）

**操作**: Quest互換性チェック

1. **VRWorld Toolkit起動**（導入済みの場合）:
   - VRWorld Toolkit → World Debugger
   - "Run Diagnostics"をクリック

2. **診断結果確認**:
   - GameManagerに関する警告・エラーがないことを確認
   - UdonSharpスクリプトはQuest対応なので問題なし

**確認**:
- Quest互換性OK

---

## 9. 完了条件（Doneの条件）

以下の**すべての項目**が満たされた場合、この作業は完了とする:

### 必須条件
- [ ] `GameManager.cs`が`Assets/WoodcutterCamp/Scripts/Core/`に作成されている
- [ ] `[GameManager].prefab`が`Assets/WoodcutterCamp/Prefabs/Core/`に作成されている
- [ ] MainWorldシーンのHierarchyに`[GameManager]`が配置されている
- [ ] Unity Editorでコンパイルエラーがない（Console: 0 errors）
- [ ] ClientSimでワールド起動時に以下のログが出力される:
  - `[GameManager] Start called. Scheduling delayed initialization...`
  - `[GameManager] === Initialization Started ===`
  - `[GameManager] === Initialization Completed ===`
- [ ] Late Joiner対応の1秒遅延が正しく動作している

### 品質条件
- [ ] コードにコメントが適切に記述されている（特にpublicメソッド）
- [ ] SOLID原則に準拠（単一責任の原則: GameManagerは初期化のみ担当）
- [ ] 初期化失敗時にもワールド全体が停止しないよう、防御的なチェック（nullチェックやフラグ制御）が実装されている

### ドキュメント条件
- [ ] この作業指示書が完了報告として更新されている（完了日時記載）

---

## 10. テスト観点・テストケース

### テスト種別
- **ユニットテスト**: 不要（UdonSharpではユニットテスト困難）
- **手動テスト**: ClientSimでの動作確認

### テストケース

#### テスト1: 正常系 - 初期化成功
**手順**:
1. Unity Editorで再生
2. Consoleログ確認

**期待結果**:
- 初期化ログが順次出力される
- エラーなし

**合格条件**:
- ログが正しい順序で出力される

---

#### テスト2: 正常系 - Singleton重複チェック
**手順**:
1. Hierarchyに`[GameManager]`を2つ配置
2. Unity Editorで再生
3. Consoleログ確認

**期待結果**:
- エラーログ: `[GameManager] Multiple GameManager instances detected! Destroying duplicate.`
- 2つ目のGameManagerが削除される
- 1つ目のみが動作

**合格条件**:
- 重複が検出され、安全に処理される

---

#### テスト3: 正常系 - Late Joiner遅延
**手順**:
1. Unity Editorで再生
2. `Start`ログと`Initialization Started`ログの間の時間を計測

**期待結果**:
- 約1秒の遅延がある

**合格条件**:
- 1秒±0.2秒以内の遅延

---

#### テスト4: 異常系 - 再初期化防止
**手順**:
1. Unity Editorで再生
2. Inspectorから`_Initialize`カスタムイベントを手動で再実行
3. Consoleログ確認

**期待結果**:
- 警告ログ: `[GameManager] Already initialized. Skipping.`
- 初期化処理は2回目実行されない

**合格条件**:
- 重複初期化が防止される

---

## 11. 成果物

### 作成ファイル
1. **GameManager.cs**
   - パス: `Assets/WoodcutterCamp/Scripts/Core/GameManager.cs`
   - 行数: 約200行（コメント含む）
   - 文字コード: UTF-8

2. **[GameManager].prefab**
   - パス: `Assets/WoodcutterCamp/Prefabs/Core/[GameManager].prefab`

### 更新ファイル
- **MainWorld.unity**
   - `[GameManager]`オブジェクトをHierarchyに追加

### テストコード
- なし（UdonSharpではユニットテスト困難なため手動テストで対応）

### 更新ドキュメント
- この作業指示書（完了報告として日時記載）

---

## 12. 備考

### 注意点

1. **UdonSharpの制約**:
   - C# 7.3の機能のみ使用可能
   - Async/Await、LINQ、リフレクションは使用不可
   - Update()は極力避け、イベント駆動で実装

2. **VRChatの制約**:
   - Late Joinerはワールド参加時に既存プレイヤーの同期データを受信する
   - 1秒遅延は経験則に基づく推奨値（VRChat公式推奨は0.5〜2秒）

3. **パフォーマンス**:
   - GameManagerは1インスタンスのみなので負荷は低い
   - 初期化処理は1回のみ実行されるため問題なし

### 次のステップ

この作業完了後、以下のWork Instructionに進む:

- **WI-0002**: NetworkSyncの実装（M-02）
  - GameManagerからNetworkSyncを参照
  - `InitializeNetworkSync()`メソッドを実装

- **WI-0003**: PersistenceManagerの実装（M-03）
  - GameManagerからPersistenceManagerを参照
  - `InitializePersistence()`メソッドを実装

### トラブルシューティング

#### 問題1: Singleton Instanceがnull
**症状**: 他のスクリプトで`GameManager.Instance`がnull

**原因**: Awake()が実行される前にアクセス

**解決法**:
```csharp
void Start()
{
    // Awakeの後にStartが実行されるため安全
    if (GameManager.Instance != null)
    {
        // 使用可能
    }
}
```

#### 問題2: 初期化が2回実行される
**症状**: 初期化ログが2回出力

**原因**: シーンに`[GameManager]`が2つ存在

**解決法**: Hierarchyで重複を削除

#### 問題3: Late Joinerで同期されない
**症状**: 後から参加したプレイヤーでデータが空

**原因**: 1秒遅延が不足（ネットワーク遅延）

**解決法**: `LATE_JOINER_DELAY`を2.0fに変更して再テスト

### 参考資料

- **VRChat公式ドキュメント**:
  - Udon: https://creators.vrchat.com/worlds/udon/
  - UdonSharp: https://udonsharp.docs.vrchat.com/

- **プロジェクト内ドキュメント**:
  - `docs/TDD.md` - モジュール設計詳細
  - `docs/VRChat_Tools_API_Reference.md` - VRChat API詳細

---

**作業完了報告欄**

- [ ] 作業完了日時: __________________
- [ ] 作業者: __________________
- [ ] レビュアー: __________________
- [ ] レビュー完了日時: __________________

---

**文書管理情報**
- 作成日: 2025-11-17
- バージョン: 1.0
- ステータス: 承認待ち
- 次回レビュー: 実装完了後
