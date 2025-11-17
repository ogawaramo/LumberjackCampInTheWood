# 作業指示書: WI-0018 - HUD Settings Implementation

**作成日**: 2025-11-17
**Phase**: Phase 2（補助機能・Week 4）
**Priority**: 低
**推定工数**: 0.5日

---

## 1. 作業ID

**WI-0018**

---

## 2. 作業名

HUD Settings実装 - HUD表示のカスタマイズ機能（透明度・サイズ・位置調整）

---

## 3. 作業目的

プレイヤーがHUD（ヘッドアップディスプレイ）の表示設定をカスタマイズできる機能を提供し、以下を実現する：

- HUDの透明度調整（0.0～1.0）により、画面の視認性を向上
- HUDのサイズ調整（0.5x～1.5x）により、プレイヤーの好みに合わせた表示
- HUDの位置プリセット（4隅）により、VR/デスクトップでの最適配置
- 設定の永続化により、次回訪問時も設定が維持される
- リアルタイムプレビュー機能により、設定変更の即座の確認が可能

これにより、すべてのプレイヤーが自分の環境・好みに合わせてHUDを最適化でき、ゲーム体験が向上する。

---

## 4. 対象

### 対象システム
- 森のきこりキャンプ - VRChatワールド

### 対象モジュール
- **M-08: HUDManager Extension** (UI Layer)

### 担当機能
- **F-27**: HUD Settings（透明度、サイズ、位置調整）

### 対象ファイルパス
- `/Assets/UdonSharp/Scripts/UI/HUDSettings.cs` (新規作成)
- `/Assets/UdonSharp/Scripts/UI/HUDManager.cs` (参照・連携)

---

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0以上
- UdonSharp 1.1.9 (stable)

### 依存作業
- **WI-0017**: HUDManager実装が完了していること
  - HUDManager.Instanceでアクセス可能
  - HUD UIが正常に表示されている
- **WI-0003**: PersistenceManager実装が完了していること
  - PlayerData保存・読み込みが動作している

### ブランチ
- `feature/phase2-auxiliary-features`

### 初期状態
- HUDが画面左下に固定表示されている
- HUD設定UIは未実装

---

## 6. 入力

### 設定変更の入力

#### 透明度スライダー変更
```csharp
// 透明度値: 0.0（完全透明）～ 1.0（不透明）
float newAlpha = 0.9f; // デフォルト
```

#### サイズスライダー変更
```csharp
// サイズ倍率: 0.5x（50%）～ 1.5x（150%）
float newScale = 1.0f; // デフォルト（100%）
```

#### 位置プリセット選択
```csharp
// 位置プリセット: 0=TopLeft, 1=TopRight, 2=BottomLeft, 3=BottomRight
int positionPreset = 2; // デフォルト（BottomLeft）
```

#### リセットボタン
```csharp
// すべての設定をデフォルト値に戻す
ResetToDefaults();
```

---

## 7. 出力

### 設定適用時のログ出力

```
[HUDSettings] Transparency set to 0.9
[HUDSettings] Scale set to 1.0x
[HUDSettings] Position set to BottomLeft
[HUDSettings] Settings applied successfully
[HUDSettings] Settings saved to PlayerData
```

### 設定読み込み時のログ出力

```
[HUDSettings] Loading HUD settings from PlayerData...
[HUDSettings] Loaded: Transparency=0.9, Scale=1.0, Position=BottomLeft
[HUDSettings] Applied saved settings to HUD
```

### リセット時のログ出力

```
[HUDSettings] Resetting HUD settings to defaults...
[HUDSettings] Defaults applied: Transparency=0.9, Scale=1.0, Position=BottomLeft
```

---

## 8. 作業手順

### Step 1: HUDSettingsクラスの基本構造作成

**目的**: HUD設定を管理する独立したコンポーネントを作成し、SOLID原則（単一責任の原則）を遵守する

**手順**:
1. Unityで新規UdonSharpスクリプトを作成
   - メニュー: `Assets > Create > UdonSharp > UdonSharpScript`
   - ファイル名: `HUDSettings.cs`
   - 配置場所: `/Assets/UdonSharp/Scripts/UI/`

2. 以下の基本構造を実装:

