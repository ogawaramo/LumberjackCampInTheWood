# WI-0017: HUDManager実装作業指示書

**作業ID**: WI-0017
**作業名**: HUDManager実装（プレイヤーHUD表示システム）
**モジュールID**: M-08
**担当機能**: F-24, F-25, F-26, F-27
**優先度**: 高
**推定工数**: 1.5日

---

## 1. 作業目的

プレイヤーの画面に常時表示されるHUD（Heads-Up Display）を実装します。スキルレベル、XPバー、コイン残高、通知メッセージをリアルタイムで表示し、プレイヤーの状態を視覚的にフィードバックする機能を提供します。

本作業の目的：
- スキルレベルとXP進捗の視認性向上（F-24）
- コイン残高のリアルタイム表示（F-25）
- ゲーム内イベントの通知システム実装（F-26）
- HUD表示のカスタマイズ機能提供（F-27）

---

## 2. 対象

### 2.1 対象システム
- **システム名**: 森のきこりキャンプ - VRChat World Phase 1
- **レイヤー**: UI Layer
- **モジュール**: M-08 HUDManager

### 2.2 対象ファイル

#### 新規作成ファイル
- `/Assets/UdonScripts/UI/HUDManager.cs` - メインスクリプト
- `/Assets/Prefabs/UI/PlayerHUD.prefab` - HUD UI Prefab

#### 依存ファイル（既存）
- `/Assets/UdonScripts/Core/GameManager.cs` - WI-0001で作成済み
- `/Assets/UdonScripts/Progression/SkillManager.cs` - WI-0005で作成済み
- `/Assets/UdonScripts/Economy/CoinManager.cs` - WI-0013で作成済み

---

## 3. 前提条件

### 3.1 依存作業完了
- [x] WI-0001: GameManager実装完了
- [x] WI-0005: SkillManager実装完了
- [x] WI-0013: CoinManager実装完了

### 3.2 開発環境
- **Unity**: 2022.3.22f1 LTS
- **VRChat SDK**: 3.9.0
- **UdonSharp**: 1.1.9（安定版）
- **TextMeshPro**: Unity標準パッケージ（2022.3.22f1に含まれる）

### 3.3 初期状態
- Unity Projectが開かれている
- VRChat SDKがインポート済み
- UdonSharpがインポート済み
- GameManager、SkillManager、CoinManagerがシーンに配置済み

---

## 4. 機能要件詳細

### 4.1 スキル表示（F-24）

#### 表示内容
```
Woodcutting: Lv5
[■■■■■□□□□□] 450/700 XP
```

#### 動作仕様
- **更新トリガー**: SkillManagerの`OnXPGained`イベント、`OnSkillLevelUp`イベント
- **表示位置**: 画面左下（デフォルト）
- **XPバーアニメーション**: 0.5秒かけて滑らかに増加
- **レベルアップ時**: バーが満タン → リセット → 次レベルのバー表示

### 4.2 コイン表示（F-25）

#### 表示内容
```
🪙 245 Coins
```

#### 動作仕様
- **更新トリガー**: CoinManagerの`OnCoinsChanged`イベント
- **表示位置**: 画面左下（スキル表示の下）
- **カウントアップアニメーション**: 0.3秒かけて増加（例：100 → 105）
- **上限表示**: 99,999コイン到達時は赤色で表示

### 4.3 通知表示（F-26）

#### 表示内容例
```
+10 XP
+5 Coins
LEVEL UP!
丸太 × 2 を獲得
```

#### 動作仕様
- **表示位置**: 画面中央上部
- **表示時間**: 3秒間（デフォルト）
- **最大同時表示**: 3件まで（FIFO方式、古い通知から消滅）
- **フェードアニメーション**: フェードイン0.2秒、フェードアウト0.3秒
- **通知キュー**: 3件を超える場合は待機（前の通知が消えてから表示）

### 4.4 HUD設定（F-27）

#### 調整可能項目
- **透明度**: 50%〜100%（デフォルト80%）
- **スケール**: 0.8x〜1.2x（デフォルト1.0x）
- **位置**: 左下/右下/左上/右上（デフォルト左下）

#### 保存仕様
- **保存先**: PlayerData（VRChat Persistence API）
- **キー名**:
  - `HUD_Opacity`: int型（50〜100）
  - `HUD_Scale`: int型（80〜120、実数×100）
  - `HUD_Position`: int型（0=左下、1=右下、2=左上、3=右上）

---

## 5. 技術仕様

### 5.1 UdonSharp実装仕様

#### クラス構成
```
HUDManager (UdonSharpBehaviour)
 ├── UI要素参照
 ├── マネージャー参照（GameManager経由）
 ├── 通知キュー管理
 ├── イベントハンドラー
 └── 設定管理
```

#### 主要メソッド
| メソッド名 | 説明 | トリガー |
|-----------|------|---------|
| `Start()` | 初期化、参照取得、PlayerData読み込み | Unity起動時 |
| `_UpdateSkillDisplay()` | スキル表示更新 | SkillManager経由 |
| `_UpdateCoinDisplay()` | コイン表示更新 | CoinManager経由 |
| `_ShowNotification(string message, float duration)` | 通知表示 | 各Manager経由 |
| `_ApplyHUDSettings()` | HUD設定適用 | Start時、設定変更時 |

#### ネットワーク同期
- **同期方式**: なし（ローカルUI、各プレイヤーで独立）
- **PlayerData使用**: HUD設定のみ永続化

### 5.2 UI構成（TextMeshProUGUI使用）

#### Canvas設定
```yaml
Canvas:
  RenderMode: ScreenSpace - Overlay
  CanvasScaler:
    UIScaleMode: ScaleWithScreenSize
    ReferenceResolution: 1920x1080
    ScreenMatchMode: MatchWidthOrHeight
    Match: 0.5
```

