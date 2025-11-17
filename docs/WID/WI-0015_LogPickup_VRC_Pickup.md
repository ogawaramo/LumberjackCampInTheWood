### 作業指示書（Work Instruction）

**1. 作業ID**
WI-0015

**2. 作業名**
LogPickup VRC_Pickup System - 丸太Pickup/Drop処理と運搬距離計測機能の実装

**3. 作業目的**
VRChat標準のVRC_Pickupコンポーネントを活用し、丸太のPickup/Drop処理と運搬距離のリアルタイム計測機能を実装する。これにより、FNC-002（運搬システム）の中核機能を提供し、プレイヤーが伐採した丸太を運搬してコインを獲得できるようにする。

**4. 対象**

* **対象システム**: 森のきこりキャンプ - Phase 1
* **対象モジュール**: M-13 LogPickup（Gameplay Layer）
* **対象ファイルパス**:
  * 新規作成: `Assets/WoodcutterCamp/Scripts/Gameplay/LogPickup.cs`
  * 編集: `Assets/WoodcutterCamp/Prefabs/LogPrefab.prefab`（VRC_Pickupコンポーネント設定）

**5. 前提条件**

* **環境**:
  * Unity 2022.3.22f1 LTS
  * VRChat SDK 3.9.0
  * UdonSharp 1.1.9
* **依存する作業指示書**:
  * **WI-0013（CoinManager）が完了していること** - コイン加算処理が必要
  * **WI-0012（LogSpawner）が完了していること** - 丸太Prefabが既に存在
* **ブランチ**: `feature/wi-0015-log-pickup`
* **初期状態**: LogPrefabが存在し、基本的なRigidbodyとColliderが設定されている

**6. 入力**

* **プレイヤーのPickup/Drop操作**:
  * VR環境: グリップボタン（押す=Pickup、離す=Drop）
  * デスクトップ環境: Eキー（1回目=Pickup、2回目=Drop）
* **丸太の配置位置**: Vector3（伐採地点または前回Dropされた位置）
* **プレイヤーの現在位置**: Vector3（Pickup時とDrop時の座標）
* **納品エリア判定**: BoxCollider（キャンプ中央の納品ゾーン 5m × 5m × 3m）

**7. 出力**

* **運搬距離データ**: float（メートル単位、0〜100m）
* **HUD表示更新**:
  * Pickup時: 「丸太運搬中 (0m)」アイコン表示
  * 運搬中: 「丸太運搬中 (23m)」距離リアルタイム更新
  * Drop時: アイコン非表示
* **移動速度変更**:
  * Pickup時: baseSpeed 4.0m/s → 3.4m/s（15%減速）
  * Drop時: baseSpeed 4.0m/s に復帰
* **コイン付与**:
  * 納品エリア内Drop時: CoinManager.AddCoins(calculatedCoins) 呼び出し
  * コイン計算式: `min(floor(運搬距離 / 10) × 3, 30)`

**8. 作業手順**

### 8.1. LogPickup.cs スクリプト作成

#### 8.1.1. ファイル作成とUdonSharpBehaviour継承

**作業内容**:
1. Unityエディタを起動し、Projectビューで `Assets/WoodcutterCamp/Scripts/Gameplay/` フォルダに移動
2. 右クリック → Create → C# Script を選択
3. ファイル名を `LogPickup` に設定
4. ファイルを開き、以下のコードを記述:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

