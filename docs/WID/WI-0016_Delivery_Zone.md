# Work Instruction: WI-0016

## 1. 作業ID
**WI-0016**

## 2. 作業名
**納品ゾーンと報酬計算の実装 (Implement Delivery Zone and Reward Calculation)**

## 3. 作業目的
丸太運搬システムにおいて、プレイヤーが丸太をキャンプ中央の納品エリアに運び込むことでコイン報酬を獲得できる仕組みを実装する。運搬距離に応じた報酬計算と、達成感を高める納品演出を実現することで、プレイヤーの作業へのモチベーションを向上させる。

## 4. 対象

### 4.1 対象システム
- **System Name**: 森のきこりキャンプ (Woodcutter Camp)
- **Platform**: VRChat (Unity 2022.3.22f1 / SDK 3.9.0 / UdonSharp 1.1.9)

### 4.2 対象モジュール
- **Module ID**: M-13 (LogPickup)
- **Related Module**: M-06 (CoinManager)

### 4.3 対象ファイルパス
- **新規作成**: `Assets/WoodcutterCamp/Scripts/Gameplay/Carrying/DeliveryZone.cs`
- **修正対象**: `Assets/WoodcutterCamp/Scripts/Gameplay/Carrying/LogPickup.cs`
- **参照先**: `Assets/WoodcutterCamp/Scripts/Economy/CoinManager.cs`

### 4.4 関連技術ドキュメント
- **TDD.md**: Module M-13 (Section 5.13)
- **FSD.md**: FNC-002 運搬システム (Section 2.3.2, 2.3.3)
- **PRDv3.md**: 運搬システム仕様 (Section 2.3)

## 5. 前提条件

### 5.1 完了している作業
- **WI-0015**: LogPickup VRC_Pickup System実装完了
  - LogPickup.csで丸太のPickup/Drop処理が実装されていること
  - 運搬距離の計測（F-44: Track Carrying Distance）が実装されていること
  - `pickupStartPosition`および`carriedDistance`変数が存在すること

- **WI-0009**: CoinManager Core Logic実装完了
  - CoinManager.csが実装され、`AddCoins(int amount)`メソッドが利用可能であること

### 5.2 環境要件
- Unity 2022.3.22f1で開発環境が構築されていること
- VRChat SDK 3.9.0がインポートされていること
- UdonSharp 1.1.9がインポートされていること
- ClientSim 1.2.7でローカルテスト可能であること

### 5.3 シーン構成
- キャンプ中央エリアに納品ゾーン用の配置スペースが確保されていること
- GameManagerオブジェクトが存在し、CoinManagerがアタッチされていること

## 6. 入力

### 6.1 納品判定の入力データ
- **丸太オブジェクト**: VRC_Pickupコンポーネント付きGameObject
- **Drop位置**: プレイヤーがDropした位置のVector3座標
- **運搬距離**: LogPickup.csで計測された`carriedDistance` (float型、単位: メートル)

### 6.2 報酬計算の入力パラメータ
- **運搬距離** (carriedDistance): 0.0 〜 1000.0メートル
- **報酬計算式**: `min(floor(distance / 10) × 3, 30)`

### 6.3 入力例
| 運搬距離 | 計算式 | 獲得コイン |
|---------|--------|-----------|
| 5m | floor(5/10) × 3 = 0 × 3 | 0コイン |
| 12m | floor(12/10) × 3 = 1 × 3 | 3コイン |
| 35m | floor(35/10) × 3 = 3 × 3 | 9コイン |
| 78m | floor(78/10) × 3 = 7 × 3 | 21コイン |
| 120m | floor(120/10) × 3 = 12 × 3 = 36 → max(30) | 30コイン（上限） |

## 7. 出力

### 7.1 納品成功時の出力
1. **丸太の消滅**
   - Drop後1秒で丸太オブジェクトが縮小アニメーション
   - 縮小完了後にDestroy（またはオブジェクトプールに返却）

2. **コイン付与**
   - CoinManager.AddCoins()経由で報酬を付与
   - HUD左下のコイン残高が更新される