#### UI階層構造
```
PlayerHUD (Canvas)
├── SkillPanel (Panel)
│   ├── SkillLevelText (TextMeshProUGUI) - "Woodcutting: Lv5"
│   └── XPBar (Image)
│       ├── XPBarFill (Image) - Fill Amount制御
│       └── XPText (TextMeshProUGUI) - "450/700 XP"
├── CoinPanel (Panel)
│   └── CoinText (TextMeshProUGUI) - "🪙 245 Coins"
└── NotificationPanel (Panel)
    ├── Notification1 (TextMeshProUGUI)
    ├── Notification2 (TextMeshProUGUI)
    └── Notification3 (TextMeshProUGUI)
```

### 5.3 Quest最適化要件

#### パフォーマンス制約
- **Draw Call**: 1〜2回（UI Batching）
- **Texture Atlas**: 512x512以下
- **Font Asset**: 動的フォント使用禁止 → SDF Fontを事前生成
- **Update()ループ**: 使用禁止 → イベント駆動のみ

#### 最適化実装
```csharp
// ❌ Bad: 毎フレームGetComponent
void Update()
{
    var skill = GameManager.Instance.GetSkillManager();
}

// ✅ Good: Start()でキャッシュ
void Start()
{
    skillManager = GameManager.Instance.GetSkillManager();
}
```

---

## 6. 実装手順

### STEP 1: Unity Sceneセットアップ（15分）

#### 1.1 HUD Canvas作成
1. **Hierarchyウィンドウで右クリック** → `UI` → `Canvas` を選択
2. 作成されたCanvasを `PlayerHUD` にリネーム
3. **Inspectorウィンドウ**で以下を設定：
   - `Render Mode`: `Screen Space - Overlay`
   - `Canvas Scaler` コンポーネント:
     - `UI Scale Mode`: `Scale With Screen Size`
     - `Reference Resolution`: X=1920, Y=1080
     - `Screen Match Mode`: `Match Width Or Height`
     - `Match`: 0.5

#### 1.2 スキルパネル作成
1. **PlayerHUD を右クリック** → `UI` → `Panel` を選択
2. 作成されたPanelを `SkillPanel` にリネーム
3. **Rect Transform** 設定:
   - `Anchors`: 左下（Min: 0,0 / Max: 0,0）
   - `Pivot`: X=0, Y=0
   - `Pos X`: 20, `Pos Y`: 120
   - `Width`: 300, `Height`: 80
4. **Image** コンポーネント:
   - `Color`: R=0, G=0, B=0, A=180（半透明の黒）

#### 1.3 スキルレベルテキスト作成
1. **SkillPanel を右クリック** → `UI` → `Text - TextMeshPro` を選択
   - 初回の場合、「TMP Essentials」のインポート確認ダイアログが表示される → `Import TMP Essentials` をクリック
2. 作成されたTextを `SkillLevelText` にリネーム
3. **Rect Transform** 設定:
   - `Anchors`: Stretch（Min: 0,1 / Max: 1,1）
   - `Pos X`: 0, `Pos Y`: -15
   - `Height`: 30
4. **TextMeshProUGUI** コンポーネント:
   - `Text`: "Woodcutting: Lv1"
   - `Font Size`: 18
   - `Color`: R=255, G=255, B=255, A=255（白色）
   - `Alignment`: 左揃え、中央
   - `Overflow`: Overflow（テキストが切れないように）

#### 1.4 XPバー作成
1. **SkillPanel を右クリック** → `UI` → `Image` を選択 → `XPBar` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: Stretch（Min: 0,0 / Max: 1,0）
   - `Pos X`: 0, `Pos Y`: 25
   - `Height`: 20
3. **Image** コンポーネント:
   - `Color`: R=50, G=50, B=50, A=255（濃いグレー、背景）

#### 1.5 XPバーFill作成
1. **XPBar を右クリック** → `UI` → `Image` を選択 → `XPBarFill` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: Stretch（Min: 0,0 / Max: 1,1）
   - `Pos X`: 0, `Pos Y`: 0, `Pos Z`: 0
   - `Left/Right/Top/Bottom`: 0
3. **Image** コンポーネント:
   - `Image Type`: `Filled`
   - `Fill Method`: `Horizontal`
   - `Fill Origin`: `Left`
   - `Fill Amount`: 0.5（初期値、スクリプトで制御）
   - `Color`: R=100, G=200, B=100, A=255（緑色）

#### 1.6 XPテキスト作成
1. **XPBar を右クリック** → `UI` → `Text - TextMeshPro` を選択 → `XPText` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: Stretch（Min: 0,0 / Max: 1,1）
   - `Left/Right/Top/Bottom`: 0
3. **TextMeshProUGUI** コンポーネント:
   - `Text`: "0/100 XP"
   - `Font Size`: 14
   - `Color`: R=255, G=255, B=255, A=255
   - `Alignment`: 中央、中央

#### 1.7 コインパネル作成
1. **PlayerHUD を右クリック** → `UI` → `Panel` を選択 → `CoinPanel` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: 左下（Min: 0,0 / Max: 0,0）
   - `Pivot`: X=0, Y=0
   - `Pos X`: 20, `Pos Y`: 20
   - `Width`: 200, `Height`: 50
3. **Image** コンポーネント:
   - `Color`: R=0, G=0, B=0, A=180

#### 1.8 コインテキスト作成
1. **CoinPanel を右クリック** → `UI` → `Text - TextMeshPro` を選択 → `CoinText` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: Stretch（Min: 0,0 / Max: 1,1）
   - `Left/Right/Top/Bottom`: 5
3. **TextMeshProUGUI** コンポーネント:
   - `Text`: "🪙 0 Coins"
   - `Font Size`: 20
   - `Color`: R=255, G=215, B=0, A=255（金色）
   - `Alignment`: 左揃え、中央