```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon;

/// <summary>
/// HUD表示設定を管理するコンポーネント
/// 透明度・サイズ・位置のカスタマイズ機能を提供
/// </summary>
public class HUDSettings : UdonSharpBehaviour
{
    #region Singleton
    public static HUDSettings Instance { get; private set; }
    #endregion

    #region Settings Default Values
    [Header("Default Settings")]
    [Tooltip("デフォルトの透明度（0.0=完全透明、1.0=不透明）")]
    public float defaultTransparency = 0.9f;

    [Tooltip("デフォルトのスケール（1.0=100%）")]
    public float defaultScale = 1.0f;

    [Tooltip("デフォルトの位置プリセット（0=TopLeft, 1=TopRight, 2=BottomLeft, 3=BottomRight）")]
    public int defaultPositionPreset = 2; // BottomLeft
    #endregion

    #region Current Settings
    private float currentTransparency;
    private float currentScale;
    private int currentPositionPreset;
    #endregion

    #region UI References
    [Header("Settings UI References")]
    [Tooltip("設定パネルのCanvas（トグルで表示/非表示）")]
    public GameObject settingsPanel;

    [Tooltip("透明度スライダー")]
    public Slider transparencySlider;

    [Tooltip("サイズスライダー")]
    public Slider scaleSlider;

    [Tooltip("位置プリセットボタン（4つ）")]
    public Button[] positionButtons;

    [Tooltip("リセットボタン")]
    public Button resetButton;

    [Tooltip("適用ボタン")]
    public Button applyButton;

    [Tooltip("キャンセルボタン")]
    public Button cancelButton;

    [Tooltip("透明度表示テキスト")]
    public Text transparencyValueText;

    [Tooltip("サイズ表示テキスト")]
    public Text scaleValueText;
    #endregion

    #region HUD References
    [Header("HUD References")]
    [Tooltip("HUDManager参照")]
    public HUDManager hudManager;

    [Tooltip("HUDのRectTransform（位置・サイズ変更用）")]
    public RectTransform hudRectTransform;

    [Tooltip("HUDのCanvasGroup（透明度制御用）")]
    public CanvasGroup hudCanvasGroup;
    #endregion

    #region PlayerData Keys
    private const string KEY_TRANSPARENCY = "HUDTransparency";
    private const string KEY_SCALE = "HUDScale";
    private const string KEY_POSITION = "HUDPosition";
    #endregion

    #region Position Presets（画面サイズに対する相対位置）
    private readonly Vector2[] positionPresets = new Vector2[]
    {
        new Vector2(0.05f, 0.95f),   // TopLeft（左上5%, 上から5%）
        new Vector2(0.95f, 0.95f),   // TopRight（右から5%, 上から5%）
        new Vector2(0.05f, 0.05f),   // BottomLeft（左から5%, 下から5%）
        new Vector2(0.95f, 0.05f)    // BottomRight（右から5%, 下から5%）
    };

    private readonly Vector2[] positionAnchors = new Vector2[]
    {
        new Vector2(0, 1),   // TopLeft
        new Vector2(1, 1),   // TopRight
        new Vector2(0, 0),   // BottomLeft
        new Vector2(1, 0)    // BottomRight
    };
    #endregion

    void Start()
    {
        // Singleton確立
        if (Instance == null)
        {
            Instance = this;
            Debug.Log("[HUDSettings] Instance created");
        }
        else
        {
            Debug.LogError("[HUDSettings] Duplicate instance detected! Destroying...");
            Destroy(gameObject);
            return;
        }

        // 初期化
        InitializeSettings();
    }

    /// <summary>
    /// 設定の初期化（デフォルト値 or PlayerData読み込み）
    /// </summary>
    private void InitializeSettings()
    {
        // HUDManager参照チェック
        if (hudManager == null)
        {
            Debug.LogError("[HUDSettings] HUDManager reference is null!");
            return;
        }

        // HUD UI参照チェック
        if (hudRectTransform == null || hudCanvasGroup == null)
        {
            Debug.LogError("[HUDSettings] HUD UI references are missing!");
            return;
        }

        // 設定パネルを初期非表示
        if (settingsPanel != null)
        {
            settingsPanel.SetActive(false);
        }

        // PlayerDataから設定読み込み（Late Joiner対応のため1秒遅延）
        SendCustomEventDelayedSeconds(nameof(_LoadSettingsFromPlayerData), 1.0f);
    }
}
```

3. Unityシーンに配置:
   - Hierarchy: `HUDManager` の子オブジェクトとして `HUDSettings` GameObject作成
   - HUDSettingsスクリプトをアタッチ

**完了条件**:
- HUDSettings.Instanceが他スクリプトからアクセス可能
- 初期化処理が正常に実行される
- Consoleに「Instance created」が表示される

---

### Step 2: PlayerData読み込み・保存機能実装

**目的**: 設定を永続化し、次回訪問時に復元する

**手順**:
1. 以下のメソッドを追加:

```csharp
    /// <summary>
    /// PlayerDataから設定を読み込む（カスタムイベント - 1秒遅延呼び出し用）
    /// </summary>
    public void _LoadSettingsFromPlayerData()
    {
        Debug.Log("[HUDSettings] Loading HUD settings from PlayerData...");

        // PersistenceManagerが存在しない場合はデフォルト値使用
        if (GameManager.Instance == null || GameManager.Instance.GetPersistenceManager() == null)
        {
            Debug.LogWarning("[HUDSettings] PersistenceManager not available, using defaults");
            currentTransparency = defaultTransparency;
            currentScale = defaultScale;
            currentPositionPreset = defaultPositionPreset;
            ApplySettingsToHUD();
            return;
        }

        // PlayerDataから読み込み（デフォルト値をフォールバック）
        PersistenceManager pm = GameManager.Instance.GetPersistenceManager();
        currentTransparency = pm.GetFloat(KEY_TRANSPARENCY, defaultTransparency);
        currentScale = pm.GetFloat(KEY_SCALE, defaultScale);
        currentPositionPreset = pm.GetInt(KEY_POSITION, defaultPositionPreset);

        // 範囲チェック（不正な値を修正）
        currentTransparency = Mathf.Clamp(currentTransparency, 0.0f, 1.0f);
        currentScale = Mathf.Clamp(currentScale, 0.5f, 1.5f);
        currentPositionPreset = Mathf.Clamp(currentPositionPreset, 0, 3);

        Debug.Log($"[HUDSettings] Loaded: Transparency={currentTransparency}, Scale={currentScale}, Position={currentPositionPreset}");

        // HUDに適用
        ApplySettingsToHUD();

        // UI更新（設定パネルが開かれていなくてもスライダーの値を同期）
        UpdateUIValues();
    }

    /// <summary>
    /// 設定をPlayerDataに保存
    /// </summary>
    private void SaveSettingsToPlayerData()
    {
        if (GameManager.Instance == null || GameManager.Instance.GetPersistenceManager() == null)
        {
            Debug.LogWarning("[HUDSettings] PersistenceManager not available, cannot save settings");
            return;
        }

        PersistenceManager pm = GameManager.Instance.GetPersistenceManager();
        pm.SetFloat(KEY_TRANSPARENCY, currentTransparency);
        pm.SetFloat(KEY_SCALE, currentScale);
        pm.SetInt(KEY_POSITION, currentPositionPreset);

        Debug.Log("[HUDSettings] Settings saved to PlayerData");
    }

    /// <summary>
    /// HUDに設定を適用（透明度・サイズ・位置）
    /// </summary>
    private void ApplySettingsToHUD()
    {
        if (hudCanvasGroup == null || hudRectTransform == null)
        {
            Debug.LogError("[HUDSettings] Cannot apply settings - HUD references are null");
            return;
        }

        // 透明度適用（CanvasGroup.alphaを使用）
        hudCanvasGroup.alpha = currentTransparency;

        // サイズ適用（RectTransform.localScaleを使用）
        hudRectTransform.localScale = Vector3.one * currentScale;

        // 位置適用（プリセットに基づいて）
        ApplyPositionPreset(currentPositionPreset);

        Debug.Log($"[HUDSettings] Applied: Transparency={currentTransparency}, Scale={currentScale}, Position={currentPositionPreset}");
    }

    /// <summary>
    /// 位置プリセットをHUDに適用
    /// </summary>
    /// <param name="presetIndex">プリセット番号（0～3）</param>
    private void ApplyPositionPreset(int presetIndex)
    {
        if (presetIndex < 0 || presetIndex >= positionPresets.Length)
        {
            Debug.LogError($"[HUDSettings] Invalid position preset index: {presetIndex}");
            return;
        }

        // アンカー設定（四隅のいずれか）
        hudRectTransform.anchorMin = positionAnchors[presetIndex];
        hudRectTransform.anchorMax = positionAnchors[presetIndex];
        hudRectTransform.pivot = positionAnchors[presetIndex];

        // 位置設定（相対位置）
        Vector2 position = positionPresets[presetIndex];
        float canvasWidth = hudRectTransform.parent.GetComponent<RectTransform>().rect.width;
        float canvasHeight = hudRectTransform.parent.GetComponent<RectTransform>().rect.height;

        hudRectTransform.anchoredPosition = new Vector2(
            (position.x - positionAnchors[presetIndex].x) * canvasWidth,
            (position.y - positionAnchors[presetIndex].y) * canvasHeight
        );
    }
```

**参照**: `WI-0003_PersistenceManager.md` のPlayerData保存・読み込みパターン

**完了条件**:
- PlayerDataから設定が正しく読み込まれる
- 設定がPlayerDataに保存される
- 次回ワールド訪問時に設定が復元される

---

### Step 3: 設定UI実装（スライダー・ボタン）

**目的**: プレイヤーが直感的に設定を変更できるUIを実装する

**手順**:
1. Unity Editorで設定UIを作成:
   - Canvas（WorldSpace）を作成し、「HUDSettingsPanel」と命名
   - Panel（背景）を追加し、半透明の暗い背景色を設定
   - 以下のUI要素を配置:

```
HUDSettingsPanel (Canvas)
├─ Background (Panel - 半透明グレー)
│
├─ Title (Text: "HUD設定")
│
├─ TransparencySection
│  ├─ Label (Text: "透明度")
│  ├─ Slider (0.0 - 1.0)
│  └─ ValueText (Text: "90%")
│
├─ ScaleSection
│  ├─ Label (Text: "サイズ")
│  ├─ Slider (0.5 - 1.5)
│  └─ ValueText (Text: "100%")
│
├─ PositionSection
│  ├─ Label (Text: "位置")
│  ├─ ButtonGrid (2x2)
│  │  ├─ TopLeftButton (Button: "左上")
│  │  ├─ TopRightButton (Button: "右上")
│  │  ├─ BottomLeftButton (Button: "左下")
│  │  └─ BottomRightButton (Button: "右下")
│
└─ ButtonRow
   ├─ ResetButton (Button: "リセット")
   ├─ ApplyButton (Button: "適用")
   └─ CancelButton (Button: "キャンセル")
```