3. **演出効果**
   - サウンド: 「チャリーン」という納品音再生
   - パーティクル: 金色パーティクルがプレイヤーに向かって飛ぶ
   - 通知: 画面中央上部に「+Xコイン」表示（1秒間）

4. **統計更新**
   - プレイヤーの累計納品数カウント+1（ランキング用）

### 7.2 納品失敗時の出力
- **エリア外でDrop**: 丸太はその場に落下、物理演算適用
- **通知メッセージ**: HUDに「納品エリアに運んでください」（2秒間表示）
- **コイン付与なし**

## 8. 作業手順

### 8.1 納品ゾーンオブジェクトの作成

#### 8.1.1 Hierarchyでオブジェクト配置
1. **Unity Hierarchyで右クリック → Create Empty**
   - オブジェクト名: `DeliveryZone`
   - 位置: キャンプ中央の倉庫前 (例: x=0, y=0, z=10)

2. **BoxColliderコンポーネントを追加**
   - Inspector → Add Component → Box Collider
   - **Size**: X=5, Y=3, Z=5 (5m × 3m × 5mの領域)
   - **Center**: X=0, Y=1.5, Z=0 (地面から1.5m上が中心)
   - **Is Trigger**: ✅ ON（チェックを入れる）

3. **視覚表示用の子オブジェクト作成**
   - DeliveryZone直下に `GroundMarker` (平面オブジェクト) を追加
   - 素材: 黄色いエミッシブマテリアル適用
   - サイズ: 5m × 5m、地面レベル (Y=0) に配置

4. **看板オブジェクトの配置**
   - DeliveryZone直下に `Sign` (3Dテキストまたはクワッド) を追加
   - テキスト内容: 「納品エリア / Delivery Zone」
   - 位置: 納品エリアの端 (例: x=2.5, y=2, z=0)

#### 8.1.2 パーティクルシステムの追加
1. **DeliveryZone直下に Particle System を追加**
   - オブジェクト名: `AmbientParticles`
   - パーティクル設定:
     - **Start Color**: 金色 (R=255, G=215, B=0, A=100)
     - **Start Size**: 0.1
     - **Start Lifetime**: 2秒
     - **Emission Rate**: 10粒/秒
     - **Shape**: Box (5m × 3m × 5m)
     - **Gravity Modifier**: -0.5（上に浮かぶ）

### 8.2 DeliveryZone.cs スクリプトの実装

#### 8.2.1 スクリプトファイルの作成
1. **Projectウィンドウで右クリック**
   - `Assets/WoodcutterCamp/Scripts/Gameplay/Carrying/` に移動
   - Create → C# Script
   - ファイル名: `DeliveryZone.cs`

2. **スクリプトをUdonSharpに変換**
   - スクリプトを選択
   - Inspector → Convert to UdonSharp (UdonSharp 1.1.9使用時)

#### 8.2.2 DeliveryZone.cs の実装コード

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon;