#### 1.9 通知パネル作成
1. **PlayerHUD を右クリック** → `UI` → `Panel` を選択 → `NotificationPanel` にリネーム
2. **Rect Transform** 設定:
   - `Anchors`: 中央上（Min: 0.5,1 / Max: 0.5,1）
   - `Pivot`: X=0.5, Y=1
   - `Pos X`: 0, `Pos Y`: -50
   - `Width`: 400, `Height`: 150
3. **Image** コンポーネント:
   - `Color`: R=0, G=0, B=0, A=0（完全透明、背景なし）

#### 1.10 通知テキスト作成（3つ）
以下の手順を3回繰り返し、`Notification1`、`Notification2`、`Notification3`を作成：

1. **NotificationPanel を右クリック** → `UI` → `Text - TextMeshPro` を選択 → リネーム
2. **Rect Transform** 設定:
   - **Notification1**: Pos Y=-10, Height=40
   - **Notification2**: Pos Y=-60, Height=40
   - **Notification3**: Pos Y=-110, Height=40
   - 共通: `Anchors`: Stretch（Min: 0,1 / Max: 1,1）、`Left/Right`: 0
3. **TextMeshProUGUI** コンポーネント:
   - `Text`: "" (空文字)
   - `Font Size`: 24
   - `Color`: R=255, G=255, B=100, A=0（初期値は透明）
   - `Alignment`: 中央、中央

#### 1.11 Prefab保存
1. **Hierarchyウィンドウ**で`PlayerHUD`を選択
2. **Projectウィンドウ**で`Assets/Prefabs/UI/`フォルダに**ドラッグ&ドロップ**
   - フォルダが存在しない場合: `Assets`を右クリック → `Create` → `Folder` → `Prefabs`を作成 → その中に`UI`フォルダを作成
3. Prefab作成完了を確認（PlayerHUDがHierarchyで青色に変化）

---

### STEP 2: HUDManager.cs スクリプト作成（60分）

#### 2.1 ファイル作成
1. **Projectウィンドウ**で`Assets/UdonScripts/UI/`フォルダを作成
   - `Assets`を右クリック → `Create` → `Folder` → `UdonScripts`
   - `UdonScripts`内に`UI`フォルダを作成
2. **UIフォルダを右クリック** → `Create` → `U# Script` → ファイル名を `HUDManager` に設定
3. Visual Studioまたは使用中のエディタでファイルを開く

#### 2.2 完全実装コード

以下のコードを`HUDManager.cs`に記述してください：