namespace WoodcutterCamp.Gameplay
{
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)] // Pickupはローカル処理のみ
    public class LogPickup : UdonSharpBehaviour
    {
        // === 外部参照 ===
        [Header("External References")]
        [Tooltip("CoinManagerへの参照（Inspectorで手動設定）")]
        public CoinManager coinManager;

        [Tooltip("HUDManagerへの参照（Inspectorで手動設定）")]
        public HUDManager hudManager;

        [Tooltip("納品ゾーンのBoxCollider（Inspectorで手動設定）")]
        public BoxCollider deliveryZone;

        // === VRC_Pickup参照 ===
        private VRC_Pickup pickup;

        // === 運搬データ ===
        private Vector3 pickupStartPosition;
        private bool isCarrying = false;
        private float currentDistance = 0f;

        // === 移動速度制御 ===
        private const float BASE_WALK_SPEED = 4.0f;
        private const float CARRYING_WALK_SPEED = 3.4f; // 15%減速

        // === 距離計測更新間隔 ===
        private const float UPDATE_INTERVAL = 0.5f; // 0.5秒ごとに距離更新
        private float nextUpdateTime = 0f;

        // === コイン計算設定 ===
        private const float DISTANCE_PER_COIN = 10f; // 10mごとに3コイン
        private const int COINS_PER_INTERVAL = 3;
        private const int MAX_COINS = 30; // 最大30コイン（100m相当）

        // === デバッグログフラグ ===
        private const bool DEBUG_LOG_ENABLED = true;

        void Start()
        {
            // VRC_Pickupコンポーネント取得
            pickup = (VRC_Pickup)GetComponent(typeof(VRC_Pickup));

            if (pickup == null)
            {
                Debug.LogError("[LogPickup] VRC_Pickup component not found on this GameObject");
                return;
            }

            // 外部参照チェック
            if (coinManager == null)
            {
                Debug.LogError("[LogPickup] CoinManager reference is not set in Inspector");
            }

            if (hudManager == null)
            {
                Debug.LogWarning("[LogPickup] HUDManager reference is not set in Inspector");
            }

            if (deliveryZone == null)
            {
                Debug.LogError("[LogPickup] DeliveryZone BoxCollider is not set in Inspector");
            }

            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log("[LogPickup] Initialization completed for: " + gameObject.name);
            }
        }

        /// <summary>
        /// VRC_Pickupイベント: プレイヤーが丸太をPickupしたとき
        /// </summary>
        public override void OnPickup()
        {
            // Pickup開始位置を記録
            pickupStartPosition = transform.position;
            isCarrying = true;
            currentDistance = 0f;
            nextUpdateTime = Time.time + UPDATE_INTERVAL;

            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log($"[LogPickup] OnPickup - Start Position: {pickupStartPosition}");
            }

            // HUD更新: 運搬中アイコン表示
            if (hudManager != null)
            {
                hudManager._ShowCarryingIcon(true, currentDistance);
            }

            // 移動速度を減速（ローカルプレイヤーのみ）
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer != null)
            {
                localPlayer.SetWalkSpeed(CARRYING_WALK_SPEED);
                localPlayer.SetRunSpeed(CARRYING_WALK_SPEED); // 走り速度も同様に減速

                if (DEBUG_LOG_ENABLED)
                {
                    Debug.Log($"[LogPickup] Walk speed reduced to {CARRYING_WALK_SPEED} m/s");
                }
            }
        }

        /// <summary>
        /// 毎フレーム実行: 運搬距離をリアルタイム計測
        /// </summary>
        void Update()
        {
            if (!isCarrying)
                return;

            // 0.5秒ごとに距離更新（CPU負荷軽減）
            if (Time.time >= nextUpdateTime)
            {
                CalculateCarryingDistance();
                nextUpdateTime = Time.time + UPDATE_INTERVAL;
            }
        }

        /// <summary>
        /// 運搬距離の計算とHUD更新
        /// </summary>
        private void CalculateCarryingDistance()
        {
            currentDistance = Vector3.Distance(pickupStartPosition, transform.position);

            // HUD更新
            if (hudManager != null)
            {
                hudManager._UpdateCarryingDistance(currentDistance);
            }

            if (DEBUG_LOG_ENABLED && currentDistance > 0f)
            {
                Debug.Log($"[LogPickup] Carrying Distance: {currentDistance:F1}m");
            }
        }

        /// <summary>
        /// VRC_Pickupイベント: プレイヤーが丸太をDropしたとき
        /// </summary>
        public override void OnDrop()
        {
            // 最終距離を計算
            CalculateCarryingDistance();

            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log($"[LogPickup] OnDrop - Final Distance: {currentDistance:F1}m");
            }

            // 納品ゾーン判定
            bool isInDeliveryZone = CheckDeliveryZone();

            if (isInDeliveryZone)
            {
                // コイン報酬計算
                int earnedCoins = CalculateCoinReward(currentDistance);

                // CoinManagerにコイン加算
                if (coinManager != null && earnedCoins > 0)
                {
                    coinManager._AddCoins(earnedCoins);

                    if (DEBUG_LOG_ENABLED)
                    {
                        Debug.Log($"[LogPickup] Delivered! Earned {earnedCoins} coins for {currentDistance:F1}m");
                    }
                }

                // 納品成功エフェクト（HUDManager経由）
                if (hudManager != null)
                {
                    hudManager._ShowDeliverySuccess(earnedCoins);
                }

                // 丸太を消滅させる（1秒後に吸い込まれるアニメーション）
                SendCustomEventDelayedSeconds(nameof(_DestroyLog), 1.0f);
            }
            else
            {
                // 納品エリア外でDrop
                if (hudManager != null)
                {
                    hudManager._ShowMessage("納品エリアに運んでください", 2.0f);
                }

                if (DEBUG_LOG_ENABLED)
                {
                    Debug.Log("[LogPickup] Dropped outside delivery zone");
                }
            }

            // 運搬状態をリセット
            isCarrying = false;
            currentDistance = 0f;

            // HUD更新: 運搬中アイコン非表示
            if (hudManager != null)
            {
                hudManager._ShowCarryingIcon(false, 0f);
            }

            // 移動速度を元に戻す
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer != null)
            {
                localPlayer.SetWalkSpeed(BASE_WALK_SPEED);
                localPlayer.SetRunSpeed(BASE_WALK_SPEED);

                if (DEBUG_LOG_ENABLED)
                {
                    Debug.Log($"[LogPickup] Walk speed restored to {BASE_WALK_SPEED} m/s");
                }
            }
        }

        /// <summary>
        /// 納品ゾーン内判定
        /// </summary>
        private bool CheckDeliveryZone()
        {
            if (deliveryZone == null)
            {
                Debug.LogError("[LogPickup] DeliveryZone reference is null");
                return false;
            }

            // 丸太の位置がBoxCollider内にあるかチェック
            Vector3 logPosition = transform.position;
            Bounds zoneBounds = deliveryZone.bounds;

            bool isInside = zoneBounds.Contains(logPosition);

            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log($"[LogPickup] CheckDeliveryZone - Log Position: {logPosition}, Zone Center: {zoneBounds.center}, Inside: {isInside}");
            }

            return isInside;
        }

        /// <summary>
        /// コイン報酬の計算
        /// </summary>
        private int CalculateCoinReward(float distance)
        {
            // 計算式: min(floor(距離 / 10) × 3, 30)
            int coins = Mathf.FloorToInt(distance / DISTANCE_PER_COIN) * COINS_PER_INTERVAL;
            coins = Mathf.Clamp(coins, 0, MAX_COINS);

            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log($"[LogPickup] CalculateCoinReward - Distance: {distance:F1}m → Coins: {coins}");
            }

            return coins;
        }

        /// <summary>
        /// 丸太を消滅させる（Udon CustomEvent）
        /// </summary>
        public void _DestroyLog()
        {
            if (DEBUG_LOG_ENABLED)
            {
                Debug.Log("[LogPickup] Log destroyed after delivery");
            }

            // オブジェクトプールに戻す（将来的な拡張）
            // 現在はシンプルにGameObjectを非アクティブ化
            gameObject.SetActive(false);
        }
    }
}
```

**注意点**:
- **UdonSharpの命名規約**: パブリックメソッドでUdon CustomEventとして呼び出すものは `_` プレフィックスを付ける（例: `_DestroyLog()`）
- **ネットワーク同期**: LogPickupは `BehaviourSyncMode.None` でローカル処理のみ（運搬距離は各プレイヤーのクライアントで独立計算）
- **パフォーマンス**: 距離計測はUpdate()で毎フレーム実行せず、0.5秒間隔で更新（CPU負荷軽減）

---

### 8.2. LogPrefabへのVRC_Pickupコンポーネント設定

#### 8.2.1. Prefabを開く

**作業内容**:
1. Projectビューで `Assets/WoodcutterCamp/Prefabs/LogPrefab.prefab` を探す
2. ダブルクリックしてPrefab編集モードで開く

#### 8.2.2. VRC_Pickupコンポーネント追加

**作業内容**:
1. HierarchyでLogPrefabのルートオブジェクトを選択
2. Inspectorビューで `Add Component` ボタンをクリック
3. 検索ボックスに `VRC Pickup` と入力
4. `VRC_Pickup` コンポーネントを選択して追加

#### 8.2.3. VRC_Pickup設定

**作業内容**:
VRC_Pickupコンポーネントの各パラメータを以下のように設定する：

| パラメータ名 | 設定値 | 理由 |
|------------|-------|------|
| **Disallow Theft** | ☐ Off | 他プレイヤーが持っている丸太を奪取可能（協力プレイ促進） |
| **Exact Gun** | None | 丸太はExact Gunなし（直接手で掴む） |
| **Exact Placement** | None | 特定位置への配置制限なし |
| **Allow Manipulation When Equipped** | ☑ On | 持っている間も位置調整可能 |
| **Orientation** | Any | どの向きでも掴める（操作性向上） |
| **Auto Hold** | Desktop Auto Hold | デスクトップ: 自動保持、VR: トリガー離すとDrop |
| **Use Text** | "丸太を掴む" | Pickup時のプロンプト表示 |
| **Proximity** | 0.5 | Pickup判定範囲 0.5メートル |
| **Throw Velocity Boost Min/Max Speed** | 0 / 0 | 投げ機能は無効化（丸太は投げない） |

**設定手順（具体的操作）**:
1. **Disallow Theft**: チェックボックスを**OFF（チェックなし）**にする
2. **Exact Gun**: ドロップダウンを `None` のままにする
3. **Exact Placement**: ドロップダウンを `None` のままにする
4. **Allow Manipulation When Equipped**: チェックボックスを**ON（チェックあり）**にする
5. **Orientation**: ドロップダウンを `Any` に設定
6. **Auto Hold**: ドロップダウンを `Desktop Auto Hold` に設定
7. **Use Text**: テキストフィールドに `丸太を掴む` と入力
8. **Proximity**: スライダーまたは数値入力で `0.5` に設定
9. **Throw Velocity Boost Min Speed**: `0` に設定
10. **Throw Velocity Boost Max Speed**: `0` に設定

#### 8.2.4. LogPickupスクリプトアタッチ

**作業内容**:
1. Inspectorビューで `Add Component` ボタンをクリック
2. 検索ボックスに `LogPickup` と入力
3. 作成した `LogPickup` スクリプトを選択して追加

#### 8.2.5. LogPickup Inspector設定

**作業内容**:
LogPickupコンポーネントのInspectorで以下の参照を設定する：

| フィールド名 | 設定方法 | 設定値 |
|-----------|--------|-------|
| **Coin Manager** | ドラッグ&ドロップ | Hierarchyの `GameManager` オブジェクト内の `CoinManager` コンポーネント |
| **Hud Manager** | ドラッグ&ドロップ | Hierarchyの `UI/HUD` オブジェクト内の `HUDManager` コンポーネント |
| **Delivery Zone** | ドラッグ&ドロップ | Hierarchyの `Camp/DeliveryZone` オブジェクトの `BoxCollider` コンポーネント |

**具体的な手順**:
1. **Coin Manager**:
   - Hierarchyビューで `GameManager` オブジェクトを探す
   - `GameManager` を展開し、`CoinManager` コンポーネントを見つける
   - `CoinManager` をクリックして、InspectorのLogPickupの **Coin Manager** フィールドにドラッグ&ドロップ
2. **Hud Manager**:
   - Hierarchyビューで `UI/HUD` オブジェクトを探す
   - `HUDManager` コンポーネントを見つける
   - `HUDManager` をInspectorのLogPickupの **Hud Manager** フィールドにドラッグ&ドロップ
3. **Delivery Zone**:
   - Hierarchyビューで `Camp/DeliveryZone` オブジェクトを探す
   - `DeliveryZone` の `BoxCollider` コンポーネントを見つける
   - `BoxCollider` をInspectorのLogPickupの **Delivery Zone** フィールドにドラッグ&ドロップ

#### 8.2.6. Prefab保存

**作業内容**:
1. InspectorまたはHierarchyの上部にある **Save** ボタンをクリック
2. Prefab編集モードを終了（Hierarchyの左上の `<` ボタンをクリック）

---

### 8.3. 納品ゾーン（DeliveryZone）の設定

#### 8.3.1. DeliveryZone GameObject作成

**作業内容**:
1. Hierarchyビューで `Camp` オブジェクトを右クリック
2. `3D Object > Cube` を選択
3. 作成されたCubeオブジェクトの名前を `DeliveryZone` に変更

#### 8.3.2. Transform設定

**作業内容**:
Inspectorで以下のTransformを設定:

| プロパティ | 設定値 | 備考 |
|----------|-------|------|
| **Position** | (0, 0.5, 0) | キャンプ中央の地面より0.5m上 |
| **Rotation** | (0, 0, 0) | 回転なし |
| **Scale** | (5, 3, 5) | 5m × 3m × 5m のボックス |

#### 8.3.3. MeshRendererを非表示化

**作業内容**:
1. Inspectorで `Mesh Renderer` コンポーネントを見つける
2. コンポーネント左側のチェックボックスを**OFF（チェックなし）**にする
3. これにより、納品ゾーンは視覚的に見えなくなるが、BoxColliderは機能する

#### 8.3.4. BoxCollider設定

**作業内容**:
1. Inspectorで `Box Collider` コンポーネントを確認
2. **Is Trigger** チェックボックスを**ON（チェックあり）**にする
   - **理由**: トリガーにすることで、丸太がゾーン内に入っても物理衝突しない（位置判定のみ）

#### 8.3.5. 納品ゾーンの視覚的フィードバック（オプション）

**作業内容**（推奨だが必須ではない）:
1. `DeliveryZone` オブジェクトに子オブジェクトとして `3D Object > Plane` を追加
2. Plane の Transform:
   - Position: (0, -1.4, 0)（ゾーンの床面）
   - Rotation: (0, 0, 0)
   - Scale: (0.5, 1, 0.5)（5m × 5mのゾーン境界を示す）
3. Plane の Material:
   - 新しいMaterialを作成（`DeliveryZoneMaterial`）
   - Shader: `Unlit/Transparent`
   - Base Color: 緑色（RGB: 0, 255, 0, Alpha: 100）
   - これにより、半透明の緑色の床が表示され、プレイヤーに納品エリアが分かりやすくなる

---

### 8.4. HUDManagerの拡張（運搬UI表示メソッド追加）

#### 8.4.1. HUDManager.cs に運搬UI表示メソッド追加

**作業内容**:
`Assets/WoodcutterCamp/Scripts/UI/HUDManager.cs` ファイルを開き、以下のメソッドを追加:

```csharp
// === 運搬UI表示 ===
[Header("Carrying UI")]
[Tooltip("運搬中アイコン（Imageコンポーネント）")]
public UnityEngine.UI.Image carryingIcon;

