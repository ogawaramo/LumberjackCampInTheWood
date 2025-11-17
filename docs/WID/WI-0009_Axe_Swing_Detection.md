# 作業指示書 - WI-0009

**作成日**: 2025-11-17
**ステータス**: 作成完了
**優先度**: 最高
**見積工数**: 1.5日（12時間）

---

## 1. 作業ID

**WI-0009**

---

## 2. 作業名

**Implement Axe Swing Detection - 斧スイング検出とコライダー衝突判定の実装（VR/Desktop両対応）**

---

## 3. 作業目的

VRとデスクトップの両環境で動作する斧スイング検出システムと木のコライダーとの衝突判定を実装する。VRC_Pickupコンポーネントを使用した入力処理、スイングクールダウン、斧の種類別設定（ScriptableObject）、OnTriggerEnterによる木との衝突判定を実装し、伐採システムの基盤を構築する。

---

## 4. 対象

### 対象システム
- **システム名**: 森のきこりキャンプ - Phase 1
- **プラットフォーム**: VRChat（Unity 2022.3.22f1 / SDK 3.9.0 / UdonSharp 1.1.9）
- **対応環境**: VR（Quest/PC VR）およびデスクトップ

### 対象モジュール/パッケージ
- **Module**: M-11 AxeInteraction
- **Layer**: Gameplay Layer
- **Functions**: F-35 (Detect Axe Swing), F-36 (Collision Detection)

### 対象ファイルパス
- **新規作成**: `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/AxeConfig.cs`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/WoodenAxe.asset`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/IronAxe.asset`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/DarkSteelAxe.asset`

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 がインストール済み
- VRChat SDK 3.9.0 がプロジェクトにインポート済み
- UdonSharp 1.1.9 がインポート済み
- **SkillEffects（M-05）の実装が完了していること（WI-0006完了）**

### 初期状態
- `/Assets/WoodcutterCamp/Scripts/Gameplay/` フォルダが作成済み
- `/Assets/WoodcutterCamp/ScriptableObjects/` フォルダが作成済み
- SkillEffects.cs が存在し、`GetDamageMultiplier()` メソッドが使用可能

### ブランチ
- 作業ブランチ: `feature/WI-0009-axe-swing-detection`（`main` からブランチ作成）

---

## 6. 入力

### 入力データ

#### VRモード入力
```csharp
// VRC_Pickup.OnPickupUseDown() イベント
// トリガーボタンを押した時に自動呼び出し
```

#### デスクトップモード入力
```csharp
// Input.GetMouseButtonDown(0) で左クリック検出
// マウス左ボタン押下時
```

### 想定する入力例
- **VR**: プレイヤーがトリガーを引く → OnPickupUseDown() → スイング開始
- **Desktop**: プレイヤーが左クリック → GetMouseButtonDown(0) == true → スイング開始

---

## 7. 出力

### 期待する出力データ

#### 1. スイング実行時
```csharp
// ログ出力
Debug.Log("[AxeController] Swing executed. Cooldown: 0.5s");

// スイングアニメーション開始（0.5秒持続）
// クールダウンタイマー設定（次のスイングまで0.5秒待機）
```

#### 2. 木のコライダーとの衝突時
```csharp
// ログ出力
Debug.Log("[AxeController] Hit tree (ID: 3). Base damage: 10");

// TreeControllerへのダメージ通知
// treeController.TakeDamage(damage, playerAPI);