2. 以下のUIイベントハンドラを実装:

```csharp
    /// <summary>
    /// 透明度スライダー変更時
    /// </summary>
    public void _OnTransparencySliderChanged()
    {
        if (transparencySlider == null) return;

        currentTransparency = transparencySlider.value;

        // リアルタイムプレビュー（即座に反映）
        if (hudCanvasGroup != null)
        {
            hudCanvasGroup.alpha = currentTransparency;
        }

        // UI表示更新
        if (transparencyValueText != null)
        {
            transparencyValueText.text = $"{Mathf.RoundToInt(currentTransparency * 100)}%";
        }

        Debug.Log($"[HUDSettings] Transparency preview: {currentTransparency}");
    }

    /// <summary>
    /// サイズスライダー変更時
    /// </summary>
    public void _OnScaleSliderChanged()
    {
        if (scaleSlider == null) return;

        currentScale = scaleSlider.value;

        // リアルタイムプレビュー（即座に反映）
        if (hudRectTransform != null)
        {
            hudRectTransform.localScale = Vector3.one * currentScale;
        }

        // UI表示更新
        if (scaleValueText != null)
        {
            scaleValueText.text = $"{Mathf.RoundToInt(currentScale * 100)}%";
        }

        Debug.Log($"[HUDSettings] Scale preview: {currentScale}x");
    }

    /// <summary>
    /// 位置プリセットボタンクリック時（汎用）
    /// </summary>
    /// <param name="presetIndex">プリセット番号（0～3）</param>
    public void _OnPositionPresetClicked(int presetIndex)
    {
        currentPositionPreset = presetIndex;

        // リアルタイムプレビュー（即座に反映）
        ApplyPositionPreset(currentPositionPreset);

        // ボタンのハイライト更新
        UpdatePositionButtonHighlight();

        string[] positionNames = { "左上", "右上", "左下", "右下" };
        Debug.Log($"[HUDSettings] Position preview: {positionNames[presetIndex]}");
    }

    // 各位置ボタン用のカスタムイベント（UdonSharp用）
    public void _OnTopLeftClicked() { _OnPositionPresetClicked(0); }
    public void _OnTopRightClicked() { _OnPositionPresetClicked(1); }
    public void _OnBottomLeftClicked() { _OnPositionPresetClicked(2); }
    public void _OnBottomRightClicked() { _OnPositionPresetClicked(3); }

    /// <summary>
    /// リセットボタンクリック時
    /// </summary>
    public void _OnResetButtonClicked()
    {
        Debug.Log("[HUDSettings] Resetting HUD settings to defaults...");

        // デフォルト値に戻す
        currentTransparency = defaultTransparency;
        currentScale = defaultScale;
        currentPositionPreset = defaultPositionPreset;

        // HUDに適用
        ApplySettingsToHUD();

        // UI更新
        UpdateUIValues();

        Debug.Log("[HUDSettings] Defaults applied");
    }

    /// <summary>
    /// 適用ボタンクリック時
    /// </summary>
    public void _OnApplyButtonClicked()
    {
        Debug.Log("[HUDSettings] Applying settings...");

        // 既にリアルタイムプレビューで適用済みのため、保存のみ
        SaveSettingsToPlayerData();

        // 設定パネルを閉じる
        CloseSettingsPanel();

        Debug.Log("[HUDSettings] Settings applied and saved");
    }

    /// <summary>
    /// キャンセルボタンクリック時
    /// </summary>
    public void _OnCancelButtonClicked()
    {
        Debug.Log("[HUDSettings] Settings cancelled");

        // 保存済みの設定を再読み込み（プレビュー変更を破棄）
        _LoadSettingsFromPlayerData();

        // 設定パネルを閉じる
        CloseSettingsPanel();
    }

    /// <summary>
    /// UI値を現在の設定に更新
    /// </summary>
    private void UpdateUIValues()
    {
        if (transparencySlider != null)
        {
            transparencySlider.value = currentTransparency;
        }

        if (scaleSlider != null)
        {
            scaleSlider.value = currentScale;
        }

        if (transparencyValueText != null)
        {
            transparencyValueText.text = $"{Mathf.RoundToInt(currentTransparency * 100)}%";
        }

        if (scaleValueText != null)
        {
            scaleValueText.text = $"{Mathf.RoundToInt(currentScale * 100)}%";
        }

        UpdatePositionButtonHighlight();
    }

    /// <summary>
    /// 位置ボタンのハイライト更新
    /// </summary>
    private void UpdatePositionButtonHighlight()
    {
        if (positionButtons == null || positionButtons.Length != 4) return;

        for (int i = 0; i < positionButtons.Length; i++)
        {
            if (positionButtons[i] == null) continue;

            // 選択中のボタンを強調表示（色変更）
            ColorBlock colors = positionButtons[i].colors;
            colors.normalColor = (i == currentPositionPreset) ? Color.green : Color.white;
            positionButtons[i].colors = colors;
        }
    }
```