namespace WoodcutterCamp.Gameplay.Carrying
{
    /// <summary>
    /// 納品ゾーンの管理
    /// 丸太がエリア内でDropされた際に報酬付与と納品演出を実行
    /// </summary>
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class DeliveryZone : UdonSharpBehaviour
    {
        // ===== 依存関係 =====
        [Header("依存モジュール")]
        [Tooltip("CoinManagerの参照（GameManagerから取得）")]
        public CoinManager coinManager;

        [Header("エフェクト設定")]
        [Tooltip("納品成功時の効果音")]
        public AudioClip deliverySound;

        [Tooltip("納品成功時のパーティクル（金色）")]
        public ParticleSystem deliveryParticles;

        [Tooltip("HUD通知用のUIテキスト（CoinNotification）")]
        public UnityEngine.UI.Text coinNotificationText;

        [Header("納品設定")]
        [Tooltip("丸太消滅までの待機時間（秒）")]
        public float logDisappearDelay = 1.0f;

        [Tooltip("納品アニメーション時間（秒）")]
        public float deliveryAnimationDuration = 1.0f;

        // ===== 内部状態 =====
        private AudioSource audioSource;
        private const int MAX_COIN_REWARD = 30; // 報酬上限
        private const int COINS_PER_10M = 3;    // 10mあたりのコイン

        // ===== 初期化 =====
        void Start()
        {
            InitializeDependencies();
            ValidateReferences();
        }

        /// <summary>
        /// 依存関係の初期化
        /// </summary>
        private void InitializeDependencies()
        {
            // CoinManagerの自動取得（未設定の場合）
            if (coinManager == null)
            {
                GameObject gameManagerObj = GameObject.Find("GameManager");
                if (gameManagerObj != null)
                {
                    coinManager = gameManagerObj.GetComponent<CoinManager>();
                }
            }

            // AudioSourceの取得または追加
            audioSource = GetComponent<AudioSource>();
            if (audioSource == null)
            {
                audioSource = gameObject.AddComponent<AudioSource>();
                audioSource.playOnAwake = false;
                audioSource.spatialBlend = 0.7f; // 3D音響
                audioSource.maxDistance = 15f;
            }
        }

        /// <summary>
        /// 必須参照のバリデーション
        /// </summary>
        private void ValidateReferences()
        {
            if (coinManager == null)
            {
                Debug.LogError("[DeliveryZone] CoinManager reference is null. Please assign in Inspector or ensure GameManager exists.");
            }

            if (deliverySound == null)
            {
                Debug.LogWarning("[DeliveryZone] DeliverySound is not assigned. No sound will play on delivery.");
            }

            if (deliveryParticles == null)
            {
                Debug.LogWarning("[DeliveryZone] DeliveryParticles is not assigned. No particle effect on delivery.");
            }
        }

        // ===== Trigger判定 =====
        /// <summary>
        /// 丸太がエリア内でDropされた際の処理
        /// LogPickup.csから呼び出される
        /// </summary>
        /// <param name="logPickup">Dropされた丸太のLogPickupコンポーネント</param>
        public void _ProcessLogDelivery(LogPickup logPickup)
        {
            if (logPickup == null)
            {
                Debug.LogWarning("[DeliveryZone] LogPickup reference is null.");
                return;
            }

            // 運搬距離を取得
            float carriedDistance = logPickup.GetCarriedDistance();

            // 報酬計算
            int coinReward = CalculateCoinReward(carriedDistance);

            // コイン付与
            if (coinManager != null && coinReward > 0)
            {
                coinManager._AddCoins(coinReward);
                Debug.Log($"[DeliveryZone] Log delivered! Distance: {carriedDistance:F1}m, Reward: {coinReward} coins");
            }

            // 演出実行
            PlayDeliveryEffects(logPickup.gameObject, coinReward);

            // 統計更新（将来的にRankingTrackerと連携）
            // RankingTracker.IncrementDeliveryCount();
        }

        /// <summary>
        /// 報酬計算式の実装
        /// </summary>
        /// <param name="distance">運搬距離（メートル）</param>
        /// <returns>獲得コイン数</returns>
        private int CalculateCoinReward(float distance)
        {
            // 入力バリデーション
            if (distance < 0f)
            {
                Debug.LogWarning($"[DeliveryZone] Invalid distance: {distance}. Clamping to 0.");
                distance = 0f;
            }

            if (distance > 1000f)
            {
                Debug.LogWarning($"[DeliveryZone] Abnormal distance: {distance}. Clamping to 1000m.");
                distance = 1000f;
            }

            // 計算: min(floor(distance / 10) × 3, 30)
            int segments = Mathf.FloorToInt(distance / 10f);
            int reward = segments * COINS_PER_10M;
            reward = Mathf.Min(reward, MAX_COIN_REWARD);

            return reward;
        }

        /// <summary>
        /// 納品演出の実行
        /// </summary>
        /// <param name="logObject">丸太のGameObject</param>
        /// <param name="coinReward">獲得コイン数</param>
        private void PlayDeliveryEffects(GameObject logObject, int coinReward)
        {
            // 1. サウンド再生
            if (audioSource != null && deliverySound != null)
            {
                audioSource.PlayOneShot(deliverySound);
            }

            // 2. パーティクル再生
            if (deliveryParticles != null)
            {
                // プレイヤー方向にパーティクルを飛ばす設定（オプション）
                VRCPlayerApi localPlayer = Networking.LocalPlayer;
                if (localPlayer != null)
                {
                    Vector3 playerPosition = localPlayer.GetPosition();
                    deliveryParticles.transform.position = logObject.transform.position;
                    // パーティクルの方向設定（カスタム実装可能）
                }
                deliveryParticles.Play();
            }

            // 3. 通知表示
            ShowCoinNotification(coinReward);

            // 4. 丸太の消滅アニメーション
            SendCustomEventDelayedSeconds(nameof(_AnimateLogDisappear), logDisappearDelay, VRC.Udon.Common.Enums.EventTiming.Update);
            _disappearingLog = logObject;
        }

        /// <summary>
        /// コイン獲得通知の表示
        /// </summary>
        /// <param name="amount">獲得コイン数</param>
        private void ShowCoinNotification(int amount)
        {
            if (coinNotificationText == null)
            {
                Debug.LogWarning("[DeliveryZone] CoinNotificationText is not assigned.");
                return;
            }

            // 通知テキストを更新
            coinNotificationText.text = $"+{amount}コイン";
            coinNotificationText.gameObject.SetActive(true);

            // 1秒後に非表示
            SendCustomEventDelayedSeconds(nameof(_HideNotification), 1.0f, VRC.Udon.Common.Enums.EventTiming.Update);
        }

        /// <summary>
        /// 通知を非表示にする
        /// </summary>
        public void _HideNotification()
        {
            if (coinNotificationText != null)
            {
                coinNotificationText.gameObject.SetActive(false);
            }
        }

        // ===== 丸太消滅アニメーション =====
        private GameObject _disappearingLog;
        private float _animationStartTime;

        /// <summary>
        /// 丸太の消滅アニメーション開始
        /// </summary>
        public void _AnimateLogDisappear()
        {
            if (_disappearingLog == null)
            {
                return;
            }

            _animationStartTime = Time.time;
            _isAnimating = true;
        }

        private bool _isAnimating = false;

        void Update()
        {
            // 消滅アニメーション処理
            if (_isAnimating && _disappearingLog != null)
            {
                float elapsed = Time.time - _animationStartTime;
                float progress = elapsed / deliveryAnimationDuration;

                if (progress < 1.0f)
                {
                    // 縮小しながら上昇
                    float scale = Mathf.Lerp(1.0f, 0.0f, progress);
                    _disappearingLog.transform.localScale = Vector3.one * scale;

                    Vector3 pos = _disappearingLog.transform.position;
                    pos.y += Time.deltaTime * 0.5f; // 上昇速度
                    _disappearingLog.transform.position = pos;
                }
                else
                {
                    // アニメーション完了
                    Destroy(_disappearingLog);
                    // オブジェクトプール使用の場合は: LogSpawner.ReturnToPool(_disappearingLog);
                    _disappearingLog = null;
                    _isAnimating = false;
                }
            }
        }
    }
}
```

### 8.3 LogPickup.cs への納品判定追加

#### 8.3.1 LogPickup.csの修正箇所

既存のLogPickup.csファイルを開き、以下の修正を追加します。

**追加するフィールド（クラス冒頭）:**
```csharp
[Header("納品設定")]
[Tooltip("納品ゾーンのレイヤーマスク")]
public LayerMask deliveryZoneLayer;