// ヒットエフェクト再生（パーティクル・サウンド）
```

#### 3. 斧設定の読み込み
```csharp
// ScriptableObjectから斧データ読み込み
// 木の斧: baseDamage=10, attackSpeed=1.0秒
// 鉄の斧: baseDamage=15, attackSpeed=0.9秒
// 黒鋼の斧: baseDamage=20, attackSpeed=0.8秒
```

---

## 8. 作業手順

### ステップ1: Gitブランチの作成

1. ターミナルを開き、プロジェクトルートに移動:
   ```bash
   cd /mnt/d/Dropbox/Dev/CLI/WCF
   ```

2. mainブランチから新しいブランチを作成:
   ```bash
   git checkout main
   git pull origin main
   git checkout -b feature/WI-0009-axe-swing-detection
   ```

3. ブランチが作成されたことを確認:
   ```bash
   git branch
   # * feature/WI-0009-axe-swing-detection と表示される
   ```

### ステップ2: Unityプロジェクトのセットアップ

1. Unityエディタを起動し、WoodcutterCampプロジェクトを開く

2. Projectウィンドウで以下のフォルダ構造を作成:
   ```
   /Assets/WoodcutterCamp/Scripts/Gameplay/
   /Assets/WoodcutterCamp/ScriptableObjects/
   /Assets/WoodcutterCamp/ScriptableObjects/Axes/
   ```

3. Consoleウィンドウを開く（`Window` → `General` → `Console`）
   - エラーがないことを確認

### ステップ3: AxeConfig ScriptableObject の作成

1. `/Assets/WoodcutterCamp/ScriptableObjects/` フォルダ内で右クリック

2. `Create` → `C# Script` を選択

3. ファイル名を `AxeConfig` に設定

4. 作成されたファイルをダブルクリックして Visual Studio Code で開く

5. 以下のコードを実装する（**完全なコード例**）:

```csharp
using UdonSharp;
using UnityEngine;

namespace WoodcutterCamp.Gameplay
{
    /// <summary>
    /// 斧の設定データを保持するScriptableObject
    /// 斧の種類ごとにアセットとして作成する
    /// </summary>
    [CreateAssetMenu(fileName = "NewAxeConfig", menuName = "WoodcutterCamp/AxeConfig")]
    public class AxeConfig : ScriptableObject
    {
        [Header("基本情報")]
        [Tooltip("斧の名前（例: 木の斧、鉄の斧）")]
        public string axeName = "木の斧";

        [Tooltip("斧のID（1=木, 2=鉄, 3=黒鋼）")]
        [Range(1, 3)]
        public int axeID = 1;

        [Header("ダメージ設定")]
        [Tooltip("基礎ダメージ量")]
        [Range(1, 100)]
        public int baseDamage = 10;

        [Tooltip("クリティカル率（0.0〜1.0）")]
        [Range(0f, 1f)]
        public float criticalRate = 0f;

        [Header("攻撃速度設定")]
        [Tooltip("攻撃速度（秒/回）- 小さいほど速い")]
        [Range(0.5f, 2.0f)]
        public float attackSpeed = 1.0f;

        [Header("ビジュアル")]
        [Tooltip("斧のプレハブ（オプション - Phase 2）")]
        public GameObject axePrefab;

        /// <summary>
        /// 設定内容をログ出力（デバッグ用）
        /// </summary>
        public void DebugPrint()
        {
            Debug.Log($"[AxeConfig] {axeName} (ID:{axeID}) - Damage:{baseDamage}, Speed:{attackSpeed}s, Crit:{criticalRate*100}%");
        }
    }
}
```

6. ファイルを保存（Ctrl+S）

7. Unityエディタに戻り、コンパイルを待つ

### ステップ4: 斧のアセット作成

#### 4-1: 木の斧（WoodenAxe.asset）

1. Projectウィンドウで `/Assets/WoodcutterCamp/ScriptableObjects/Axes/` フォルダを開く

2. フォルダ内で右クリック → `Create` → `WoodcutterCamp` → `AxeConfig` を選択

3. 作成されたアセット名を `WoodenAxe` に変更

4. アセットを選択し、Inspectorで以下の値を設定:
   - **Axe Name**: `木の斧`
   - **Axe ID**: `1`
   - **Base Damage**: `10`
   - **Critical Rate**: `0`
   - **Attack Speed**: `1.0`

5. 設定を保存（Ctrl+S）

#### 4-2: 鉄の斧（IronAxe.asset）

1. 同じフォルダ内で右クリック → `Create` → `WoodcutterCamp` → `AxeConfig`

2. アセット名を `IronAxe` に変更

3. Inspectorで以下の値を設定:
   - **Axe Name**: `鉄の斧`
   - **Axe ID**: `2`
   - **Base Damage**: `15`
   - **Critical Rate**: `0`
   - **Attack Speed**: `0.9`

#### 4-3: 黒鋼の斧（DarkSteelAxe.asset）