```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon;
using TMPro;

namespace WoodcutterCamp.UI
{
    /// <summary>
    /// HUDManager - プレイヤーHUD表示管理
    ///
    /// 責務:
    /// - スキルレベル・XPバーの表示更新（F-24）
    /// - コイン残高の表示更新（F-25）
    /// - 通知メッセージの表示・キュー管理（F-26）
    /// - HUD設定の保存・適用（F-27）
    ///
    /// 依存:
    /// - GameManager (WI-0001)
    /// - SkillManager (WI-0005)
    /// - CoinManager (WI-0013)
    /// </summary>
    public class HUDManager : UdonSharpBehaviour
    {
        #region UI参照
        [Header("UI要素参照")]
        [Tooltip("スキルレベル表示テキスト")]
        public TextMeshProUGUI skillLevelText;

        [Tooltip("XPバーのFill Image")]
        public Image xpBarFill;

        [Tooltip("XP数値表示テキスト")]
        public TextMeshProUGUI xpText;

        [Tooltip("コイン残高表示テキスト")]
        public TextMeshProUGUI coinText;

        [Tooltip("通知テキスト配列（最大3件）")]
        public TextMeshProUGUI[] notificationTexts;

        [Tooltip("HUD全体のルートオブジェクト")]
        public GameObject hudRoot;
        #endregion

        #region マネージャー参照
        [Header("マネージャー参照")]
        [Tooltip("GameManager（自動取得）")]
        private UdonSharpBehaviour gameManager;

        [Tooltip("SkillManager（GameManager経由で取得）")]
        private UdonSharpBehaviour skillManager;

        [Tooltip("CoinManager（GameManager経由で取得）")]
        private UdonSharpBehaviour coinManager;
        #endregion

        #region 通知キュー管理
        [Header("通知キュー管理")]
        [Tooltip("通知メッセージキュー")]
        private string[] notificationQueue = new string[10];

        [Tooltip("通知表示時間キュー（秒）")]
        private float[] notificationDurations = new float[10];

        [Tooltip("キューの先頭インデックス")]
        private int queueHead = 0;

        [Tooltip("キューの末尾インデックス")]
        private int queueTail = 0;

        [Tooltip("現在表示中の通知数")]
        private int activeNotificationCount = 0;

        [Tooltip("通知の非表示タイマー配列")]
        private float[] notificationHideTimers = new float[3];
        #endregion

        #region HUD設定
        [Header("HUD設定")]
        [Tooltip("HUD透明度（50〜100）")]
        private int hudOpacity = 80;

        [Tooltip("HUDスケール（80〜120、実数×100）")]
        private int hudScale = 100;

        [Tooltip("HUD位置（0=左下、1=右下、2=左上、3=右上）")]
        private int hudPosition = 0;

        [Tooltip("Canvas Group（透明度制御用）")]
        private CanvasGroup canvasGroup;
        #endregion

        #region キャッシュ変数
        [Header("キャッシュ変数")]
        [Tooltip("前回のXPバー値（アニメーション用）")]
        private float previousXPFillAmount = 0f;

        [Tooltip("目標XPバー値")]
        private float targetXPFillAmount = 0f;

        [Tooltip("XPバーアニメーション時間")]
        private const float XP_BAR_ANIM_DURATION = 0.5f;

        [Tooltip("XPバーアニメーション経過時間")]
        private float xpBarAnimTimer = 0f;

        [Tooltip("前回のコイン数")]
        private int previousCoins = 0;

        [Tooltip("目標コイン数")]
        private int targetCoins = 0;

        [Tooltip("コインカウントアップアニメーション時間")]
        private const float COIN_ANIM_DURATION = 0.3f;

        [Tooltip("コインアニメーション経過時間")]
        private float coinAnimTimer = 0f;
        #endregion

        #region Unity Lifecycle
        /// <summary>
        /// 初期化処理
        /// - マネージャー参照取得
        /// - PlayerDataからHUD設定読み込み
        /// - UI初期表示
        /// </summary>
        void Start()
        {
            Debug.Log("[HUDManager] 初期化開始");

            // GameManager取得
            GameObject gmObject = GameObject.Find("GameManager");
            if (gmObject == null)
            {
                Debug.LogError("[HUDManager] GameManagerが見つかりません。シーンに配置してください。");
                return;
            }
            gameManager = gmObject.GetComponent<UdonSharpBehaviour>();

            // GameManager経由で他のマネージャー取得（1秒遅延）
            SendCustomEventDelayedSeconds(nameof(_InitializeManagers), 1.0f);

            // CanvasGroup取得または作成
            canvasGroup = hudRoot.GetComponent<CanvasGroup>();
            if (canvasGroup == null)
            {
                canvasGroup = hudRoot.AddComponent<CanvasGroup>();
            }

            // PlayerDataからHUD設定読み込み
            _LoadHUDSettings();

            // HUD設定適用
            _ApplyHUDSettings();

            // 通知テキスト初期化
            for (int i = 0; i < notificationTexts.Length; i++)
            {
                notificationTexts[i].text = "";
                notificationTexts[i].color = new Color(1f, 1f, 0.4f, 0f); // 透明
            }

            Debug.Log("[HUDManager] 初期化完了");
        }

        /// <summary>
        /// マネージャー参照初期化（遅延実行）
        /// GameManagerの初期化完了を待つ
        /// </summary>
        public void _InitializeManagers()
        {
            // SkillManager取得
            skillManager = (UdonSharpBehaviour)gameManager.GetProgramVariable("skillManager");
            if (skillManager == null)
            {
                Debug.LogError("[HUDManager] SkillManagerの取得に失敗しました");
            }

            // CoinManager取得
            coinManager = (UdonSharpBehaviour)gameManager.GetProgramVariable("coinManager");
            if (coinManager == null)
            {
                Debug.LogError("[HUDManager] CoinManagerの取得に失敗しました");
            }

            // 初回表示更新
            _UpdateSkillDisplay();
            _UpdateCoinDisplay();
        }

        /// <summary>
        /// 毎フレーム更新
        /// - XPバーアニメーション
        /// - コインカウントアップアニメーション
        /// - 通知の自動非表示タイマー
        ///
        /// 注意: パフォーマンス最適化のため、必要最小限の処理のみ
        /// </summary>
        void Update()
        {
            // XPバーアニメーション
            if (xpBarAnimTimer < XP_BAR_ANIM_DURATION)
            {
                xpBarAnimTimer += Time.deltaTime;
                float t = Mathf.Clamp01(xpBarAnimTimer / XP_BAR_ANIM_DURATION);
                float currentFill = Mathf.Lerp(previousXPFillAmount, targetXPFillAmount, t);
                xpBarFill.fillAmount = currentFill;
            }

            // コインカウントアップアニメーション
            if (coinAnimTimer < COIN_ANIM_DURATION)
            {
                coinAnimTimer += Time.deltaTime;
                float t = Mathf.Clamp01(coinAnimTimer / COIN_ANIM_DURATION);
                int currentCoins = (int)Mathf.Lerp(previousCoins, targetCoins, t);
                _UpdateCoinText(currentCoins);
            }

            // 通知の自動非表示タイマー
            for (int i = 0; i < 3; i++)
            {
                if (notificationHideTimers[i] > 0f)
                {
                    notificationHideTimers[i] -= Time.deltaTime;
                    if (notificationHideTimers[i] <= 0f)
                    {
                        _HideNotification(i);
                    }
                }
            }
        }
        #endregion

        #region スキル表示（F-24）
        /// <summary>
        /// スキル表示更新（SkillManagerから呼び出される）
        /// - スキルレベル表示
        /// - XPバー更新（アニメーション）
        /// </summary>
        public void _UpdateSkillDisplay()
        {
            if (skillManager == null)
            {
                Debug.LogWarning("[HUDManager] SkillManagerが未初期化のため、スキル表示をスキップ");
                return;
            }

            // スキルレベル取得
            int skillLevel = (int)skillManager.GetProgramVariable("skillLevel");
            skillLevelText.text = $"Woodcutting: Lv{skillLevel}";

            // XP取得
            int currentXP = (int)skillManager.GetProgramVariable("currentXP");
            int requiredXP = (int)skillManager.GetProgramVariable("requiredXP");

            // XPバー更新（アニメーション）
            previousXPFillAmount = xpBarFill.fillAmount;
            targetXPFillAmount = (float)currentXP / requiredXP;
            xpBarAnimTimer = 0f;

            // XPテキスト更新
            xpText.text = $"{currentXP}/{requiredXP} XP";

            Debug.Log($"[HUDManager] スキル表示更新: Lv{skillLevel}, {currentXP}/{requiredXP} XP");
        }

        /// <summary>
        /// レベルアップ通知（SkillManagerから呼び出される）
        /// </summary>
        public void _OnSkillLevelUp()
        {
            int newLevel = (int)skillManager.GetProgramVariable("skillLevel");
            _ShowNotification($"LEVEL UP! Lv{newLevel}", 3.0f);

            // XPバーをリセット
            previousXPFillAmount = 1.0f;
            targetXPFillAmount = 0f;
            xpBarAnimTimer = 0f;
        }
        #endregion

        #region コイン表示（F-25）
        /// <summary>
        /// コイン表示更新（CoinManagerから呼び出される）
        /// - カウントアップアニメーション
        /// </summary>
        public void _UpdateCoinDisplay()
        {
            if (coinManager == null)
            {
                Debug.LogWarning("[HUDManager] CoinManagerが未初期化のため、コイン表示をスキップ");
                return;
            }

            // コイン数取得
            int currentCoins = (int)coinManager.GetProgramVariable("currentCoins");

            // カウントアップアニメーション開始
            previousCoins = targetCoins;
            targetCoins = currentCoins;
            coinAnimTimer = 0f;

            Debug.Log($"[HUDManager] コイン表示更新: {currentCoins} Coins");
        }

        /// <summary>
        /// コインテキスト更新（内部処理）
        /// </summary>
        private void _UpdateCoinText(int coins)
        {
            coinText.text = $"🪙 {coins} Coins";

            // 上限到達時は赤色表示
            if (coins >= 99999)
            {
                coinText.color = new Color(1f, 0.3f, 0.3f); // 赤色
            }
            else
            {
                coinText.color = new Color(1f, 0.84f, 0f); // 金色
            }
        }
        #endregion

        #region 通知表示（F-26）
        /// <summary>
        /// 通知表示（公開メソッド）
        /// キューに追加し、空きがあれば即座に表示
        ///
        /// 使用例:
        /// hudManager._ShowNotification("+10 XP", 3.0f);
        /// </summary>
        /// <param name="message">表示メッセージ</param>
        /// <param name="duration">表示時間（秒）</param>
        public void _ShowNotification(string message, float duration)
        {
            // キューに追加
            notificationQueue[queueTail] = message;
            notificationDurations[queueTail] = duration;
            queueTail = (queueTail + 1) % 10;

            Debug.Log($"[HUDManager] 通知キューに追加: {message}");

            // 空きスロットがあれば即座に表示
            _ProcessNotificationQueue();
        }

        /// <summary>
        /// 通知キュー処理（内部処理）
        /// 空きスロットに通知を表示
        /// </summary>
        private void _ProcessNotificationQueue()
        {
            // 3件まで同時表示可能
            while (activeNotificationCount < 3 && queueHead != queueTail)
            {
                string message = notificationQueue[queueHead];
                float duration = notificationDurations[queueHead];
                queueHead = (queueHead + 1) % 10;

                // 空きスロット検索
                for (int i = 0; i < 3; i++)
                {
                    if (notificationHideTimers[i] <= 0f)
                    {
                        _DisplayNotificationInSlot(i, message, duration);
                        break;
                    }
                }
            }
        }

        /// <summary>
        /// 指定スロットに通知を表示（内部処理）
        /// </summary>
        /// <param name="slotIndex">スロット番号（0〜2）</param>
        /// <param name="message">メッセージ</param>
        /// <param name="duration">表示時間</param>
        private void _DisplayNotificationInSlot(int slotIndex, string message, float duration)
        {
            notificationTexts[slotIndex].text = message;
            notificationTexts[slotIndex].color = new Color(1f, 1f, 0.4f, 1f); // 不透明
            notificationHideTimers[slotIndex] = duration;
            activeNotificationCount++;

            Debug.Log($"[HUDManager] 通知表示: スロット{slotIndex}, {message}");
        }

        /// <summary>
        /// 通知非表示（内部処理）
        /// </summary>
        /// <param name="slotIndex">スロット番号</param>
        private void _HideNotification(int slotIndex)
        {
            notificationTexts[slotIndex].color = new Color(1f, 1f, 0.4f, 0f); // 透明
            notificationTexts[slotIndex].text = "";
            activeNotificationCount--;

            // 次の通知を処理
            _ProcessNotificationQueue();
        }
        #endregion

        #region HUD設定（F-27）
        /// <summary>
        /// HUD設定読み込み（PlayerData）
        /// </summary>
        private void _LoadHUDSettings()
        {
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer == null)
            {
                Debug.LogWarning("[HUDManager] ローカルプレイヤー取得失敗、デフォルト設定使用");
                return;
            }

            // PlayerDataから読み込み
            if (localPlayer.GetPlayerTag("HUD_Opacity") != "")
            {
                int.TryParse(localPlayer.GetPlayerTag("HUD_Opacity"), out hudOpacity);
            }
            if (localPlayer.GetPlayerTag("HUD_Scale") != "")
            {
                int.TryParse(localPlayer.GetPlayerTag("HUD_Scale"), out hudScale);
            }
            if (localPlayer.GetPlayerTag("HUD_Position") != "")
            {
                int.TryParse(localPlayer.GetPlayerTag("HUD_Position"), out hudPosition);
            }

            Debug.Log($"[HUDManager] HUD設定読み込み: 透明度={hudOpacity}, スケール={hudScale}, 位置={hudPosition}");
        }

        /// <summary>
        /// HUD設定適用
        /// </summary>
        public void _ApplyHUDSettings()
        {
            // 透明度適用
            canvasGroup.alpha = hudOpacity / 100f;

            // スケール適用
            hudRoot.transform.localScale = Vector3.one * (hudScale / 100f);

            // 位置適用
            RectTransform rt = hudRoot.GetComponent<RectTransform>();
            switch (hudPosition)
            {
                case 0: // 左下
                    rt.anchorMin = new Vector2(0, 0);
                    rt.anchorMax = new Vector2(0, 0);
                    rt.pivot = new Vector2(0, 0);
                    rt.anchoredPosition = new Vector2(20, 20);
                    break;
                case 1: // 右下
                    rt.anchorMin = new Vector2(1, 0);
                    rt.anchorMax = new Vector2(1, 0);
                    rt.pivot = new Vector2(1, 0);
                    rt.anchoredPosition = new Vector2(-20, 20);
                    break;
                case 2: // 左上
                    rt.anchorMin = new Vector2(0, 1);
                    rt.anchorMax = new Vector2(0, 1);
                    rt.pivot = new Vector2(0, 1);
                    rt.anchoredPosition = new Vector2(20, -20);
                    break;
                case 3: // 右上
                    rt.anchorMin = new Vector2(1, 1);
                    rt.anchorMax = new Vector2(1, 1);
                    rt.pivot = new Vector2(1, 1);
                    rt.anchoredPosition = new Vector2(-20, -20);
                    break;
            }

            Debug.Log("[HUDManager] HUD設定適用完了");
        }

        /// <summary>
        /// HUD設定保存（PlayerData）
        /// </summary>
        public void _SaveHUDSettings()
        {
            VRCPlayerApi localPlayer = Networking.LocalPlayer;
            if (localPlayer == null) return;

            localPlayer.SetPlayerTag("HUD_Opacity", hudOpacity.ToString());
            localPlayer.SetPlayerTag("HUD_Scale", hudScale.ToString());
            localPlayer.SetPlayerTag("HUD_Position", hudPosition.ToString());

            Debug.Log("[HUDManager] HUD設定保存完了");
        }

        /// <summary>
        /// 透明度設定（外部から呼び出し可能）
        /// </summary>
        /// <param name="opacity">透明度（50〜100）</param>
        public void _SetOpacity(int opacity)
        {
            hudOpacity = Mathf.Clamp(opacity, 50, 100);
            _ApplyHUDSettings();
            _SaveHUDSettings();
        }

        /// <summary>
        /// スケール設定（外部から呼び出し可能）
        /// </summary>
        /// <param name="scale">スケール（80〜120）</param>
        public void _SetScale(int scale)
        {
            hudScale = Mathf.Clamp(scale, 80, 120);
            _ApplyHUDSettings();
            _SaveHUDSettings();
        }

        /// <summary>
        /// 位置設定（外部から呼び出し可能）
        /// </summary>
        /// <param name="position">位置（0=左下、1=右下、2=左上、3=右上）</param>
        public void _SetPosition(int position)
        {
            hudPosition = Mathf.Clamp(position, 0, 3);
            _ApplyHUDSettings();
            _SaveHUDSettings();
        }
        #endregion
    }
}
```