private DeliveryZone currentDeliveryZone; // エリア内判定用
```

**OnDrop()メソッドに納品判定を追加:**
```csharp
public override void OnDrop()
{
    if (!isPickedUp)
    {
        return;
    }

    // 運搬距離の最終計測
    Vector3 dropPosition = transform.position;
    carriedDistance = Vector3.Distance(pickupStartPosition, dropPosition);

    Debug.Log($"[LogPickup] Log dropped. Total distance: {carriedDistance:F1}m");

    // === 納品判定 ===
    if (currentDeliveryZone != null)
    {
        // エリア内でDropされた
        Debug.Log("[LogPickup] Log dropped in delivery zone!");
        currentDeliveryZone._ProcessLogDelivery(this);
    }
    else
    {
        // エリア外でDropされた
        Debug.Log("[LogPickup] Log dropped outside delivery zone.");
        ShowErrorMessage("納品エリアに運んでください");
    }

    // 状態リセット
    isPickedUp = false;
    currentOwner = null;
}

/// <summary>
/// エラーメッセージ表示（簡易実装）
/// </summary>
private void ShowErrorMessage(string message)
{
    // HUDManagerやNotificationSystemと連携
    Debug.LogWarning($"[LogPickup] {message}");
    // 実装例: HUDManager.ShowNotification(message, 2.0f);
}