1. 同じフォルダ内で右クリック → `Create` → `WoodcutterCamp` → `AxeConfig`

2. アセット名を `DarkSteelAxe` に変更

3. Inspectorで以下の値を設定:
   - **Axe Name**: `黒鋼の斧`
   - **Axe ID**: `3`
   - **Base Damage**: `20`
   - **Critical Rate**: `0.1` (10%)
   - **Attack Speed**: `0.8`

### ステップ5: AxeController.cs の実装

1. `/Assets/WoodcutterCamp/Scripts/Gameplay/` フォルダ内で右クリック

2. `Create` → `C# Script` を選択

3. ファイル名を `AxeController` に設定

4. 作成されたファイルをダブルクリックして Visual Studio Code で開く

5. 以下のコードを実装する（**完全なコード例**）:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

namespace WoodcutterCamp.Gameplay
{
    /// <summary>
    /// M-11 AxeController - 斧のスイング検出とコライダー衝突判定
    /// VR/Desktop両対応
    /// </summary>
    [RequireComponent(typeof(VRC_Pickup))]
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class AxeController : UdonSharpBehaviour
    {
        // ========================================
        // インスペクター設定
        // ========================================

        [Header("斧の設定")]
        [Tooltip("斧の設定データ（ScriptableObject）")]
        public AxeConfig axeConfig;

        [Header("コライダー設定")]
        [Tooltip("斧の刃部分のコライダー（Trigger設定必須）")]
        public Collider bladeCollider;

        [Header("デバッグ設定")]
        [Tooltip("デバッグログを出力するか")]
        public bool enableDebugLog = true;

        // ========================================
        // プライベート変数
        // ========================================

        // コンポーネント参照
        private VRC_Pickup pickupComponent;
        private VRCPlayerApi localPlayer;

        // スイング状態管理
        private bool isSwinging = false;
        private float lastSwingTime = 0f;
        private float cooldownDuration = 0.5f; // 基本クールダウン（0.5秒固定）

        // デスクトップモード判定
        private bool isDesktopMode = false;

        // 衝突判定フラグ
        private bool canHit = false;

        // ========================================
        // 初期化
        // ========================================

        void Start()
        {
            // コンポーネント取得
            pickupComponent = (VRC_Pickup)GetComponent(typeof(VRC_Pickup));
            if (pickupComponent == null)
            {
                Debug.LogError("[AxeController] VRC_Pickup component not found!");
                return;
            }

            // ローカルプレイヤー取得
            localPlayer = Networking.LocalPlayer;
            if (localPlayer == null)
            {
                Debug.LogWarning("[AxeController] LocalPlayer not found (Editor mode?)");
            }

            // デスクトップモード判定
            isDesktopMode = !localPlayer.IsUserInVR();

            // 斧設定の検証
            if (axeConfig == null)
            {
                Debug.LogError("[AxeController] AxeConfig is not assigned!");
                return;
            }

            // ブレードコライダー検証
            if (bladeCollider == null)
            {
                Debug.LogError("[AxeController] BladeCollider is not assigned!");
                return;
            }

            // コライダーがTriggerであることを確認
            if (!bladeCollider.isTrigger)
            {
                Debug.LogWarning("[AxeController] BladeCollider is not a Trigger. Setting to Trigger.");
                bladeCollider.isTrigger = true;
            }

            // 初期化ログ
            if (enableDebugLog)
            {
                Debug.Log($"[AxeController] Initialized: {axeConfig.axeName} (Damage:{axeConfig.baseDamage}, Speed:{axeConfig.attackSpeed}s)");
                Debug.Log($"[AxeController] Mode: {(isDesktopMode ? "Desktop" : "VR")}");
            }
        }

        // ========================================
        // F-35: スイング検出（VR/Desktop両対応）
        // ========================================

        /// <summary>
        /// VRモード: トリガー押下時（VRC_Pickup イベント）
        /// </summary>
        public override void OnPickupUseDown()
        {
            // VRモードのみ実行
            if (!isDesktopMode)
            {
                ExecuteSwing();
            }
        }

        /// <summary>
        /// デスクトップモード: Update() で左クリック検出
        /// </summary>
        void Update()
        {
            // デスクトップモードかつ斧を持っている場合のみ
            if (isDesktopMode && pickupComponent != null && pickupComponent.IsHeld)
            {
                // 左クリック検出
                if (Input.GetMouseButtonDown(0))
                {
                    ExecuteSwing();
                }
            }
        }

        /// <summary>
        /// スイング実行処理（VR/Desktop共通）
        /// </summary>
        private void ExecuteSwing()
        {
            // クールダウン中はスキップ
            float currentTime = Time.time;
            if (currentTime - lastSwingTime < cooldownDuration)
            {
                if (enableDebugLog)
                {
                    Debug.Log($"[AxeController] Swing blocked: cooldown ({cooldownDuration - (currentTime - lastSwingTime):F2}s remaining)");
                }
                return;
            }

            // スイング開始
            isSwinging = true;
            lastSwingTime = currentTime;
            canHit = true; // 衝突判定を有効化

            if (enableDebugLog)
            {
                Debug.Log($"[AxeController] Swing executed. Cooldown: {cooldownDuration}s");
            }

            // スイング持続時間後に状態をリセット
            float swingDuration = 0.5f; // VR酔い防止のため0.5秒以上
            SendCustomEventDelayedSeconds(nameof(_EndSwing), swingDuration);
        }

        /// <summary>
        /// スイング終了処理（UdonSharp CustomEvent）
        /// </summary>
        public void _EndSwing()
        {
            isSwinging = false;
            canHit = false; // 衝突判定を無効化

            if (enableDebugLog)
            {
                Debug.Log("[AxeController] Swing ended.");
            }
        }

        // ========================================
        // F-36: コライダー衝突判定
        // ========================================

        /// <summary>
        /// OnTriggerEnter: 斧の刃が木のコライダーに衝突した時
        /// </summary>
        /// <param name="other">衝突したコライダー</param>
        void OnTriggerEnter(Collider other)
        {
            // スイング中かつ衝突判定が有効な時のみ処理
            if (!canHit || !isSwinging)
            {
                return;
            }

            // 木のタグをチェック
            if (other.CompareTag("Tree"))
            {
                if (enableDebugLog)
                {
                    Debug.Log($"[AxeController] Hit tree: {other.gameObject.name}");
                }

                // 衝突判定を一時的に無効化（連続ヒット防止）
                canHit = false;

                // TODO: WI-0010でダメージ計算実装
                // 現状は基礎ダメージのみログ出力
                int damage = axeConfig.baseDamage;
                Debug.Log($"[AxeController] Base damage: {damage}");

                // TODO: TreeControllerへのダメージ通知
                // TreeController treeController = other.GetComponent<TreeController>();
                // if (treeController != null)
                // {
                //     treeController.TakeDamage(damage, localPlayer);
                // }
            }
        }

        // ========================================
        // デバッグ用メソッド
        // ========================================

        /// <summary>
        /// 斧設定をログ出力（デバッグ用）
        /// </summary>
        public void _DebugPrintAxeConfig()
        {
            if (axeConfig != null)
            {
                Debug.Log("========================================");
                Debug.Log("[AxeController] Axe Configuration:");
                Debug.Log($"  Name: {axeConfig.axeName}");
                Debug.Log($"  ID: {axeConfig.axeID}");
                Debug.Log($"  Base Damage: {axeConfig.baseDamage}");
                Debug.Log($"  Critical Rate: {axeConfig.criticalRate * 100}%");
                Debug.Log($"  Attack Speed: {axeConfig.attackSpeed}s");
                Debug.Log("========================================");
            }
            else
            {
                Debug.LogError("[AxeController] AxeConfig is not assigned!");
            }
        }

        /// <summary>
        /// 強制的にスイングを実行（テスト用）
        /// </summary>
        public void _DebugForceSwing()
        {
            Debug.Log("[AxeController] Force swing triggered (Debug)");
            ExecuteSwing();
        }
    }
}
```

6. ファイルを保存（Ctrl+S）

7. Unityエディタに戻り、コンパイルを待つ

8. UdonSharpの手動コンパイルを実行:
   - メニュー → `UdonSharp` → `Compile All UdonSharp Programs`

### ステップ6: Unityシーンへの配置とコライダー設定

#### 6-1: 斧オブジェクトの作成

1. Hierarchyウィンドウで右クリック → `Create Empty` を選択

2. オブジェクト名を `WoodenAxe` に変更

3. Inspectorで `Add Component` をクリック

4. 検索ボックスに `AxeController` と入力し、スクリプトをアタッチ

5. 検索ボックスに `VRC_Pickup` と入力し、コンポーネントをアタッチ

#### 6-2: 斧のビジュアル作成（簡易版）

1. Hierarchyで `WoodenAxe` を右クリック → `3D Object` → `Cube` を選択

2. Cube名を `Handle` に変更

3. Transformを以下のように設定:
   - Position: (0, 0, 0)
   - Rotation: (0, 0, 0)
   - Scale: (0.05, 0.5, 0.05) ← 柄の部分

4. 再度 `WoodenAxe` を右クリック → `3D Object` → `Cube` を選択

5. Cube名を `Blade` に変更

6. Transformを以下のように設定:
   - Position: (0, 0.3, 0)
   - Rotation: (0, 0, 0)
   - Scale: (0.15, 0.1, 0.02) ← 刃の部分

#### 6-3: ブレードコライダーの設定

1. Hierarchyで `Blade` を選択

2. Inspectorで既存のBox Collider コンポーネントを確認

3. Box Collider の設定を変更:
   - **Is Trigger**: ✓（チェックを入れる） ← 重要！
   - Center: (0, 0, 0)
   - Size: (0.15, 0.1, 0.02)

4. `Blade` オブジェクトに **Tag** を設定:
   - Inspector上部の Tag ドロップダウン → `Add Tag...`
   - Tags リストの `+` ボタンをクリック
   - 新しいタグ名: `AxeBlade` を入力
   - 保存後、`Blade` オブジェクトを再度選択
   - Tag ドロップダウンから `AxeBlade` を選択

#### 6-4: AxeControllerの設定

1. Hierarchyで `WoodenAxe` を選択

2. Inspectorで `AxeController` コンポーネントを確認

3. 以下のフィールドを設定:
   - **Axe Config**: Projectウィンドウから `/Assets/WoodcutterCamp/ScriptableObjects/Axes/WoodenAxe.asset` をドラッグ&ドロップ
   - **Blade Collider**: Hierarchyから `Blade` オブジェクトのBox Colliderをドラッグ&ドロップ
   - **Enable Debug Log**: ✓（チェックを入れる）

4. `VRC_Pickup` コンポーネントの設定:
   - **Orientation**: Any
   - **Auto Hold**: Off（トリガー離すとDrop）

#### 6-5: テスト用の木オブジェクト作成

1. Hierarchyで右クリック → `3D Object` → `Cube` を選択

2. Cube名を `TestTree` に変更

3. Transformを以下のように設定:
   - Position: (2, 1, 0) ← 斧から少し離す
   - Scale: (0.5, 2, 0.5) ← 木の幹

4. **Tag** を設定:
   - Inspector上部の Tag ドロップダウン → `Add Tag...`
   - 新しいタグ名: `Tree` を入力
   - 保存後、`TestTree` オブジェクトを再度選択
   - Tag ドロップダウンから `Tree` を選択

5. Box Collider の設定を確認:
   - **Is Trigger**: ✓（チェックを入れる）

### ステップ7: テストの実行

#### テスト方法A: Unity Playモード（デスクトップ）

1. Unityエディタで Playモード に入る（上部の▶ボタン）

2. Sceneビューで斧オブジェクト（WoodenAxe）を見つける

3. 以下の操作でテスト:
   - **Eキー**: 斧をPickup
   - **左クリック**: スイング実行
   - 斧を振って木（TestTree）に当てる

4. Consoleウィンドウで以下のログを確認:
   ```
   [AxeController] Initialized: 木の斧 (Damage:10, Speed:1.0s)
   [AxeController] Mode: Desktop
   [AxeController] Swing executed. Cooldown: 0.5s
   [AxeController] Hit tree: TestTree
   [AxeController] Base damage: 10
   [AxeController] Swing ended.
   ```

5. **クールダウンテスト**: 0.5秒以内に連続クリック
   - 期待結果: 2回目のスイングがブロックされる
   - ログ: `Swing blocked: cooldown (0.3s remaining)` など

6. Playモードを終了（▶ボタンを再度クリック）

#### テスト方法B: VRCClientSimを使用したVRモードテスト

1. VRChat SDK → `ClientSim` → `Enable ClientSim` をクリック

2. ClientSimウィンドウで以下を確認:
   - VR Mode: On
   - Player Position: 斧の近く

3. ClientSim Playモードで実行:
   - VRコントローラーで斧をPickup（Grip）
   - トリガーボタン押下でスイング
   - 木に当てる

4. Consoleウィンドウでログを確認（デスクトップと同様）

#### テスト方法C: デバッグメソッドを使ったテスト

1. Playモード中、Hierarchyで `WoodenAxe` を選択

2. Inspectorの UdonBehaviour セクションで以下を確認:
   - `_DebugPrintAxeConfig` メソッドが表示される
   - メソッド名の右にある `Send Custom Event` ボタンをクリック

3. Console ウィンドウで以下のようなログが出力されることを確認:
   ```
   ========================================
   [AxeController] Axe Configuration:
     Name: 木の斧
     ID: 1
     Base Damage: 10
     Critical Rate: 0%
     Attack Speed: 1.0s
   ========================================
   ```

4. `_DebugForceSwing` メソッドを実行して強制スイング

---

## 9. 完了条件（Doneの条件）

以下のすべての条件を満たすこと:

- [ ] `AxeConfig.cs` ファイルが `/Assets/WoodcutterCamp/ScriptableObjects/` に作成されている
- [ ] `AxeController.cs` ファイルが `/Assets/WoodcutterCamp/Scripts/Gameplay/` に作成されている
- [ ] Unityエディタでコンパイルエラーが0件である
- [ ] UdonSharp 1.1.9 でビルドが成功する
- [ ] 3つの斧アセット（WoodenAxe, IronAxe, DarkSteelAxe）が正しく作成されている
- [ ] VRモードでトリガー入力が正しく検出される
- [ ] デスクトップモードで左クリック入力が正しく検出される
- [ ] スイングクールダウン（0.5秒）が正しく機能する
- [ ] OnTriggerEnter で木のコライダーとの衝突が検出される
- [ ] ブレードコライダーがTriggerとして設定されている
- [ ] 木オブジェクトに `Tree` タグが設定されている
- [ ] デバッグログで斧の設定情報が正しく表示される
- [ ] Playモードでテストケースが全て成功している

---

## 10. テスト観点/テストケース

### テスト種別
- **ユニットテスト（手動）**: Unityエディタ上での動作確認
- **統合テスト**: VR/Desktop両環境での入力検出テスト

### 正常系テストケース

| テストID | テスト内容 | 環境 | 操作 | 期待結果 | 確認方法 |
|---------|----------|------|------|---------|---------|
| TC-001 | デスクトップ左クリック検出 | Desktop | 斧をPickup後、左クリック | スイング実行ログ表示 | Console |
| TC-002 | VRトリガー検出 | VR | 斧をPickup後、トリガー | スイング実行ログ表示 | Console |
| TC-003 | クールダウン正常動作 | Both | 0.5秒以内に2回スイング | 2回目がブロックされる | Console |
| TC-004 | 木との衝突検出 | Both | 斧を振って木に当てる | Hit treeログ表示 | Console |
| TC-005 | 基礎ダメージ取得 | Both | 木に当てる | Base damage: 10 表示 | Console |
| TC-006 | 斧設定の読み込み | Both | Playモード開始 | Initialized ログ表示 | Console |

### 異常系テストケース

| テストID | テスト内容 | 環境 | 操作 | 期待結果 | 確認方法 |
|---------|----------|------|------|---------|---------|
| TC-007 | AxeConfig未設定 | Both | Axe Configを空にしてPlay | Errorログ表示、動作停止 | Console |
| TC-008 | BladeCollider未設定 | Both | Blade Colliderを空にしてPlay | Errorログ表示 | Console |
| TC-009 | BladeColliderがTriggerでない | Both | Is Triggerをオフにする | Warningログ表示、自動修正 | Console |
| TC-010 | 木タグ未設定 | Both | 木のTagを削除して衝突 | Hit検出されない | Console |
| TC-011 | 斧を持っていない状態でクリック | Desktop | Pickupせずに左クリック | スイング実行されない | Console |
| TC-012 | スイング中でない衝突 | Both | スイング終了後に木に触れる | Hit検出されない | Console |

### テスト実施手順

1. Unityエディタでメインシーン `WoodcutterCamp` を開く
2. Playモード に入る
3. 以下の順序でテストを実施:
   - **TC-001〜TC-006**: 正常系テスト
   - **TC-007〜TC-012**: 異常系テスト（設定を変更して実施）
4. Console ウィンドウですべての期待ログが出力されることを確認
5. Playモードを終了

---

## 11. 成果物

### 変更ファイル
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/AxeConfig.cs`
- **新規作成**: `/Assets/WoodcutterCamp/Scripts/Gameplay/AxeController.cs`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/WoodenAxe.asset`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/IronAxe.asset`
- **新規作成**: `/Assets/WoodcutterCamp/ScriptableObjects/Axes/DarkSteelAxe.asset`
- **新規作成（オプション）**: `/Assets/WoodcutterCamp/Prefabs/Gameplay/WoodenAxe.prefab`