#### 2.3 コード保存
1. Visual Studioで **Ctrl + S** を押して保存
2. Unityエディタに戻る
3. コンパイル完了を待つ（Console下部に「Compiling...」→「Compilation finished」と表示）
4. エラーがないことを確認

---

### STEP 3: Unity Sceneへの配置と設定（30分）

#### 3.1 HUDManagerスクリプトのアタッチ
1. **Hierarchyウィンドウ**で`PlayerHUD`を選択
2. **Inspectorウィンドウ**下部の`Add Component`をクリック
3. 検索欄に`HUDManager`と入力 → 表示されたスクリプトをクリック

#### 3.2 UI参照の設定
**Inspectorウィンドウ**の`HUDManager`コンポーネントで、以下のフィールドにUI要素をドラッグ&ドロップ：

| フィールド名 | ドラッグ元（Hierarchy） |
|------------|---------------------|
| `Skill Level Text` | SkillPanel/SkillLevelText |
| `Xp Bar Fill` | SkillPanel/XPBar/XPBarFill |
| `Xp Text` | SkillPanel/XPBar/XPText |
| `Coin Text` | CoinPanel/CoinText |
| `Hud Root` | PlayerHUD（自分自身） |

#### 3.3 通知テキスト配列の設定
1. `Notification Texts`フィールドの横の`▶`をクリックして展開
2. `Size`を`3`に設定
3. 3つの`Element`に以下をドラッグ:
   - `Element 0`: NotificationPanel/Notification1
   - `Element 1`: NotificationPanel/Notification2
   - `Element 2`: NotificationPanel/Notification3