[Tooltip("運搬距離テキスト（TextMeshProUGUI）")]
public TMPro.TextMeshProUGUI carryingDistanceText;

/// <summary>
/// 運搬中アイコンの表示/非表示
/// </summary>
public void _ShowCarryingIcon(bool show, float distance)
{
    if (carryingIcon != null)
    {
        carryingIcon.gameObject.SetActive(show);
    }

    if (show && carryingDistanceText != null)
    {
        carryingDistanceText.text = $"丸太運搬中 ({distance:F0}m)";
    }
}

/// <summary>
/// 運搬距離のリアルタイム更新
/// </summary>
public void _UpdateCarryingDistance(float distance)
{
    if (carryingDistanceText != null)
    {
        carryingDistanceText.text = $"丸太運搬中 ({distance:F0}m)";
    }
}

/// <summary>
/// 納品成功エフェクト表示
/// </summary>
public void _ShowDeliverySuccess(int earnedCoins)
{
    // 画面中央上部に「+15コイン」通知を1秒間表示
    _ShowNotification($"+{earnedCoins}コイン", 1.0f);

    // 金色のパーティクルエフェクト再生（オプション）
    // TODO: ParticleSystemの再生処理を追加
}

/// <summary>
/// メッセージ表示（汎用）
/// </summary>
public void _ShowMessage(string message, float duration)
{
    // 既存の通知システムを利用
    _ShowNotification(message, duration);
}
```

#### 8.4.2. HUDのUI要素作成

**作業内容**:
1. Hierarchyビューで `UI/HUD` Canvas を選択
2. 右クリック → `UI > Image` を選択し、名前を `CarryingIcon` に変更
3. **RectTransform設定**:
   - Anchor: Right-Bottom（右下）
   - Anchor Presets: クリックして Alt+Shift を押しながら右下のプリセットを選択
   - Pos X: -100
   - Pos Y: 100
   - Width: 64
   - Height: 64
4. **Image設定**:
   - Source Image: 丸太のアイコン画像（または一時的にUnityのデフォルトSprite）
   - Color: 白色（255, 255, 255, 255）
5. `CarryingIcon` を子オブジェクトとして `UI > Text - TextMeshPro` を追加し、名前を `DistanceText` に変更
6. **DistanceText RectTransform設定**:
   - Anchor: Bottom-Center
   - Pos X: 0
   - Pos Y: -30
   - Width: 150
   - Height: 30
7. **TextMeshProUGUI設定**:
   - Font: デフォルトフォントまたは日本語フォント
   - Font Size: 18
   - Alignment: Center
   - Text: "丸太運搬中 (0m)"

#### 8.4.3. HUDManagerのInspector設定

**作業内容**:
1. Hierarchyで `UI/HUD` の `HUDManager` コンポーネントを選択
2. Inspectorで以下のフィールドに参照を設定:
   - **Carrying Icon**: Hierarchyの `CarryingIcon` Imageをドラッグ&ドロップ
   - **Carrying Distance Text**: Hierarchyの `DistanceText` TextMeshProUGUIをドラッグ&ドロップ

---

### 8.5. テストシーンの準備

#### 8.5.1. テストシーンにLogPrefab配置

**作業内容**:
1. Hierarchyビューで `Forest` エリアを選択
2. Projectビューから `LogPrefab.prefab` をドラッグして、地面に配置
3. Position: (10, 0.5, 15)（キャンプから約15m離れた位置）

#### 8.5.2. ClientSimでのテスト準備

**作業内容**:
1. メニューバーから `VRChat SDK > Utilities > ClientSim Settings` を開く
2. **Enable ClientSim** が ON になっていることを確認
3. Unity Playモードで再生

---

### 8.6. ビルドとテスト

#### 8.6.1. UdonSharpコンパイル確認

**作業内容**:
1. メニューバーから `UdonSharp > Compile All UdonSharp Programs` を実行
2. Consoleにエラーが表示されないことを確認
3. エラーがある場合:
   - エラーメッセージを読み、該当行を修正
   - 再度コンパイル実行

#### 8.6.2. Playモードテスト

**作業内容**:
1. Unity Playモードを開始（エディタ上部の▶ボタン）
2. **Pickup機能テスト**:
   - WASDキーで移動し、丸太に近づく（0.5m以内）
   - 画面中央に "丸太を掴む [E]" プロンプトが表示されることを確認
   - Eキーを押し、丸太がカメラ前に固定されることを確認
   - HUD右下に「丸太運搬中 (0m)」アイコンが表示されることを確認
3. **距離計測テスト**:
   - 丸太を持ったまま移動
   - HUDの距離表示が0.5秒ごとに更新されることを確認
   - Consoleに `[LogPickup] Carrying Distance: XX.Xm` ログが表示されることを確認
4. **Drop機能テスト**:
   - 納品ゾーン外（キャンプから離れた場所）でEキーを押す
   - 丸太がその場に落下することを確認
   - HUDに「納品エリアに運んでください」メッセージが表示されることを確認
   - 運搬中アイコンが非表示になることを確認
5. **納品機能テスト**:
   - 丸太を再度Pickup
   - キャンプ中央の納品ゾーン（DeliveryZone）に移動
   - Eキーを押す
   - 1秒後に丸太が消滅することを確認
   - HUD中央上部に「+XXコイン」通知が表示されることを確認
   - Consoleに `[LogPickup] Delivered! Earned XX coins for XX.Xm` ログが表示されることを確認
6. **移動速度変更テスト**:
   - Consoleログで `[LogPickup] Walk speed reduced to 3.4 m/s` と `[LogPickup] Walk speed restored to 4.0 m/s` が表示されることを確認
   - 実際の移動速度が変化していることを体感（丸太持ち時は遅い）

#### 8.6.3. コイン計算ロジックテスト

**作業内容**:
以下の距離でテストし、正しいコイン数が付与されることを確認:

| テストケース | 運搬距離 | 期待コイン数 | 計算式 |
|------------|---------|------------|--------|
| TC-01 | 5m | 0コイン | floor(5/10) × 3 = 0 |
| TC-02 | 10m | 3コイン | floor(10/10) × 3 = 3 |
| TC-03 | 25m | 6コイン | floor(25/10) × 3 = 6 |
| TC-04 | 50m | 15コイン | floor(50/10) × 3 = 15 |
| TC-05 | 100m | 30コイン | floor(100/10) × 3 = 30 |
| TC-06 | 150m | 30コイン（上限） | floor(150/10) × 3 = 45 → clamp(30) |

**テスト手順**:
1. Sceneビューで丸太を各距離に配置
2. 納品ゾーンまで運搬してDrop
3. Consoleログで付与されたコイン数を確認
4. 期待値と一致することを確認

---

## **9. 完了条件（Doneの条件）**

### 9.1. 実装完了条件

- ✅ `LogPickup.cs` スクリプトが作成され、すべてのメソッドが実装されている
- ✅ LogPrefabに `VRC_Pickup` コンポーネントが正しく設定されている
- ✅ LogPrefabに `LogPickup` スクリプトがアタッチされ、Inspector参照がすべて設定されている
- ✅ 納品ゾーン（DeliveryZone）が作成され、BoxColliderが設定されている
- ✅ HUDManagerに運搬UI表示メソッドが追加されている
- ✅ HUDに運搬中アイコンと距離テキストが配置されている

### 9.2. 機能動作確認

- ✅ プレイヤーが丸太に近づくと「丸太を掴む [E]」プロンプトが表示される
- ✅ Eキー（VRではグリップボタン）で丸太をPickupできる
- ✅ Pickup時にHUD右下に「丸太運搬中 (0m)」アイコンが表示される
- ✅ 運搬中、移動速度が4.0m/s → 3.4m/sに減速される
- ✅ 運搬中、距離表示が0.5秒ごとに更新される
- ✅ 納品エリア外でDropすると、丸太が地面に落下し、「納品エリアに運んでください」メッセージが表示される
- ✅ 納品エリア内でDropすると、運搬距離に応じたコインが付与される
- ✅ 納品成功時、画面中央上部に「+XXコイン」通知が表示される
- ✅ 納品成功時、丸太が1秒後に消滅する
- ✅ Drop後、移動速度が4.0m/sに戻る
- ✅ Drop後、運搬中アイコンが非表示になる

### 9.3. コイン計算検証

- ✅ 10m運搬 → 3コイン付与
- ✅ 50m運搬 → 15コイン付与
- ✅ 100m運搬 → 30コイン付与（上限）
- ✅ 150m運搬 → 30コイン付与（上限維持）
- ✅ 5m未満 → 0コイン付与

### 9.4. エラーハンドリング

- ✅ CoinManager参照がnullの場合、エラーログが出力され、コイン付与がスキップされる
- ✅ HUDManager参照がnullの場合、警告ログが出力され、UI更新がスキップされる
- ✅ DeliveryZone参照がnullの場合、エラーログが出力され、納品判定が失敗する

### 9.5. パフォーマンス

- ✅ 距離計測がUpdate()で0.5秒間隔のため、CPU負荷が低い（Profilerで確認）
- ✅ Pickup/Drop処理が50ms以内に完了する（Consoleのタイムスタンプで確認）

---

**10. テスト観点／テストケース**

### 10.1. テスト種別

- **単体テスト（Unit Test）**: LogPickup.csの各メソッド単独動作確認
- **結合テスト（Integration Test）**: CoinManager、HUDManagerとの連携確認
- **システムテスト（System Test）**: ClientSimでの実プレイテスト

### 10.2. テストケース

#### 10.2.1. 正常系テストケース

| TC ID | テストケース名 | 操作手順 | 期待結果 |
|-------|-------------|---------|---------|
| TC-P-01 | 丸太Pickup | 1. 丸太に近づく<br>2. Eキー押下 | 丸太がカメラ前に固定される<br>HUDに「丸太運搬中 (0m)」表示<br>移動速度が3.4m/sに減速 |
| TC-P-02 | 距離計測 | 1. 丸太をPickup<br>2. 30m移動 | HUDの距離表示が30m前後になる<br>Consoleに距離ログが0.5秒ごとに出力 |
| TC-P-03 | 納品エリア外Drop | 1. 丸太をPickup<br>2. 納品エリア外でEキー押下 | 丸太が地面に落下<br>「納品エリアに運んでください」メッセージ表示<br>コインは付与されない |
| TC-P-04 | 納品エリア内Drop（10m運搬） | 1. 丸太をPickup<br>2. 10m移動<br>3. 納品エリア内でEキー押下 | 3コイン付与<br>「+3コイン」通知表示<br>丸太が1秒後に消滅 |
| TC-P-05 | 納品エリア内Drop（50m運搬） | 1. 丸太をPickup<br>2. 50m移動<br>3. 納品エリア内でEキー押下 | 15コイン付与<br>「+15コイン」通知表示 |
| TC-P-06 | 納品エリア内Drop（100m運搬） | 1. 丸太をPickup<br>2. 100m移動<br>3. 納品エリア内でEキー押下 | 30コイン付与（上限）<br>「+30コイン」通知表示 |
| TC-P-07 | 納品エリア内Drop（150m運搬） | 1. 丸太をPickup<br>2. 150m移動<br>3. 納品エリア内でEキー押下 | 30コイン付与（上限維持）<br>「+30コイン」通知表示 |
| TC-P-08 | Drop後の移動速度復帰 | 1. 丸太をPickup<br>2. Drop（納品エリア内外問わず） | 移動速度が4.0m/sに戻る<br>Consoleに速度復帰ログ出力 |
| TC-P-09 | Drop後のUI非表示 | 1. 丸太をPickup<br>2. Drop | HUDの運搬中アイコンが非表示になる |

#### 10.2.2. 異常系テストケース

| TC ID | テストケース名 | 操作手順 | 期待結果 |
|-------|-------------|---------|---------|
| TC-E-01 | CoinManager参照null | 1. LogPickupのInspectorでCoinManager参照を削除<br>2. 納品エリア内Drop | Consoleに「CoinManager reference is not set」エラーログ<br>コイン付与はスキップされ、クラッシュしない |
| TC-E-02 | HUDManager参照null | 1. LogPickupのInspectorでHUDManager参照を削除<br>2. 丸太をPickup | Consoleに「HUDManager reference is not set」警告ログ<br>UI更新はスキップされ、Pickup処理は正常動作 |
| TC-E-03 | DeliveryZone参照null | 1. LogPickupのInspectorでDeliveryZone参照を削除<br>2. 納品エリア内でDrop | Consoleに「DeliveryZone reference is null」エラーログ<br>納品判定が失敗し、コイン付与なし |
| TC-E-04 | VRC_Pickupコンポーネントなし | 1. LogPrefabからVRC_Pickupコンポーネントを削除<br>2. ゲーム開始 | Consoleに「VRC_Pickup component not found」エラーログ<br>Pickup機能が動作しない |
| TC-E-05 | 5m未満の運搬 | 1. 丸太をPickup<br>2. 3m移動<br>3. 納品エリア内でDrop | 0コイン付与<br>「+0コイン」通知は表示されない（コイン付与なし） |
| TC-E-06 | 運搬中にワールド退出 | 1. 丸太をPickup<br>2. Playモード停止 | エラーなく終了<br>次回起動時に丸太は元の位置に存在 |

#### 10.2.3. VR環境テストケース（VRヘッドセットでのテスト）

| TC ID | テストケース名 | 操作手順 | 期待結果 |
|-------|-------------|---------|---------|
| TC-VR-01 | VRでのPickup | 1. 丸太に近づく<br>2. グリップボタン押下 | 丸太が手の位置に固定される<br>HUD表示更新 |
| TC-VR-02 | VRでのDrop | 1. 丸太をPickup<br>2. グリップボタン離す | 丸太がその場に落下<br>納品判定が正常動作 |
| TC-VR-03 | VRでの移動速度体感 | 1. 丸太をPickup<br>2. 移動 | 明らかに移動が遅くなることを体感できる |

---

**11. 成果物**

### 11.1. 新規作成ファイル

- ✅ `Assets/WoodcutterCamp/Scripts/Gameplay/LogPickup.cs`
- ✅ `Assets/WoodcutterCamp/Prefabs/DeliveryZone.prefab`（納品ゾーン）
- ✅ HUDの運搬UI要素（`CarryingIcon` Image、`DistanceText` TextMeshProUGUI）

### 11.2. 編集ファイル

- ✅ `Assets/WoodcutterCamp/Prefabs/LogPrefab.prefab`（VRC_Pickupコンポーネント追加、LogPickupスクリプトアタッチ）
- ✅ `Assets/WoodcutterCamp/Scripts/UI/HUDManager.cs`（運搬UI表示メソッド追加）

### 11.3. Unityシーン変更

- ✅ テストシーンに `DeliveryZone` GameObject追加
- ✅ テストシーンに `LogPrefab` 配置

### 11.4. ドキュメント

- ✅ 本作業指示書（WI-0015_LogPickup_VRC_Pickup.md）

---

**12. 備考**

### 12.1. 技術的注意点

#### VRC_Pickupの仕様
- **Auto Hold設定**: `Desktop Auto Hold` により、デスクトップでは1回目のEキーでPickup、2回目でDropとなる。VRではトリガーを離すとDropされる。
- **Orientation: Any**: プレイヤーがどの角度からでも丸太を掴めるため、操作性が向上する。
- **Proximity: 0.5m**: 丸太から0.5m以内に近づくとPickupプロンプトが表示される。この値が大きすぎると遠くから掴めてしまい不自然、小さすぎると掴みにくい。

#### 移動速度変更の制約
- `SetWalkSpeed()` と `SetRunSpeed()` はローカルプレイヤーにのみ適用される（他プレイヤーには見えない）
- VRChatのデフォルト移動速度は4.0m/sのため、15%減速で3.4m/sとした
- Quest 2環境では移動速度変更による酔いのリスクがあるため、減速量は15%に抑えた（PRDの推奨値）

#### パフォーマンス最適化
- 距離計測を毎フレーム実行すると、10人が同時運搬した場合にCPU負荷が高くなる
- 0.5秒間隔の更新により、体感できる精度を維持しつつCPU負荷を約1/30に削減

#### オブジェクトプール対応（将来拡張）
- 現在は納品成功時に `gameObject.SetActive(false)` で丸太を非アクティブ化
- 将来的には、LogSpawnerのオブジェクトプールに戻す処理を追加することで、メモリ効率が向上

### 12.2. FSD仕様との対応

本作業指示書は、FSD.md「FNC-002: 運搬システム」の以下の仕様を実装する：

| FSD仕様 | 実装内容 |
|---------|---------|
| 3.1 正常系 - 1. 丸太のPickup | VRC_Pickup.OnPickup()イベントで実装 |
| 3.1 正常系 - 2. 運搬 | Update()で距離計測、HUD更新 |
| 3.1 正常系 - 3. 納品ゾーンでのDrop | VRC_Pickup.OnDrop()イベント + BoxCollider判定 |
| 3.1 正常系 - 4. 報酬計算 | CalculateCoinReward()メソッド、計算式準拠 |
| 3.1 異常系 - ケース1 | CheckDeliveryZone()でfalse時のメッセージ表示 |

### 12.3. 依存作業との連携

- **WI-0013（CoinManager）**: `CoinManager._AddCoins(int amount)` メソッドが実装済みであることが前提
- **WI-0012（LogSpawner）**: LogPrefabが既に存在し、Rigidbody・Colliderが設定されていることが前提
- **WI-0008（HUDManager）**: 基本的なHUD表示機能が実装されていることが前提（通知表示メソッドなど）

### 12.4. 次の作業（WI-0016）への準備

- 本WI完了後、WI-0016「Delivery Zone and Reward Calculation」で以下を実装予定:
  - 納品時のアニメーション演出（丸太が縮小しながら上昇）
  - 金色のパーティクルエフェクト
  - ランキングトラッカーへの運搬数記録
  - より詳細な納品UI（累計運搬距離表示など）

### 12.5. デバッグログについて

- 本実装ではデバッグログを多用している（`DEBUG_LOG_ENABLED = true`）
- 開発中は詳細なログ出力により問題の特定が容易
- 本番ビルド時は `DEBUG_LOG_ENABLED = false` に変更することで、ログ出力を無効化し、パフォーマンスを向上させる

### 12.6. Quest最適化について

- 本機能は Quest 2環境での60fps維持を目標としている
- 距離計測の0.5秒間隔更新により、CPU負荷を抑制
- HUDのUI要素は最小限に抑え、描画負荷を軽減

---

**作業指示書作成日**: 2025-11-17
**作業指示書バージョン**: 1.0
**作成者**: Claude (Anthropic)
**レビュー状態**: 未レビュー（実装開始前にレビュー必要）