### 追加テストコード
- AxeController.cs 内の `_DebugPrintAxeConfig()` メソッド
- AxeController.cs 内の `_DebugForceSwing()` メソッド

### 更新したドキュメント
- なし（本作業指示書のみ）

---

## 12. 備考

### 注意点

#### VR/Desktop両対応の実装
- **VRモード**: `OnPickupUseDown()` でトリガー検出
- **デスクトップモード**: `Update()` で `Input.GetMouseButtonDown(0)` 検出
- **モード判定**: `Networking.LocalPlayer.IsUserInVR()` で自動判定
- **重要**: VRCInputManagerは使用不可（UdonSharp制約のため）

#### VRC_Pickupコンポーネントの設定
- **Orientation**: Any（どの向きでも掴める）
- **AutoHold**: Off（トリガー離すとDrop）
- **Proximity**: 0.5m（掴み判定範囲）

#### コライダーのTrigger設定
- **ブレードコライダー**: 必ず `Is Trigger = true` に設定
- **木のコライダー**: 必ず `Is Trigger = true` に設定
- **タグ設定**: 木は `Tree` タグ、刃は `AxeBlade` タグ（オプション）

#### スイングアニメーション
- **Phase 1では未実装**: ログ出力のみ
- **Phase 2で実装予定**: Animatorを使った斧のスイングアニメーション
- **VR酔い防止**: スイング持続時間は0.5秒以上を推奨