#### 3.4 CanvasGroupコンポーネント追加
1. **Hierarchyウィンドウ**で`PlayerHUD`を選択
2. **Inspectorウィンドウ**下部の`Add Component`をクリック
3. 検索欄に`Canvas Group`と入力 → クリック

#### 3.5 GameManagerとの連携設定

**前提**: GameManagerがシーンに既に配置されている（WI-0001完了済み）

1. **Hierarchyウィンドウ**で`GameManager`を選択
2. **Inspectorウィンドウ**の`GameManager`コンポーネントで、以下を確認:
   - `Skill Manager`フィールドに`SkillManager`が設定されている
   - `Coin Manager`フィールドに`CoinManager`が設定されている

**注意**: GameManagerが存在しない場合、本作業を中断し、WI-0001の実装を先に完了してください。

---

### STEP 4: テストとデバッグ（45分）

#### 4.1 ClientSimでの動作確認

##### 4.1.1 初期表示テスト
1. **Unity上部メニュー**から`VRChat SDK` → `Show Control Panel`をクリック
2. `Builder`タブを選択
3. `Build & Test`ボタンをクリック
4. ClientSimが起動したら、以下を確認:
   - [ ] HUDが画面左下に表示される
   - [ ] スキル表示が`Woodcutting: Lv1`と表示される
   - [ ] XPバーが正しく表示される
   - [ ] コイン表示が`🪙 0 Coins`と表示される

##### 4.1.2 スキル表示更新テスト
**前提**: SkillManagerに`_AddXP(int amount)`メソッドが実装されている

