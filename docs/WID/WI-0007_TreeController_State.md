# 作業指示書（Work Instruction）

## 1. 作業ID
**WI-0007**

## 2. 作業名
**TreeController State Machine実装 - 木の状態管理とネットワーク同期**

## 3. 作業目的
木オブジェクトの状態管理（HP、倒木フラグ、リスポーンタイマー）をネットワーク同期（Manual Sync）で実装し、マルチプレイヤー環境での木の状態の一貫性を保証する。Master Clientでのみダメージ処理を実行し、すべてのクライアントに状態変化を同期する。

---

## 4. 対象

- **対象システム**: 森のきこりキャンプ（Woodcutter Camp）VRChatワールド Phase 1
- **対象モジュール**: M-10 TreeController（Gameplay Layer）
- **対象ファイルパス**:
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`（新規作成）
  - `/Assets/WoodcutterCamp/ScriptableObjects/TreeConfig.asset`（新規作成）

---

## 5. 前提条件

### 環境要件
- Unity 2022.3.22f1
- VRChat SDK 3.9.0
- UdonSharp 1.1.9

### 完了している依存作業
- **WI-0001**: GameManager Singleton実装完了
- **WI-0002**: NetworkSync Batching System実装完了

### 初期状態
- Unityプロジェクトが開いている
- VRChat SDKがインストール済み
- シーンに木オブジェクト（Pine/Oak/Ancient）が配置済み
- 各木オブジェクトにはColliderが設定済み

### ブランチ
- `feature/woodcutting-system`（既存ブランチから派生）

---

## 6. 入力

### AxeInteractionからのダメージ入力
```csharp
// M-11 AxeInteractionから呼び出される
public void _TakeDamage(int damage, VRCPlayerApi attacker);
```

**パラメータ**:
- `damage`: ダメージ量（1〜1000の整数）
- `attacker`: 攻撃したプレイヤー（VRCPlayerApi）

### 想定するダメージ値
- 基礎ダメージ10（木の斧）× スキル倍率（1.0〜1.5）= **10〜15ダメージ/回**
- クリティカル発生時: **20〜30ダメージ/回**
- 協力伐採ボーナス時: **+20%**

---

## 7. 出力

### 状態変化イベント
1. **HP減少時**: HP更新、ネットワーク同期
2. **HP=0時**: 倒木アニメーション開始イベント発火
3. **倒木完了時**: LogSpawnerへの丸太生成イベント通知
4. **リスポーン時**: 木の再生成、HP満タン復元

### ネットワーク同期データ
```csharp
[UdonSynced] private int currentHP;        // 現在HP（0〜100）
[UdonSynced] private bool isFallen;        // 倒木フラグ
[UdonSynced] private float respawnTimer;   // リスポーンまでの残り時間（秒）
```

### UIへの出力
- 木の上部にHPバー表示（WorldSpace Canvas）
- ダメージ数値表示（「-12」など）

---

## 8. 作業手順

### 手順1: TreeConfigScriptableObject作成
**目的**: 木の種類ごとの設定データを管理する

**操作手順**:
1. Unityエディタで、Projectウィンドウを開く
2. `/Assets/WoodcutterCamp/ScriptableObjects/`フォルダを右クリック
3. `Create > ScriptableObject > TreeConfig`を選択（後述のスクリプト作成後）
4. 3種類のTreeConfigを作成:
   - `PineTreeConfig.asset`
   - `OakTreeConfig.asset`
   - `AncientTreeConfig.asset`

**TreeConfig.cs スクリプト**:
```csharp
using UnityEngine;

[CreateAssetMenu(fileName = "TreeConfig", menuName = "WoodcutterCamp/TreeConfig", order = 1)]
public class TreeConfig : ScriptableObject
{
    [Header("基本設定")]
    public string treeName;           // 木の名前（"Pine", "Oak", "Ancient"）
    public int maxHP;                 // 最大HP
    public float respawnTime;         // リスポーン時間（秒）

    [Header("丸太生成設定")]
    public int minLogCount;           // 最小丸太生成数
    public int maxLogCount;           // 最大丸太生成数