#### UdonSharpの制約
- `Input.GetMouseButtonDown()` は使用可能
- `VRC_Pickup` コンポーネントは必須
- `OnTriggerEnter()` は自動呼び出し（UdonSharpが対応）
- **Update()を最小限に**: デスクトップモードの入力検出のみ使用

### 制約事項
- **スイングクールダウン**: 0.5秒固定（Phase 1）
- **ダメージ計算**: Phase 1では基礎ダメージのみログ出力（WI-0010で実装）
- **エフェクト**: Phase 1では未実装（WI-0011で実装）
- **アニメーション**: Phase 1では未実装

### 依存関係
- **依存先**: SkillEffects（M-05） ← WI-0006で実装済み
- **依存元**: TreeController（M-10） ← WI-0010でダメージ通知を実装予定
- **次の作業**: WI-0010 でダメージ計算とTreeControllerへの通知を実装

### 参考資料
- **TDD.md**: Module M-11定義（Line 485-523）
- **FSD.md**: 伐採システム詳細（Line 21-263）
- **PRD.md**: 斧の仕様（Line 75-98）
- **VRChat SDK Documentation**: https://creators.vrchat.com/worlds/udon/

### トラブルシューティング

#### 問題: スイングが検出されない（VRモード）
**原因**: VRC_Pickupコンポーネントが正しく設定されていない
**対処法**:
- VRC_Pickupコンポーネントが斧オブジェクトにアタッチされているか確認
- AutoHold が Off になっているか確認
- ClientSimでVRモードが有効になっているか確認