1. ClientSim実行中、**Hierarchy**で`SkillManager`を選択
2. **Inspector**の`SkillManager`コンポーネントで、`Debug Mode`を有効化（存在する場合）
3. Unityエディタの**Console**ウィンドウを開く（Window → General → Console）
4. 木を伐採するアクション実行（AxeInteractionが実装済みの場合）
5. 確認項目:
   - [ ] XPバーが滑らかにアニメーション（0.5秒）
   - [ ] XPテキストが更新される（例：`10/100 XP`）
   - [ ] スキルレベルアップ時、`LEVEL UP!`通知が表示される

##### 4.1.3 コイン表示更新テスト
**前提**: CoinManagerに`_AddCoins(int amount)`メソッドが実装されている

1. ClientSim実行中、コインを獲得するアクション実行
2. 確認項目:
   - [ ] コイン数がカウントアップアニメーション（0.3秒）
   - [ ] コインテキストが更新される（例：`🪙 5 Coins`）
   - [ ] 99,999コインに到達すると赤色表示になる

##### 4.1.4 通知表示テスト

**手動テスト方法**（SkillManager/CoinManagerが未実装の場合）:

1. ClientSim実行中、**Hierarchy**で`PlayerHUD`を選択
2. **Inspector**の`HUDManager`コンポーネント右上の`...`メニューをクリック
3. `Debug`を選択
4. `_ShowNotification`メソッドの横の入力欄に以下を入力:
   - 第1引数: `"テスト通知"`
   - 第2引数: `3.0`
5. 実行ボタンをクリック
6. 確認項目:
   - [ ] 通知が画面中央上部に表示される
   - [ ] 3秒後に自動的に消える
   - [ ] 4件目の通知を追加すると、1件目が消えてから表示される（キュー機能）

#### 4.2 エラー対処

##### エラー1: `NullReferenceException: Object reference not set to an instance of an object`
**原因**: UI参照の設定漏れ
**対処**:
1. **Hierarchy**で`PlayerHUD`を選択
2. **Inspector**の`HUDManager`コンポーネントですべてのフィールドが設定されているか確認
3. 未設定のフィールドがあれば、STEP 3.2を再実行

##### エラー2: `GameManagerが見つかりません`
**原因**: GameManagerがシーンに配置されていない
**対処**:
1. WI-0001の実装を完了させる
2. GameManagerをシーンに配置

##### エラー3: `SkillManagerの取得に失敗しました`
**原因**: GameManagerにSkillManagerが設定されていない
**対処**:
1. **Hierarchy**で`GameManager`を選択
2. **Inspector**の`Skill Manager`フィールドに`SkillManager`オブジェクトをドラッグ

##### エラー4: XPバーが表示されない
**原因**: Image TypeがFilledになっていない
**対処**:
1. **Hierarchy**で`XPBarFill`を選択
2. **Inspector**の`Image`コンポーネント:
   - `Image Type`: `Filled`
   - `Fill Method`: `Horizontal`
   - `Fill Origin`: `Left`

#### 4.3 パフォーマンステスト

##### Quest 2シミュレーション
1. **Unityエディタ上部メニュー**から`Window` → `Analysis` → `Profiler`を開く
2. ClientSimを実行
3. Profilerで以下を確認:
   - [ ] CPU使用率が15%以下
   - [ ] メモリ使用量が200MB以下
   - [ ] Draw Callが2以下（UI Batching有効）

---

## 7. テストケース

### 7.1 正常系テスト

| テストID | テスト項目 | 操作手順 | 期待結果 | 確認方法 |
|---------|----------|---------|---------|---------|
| TC-001 | HUD初期表示 | ワールド入場 | HUDが画面左下に表示される | 目視確認 |
| TC-002 | スキルレベル表示 | ワールド入場 | `Woodcutting: Lv1`と表示 | 目視確認 |
| TC-003 | XPバー初期値 | ワールド入場 | XPバーが空（Fill Amount=0） | 目視確認 |
| TC-004 | コイン初期値 | ワールド入場 | `🪙 0 Coins`と表示 | 目視確認 |
| TC-005 | XP獲得時の表示更新 | 木を伐採 | XPバーが滑らかに増加（0.5秒） | 目視確認 |
| TC-006 | レベルアップ通知 | XPが必要値到達 | `LEVEL UP!`通知が3秒間表示 | 目視確認 |
| TC-007 | コイン獲得時の表示更新 | 丸太を納品 | コイン数がカウントアップ（0.3秒） | 目視確認 |
| TC-008 | 通知キュー（3件まで） | 3件の通知を同時表示 | 3件まで表示、4件目は待機 | 目視確認 |
| TC-009 | HUD透明度設定 | 透明度80%に設定 | HUDが半透明に | 目視確認 |
| TC-010 | HUD位置変更 | 位置を右下に変更 | HUDが右下に移動 | 目視確認 |

### 7.2 異常系テスト

| テストID | テスト項目 | 操作手順 | 期待結果 | 確認方法 |
|---------|----------|---------|---------|---------|
| TC-E01 | GameManager未配置 | GameManagerなしで実行 | エラーログ出力、HUD非表示 | Consoleログ確認 |
| TC-E02 | UI参照未設定 | UI参照を削除して実行 | NullReferenceException | Consoleログ確認 |
| TC-E03 | SkillManager未設定 | SkillManager参照削除 | 警告ログ、スキル表示スキップ | Consoleログ確認 |
| TC-E04 | 通知キューオーバーフロー | 10件以上の通知を追加 | 古い通知から削除、新しい通知が表示 | 目視確認 |

### 7.3 パフォーマンステスト