    [Header("報酬設定")]
    public int baseXP;                // 基礎経験値
    public int baseCoins;             // 基礎コイン
}
```

**設定値（PRDv3.md Section 2.1.1参照）**:
- **Pine（小）**: maxHP=30, respawnTime=180秒, minLog=1, maxLog=2, baseXP=10, baseCoins=3
- **Oak（中）**: maxHP=60, respawnTime=300秒, minLog=2, maxLog=3, baseXP=20, baseCoins=5
- **Ancient（大）**: maxHP=100, respawnTime=600秒, minLog=3, maxLog=4, baseXP=40, baseCoins=10

**確認方法**:
- Inspectorウィンドウで各TreeConfigの値が正しく設定されているか確認
- スクリプトエラーがないことをConsoleで確認

---

### 手順2: TreeController.cs UdonSharpスクリプト作成
**目的**: 木の状態管理とネットワーク同期を実装

**操作手順**:
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/`フォルダを作成
2. フォルダ内で右クリック → `Create > UdonSharp > UdonSharpBehaviour Script`
3. ファイル名を`TreeController.cs`に変更

**TreeController.cs 完全実装コード**:
```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;
using VRC.Udon.Common.Interfaces;

/// <summary>
/// Module M-10: TreeController
/// 木の状態管理（HP、倒木、リスポーン）とネットワーク同期を担当
/// </summary>
[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]
public class TreeController : UdonSharpBehaviour
{
    // ========================================
    // Inspector設定
    // ========================================
    [Header("木の設定")]
    public TreeConfig treeConfig;

    [Header("ビジュアル設定")]
    public GameObject standingVisual;     // 立木のビジュアル
    public GameObject fallenVisual;       // 倒木のビジュアル（非表示状態で配置）
    public Transform hpBarCanvas;         // HPバー用WorldSpace Canvas
    public UnityEngine.UI.Image hpBarFill; // HPバーのFill Image
    public TextMesh damageText;           // ダメージ数値表示用

    [Header("コンポーネントキャッシュ")]
    public Collider treeCollider;         // 木のコライダー
    public AudioSource audioSource;       // 効果音再生用

    [Header("効果音")]
    public AudioClip hitSound;            // ヒット音
    public AudioClip creakSound;          // きしみ音
    public AudioClip fallSound;           // 倒木音
    public AudioClip respawnSound;        // リスポーン音

    // ========================================
    // ネットワーク同期変数（Manual Sync）
    // ========================================
    [UdonSynced] private int currentHP;
    [UdonSynced] private bool isFallen;
    [UdonSynced] private float respawnTimer;

    // ========================================
    // 状態管理（ローカル）
    // ========================================
    private TreeState currentState;
    private bool isInitialized = false;
    private float nextUpdateTime = 0;

    // ========================================
    // 定数
    // ========================================
    private const float HP_BAR_UPDATE_INTERVAL = 0.1f;  // HPバー更新間隔
    private const float LOW_HP_THRESHOLD = 0.3f;        // 低HP警告閾値（30%以下）
    private const float DAMAGE_TEXT_DURATION = 1.0f;    // ダメージ表示時間

    /// <summary>
    /// 木の状態を表すenum
    /// </summary>
    private enum TreeState
    {
        Standing = 0,  // 立木（攻撃可能）
        Falling = 1,   // 倒木中（攻撃不可）
        Fallen = 2     // 倒木完了（攻撃可能、丸太化待ち）
    }

    // ========================================
    // 初期化
    // ========================================
    void Start()
    {
        Initialize();
    }

    private void Initialize()
    {
        if (isInitialized) return;

        // コンポーネントキャッシング（GetComponent最適化）
        if (treeCollider == null)
            treeCollider = GetComponent<Collider>();
        if (audioSource == null)
            audioSource = GetComponent<AudioSource>();

        // バリデーション
        if (treeConfig == null)
        {
            Debug.LogError("[TreeController] TreeConfigが設定されていません！", this);
            return;
        }

        // 初期状態設定
        currentHP = treeConfig.maxHP;
        currentState = TreeState.Standing;
        isFallen = false;
        respawnTimer = 0;

        // ビジュアル初期化
        standingVisual.SetActive(true);
        fallenVisual.SetActive(false);

        // HPバー初期化
        UpdateHPBar();

        isInitialized = true;
        Debug.Log($"[TreeController] {treeConfig.treeName} initialized with HP={currentHP}", this);
    }

    // ========================================
    // ダメージ処理（M-11 AxeInteractionから呼び出される）
    // ========================================

    /// <summary>
    /// Function F-31: Take Damage from Axe
    /// ダメージを受け取り、Master Clientのみで処理を実行
    /// </summary>
    public void _TakeDamage(int damage, VRCPlayerApi attacker)
    {
        // バリデーション
        if (!isInitialized)
        {
            Debug.LogWarning("[TreeController] 初期化されていません");
            return;
        }

        if (damage < 0 || damage > 1000)
        {
            Debug.LogWarning($"[TreeController] 異常なダメージ値: {damage}。1-1000にクランプします");
            damage = Mathf.Clamp(damage, 0, 1000);
        }

        // 倒木中は攻撃無効
        if (currentState == TreeState.Falling)
        {
            Debug.Log("[TreeController] 倒木アニメーション中のため攻撃無効");
            return;
        }

        // Master Clientのみがダメージ処理を実行（TDD Section 8.3参照）
        if (!Networking.IsMaster)
        {
            Debug.Log("[TreeController] Master Clientではないため、ダメージ処理をスキップ");
            return;
        }

        // HP減算
        currentHP -= damage;
        if (currentHP < 0) currentHP = 0;

        Debug.Log($"[TreeController] ダメージ{damage}を受けた。残りHP={currentHP}/{treeConfig.maxHP}");

        // ダメージ表示（ローカル）
        ShowDamageText(damage);

        // 効果音再生（ローカル）
        PlayHitSound();

        // HPバー更新
        UpdateHPBar();

        // HP=0の場合、倒木処理
        if (currentHP <= 0 && currentState == TreeState.Standing)
        {
            StartFalling(attacker);
        }
        else if (currentHP <= treeConfig.maxHP * LOW_HP_THRESHOLD && currentState == TreeState.Standing)
        {
            // 低HP時のきしみ音
            PlayCreakSound();
        }

        // ネットワーク同期（Function F-34）
        RequestSerialization();
    }

    // ========================================
    // 倒木処理（Function F-32）
    // ========================================

    /// <summary>
    /// Function F-32: Start Falling Animation
    /// 倒木アニメーションを開始（Master Clientのみ）
    /// </summary>
    private void StartFalling(VRCPlayerApi lastAttacker)
    {
        if (currentState != TreeState.Standing) return;

        Debug.Log($"[TreeController] {treeConfig.treeName}が倒れます");

        currentState = TreeState.Falling;
        isFallen = true;

        // コライダー無効化（倒木中は攻撃不可）
        treeCollider.enabled = false;

        // 倒木音再生
        PlayFallSound();

        // 倒木アニメーション（2秒間の簡易回転）
        SendCustomEventDelayedSeconds(nameof(_OnFallingComplete), 2.0f);

        // ネットワーク同期
        RequestSerialization();
    }

    /// <summary>
    /// 倒木アニメーション完了時の処理
    /// </summary>
    public void _OnFallingComplete()
    {
        if (currentState != TreeState.Falling) return;

        Debug.Log("[TreeController] 倒木完了");

        currentState = TreeState.Fallen;

        // ビジュアル切り替え
        standingVisual.SetActive(false);
        fallenVisual.SetActive(true);

        // HPバー非表示
        if (hpBarCanvas != null)
            hpBarCanvas.gameObject.SetActive(false);

        // M-12 LogSpawnerへの通知（イベント発火）
        // ※ WI-0012でLogSpawnerが実装された後に有効化
        // logSpawner.SendCustomEvent("_SpawnLogs");

        // リスポーンタイマー開始
        StartRespawnTimer();

        // ネットワーク同期
        RequestSerialization();
    }

    // ========================================
    // リスポーン処理（Function F-33）
    // ========================================

    /// <summary>
    /// Function F-33: Respawn Tree after Timer
    /// リスポーンタイマーを開始
    /// </summary>
    private void StartRespawnTimer()
    {
        if (!Networking.IsMaster) return; // Master Clientのみ

        respawnTimer = treeConfig.respawnTime;
        Debug.Log($"[TreeController] リスポーンタイマー開始: {respawnTimer}秒");
    }

    /// <summary>
    /// リスポーンタイマー更新（Update内で呼ばれる）
    /// </summary>
    private void UpdateRespawnTimer()
    {
        if (!Networking.IsMaster) return;
        if (currentState != TreeState.Fallen) return;
        if (respawnTimer <= 0) return;

        respawnTimer -= Time.deltaTime;

        if (respawnTimer <= 0)
        {
            respawnTimer = 0;
            RespawnTree();
        }
    }

    /// <summary>
    /// 木のリスポーン処理
    /// </summary>
    private void RespawnTree()
    {
        if (!Networking.IsMaster) return;

        Debug.Log($"[TreeController] {treeConfig.treeName}がリスポーンします");

        // 状態リセット
        currentState = TreeState.Standing;
        currentHP = treeConfig.maxHP;
        isFallen = false;
        respawnTimer = 0;

        // ビジュアル復元
        standingVisual.SetActive(true);
        fallenVisual.SetActive(false);

        // コライダー有効化
        treeCollider.enabled = true;

        // HPバー表示復元
        if (hpBarCanvas != null)
            hpBarCanvas.gameObject.SetActive(true);
        UpdateHPBar();

        // リスポーン音再生
        PlayRespawnSound();

        // ネットワーク同期
        RequestSerialization();
    }

    // ========================================
    // ネットワーク同期（Function F-34）
    // ========================================

    /// <summary>
    /// Function F-34: Sync Tree State
    /// ネットワーク同期データ受信時の処理（Late Joiner対応）
    /// </summary>
    public override void OnDeserialization()
    {
        Debug.Log($"[TreeController] ネットワーク同期受信: HP={currentHP}, isFallen={isFallen}");

        // 状態を同期データから復元
        if (isFallen && currentState == TreeState.Standing)
        {
            // 倒木状態に遷移
            currentState = TreeState.Fallen;
            standingVisual.SetActive(false);
            fallenVisual.SetActive(true);
            treeCollider.enabled = false;
            if (hpBarCanvas != null)
                hpBarCanvas.gameObject.SetActive(false);
        }
        else if (!isFallen && currentState != TreeState.Standing)
        {
            // 立木状態に復元
            currentState = TreeState.Standing;
            standingVisual.SetActive(true);
            fallenVisual.SetActive(false);
            treeCollider.enabled = true;
            if (hpBarCanvas != null)
                hpBarCanvas.gameObject.SetActive(true);
        }

        // HPバー更新
        UpdateHPBar();
    }

    // ========================================
    // UI更新
    // ========================================

    /// <summary>
    /// HPバーの更新
    /// </summary>
    private void UpdateHPBar()
    {
        if (hpBarFill == null) return;

        float hpRatio = (float)currentHP / treeConfig.maxHP;
        hpBarFill.fillAmount = hpRatio;

        // HP30%以下で黄色に変化（PRDv3.md Section 2.1）
        if (hpRatio <= LOW_HP_THRESHOLD)
        {
            hpBarFill.color = Color.yellow;
        }
        else
        {
            hpBarFill.color = Color.green;
        }
    }

    /// <summary>
    /// ダメージ数値の表示
    /// </summary>
    private void ShowDamageText(int damage)
    {
        if (damageText == null) return;

        damageText.text = $"-{damage}";
        damageText.gameObject.SetActive(true);

        // 1秒後に非表示
        SendCustomEventDelayedSeconds(nameof(_HideDamageText), DAMAGE_TEXT_DURATION);
    }

    public void _HideDamageText()
    {
        if (damageText != null)
            damageText.gameObject.SetActive(false);
    }

    // ========================================
    // 効果音再生
    // ========================================

    private void PlayHitSound()
    {
        if (audioSource != null && hitSound != null)
        {
            audioSource.PlayOneShot(hitSound);
        }
    }

    private void PlayCreakSound()
    {
        if (audioSource != null && creakSound != null)
        {
            audioSource.PlayOneShot(creakSound);
        }
    }

    private void PlayFallSound()
    {
        if (audioSource != null && fallSound != null)
        {
            audioSource.PlayOneShot(fallSound);
        }
    }

    private void PlayRespawnSound()
    {
        if (audioSource != null && respawnSound != null)
        {
            audioSource.PlayOneShot(respawnSound);
        }
    }

    // ========================================
    // Update処理（最小限に抑える）
    // ========================================

    void Update()
    {
        // リスポーンタイマー更新（Master Clientのみ）
        if (Networking.IsMaster)
        {
            UpdateRespawnTimer();
        }
    }
}
```