#### 問題: スイングが検出されない（デスクトップモード）
**原因**: Update()が呼ばれていない、またはPickup状態が正しくない
**対処法**:
- pickupComponent.IsHeld が true になっているか確認
- enableDebugLog を true にしてログを確認
- _DebugForceSwing() メソッドで強制実行してみる

#### 問題: 木との衝突が検出されない
**原因**: コライダーのTrigger設定が間違っている
**対処法**:
- ブレードコライダーの Is Trigger が true になっているか確認
- 木のコライダーの Is Trigger が true になっているか確認
- 木のTagが `Tree` になっているか確認
- OnTriggerEnter() のログが出ているか確認

#### 問題: クールダウンが機能しない
**原因**: lastSwingTime が正しく更新されていない
**対処法**:
- ExecuteSwing() 内で lastSwingTime = Time.time が実行されているか確認
- デバッグログで cooldown remaining の値を確認

#### 問題: AxeConfigが読み込まれない
**原因**: ScriptableObjectが正しく設定されていない
**対処法**:
- Inspectorで Axe Config フィールドが空でないか確認
- WoodenAxe.asset が正しく作成されているか確認
- _DebugPrintAxeConfig() で設定内容を確認

---

## 13. 実装完了後の次ステップ

### 1. Gitコミット