/// <summary>
/// 運搬距離を取得（DeliveryZoneから呼ばれる）
/// </summary>
public float GetCarriedDistance()
{
    return carriedDistance;
}
```

**Trigger判定の追加（OnTriggerStay/Exit）:**
```csharp
/// <summary>
/// 納品エリアに入った際の判定
/// </summary>
void OnTriggerStay(Collider other)
{
    // 納品ゾーンのコライダーと接触中
    if (((1 << other.gameObject.layer) & deliveryZoneLayer) != 0)
    {
        DeliveryZone zone = other.GetComponent<DeliveryZone>();
        if (zone != null)
        {
            currentDeliveryZone = zone;
        }
    }
}

/// <summary>
/// 納品エリアから出た際の判定
/// </summary>
void OnTriggerExit(Collider other)
{
    if (((1 << other.gameObject.layer) & deliveryZoneLayer) != 0)
    {
        currentDeliveryZone = null;
    }
}
```

### 8.4 Unity Editorでの設定作業

#### 8.4.1 レイヤー設定
1. **Unity上部メニュー → Edit → Project Settings → Tags and Layers**
2. **新しいレイヤー追加**:
   - Layer 10: `DeliveryZone`
3. **DeliveryZoneオブジェクトにレイヤー適用**:
   - HierarchyでDeliveryZoneを選択
   - Inspector → Layer → DeliveryZone

#### 8.4.2 コンポーネント設定
1. **DeliveryZoneオブジェクトを選択**
2. **Add Component → DeliveryZone (Script)**
3. **Inspectorで参照を設定**:
   - **Coin Manager**: GameManagerオブジェクトからドラッグ
   - **Delivery Sound**: プロジェクト内の「チャリーン」音声ファイルをドラッグ
   - **Delivery Particles**: DeliveryZone/AmbientParticlesをドラッグ
   - **Coin Notification Text**: HUD Canvas内のテキストオブジェクトをドラッグ
   - **Log Disappear Delay**: 1.0
   - **Delivery Animation Duration**: 1.0

4. **丸太Prefabの設定**:
   - 丸太Prefabを選択
   - LogPickupコンポーネントのInspectorで:
     - **Delivery Zone Layer**: DeliveryZoneレイヤーを選択

### 8.5 テスト用サウンド・パーティクルの準備

#### 8.5.1 納品音の追加
1. **フリー音源サイトから「コイン音」をダウンロード**
   - 推奨サイト: Freesound.org, OpenGameArt.org
   - 検索ワード: "coin drop", "cash register"
2. **Unityにインポート**:
   - `Assets/WoodcutterCamp/Audio/SFX/` にファイル配置
   - Inspectorで **Force To Mono** にチェック
   - **Load Type**: Compressed In Memory
3. **DeliveryZoneのDelivery Soundに割り当て**

#### 8.5.2 パーティクルエフェクトの調整
1. **AmbientParticlesの設定確認**:
   - Start Color: 金色 (#FFD700)
   - Emission: Rate over Time = 10
   - Shape: Box (5, 3, 5)
2. **追加演出（オプション）**:
   - Sub Emitterを追加し、プレイヤー方向に飛ぶエフェクトを実装

### 8.6 ClientSimでのローカルテスト

#### 8.6.1 テストシナリオ実行
1. **Unityエディタでシーンを再生**
2. **ClientSimを有効化**（VRChat SDKパネルから）
3. **テスト手順**:
   - 木を伐採して丸太を生成
   - 丸太をPickup
   - 納品エリアまで運搬（距離を変えて複数回テスト）
   - 納品エリア内でDrop

#### 8.6.2 確認項目
- [ ] 納品エリア内でDropすると「チャリーン」音が再生される
- [ ] 丸太が1秒後に縮小しながら消える
- [ ] パーティクルが金色で表示される
- [ ] HUDに「+Xコイン」通知が1秒間表示される
- [ ] コイン残高が正しく増加している（CoinManagerと連携）
- [ ] 納品エリア外でDropすると丸太が地面に落ちる
- [ ] エラーメッセージ「納品エリアに運んでください」が表示される

### 8.7 報酬計算式のユニットテスト（オプション）

#### 8.7.1 テストスクリプト作成
Unityエディタ上でテストを実行する場合、以下のテストコードを作成します。

**ファイルパス**: `Assets/WoodcutterCamp/Tests/Editor/DeliveryZoneTests.cs`

```csharp
using NUnit.Framework;
using UnityEngine;