**コード解説**:
1. **Manual Sync使用**: `[UdonBehaviourSyncMode(BehaviourSyncMode.Manual)]`で手動同期
2. **Master Client検証**: `Networking.IsMaster`でダメージ処理をMaster Clientのみに限定
3. **GetComponentキャッシング**: Start()で一度だけコンポーネント取得（TDD Section 8.2参照）
4. **Update()回避**: リスポーンタイマーのみUpdate内で処理、他はイベント駆動
5. **異常値チェック**: ダメージ1000以上は自動拒否（TDD Section 8.3）

**確認方法**:
- Unityエディタでスクリプトをコンパイル
- Consoleにエラーがないことを確認
- UdonSharp → Compile All UdonSharp Programs

---

### 手順3: 木オブジェクトへのTreeController設定
**目的**: シーン内の木オブジェクトにTreeControllerを適用

**操作手順**:
1. Hierarchyウィンドウで木オブジェクト（Pine/Oak/Ancient）を選択
2. Inspectorウィンドウで`Add Component`をクリック
3. `TreeController`を検索して追加
4. Inspectorで以下を設定:
   - **Tree Config**: 対応するTreeConfig（PineTreeConfig/OakTreeConfig/AncientTreeConfig）
   - **Standing Visual**: 立木のGameObject
   - **Fallen Visual**: 倒木のGameObject（初期非表示）
   - **Tree Collider**: 木のCollider
   - **Audio Source**: 効果音再生用AudioSource