3. Unity Editorでイベント接続:
   - `TransparencySlider` の `OnValueChanged` → `HUDSettings._OnTransparencySliderChanged()`
   - `ScaleSlider` の `OnValueChanged` → `HUDSettings._OnScaleSliderChanged()`
   - 各位置ボタンの `OnClick` → それぞれのカスタムイベント
   - `ResetButton` の `OnClick` → `HUDSettings._OnResetButtonClicked()`
   - `ApplyButton` の `OnClick` → `HUDSettings._OnApplyButtonClicked()`
   - `CancelButton` の `OnClick` → `HUDSettings._OnCancelButtonClicked()`

**重要**: UdonSharpでは、Unityイベント（OnValueChanged, OnClickなど）は **パブリックメソッド** かつ **パラメータなし** である必要があるため、上記のようにカスタムイベント名を使用します。

**完了条件**:
- スライダー操作でリアルタイムにHUDが変化する
- ボタンクリックで位置が変更される
- UI表示が現在の設定値を正しく反映している

---

### Step 4: 設定パネルの開閉機能実装

**目的**: VR/デスクトップ両対応の設定パネル表示・非表示機能を実装する

**手順**:
1. 以下のメソッドを追加:

```csharp
    /// <summary>
    /// 設定パネルを開く
    /// </summary>
    public void OpenSettingsPanel()
    {
        if (settingsPanel == null) return;

        settingsPanel.SetActive(true);
        Debug.Log("[HUDSettings] Settings panel opened");

        // 現在の設定値をUIに反映
        UpdateUIValues();
    }

    /// <summary>
    /// 設定パネルを閉じる
    /// </summary>
    public void CloseSettingsPanel()
    {
        if (settingsPanel == null) return;

        settingsPanel.SetActive(false);
        Debug.Log("[HUDSettings] Settings panel closed");
    }

    /// <summary>
    /// 設定パネルのトグル（開いている場合は閉じる、閉じている場合は開く）
    /// </summary>
    public void ToggleSettingsPanel()
    {
        if (settingsPanel == null) return;

        if (settingsPanel.activeSelf)
        {
            CloseSettingsPanel();
        }
        else
        {
            OpenSettingsPanel();
        }
    }
```

2. VR/デスクトップ入力対応の実装:

**オプション1: ワールドUIボタンから開く**
- キャンプ中央の看板やメニューにボタンを配置
- ボタンクリック → `HUDSettings.Instance.OpenSettingsPanel()`

**オプション2: VRメニューから開く**
- VRChatのAction Menu（Radial Menu）にカスタムメニュー追加
- メニュー選択 → 設定パネル表示
- 参考: `VRChat SDK > Examples > ActionMenuAPI`

**完了条件**:
- 設定パネルが正しく開閉する
- VR/デスクトップ両方でアクセス可能
- パネル表示時に現在の設定値が反映されている

---

### Step 5: 入力バリデーションとエラーハンドリング

**目的**: 不正な設定値を防ぎ、エラー時の安定性を確保する

**手順**:
1. 以下のバリデーション機能を実装:

```csharp
    /// <summary>
    /// 設定値のバリデーション
    /// </summary>
    /// <returns>すべての設定が有効な場合はtrue</returns>
    private bool ValidateSettings()
    {
        bool isValid = true;

        // 透明度チェック（0.0～1.0）
        if (currentTransparency < 0.0f || currentTransparency > 1.0f)
        {
            Debug.LogWarning($"[HUDSettings] Invalid transparency: {currentTransparency}, clamping to 0.0-1.0");
            currentTransparency = Mathf.Clamp(currentTransparency, 0.0f, 1.0f);
            isValid = false;
        }

        // スケールチェック（0.5～1.5）
        if (currentScale < 0.5f || currentScale > 1.5f)
        {
            Debug.LogWarning($"[HUDSettings] Invalid scale: {currentScale}, clamping to 0.5-1.5");
            currentScale = Mathf.Clamp(currentScale, 0.5f, 1.5f);
            isValid = false;
        }

        // 位置プリセットチェック（0～3）
        if (currentPositionPreset < 0 || currentPositionPreset > 3)
        {
            Debug.LogWarning($"[HUDSettings] Invalid position preset: {currentPositionPreset}, resetting to default");
            currentPositionPreset = defaultPositionPreset;
            isValid = false;
        }

        return isValid;
    }

    /// <summary>
    /// Null参照チェック（HUD UI要素）
    /// </summary>
    /// <returns>すべての必須参照が有効な場合はtrue</returns>
    private bool ValidateReferences()
    {
        if (hudRectTransform == null)
        {
            Debug.LogError("[HUDSettings] HUD RectTransform reference is null!");
            return false;
        }

        if (hudCanvasGroup == null)
        {
            Debug.LogError("[HUDSettings] HUD CanvasGroup reference is null!");
            return false;
        }

        return true;
    }
```