namespace WoodcutterCamp.Tests
{
    public class DeliveryZoneTests
    {
        [Test]
        public void CalculateCoinReward_10m_Returns3Coins()
        {
            // 10m運搬 → 3コイン
            int reward = CalculateReward(10f);
            Assert.AreEqual(3, reward);
        }

        [Test]
        public void CalculateCoinReward_35m_Returns9Coins()
        {
            // 35m運搬 → floor(35/10) × 3 = 9コイン
            int reward = CalculateReward(35f);
            Assert.AreEqual(9, reward);
        }

        [Test]
        public void CalculateCoinReward_120m_Returns30CoinsMax()
        {
            // 120m運搬 → 上限30コイン
            int reward = CalculateReward(120f);
            Assert.AreEqual(30, reward);
        }

        [Test]
        public void CalculateCoinReward_NegativeDistance_Returns0()
        {
            // 負の距離は0として扱う
            int reward = CalculateReward(-10f);
            Assert.AreEqual(0, reward);
        }

        // テスト用ヘルパーメソッド
        private int CalculateReward(float distance)
        {
            const int MAX_COIN_REWARD = 30;
            const int COINS_PER_10M = 3;

            if (distance < 0f) distance = 0f;
            if (distance > 1000f) distance = 1000f;

            int segments = Mathf.FloorToInt(distance / 10f);
            int reward = segments * COINS_PER_10M;
            return Mathf.Min(reward, MAX_COIN_REWARD);
        }
    }
}
```

#### 8.7.2 ユニットテスト実行
1. **Unity上部メニュー → Window → General → Test Runner**
2. **EditModeタブを選択**
3. **Run All** をクリック
4. **全テストが通過することを確認**

## 9. 完了条件（Done の条件）

### 9.1 実装完了条件
- [ ] **DeliveryZone.csの実装が完了している**
  - 納品判定メソッド `_ProcessLogDelivery()` が実装されている
  - 報酬計算メソッド `CalculateCoinReward()` が正しく動作する
  - 納品演出メソッド `PlayDeliveryEffects()` が実装されている

- [ ] **LogPickup.csに納品判定が追加されている**
  - OnDrop()でcurrentDeliveryZoneの判定が実装されている
  - OnTriggerStay/Exitで納品エリア判定が実装されている
  - GetCarriedDistance()メソッドが実装されている

- [ ] **Unity Sceneに納品ゾーンが配置されている**
  - DeliveryZoneオブジェクトがキャンプ中央に配置されている
  - BoxCollider (isTrigger=true) が設定されている
  - 視覚表示（地面マーカー、看板）が配置されている

### 9.2 動作確認完了条件
- [ ] **ClientSimで納品処理が正常動作する**
  - 納品エリア内でDropすると報酬が付与される
  - 報酬計算が仕様通り（10m = 3コイン、上限30コイン）
  - 納品エリア外でDropするとエラーメッセージが表示される

- [ ] **演出が正しく再生される**
  - 納品音「チャリーン」が再生される
  - 丸太が1秒後に縮小アニメーションで消える
  - 金色パーティクルが表示される
  - HUD通知「+Xコイン」が1秒間表示される

- [ ] **CoinManagerと正常に連携している**
  - 納品時にCoinManager.AddCoins()が呼ばれる
  - HUD左下のコイン残高が正しく更新される

### 9.3 品質基準
- [ ] **エラーハンドリングが実装されている**
  - coinManagerがnullの場合のエラーログ出力
  - 異常な運搬距離（負値、1000m超）のクランプ処理

- [ ] **パフォーマンス基準を満たす**
  - 納品処理が50ms以内に完了する
  - Update()での処理が最小限（アニメーション中のみ実行）

- [ ] **コードが規約に準拠している**
  - UdonSharp命名規則に従っている（Udon CustomEventは "_" プレフィックス）
  - デバッグログに "[DeliveryZone]" プレフィックスが付いている

## 10. テスト観点／テストケース

### 10.1 テスト種別
- **単体テスト (Unit Test)**: 報酬計算ロジックのテスト
- **結合テスト (Integration Test)**: LogPickup ↔ DeliveryZone ↔ CoinManagerの連携テスト
- **UIテスト (UI Test)**: 演出（サウンド、パーティクル、通知）の動作確認
- **パフォーマンステスト**: 連続納品時のFPS維持確認

### 10.2 正常系テストケース

| No. | テストケース | 入力 | 期待される出力 | 確認方法 |
|-----|------------|------|---------------|---------|
| TC-001 | 10m運搬後の納品 | 運搬距離=10m | +3コイン付与、演出再生 | ClientSim実行、コンソールログ確認 |
| TC-002 | 35m運搬後の納品 | 運搬距離=35m | +9コイン付与、演出再生 | ClientSim実行、HUD表示確認 |
| TC-003 | 78m運搬後の納品 | 運搬距離=78m | +21コイン付与 | ClientSim実行 |
| TC-004 | 100m運搬後の納品 | 運搬距離=100m | +30コイン付与（上限） | ClientSim実行 |
| TC-005 | 120m運搬後の納品 | 運搬距離=120m | +30コイン付与（上限） | ClientSim実行 |
| TC-006 | 納品音の再生 | 納品成功 | 「チャリーン」音再生 | 音声出力確認 |
| TC-007 | パーティクル表示 | 納品成功 | 金色パーティクル表示 | 視覚確認 |
| TC-008 | 丸太消滅アニメーション | 納品成功 | 1秒後に縮小して消える | 視覚確認 |
| TC-009 | HUD通知表示 | 納品成功（9コイン） | 「+9コイン」表示1秒間 | HUD確認 |

### 10.3 異常系テストケース

| No. | テストケース | 入力 | 期待される出力 | 確認方法 |
|-----|------------|------|---------------|---------|
| TC-101 | 納品エリア外でDrop | エリア外でDrop | エラーメッセージ表示、コイン付与なし | コンソール確認 |
| TC-102 | 運搬距離0mでの納品 | 運搬距離=0m | +0コイン付与 | コンソール確認 |
| TC-103 | 運搬距離5m未満での納品 | 運搬距離=5m | +0コイン付与 | コンソール確認 |
| TC-104 | 異常な運搬距離（負値） | 運搬距離=-10m | 0mとしてクランプ、+0コイン | コンソール警告確認 |
| TC-105 | 異常な運搬距離（1000m超） | 運搬距離=1500m | 1000mとしてクランプ、+30コイン | コンソール警告確認 |
| TC-106 | CoinManagerがnull | coinManager未設定 | エラーログ出力、クラッシュなし | コンソールエラー確認 |
| TC-107 | 納品音が未設定 | deliverySound未設定 | 警告ログ出力、処理継続 | コンソール警告確認 |

### 10.4 性能テストケース

| No. | テストケース | 条件 | 期待値 | 確認方法 |
|-----|------------|------|--------|---------|
| TC-201 | 納品処理時間 | 1回の納品処理 | 50ms以内に完了 | Profiler測定 |
| TC-202 | 連続納品時のFPS | 10個の丸太を連続納品 | Quest 2で60fps維持 | FPS Monitor確認 |
| TC-203 | 複数プレイヤー同時納品 | 5人が同時に納品 | 同期エラーなし | マルチプレイテスト |

## 11. 成果物

### 11.1 新規作成ファイル
- **Scripts**:
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Carrying/DeliveryZone.cs`
- **Tests** (オプション):
  - `/Assets/WoodcutterCamp/Tests/Editor/DeliveryZoneTests.cs`