**HPバー用WorldSpace Canvas設定**:
1. 木オブジェクトの子オブジェクトとして`Canvas`を作成
2. Canvasの設定:
   - **Render Mode**: World Space
   - **Width**: 100
   - **Height**: 20
   - **Scale**: (0.01, 0.01, 0.01)
3. Canvasの子として`Image`（HPバー背景）を作成
   - **Anchor**: Stretch
   - **Color**: 黒（半透明）
4. Imageの子として`Fill Image`を作成
   - **Image Type**: Filled
   - **Fill Method**: Horizontal
   - **Fill Amount**: 1.0
   - **Color**: 緑
5. TreeControllerの`Hp Bar Canvas`にCanvasをドラッグ
6. `Hp Bar Fill`にFill Imageをドラッグ

**ダメージテキスト設定**:
1. Canvasの子として`3D Text`（TextMesh）を作成
2. TextMeshの設定:
   - **Text**: 空欄
   - **Font Size**: 50
   - **Anchor**: Middle Center
   - **Alignment**: Center
   - **Color**: 赤
3. 初期状態で非表示（`SetActive(false)`）
4. TreeControllerの`Damage Text`にドラッグ

**確認方法**:
- Inspectorで全フィールドが設定されているか確認
- 警告マーク（⚠️）がないことを確認
- Prefab化する場合は`Create Prefab`