```bash
git add .
git commit -m "feat: Implement Axe Swing Detection and Collision (WI-0009)

- Add AxeConfig ScriptableObject for axe settings
- Create 3 axe assets (Wooden, Iron, DarkSteel)
- Implement AxeController with VR/Desktop input support
- Add swing cooldown system (0.5s)
- Implement OnTriggerEnter collision detection with trees
- Add debug methods for testing
- VR酔い防止: Swing duration 0.5s+

Refs: TDD.md M-11, FSD.md F-35/F-36, PRD.md Section 2.1.2"
```

### 2. プルリクエスト作成

- ブランチ `feature/WI-0009-axe-swing-detection` から `main` へのPRを作成
- **PRタイトル**: `[WI-0009] Implement Axe Swing Detection (VR/Desktop Both)`
- **PR説明**:
  ```markdown
  ## 概要
  VR/Desktop両対応の斧スイング検出システムと木のコライダー衝突判定を実装

  ## 変更内容
  - AxeConfig ScriptableObject 新規作成
  - 3種類の斧アセット作成（木/鉄/黒鋼）
  - AxeController.cs 実装（F-35, F-36）
  - VR: OnPickupUseDown() でトリガー検出
  - Desktop: Input.GetMouseButtonDown(0) で左クリック検出
  - スイングクールダウン: 0.5秒
  - OnTriggerEnter: 木との衝突検出

  ## テスト結果
  - VRモードテスト: ClientSimで動作確認済み
  - デスクトップモードテスト: Playモードで動作確認済み
  - クールダウンテスト: 0.5秒以内の連続スイングがブロックされることを確認
  - 衝突検出テスト: 木との衝突が正しく検出されることを確認

  ## スクリーンショット
  （Consoleのデバッグログ出力を添付）

  ## チェックリスト
  - [x] コンパイルエラー0件
  - [x] UdonSharpビルド成功
  - [x] VR/Desktop両環境でテスト実施済み
  - [x] XMLコメント記載済み
  - [x] デバッグメソッド実装済み
  ```
- レビュアーにコードレビューを依頼

### 3. 次の作業

- **WI-0010**: Damage Calculation System実装（M-11 F-37）
  - **SkillEffectsを呼び出してダメージ計算を行う**
  - 最終ダメージ = 基礎ダメージ × skillEffects.GetDamageMultiplier(level) × 斧倍率
  - クリティカル判定実装
  - TreeControllerへのダメージ通知
  - XP・コイン付与処理

---

**作成者**: VRChat World Development Team
**最終更新**: 2025-11-17
**関連Work Instructions**: WI-0006（SkillEffects - 前提条件）, WI-0010（Damage Calculation - 次作業）
**バージョン**: 1.0