### 11.2 変更ファイル
- `/Assets/WoodcutterCamp/Scripts/Gameplay/Carrying/LogPickup.cs`
  - OnDrop()メソッドに納品判定追加
  - OnTriggerStay/Exitメソッド追加
  - GetCarriedDistance()メソッド追加

### 11.3 Unity Scene設定
- **DeliveryZoneオブジェクト配置**:
  - Position: キャンプ中央 (例: x=0, y=0, z=10)
  - BoxCollider設定: Size(5,3,5), isTrigger=true
  - 視覚表示（GroundMarker, Sign）配置
  - AmbientParticles設定

### 11.4 アセット追加
- **Audio**:
  - `/Assets/WoodcutterCamp/Audio/SFX/delivery_coin.wav` (納品音)
- **Particles**:
  - DeliveryZone/AmbientParticles (金色パーティクル)

### 11.5 更新ドキュメント
- 本Work Instruction (WI-0016_Delivery_Zone.md)
- 将来的に更新予定:
  - `/docs/TECHNICAL_DETAILS.md` (納品システムの技術詳細追記)

## 12. 備考

### 12.1 注意事項
1. **オブジェクトプールとの統合**:
   - 現在の実装ではDestroy()で丸太を削除していますが、WI-0012で実装されたオブジェクトプール（LogSpawner）がある場合は、`Destroy(_disappearingLog)`を`LogSpawner.ReturnToPool(_disappearingLog)`に置き換えてください。