---

### 手順4: ClientSimでのローカルテスト
**目的**: ネットワーク同期なしの基本動作を確認

**操作手順**:
1. Unityエディタで再生ボタン（▶️）をクリック
2. ClientSimで木に近づく
3. 木のColliderをクリック（またはVRコントローラートリガー）
4. ダメージテキストが表示されることを確認
5. HPバーが減少することを確認
6. HP=0で倒木アニメーション開始を確認
7. ビジュアルが立木→倒木に切り替わることを確認
8. リスポーンタイマー後に木が復元されることを確認

**確認項目**:
- [ ] ダメージテキスト「-10」などが表示される
- [ ] HPバーが緑→黄色（30%以下）に変化
- [ ] HP=0で倒木音が再生される
- [ ] 倒木ビジュアルに切り替わる
- [ ] リスポーンタイマー（Pine: 180秒）後に復元

**デバッグログ確認**:
```
[TreeController] Pine initialized with HP=30
[TreeController] ダメージ10を受けた。残りHP=20/30
[TreeController] Pineが倒れます
[TreeController] 倒木完了
[TreeController] リスポーンタイマー開始: 180秒
[TreeController] Pineがリスポーンします
```

---

### 手順5: ネットワーク同期テスト（Build & Test）
**目的**: マルチプレイヤー環境でのネットワーク同期を確認

**操作手順**:
1. VRChat SDK → Build & Test
2. ワールドをビルド
3. VRChatクライアントで起動
4. 2つのVRChatクライアントを起動（PC2台またはVR+デスクトップ）
5. 同じインスタンスに参加

**テストシナリオ**:

**シナリオ1: Master Clientでのダメージ処理**
1. Master Client（最初に入った人）が木を攻撃
2. 他のクライアントでHPバーが同期されることを確認
3. Master ClientでHP=0にする
4. 他のクライアントで倒木アニメーションが再生されることを確認

**シナリオ2: Late Joiner同期**
1. Master Clientが木を倒す
2. 新しいプレイヤーがインスタンスに参加
3. 新規プレイヤーで木が倒木状態で表示されることを確認

**シナリオ3: Master Client退出時の引き継ぎ**
1. Master Clientが木を50%削る
2. Master Clientが退出
3. 新Master Clientに状態が引き継がれることを確認
4. 他のプレイヤーが攻撃を継続できることを確認

**確認項目**:
- [ ] 全クライアントでHPが同期される（±1秒以内）
- [ ] Late Joinerも正しい状態で表示される
- [ ] Master Client退出後も木の状態が保持される
- [ ] 帯域使用量が5KB/秒以下（VRChat SDK Control Panelで確認）

---

### 手順6: パフォーマンステスト（Quest最適化確認）
**目的**: Quest 2環境で60fps維持を確認

**操作手順**:
1. VRChat SDK → Build & Publish
2. Quest 2デバイスでワールドを起動
3. 10人のプレイヤー（ボット可）が同時に木を伐採するシミュレーション
4. FPS値をモニタリング

**Quest最適化チェックリスト**:
- [ ] 木のポリゴン数: Pine=500, Oak=800, Ancient=1200以内
- [ ] LOD設定: LOD0/LOD1/LOD2の3段階設定
- [ ] テクスチャ: 1024x1024以下、ASTC圧縮
- [ ] HPバーCanvas: 最小限のUI要素のみ
- [ ] Update()処理: リスポーンタイマーのみ

**パフォーマンス計測**:
```
Quest 2環境:
- 10人同時伐採: 60fps維持
- メモリ使用量: 100MB以下
- ネットワーク帯域: 5KB/秒以下
```

**確認方法**:
- VRChat内蔵パフォーマンスモニター（メニュー → Performance）
- または外部ツール（VRCLens等）

---

## 9. 完了条件（Doneの条件）

### 必須要件
1. **TreeController.cs**が正常にコンパイルされる
2. **TreeConfig ScriptableObject**が3種類作成され、正しい値が設定されている
3. シーン内の木オブジェクトにTreeControllerが適用されている
4. ClientSimで以下の動作が確認できる:
   - ダメージを受けるとHPが減少
   - HP=0で倒木アニメーション開始
   - 倒木ビジュアルに切り替わる
   - リスポーンタイマー後に木が復元
5. ネットワーク同期が正常に動作する:
   - 全クライアントでHPが同期される
   - Late Joinerも正しい状態で表示される
   - Master Client退出後も状態が保持される
6. Quest 2環境で60fps維持
7. Consoleにエラーログが出力されない

### 品質要件
1. コードがSOLID原則に準拠している（単一責任、依存性逆転）
2. エラーハンドリングが適切に実装されている（異常値チェック、Null参照回避）
3. デバッグログが適切に出力されている（`[TreeController]`プレフィックス）
4. コメントが日本語で記述されている

---

## 10. テスト観点／テストケース

### テスト種別
- **単体テスト**: ClientSimでの基本動作確認
- **統合テスト**: ネットワーク同期テスト
- **負荷テスト**: Quest 2パフォーマンステスト

### 正常系テストケース

#### TC-001: 通常ダメージ処理
**前提条件**: 木のHP=30（満タン）
**操作**: ダメージ10を与える
**期待結果**: HP=20に減少、HPバー更新、ダメージテキスト表示

#### TC-002: 倒木処理
**前提条件**: 木のHP=10
**操作**: ダメージ10を与える
**期待結果**: HP=0、倒木アニメーション開始、ビジュアル切り替え

#### TC-003: リスポーン処理
**前提条件**: 木が倒木状態、リスポーンタイマー=0
**操作**: リスポーンタイマーが0になる
**期待結果**: 木が立木状態に復元、HP満タン

#### TC-004: ネットワーク同期
**前提条件**: 2クライアント接続
**操作**: Master Clientが木を攻撃
**期待結果**: 他クライアントでHPが同期される（±1秒以内）