2. 各設定適用メソッドにバリデーションを追加:

```csharp
    /// <summary>
    /// 設定をPlayerDataに保存（バリデーション付き）
    /// </summary>
    private void SaveSettingsToPlayerData()
    {
        // バリデーション実行
        if (!ValidateSettings())
        {
            Debug.LogWarning("[HUDSettings] Settings were invalid and have been corrected before saving");
        }

        if (!ValidateReferences())
        {
            Debug.LogError("[HUDSettings] Cannot save settings - missing references");
            return;
        }

        if (GameManager.Instance == null || GameManager.Instance.GetPersistenceManager() == null)
        {
            Debug.LogWarning("[HUDSettings] PersistenceManager not available, cannot save settings");
            return;
        }

        PersistenceManager pm = GameManager.Instance.GetPersistenceManager();
        pm.SetFloat(KEY_TRANSPARENCY, currentTransparency);
        pm.SetFloat(KEY_SCALE, currentScale);
        pm.SetInt(KEY_POSITION, currentPositionPreset);

        Debug.Log("[HUDSettings] Settings saved to PlayerData");
    }
```

**完了条件**:
- 不正な設定値が自動補正される
- Null参照エラーが検出され、適切にログ出力される
- エラー時もゲームプレイが継続可能

---

### Step 6: HUDManagerとの統合

**目的**: HUDManagerから設定機能にアクセスできるようにする

**手順**:
1. `HUDManager.cs` に以下のコードを追加:

```csharp
public class HUDManager : UdonSharpBehaviour
{
    // 既存コード...

    [Header("Settings")]
    public HUDSettings hudSettings;

    void Start()
    {
        // 既存の初期化処理...

        // HUDSettings参照確認
        if (hudSettings == null)
        {
            hudSettings = HUDSettings.Instance;
        }

        if (hudSettings == null)
        {
            Debug.LogWarning("[HUDManager] HUDSettings not found - settings feature unavailable");
        }
        else
        {
            Debug.Log("[HUDManager] HUDSettings connected");
        }
    }

    // Getter for other modules
    public HUDSettings GetHUDSettings()
    {
        return hudSettings;
    }
}
```

2. Unityエディタで:
   - HUDManagerオブジェクトを選択
   - Inspector > HUDManager > HUD Settings フィールドに HUDSettingsオブジェクトをドラッグ&ドロップ
   - HUDSettingsオブジェクトを選択
   - Inspector > HUDSettings > HUD Manager フィールドに HUDManagerオブジェクトをドラッグ&ドロップ
   - HUD Rect Transform フィールドに HUDのRectTransformをドラッグ&ドロップ
   - HUD Canvas Group フィールドに HUDのCanvasGroupコンポーネントをドラッグ&ドロップ

3. HUDにCanvasGroupコンポーネントを追加（透明度制御用）:
   - HUDのルートGameObjectを選択
   - Inspector > Add Component > Canvas Group
   - 以下のプロパティを設定:
     - Alpha: 0.9（デフォルト透明度）
     - Interactable: ✓
     - Block Raycasts: ✓

**完了条件**:
- HUDManagerからHUDSettingsが参照可能
- HUDSettingsからHUDManager・HUD UI要素が参照可能
- Consoleに「HUDSettings connected」と表示される

---

## 9. 完了条件（Doneの条件）

以下のすべてを満たすこと：

- [ ] HUDSettingsクラスが実装され、Singletonとして動作する
- [ ] 透明度スライダー（0.0～1.0）が動作し、リアルタイムプレビューが機能する
- [ ] サイズスライダー（0.5x～1.5x）が動作し、リアルタイムプレビューが機能する
- [ ] 位置プリセットボタン（4隅）が動作し、即座に位置が変更される
- [ ] リセットボタンがデフォルト値に戻す機能を提供する
- [ ] 適用ボタンが設定をPlayerDataに保存する
- [ ] キャンセルボタンがプレビュー変更を破棄する
- [ ] 設定がPlayerDataに正しく保存される
- [ ] 次回ワールド訪問時に設定が復元される
- [ ] 設定パネルが正しく開閉する
- [ ] VR/デスクトップ両方で操作可能
- [ ] 不正な設定値が自動補正される
- [ ] HUDManagerと正しく統合されている
- [ ] Unity Consoleにエラーが表示されない（Warning以下は許容）
- [ ] VRChat ClientSimでテストが成功する

---

## 10. テスト観点／テストケース

### テスト種別
- **機能テスト**: Unity EditorでのPlay Mode Test
- **統合テスト**: VRChat ClientSimでの動作確認
- **Quest互換性テスト**: Quest向けビルドでの動作確認

### テストケース

#### 正常系テスト