2. **HUD通知システムとの統合**:
   - 本WIでは簡易的にUI.Textを直接操作していますが、WI-0017で実装されるHUDManagerがある場合は、`HUDManager.ShowNotification()`メソッドを使用してください。

3. **ネットワーク同期について**:
   - 納品処理はローカルで完結するため、`[UdonSynced]`は使用していません。
   - ただし、統計情報（総納品数など）をRankingTrackerと連携する場合は、WI-0021実装時に同期処理を追加する必要があります。

### 12.2 今後の拡張計画
- **Phase 2 拡張要素**:
  - 太丸太協力運搬（2人以上で運ぶシステム）
  - Carryingスキルによる報酬ボーナス
  - 納品先の多様化（特殊納品クエスト）

### 12.3 既知の制限事項
- **報酬上限**: 現在の実装では100m以上運搬しても30コイン固定です。Phase 2で距離ボーナス追加を検討。
- **丸太の同時納品**: 複数の丸太を同時にDropした場合、アニメーション処理が重複する可能性があります。商用リリース前に最適化が必要です。

### 12.4 関連Work Instructions
- **前提WI**:
  - WI-0015: Implement LogPickup VRC_Pickup System
  - WI-0009: Implement CoinManager Core Logic
- **後続WI（連携予定）**:
  - WI-0017: Implement HUDManager（通知システム統合）
  - WI-0021: Implement RankingTracker（統計連携）

### 12.5 参考資料
- **VRChat公式ドキュメント**:
  - [VRC_Pickup Documentation](https://creators.vrchat.com/worlds/components/vrc_pickup/)
  - [OnTriggerStay in Udon](https://udonsharp.docs.vrchat.com/events/#ontriggerstay)
- **Unity公式ドキュメント**:
  - [BoxCollider.isTrigger](https://docs.unity3d.com/ScriptReference/BoxCollider-isTrigger.html)
  - [ParticleSystem.Play](https://docs.unity3d.com/ScriptReference/ParticleSystem.Play.html)

---

**作業指示書作成日**: 2025-11-17
**作成者**: VRChat World Development Team
**レビュー者**: （実装開始前にレビュー必要）
**承認者**: （実装開始前に承認必要）