#### TC-005: Late Joiner同期
**前提条件**: 木が倒木状態
**操作**: 新規プレイヤーがインスタンス参加
**期待結果**: 新規プレイヤーで木が倒木状態で表示

### 異常系テストケース

#### TC-E001: 異常なダメージ値（負値）
**前提条件**: 木のHP=30
**操作**: ダメージ-10を与える
**期待結果**: ダメージ0にクランプ、HPは減少しない、警告ログ出力

#### TC-E002: 異常なダメージ値（1000超過）
**前提条件**: 木のHP=30
**操作**: ダメージ9999を与える
**期待結果**: ダメージ1000にクランプ、HP=0、警告ログ出力

#### TC-E003: 倒木中の攻撃
**前提条件**: 木が倒木アニメーション中（Falling状態）
**操作**: 攻撃を試みる
**期待結果**: ダメージ無効、「倒木アニメーション中のため攻撃無効」ログ出力

#### TC-E004: TreeConfig未設定
**前提条件**: TreeControllerのTreeConfigフィールドがnull
**操作**: シーン再生
**期待結果**: エラーログ出力、処理中断

#### TC-E005: Master Client以外からのダメージ処理
**前提条件**: プレイヤーがMaster Clientではない
**操作**: _TakeDamage()を呼び出す
**期待結果**: ダメージ処理スキップ、「Master Clientではない」ログ出力

---

## 11. 成果物

### 変更ファイル
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`（新規）
2. `/Assets/WoodcutterCamp/ScriptableObjects/TreeConfig.cs`（新規）
3. `/Assets/WoodcutterCamp/ScriptableObjects/PineTreeConfig.asset`（新規）
4. `/Assets/WoodcutterCamp/ScriptableObjects/OakTreeConfig.asset`（新規）
5. `/Assets/WoodcutterCamp/ScriptableObjects/AncientTreeConfig.asset`（新規）
6. シーン内の木オブジェクト（TreeController適用済み）

### 追加テストコード
本作業ではUdon/UdonSharpの制約により自動テストコードは作成しませんが、上記テストケースを手動で実行してください。

### 更新したドキュメント
- 本WI-0007作業指示書（完了報告時に更新）

---

## 12. 備考

### 注意点
1. **Master Clientの重要性**: ダメージ処理は必ずMaster Clientで実行すること。クライアント側での不正な値変更を防ぐため。
2. **RequestSerialization()の呼び出し**: Manual Syncの場合、状態変更後に必ず`RequestSerialization()`を呼び出すこと。
3. **Late Joiner対応**: `OnDeserialization()`で状態を正しく復元すること。
4. **Quest最適化**: Update()内の処理は最小限に抑え、イベント駆動で実装すること。
5. **帯域制限**: 同期変数は必要最小限に留め、5KB/秒以下を維持すること。

### 制約事項
- UdonSharpではasync/awaitが使用できないため、Coroutineで代替
- Dictionaryなどの一部コレクション型は使用できない可能性がある
- Update()の実行コストが高いため、タイマーベースで間隔を空ける

### 次の作業との連携
- **WI-0008** (Tree Falling Animation): 倒木アニメーションの詳細実装
- **WI-0009** (AxeInteraction Detection): 斧からのダメージ入力実装
- **WI-0012** (LogSpawner): 倒木完了時の丸太生成実装

### 参考資料
- **TDD.md**: Section 5.10（M-10 TreeController仕様）
- **FSD.md**: Section 3.1（FNC-001 伐採システム仕様）
- **PRDv3.md**: Section 2.1.1（木の仕様）
- **VRChat_Tools_API_Reference.md**: Section 2.3.4（UdonSharp ネットワーク同期）
- VRChat公式ドキュメント: https://creators.vrchat.com/worlds/udon/networking/

### 推定作業時間
- **合計**: 2日（16時間）
  - TreeConfig作成: 1時間
  - TreeController.cs実装: 6時間
  - 木オブジェクト設定: 2時間
  - ローカルテスト: 2時間
  - ネットワーク同期テスト: 3時間
  - Quest最適化: 2時間

---

**作成者**: VRChat World Development Team
**作成日**: 2025-11-17
**最終更新**: 2025-11-17
**ステータス**: 作業指示確定
**優先度**: 最高
**依存関係**: WI-0001, WI-0002
**関連モジュール**: M-10 TreeController, M-02 NetworkSync
**関連機能**: F-31, F-32, F-33, F-34