**TC-001: 透明度変更のリアルタイムプレビュー**
- 手順:
  1. 設定パネルを開く
  2. 透明度スライダーを0.5に変更
  3. HUDの透明度を視覚確認
- 期待結果: HUDが即座に50%透明になる
- 確認方法: 視覚確認、Consoleに「Transparency preview: 0.5」が表示される

**TC-002: サイズ変更のリアルタイムプレビュー**
- 手順:
  1. 設定パネルを開く
  2. サイズスライダーを1.5に変更
  3. HUDのサイズを視覚確認
- 期待結果: HUDが即座に150%に拡大される
- 確認方法: 視覚確認、Consoleに「Scale preview: 1.5x」が表示される

**TC-003: 位置プリセット変更**
- 手順:
  1. 設定パネルを開く
  2. 「右上」ボタンをクリック
  3. HUDの位置を視覚確認
- 期待結果: HUDが画面右上に即座に移動する
- 確認方法: 視覚確認、Consoleに「Position preview: 右上」が表示される

**TC-004: 設定の保存と復元**
- 手順:
  1. 設定パネルを開く
  2. 透明度0.7、サイズ1.2、位置「左上」に変更
  3. 「適用」ボタンをクリック
  4. ワールドを退出
  5. 再度ワールドに入場
  6. HUDの設定を確認
- 期待結果: 前回の設定（透明度0.7、サイズ1.2、位置「左上」）が復元されている
- 確認方法: 視覚確認、Consoleに「Loaded: Transparency=0.7, Scale=1.2, Position=0」が表示される

**TC-005: リセットボタン**
- 手順:
  1. 設定パネルを開く
  2. 透明度0.3、サイズ0.5に変更
  3. 「リセット」ボタンをクリック
  4. HUDの設定を確認
- 期待結果: デフォルト値（透明度0.9、サイズ1.0、位置「左下」）に戻る
- 確認方法: 視覚確認、Consoleに「Defaults applied」が表示される

**TC-006: キャンセルボタン**
- 手順:
  1. 設定パネルを開く
  2. 透明度0.3に変更（プレビューで即座に反映）
  3. 「キャンセル」ボタンをクリック
  4. HUDの透明度を確認
- 期待結果: プレビュー変更が破棄され、保存済みの設定（0.9）に戻る
- 確認方法: 視覚確認、Consoleに「Settings cancelled」が表示される

#### 異常系テスト

**TC-007: 不正な透明度値の自動補正**
- 手順:
  1. 手動でcurrentTransparencyに -0.5 を設定（スクリプト直接編集）
  2. ValidateSettings()を実行
  3. currentTransparencyの値を確認
- 期待結果: 0.0に自動補正される
- 確認方法: Consoleに「Invalid transparency: -0.5, clamping to 0.0-1.0」が表示される

**TC-008: PersistenceManager未利用時のフォールバック**
- 手順:
  1. PersistenceManagerを無効化
  2. ワールド入場
  3. HUDの設定を確認
- 期待結果: デフォルト値が適用される
- 確認方法: Consoleに「PersistenceManager not available, using defaults」が表示される

**TC-009: Null参照エラーハンドリング**
- 手順:
  1. HUDRectTransformの参照を削除
  2. ApplySettingsToHUD()を実行
  3. Consoleを確認
- 期待結果: エラーログが出力され、処理が安全に中断される
- 確認方法: Consoleに「Cannot apply settings - HUD references are null」が表示される

#### パフォーマンステスト

**TC-010: Quest環境でのスライダー操作レスポンス**
- 手順:
  1. Quest向けビルドを作成
  2. Quest 2デバイスでワールド入場
  3. 設定パネルを開く
  4. スライダーを連続して素早く操作
- 期待結果: 60fpsを維持し、遅延なくプレビューが更新される
- 確認方法: Quest 2の開発者モードでfpsを確認

---

## 11. 成果物

### 変更ファイル
- `/Assets/UdonSharp/Scripts/UI/HUDSettings.cs` (新規作成、約350行)
- `/Assets/UdonSharp/Scripts/UI/HUDManager.cs` (参照追加、約10行追加)

### 追加UIプレハブ
- `/Assets/Prefabs/UI/HUDSettingsPanel.prefab` (設定パネルUI)

### 更新したドキュメント
- なし（本作業指示書が完了後のドキュメントとなる）

---

## 12. 備考

### 注意点

#### VRChat UI制約
- **WorldSpace Canvas**: 設定パネルはWorldSpace Canvasとして実装すること（VRで操作可能にするため）
- **レイキャスト**: VRコントローラーのレーザーポインターで操作可能なように、CanvasのRender Modeを「World Space」に設定
- **距離**: プレイヤーから1.5～2m程度の距離に配置（VRで読みやすい距離）

#### CanvasGroup vs Material.alpha
- **推奨**: CanvasGroup.alphaを使用（すべての子要素に一括適用）
- **非推奨**: Material.alphaを個別設定（パフォーマンス低下、管理複雑化）