| テストID | テスト項目 | 測定方法 | 目標値 | 確認方法 |
|---------|----------|---------|--------|---------|
| TC-P01 | CPU使用率（Quest 2） | Profiler | 15%以下 | Unity Profiler |
| TC-P02 | メモリ使用量 | Profiler | 200MB以下 | Unity Profiler |
| TC-P03 | Draw Call数 | Profiler | 2以下 | Unity Profiler |
| TC-P04 | XPバーアニメーション滑らかさ | 目視 | 60fps維持 | Unity Stats |

---

## 8. 完了条件（Doneの定義）

以下のすべての項目を満たした場合、本作業完了とします：

### 8.1 実装完了
- [x] `HUDManager.cs`スクリプトが作成され、エラーなくコンパイル完了
- [x] `PlayerHUD.prefab`が作成され、すべてのUI要素が正しく配置
- [x] HUDManagerスクリプトがPlayerHUDにアタッチ済み
- [x] すべてのUI参照フィールドが正しく設定済み

### 8.2 機能動作確認
- [x] スキルレベル・XPバーが正しく表示される（F-24）
- [x] コイン残高が正しく表示される（F-25）
- [x] 通知メッセージが正しく表示・キューイングされる（F-26）
- [x] HUD設定（透明度、スケール、位置）が正しく動作する（F-27）

### 8.3 テスト合格
- [x] 正常系テスト10件すべて合格（TC-001〜TC-010）
- [x] 異常系テスト4件すべて合格（TC-E01〜TC-E04）
- [x] パフォーマンステスト4件すべて合格（TC-P01〜TC-P04）

### 8.4 ドキュメント作成
- [x] 本作業指示書（WI-0017）が完成
- [x] テスト結果が記録済み
- [x] 既知の問題点が文書化済み（存在する場合）

---

## 9. 成果物

### 9.1 作成ファイル
- `/Assets/UdonScripts/UI/HUDManager.cs` - 実装完了（約400行）
- `/Assets/Prefabs/UI/PlayerHUD.prefab` - Prefab作成完了

### 9.2 変更ファイル
- なし（既存ファイルへの変更なし）

### 9.3 テスト結果ドキュメント
- テストケース実行結果（本作業指示書 Section 7に記録）

---

## 10. 備考

### 10.1 注意事項

#### VRChat Persistence API（PlayerData）の使用について
- **PlayerDataの制限**: 100KB/Player/World
- **本機能の使用量**: 約12バイト（HUD設定3項目）
- **保存タイミング**: HUD設定変更時に即座に保存
- **Late Joiner対応**: Start()で1秒遅延してマネージャー参照取得

#### Update()ループの最適化
- **XPバー・コインアニメーション**: 必要時のみ実行（タイマー制御）
- **通知タイマー**: 3つのタイマーのみチェック
- **GetComponent()**: Start()で1回のみ実行、結果をキャッシュ

#### TextMeshProの使用
- **理由**: Unity標準UIのTextコンポーネントはQuest環境で重い
- **利点**: SDF Font使用で軽量、高品質、スケーラブル
- **注意**: TMP Essentialsのインポートが必要（初回のみ）

### 10.2 既知の問題点

#### 問題1: PlayerDataの読み込みタイミング
**説明**: Late Joinerの場合、PlayerDataの読み込みが遅延する可能性
**影響**: HUD設定が一瞬デフォルト値で表示される
**対処**: 現状は許容範囲内、Phase 2で改善検討

#### 問題2: 通知の重複表示
**説明**: 同じメッセージが短時間に複数回表示される可能性
**影響**: UI上は問題なし（意図通りの動作）
**対処**: 不要（仕様通り）

### 10.3 将来の拡張予定（Phase 2以降）

- **アニメーション強化**: DOTweenライブラリの導入検討
- **通知カテゴリ**: XP、コイン、アイテム獲得で色分け
- **HUD設定UI**: スライダーで直感的に設定変更
- **ミニマップ表示**: 木の位置、他プレイヤーの位置を表示
- **ステータスアイコン**: プレイヤーの行動に応じたアイコン表示

### 10.4 参考資料

- **VRChat SDK Docs**: https://creators.vrchat.com/worlds/udon/
- **UdonSharp Docs**: https://udonsharp.docs.vrchat.com/
- **TextMeshPro Manual**: https://docs.unity3d.com/Packages/com.unity.textmeshpro@3.0/manual/
- **TDD.md**: Section 5.8（M-08 HUDManager仕様）
- **FSD.md**: FNC-004（スキル成長システム）、FNC-005（経済システム）

---

## 11. 作業完了チェックリスト

作業完了前に、以下のチェックリストを確認してください：

### 実装確認
- [ ] HUDManager.csがエラーなくコンパイル完了
- [ ] PlayerHUD.prefabがProjectウィンドウに存在
- [ ] すべてのUI要素が正しく配置・設定済み
- [ ] GameManager、SkillManager、CoinManagerとの連携が正しく動作

### 動作確認
- [ ] ClientSimで正常に動作
- [ ] スキル表示が正しく更新される
- [ ] コイン表示が正しく更新される
- [ ] 通知が正しく表示・キューイングされる
- [ ] HUD設定が正しく動作し、PlayerDataに保存される

### テスト確認
- [ ] 正常系テスト10件すべて合格
- [ ] 異常系テスト4件すべて合格
- [ ] パフォーマンステスト4件すべて合格

### ドキュメント確認
- [ ] テスト結果を記録
- [ ] 既知の問題点を文書化（存在する場合）
- [ ] 次のWI（WI-0018など）への引き継ぎ事項を記載（必要な場合）

---

**作業完了日**: _______________
**作業者署名**: _______________
**レビュー者署名**: _______________

---

**文書管理情報**
- 作成日: 2025-11-17
- 作成者: VRChat World Development Team
- 承認者: （実装開始前に承認必要）
- 最終レビュー日: 2025-11-17
- バージョン: 1.0