#### RectTransform位置調整
- **アンカー設定**: 四隅に固定することで、画面解像度変更に対応
- **相対位置**: パーセンテージベースで位置を保存（絶対座標は避ける）

#### Quest最適化
- **UI Draw Call**: 設定パネルは1つのCanvasにまとめ、Draw Callを最小化
- **テクスチャ**: UIテクスチャは512x512以下、ASTC圧縮
- **フォント**: Dynamic Fontは避け、Bitmap Fontを使用（パフォーマンス向上）

#### UdonSharpイベントハンドリング
- **命名規則**: カスタムイベントは `_` プレフィックス + PascalCase（例: `_OnApplyButtonClicked()`）
- **パラメータ**: Unityイベント（OnClick, OnValueChangedなど）はパラメータなしメソッドのみ対応
- **遅延呼び出し**: SendCustomEventDelayedSecondsは、Late Joiner対応に有効

#### PlayerData保存タイミング
- **推奨**: 「適用」ボタンクリック時のみ保存（頻繁な保存を避ける）
- **非推奨**: スライダー変更ごとに保存（PlayerData書き込み負荷）

### パフォーマンス考慮

**UI更新頻度**
- リアルタイムプレビュー: スライダー変更時のみ更新
- バリデーション: 設定適用時のみ実行
- PlayerData保存: 「適用」ボタンクリック時のみ

**メモリ使用量**
- 設定パネルUI: 非表示時はアクティブ状態だがレイキャストオフ
- テクスチャキャッシング: UI要素のGetComponent()をStart()で1回のみ実行

### セキュリティ

**設定値の範囲制限**
- 透明度: 0.0～1.0（視認不可を防ぐため最小0.3を推奨）
- スケール: 0.5～1.5（極端なサイズ変更を防ぐ）
- 位置: プリセット4種のみ（自由配置は不可）

**PlayerDataサイズ**
- 本機能の使用量: 約12バイト（float × 2 + int × 1）
- 全体制限: 70KB以内（PersistenceManagerで管理）

### デバッグ方法

**ClientSimでのテスト**
```
1. Unity EditorでPlay Mode起動
2. ClientSimで設定パネルを開く
3. スライダー操作でHUD変化を確認
4. Consoleでログ確認
```

**VRChatクライアントでのテスト**
```
1. ワールドをビルド・アップロード
2. VRChatクライアントで入場
3. 設定パネルを開く（VRメニューまたはワールドUI）
4. VRコントローラーで操作確認
5. 一度退出し、再入場して設定復元を確認
```

### トラブルシューティング

**問題**: スライダーが動かない
```
解決法:
1. Sliderコンポーネントの設定確認（Min/Max値）
2. OnValueChangedイベントが正しく接続されているか確認
3. UdonSharpコンパイルエラーがないか確認
```

**問題**: 設定が保存されない
```
解決法:
1. PersistenceManagerが正常に動作しているか確認
2. PlayerDataキーが正しいか確認（重複チェック）
3. OnPlayerRestoredが呼ばれているか確認（Late Joiner対応）
```

**問題**: VRで設定パネルが操作できない
```
解決法:
1. CanvasのRender Modeが「World Space」になっているか確認
2. GraphicRaycasterコンポーネントが有効か確認
3. パネルの距離・角度が適切か確認（1.5～2m、正面向き）
```

### 将来の拡張

**Phase 3以降の追加機能候補**
- HUD要素の個別表示/非表示切り替え
- カスタムカラーテーマ
- アニメーション速度調整
- フォントサイズ調整
- 自由配置モード（ドラッグ&ドロップ）

### 参考ドキュメント
- `TDD.md` Section 5.8（Module M-08定義）
- `FSD.md` FNC-005（経済システム - ショップUI参考）
- `WI-0017_HUDManager.md`（HUDManager実装）
- `WI-0003_PersistenceManager.md`（PlayerData保存・読み込み）
- `REF.md` Section 2.3.4（UdonSharpネットワーク同期 - PlayerData使用例）
- 公式: https://creators.vrchat.com/worlds/udon/persistence/player-data/

---

**作業開始前チェックリスト**:
- [ ] WI-0017（HUDManager）が完了している
- [ ] WI-0003（PersistenceManager）が完了している
- [ ] Unity 2022.3.22f1 LTS環境が準備されている
- [ ] VRChat SDK 3.9.0以上がインストールされている
- [ ] UdonSharp 1.1.9がインストールされている
- [ ] VRChat ClientSimがセットアップされている

**作業完了後チェックリスト**:
- [ ] すべてのテストケースが成功している
- [ ] Unity Consoleにエラーがない（Warning以下は許容）
- [ ] ClientSimでの動作確認が完了している
- [ ] HUDManagerとの統合が確認されている
- [ ] 本作業指示書の「完了条件」がすべて満たされている
- [ ] Quest向けビルドでのテストが完了している

---

**承認**:
- [ ] 技術レビュー承認
- [ ] コードレビュー承認
- [ ] 動作テスト承認
- [ ] Phase 2実装への統合承認
