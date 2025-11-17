# 作業指示書（Work Instruction）

## 1. 作業ID
**WI-0012**

## 2. 作業名
**LogSpawner実装 - オブジェクトプール管理と丸太生成システム**

## 3. 作業目的
倒木完了時に丸太オブジェクトをスポーンする機能を実装する。オブジェクトプール方式で最大30個の丸太を効率的に管理し、古い丸太から自動削除（FIFO）することでメモリとパフォーマンスを最適化する。Quest 2環境で60fps維持を実現する。

---

## 4. 対象

- **対象システム**: 森のきこりキャンプ（Woodcutter Camp）VRChatワールド Phase 1
- **対象モジュール**: M-12 LogSpawner（Gameplay Layer）
- **対象ファイルパス**:
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/LogSpawner.cs`（新規作成）
  - `/Assets/WoodcutterCamp/Prefabs/Log.prefab`（新規作成）
  - `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`（LogSpawner連携追加）

---

## 5. 前提条件

### 環境要件
- Unity 2022.3.22f1
- VRChat SDK 3.9.0
- UdonSharp 1.1.9

### 完了している依存作業
- **WI-0001**: GameManager Singleton実装完了
- **WI-0002**: NetworkSync Batching System実装完了
- **WI-0007**: TreeController State Machine実装完了

### 初期状態
- Unityプロジェクトが開いている
- TreeControllerが実装済みで、倒木処理が動作している
- シーンに木オブジェクトが配置済み
- 丸太の3Dモデル（Cylinderまたはカスタムメッシュ）が準備済み

### ブランチ
- `feature/woodcutting-system`（既存ブランチから継続）

---

## 6. 入力

### TreeControllerからの倒木完了イベント
```csharp
// M-10 TreeControllerから呼び出される
public void _SpawnLogs(Vector3 treePosition, int treeType, int skillLevel);
```

**パラメータ**:
- `treePosition`: 倒木の中心位置（Vector3）
- `treeType`: 木の種類（0=Pine, 1=Oak, 2=Ancient）
- `skillLevel`: 伐採者のスキルレベル（1〜10）

### 木の種類別丸太生成数（FSD.md Section 3.1参照）
- **Pine（小）**: 1〜2本（基本）+ スキルボーナス
- **Oak（中）**: 2〜3本（基本）+ スキルボーナス
- **Ancient（大）**: 3〜4本（基本）+ スキルボーナス

### スキルボーナス条件（TDD.md Section 5.5参照）
- **Lv3**: 丸太+1本（10%確率）
- **Lv6**: 丸太+1本（20%確率）
- **Lv9**: 丸太+1本（30%確率）

---

## 7. 出力

### 生成される丸太オブジェクト
- **サイズ**: 長さ2m、直径0.3mの円柱（Cylinder）
- **位置**: 倒木中心から1m間隔で配置、ランダムな力で散らばる（-0.5〜0.5mのオフセット）
- **物理特性**: Rigidbody（質量5kg）、CapsuleCollider
- **VRC_Pickup設定**: AutoHold = Off、Orientation = Any

### オブジェクトプール管理
- **プールサイズ**: 40個（予備10個含む）
- **最大同時存在数**: 30個
- **削除方式**: FIFO（First-In-First-Out）- 古い丸太から自動削除

### ネットワーク同期
- 丸太の位置と状態は**ローカル処理**（同期不要）
- 生成イベントのみTreeControllerの同期に依存

---

## 8. 作業手順

### 手順1: 丸太Prefab作成
**目的**: VRC_Pickup対応の丸太オブジェクトを作成

**操作手順**:
1. Hierarchyウィンドウで空のGameObjectを作成し、名前を`Log`に変更
2. `Log`の子として`GameObject > 3D Object > Cylinder`を追加
3. Cylinderの設定:
   - **Scale**: (0.3, 1.0, 0.3) - 直径0.3m、長さ2m
   - **Rotation**: (0, 0, 90) - 横向きに回転
   - **Material**: 木材テクスチャを適用（茶色系、木目模様）
4. `Log`に`Rigidbody`コンポーネントを追加
   - **Mass**: 5kg
   - **Drag**: 0.5
   - **Angular Drag**: 0.3
   - **Use Gravity**: ✅ チェック
5. `Log`に`Capsule Collider`を追加
   - **Radius**: 0.15
   - **Height**: 2.0
   - **Direction**: Z-Axis
6. `Log`に`VRC_Pickup`コンポーネントを追加
   - **Auto Hold**: Off（トリガー離すとDrop）
   - **Orientation**: Any（任意の向きで持てる）
   - **Use Text**: "丸太を掴む"
7. Prefab化
   - `Log`オブジェクトを`/Assets/WoodcutterCamp/Prefabs/`にドラッグしてPrefab作成
   - Hierarchy内の`Log`は削除（実行時にスポーンするため）

**確認方法**:
- Prefabをダブルクリックして編集モードで開く
- Inspectorで全コンポーネントが正しく設定されているか確認
- シーンに一時的に配置し、Rigidbodyの物理挙動（落下）を確認
- VRC_Pickupでデスクトップモード（E キー）とVRモード（グリップボタン）の両方でPickup可能か確認

---

### 手順2: LogSpawner.cs UdonSharpスクリプト作成
**目的**: オブジェクトプール管理と丸太生成ロジックを実装

**操作手順**:
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/`フォルダを開く
2. フォルダ内で右クリック → `Create > UdonSharp > UdonSharpBehaviour Script`
3. ファイル名を`LogSpawner.cs`に変更

**LogSpawner.cs 完全実装コード**:
```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

/// <summary>
/// Module M-12: LogSpawner
/// 丸太のスポーン処理とオブジェクトプール管理を担当
/// Functions: F-40 (Spawn Logs), F-41 (Object Pool), F-42 (Calculate Position)
/// </summary>
public class LogSpawner : UdonSharpBehaviour
{
    // ========================================
    // Inspector設定
    // ========================================
    [Header("丸太Prefab")]
    [Tooltip("スポーンする丸太のPrefab")]
    public GameObject logPrefab;

    [Header("オブジェクトプール設定")]
    [Tooltip("プールサイズ（予備含む）")]
    public int poolSize = 40;

    [Tooltip("最大同時存在数（古いものから削除）")]
    public int maxActiveLogs = 30;

    [Header("スポーン位置設定")]
    [Tooltip("丸太の間隔（メートル）")]
    public float logSpacing = 1.0f;

    [Tooltip("ランダムオフセットの最大値（メートル）")]
    public float randomOffset = 0.5f;

    [Header("物理設定")]
    [Tooltip("丸太に加える初期力の強さ")]
    public float spawnForce = 2.0f;

    // ========================================
    // オブジェクトプール管理
    // ========================================
    private GameObject[] logPool;          // 丸太オブジェクトのプール
    private float[] logSpawnTimes;         // 各丸太の生成時刻（FIFO判定用）
    private int activeLogCount = 0;        // 現在アクティブな丸太数
    private bool isInitialized = false;

    // ========================================
    // 定数
    // ========================================
    private const int TREE_TYPE_PINE = 0;
    private const int TREE_TYPE_OAK = 1;
    private const int TREE_TYPE_ANCIENT = 2;

    // 木の種類別丸太生成数設定（FSD.md Section 3.1参照）
    private readonly int[] minLogCounts = { 1, 2, 3 };  // Pine, Oak, Ancient
    private readonly int[] maxLogCounts = { 2, 3, 4 };

    // スキルボーナス設定（TDD.md Section 5.5参照）
    private readonly int[] bonusSkillLevels = { 3, 6, 9 };
    private readonly float[] bonusChances = { 0.1f, 0.2f, 0.3f };

    // ========================================
    // 初期化
    // ========================================
    void Start()
    {
        InitializeObjectPool();
    }

    /// <summary>
    /// Function F-41: Object Pool Management
    /// オブジェクトプールの初期化
    /// </summary>
    private void InitializeObjectPool()
    {
        if (isInitialized) return;

        // バリデーション
        if (logPrefab == null)
        {
            Debug.LogError("[LogSpawner] logPrefabが設定されていません！", this);
            return;
        }

        if (poolSize <= 0 || maxActiveLogs <= 0)
        {
            Debug.LogError($"[LogSpawner] 無効なプール設定: poolSize={poolSize}, maxActiveLogs={maxActiveLogs}", this);
            return;
        }

        if (maxActiveLogs > poolSize)
        {
            Debug.LogWarning($"[LogSpawner] maxActiveLogs({maxActiveLogs})がpoolSize({poolSize})を超えています。poolSizeに調整します");
            maxActiveLogs = poolSize;
        }

        // オブジェクトプール配列初期化（UdonSharpではList<T>が使えないため配列で実装）
        logPool = new GameObject[poolSize];
        logSpawnTimes = new float[poolSize];

        // プール内のオブジェクトを事前生成
        for (int i = 0; i < poolSize; i++)
        {
            GameObject logObj = Instantiate(logPrefab);
            logObj.name = $"Log_{i:D3}";  // Log_000, Log_001, ...
            logObj.SetActive(false);      // 初期状態は非アクティブ
            logPool[i] = logObj;
            logSpawnTimes[i] = -1f;       // -1 = 未使用状態
        }

        activeLogCount = 0;
        isInitialized = true;

        Debug.Log($"[LogSpawner] オブジェクトプール初期化完了: poolSize={poolSize}, maxActiveLogs={maxActiveLogs}");
    }

    // ========================================
    // 丸太生成処理（TreeControllerから呼び出される）
    // ========================================

    /// <summary>
    /// Function F-40: Spawn Logs from Fallen Tree
    /// 倒木から丸太をスポーンする
    /// </summary>
    /// <param name="treePosition">倒木の中心位置</param>
    /// <param name="treeType">木の種類（0=Pine, 1=Oak, 2=Ancient）</param>
    /// <param name="skillLevel">伐採者のスキルレベル（1〜10）</param>
    public void _SpawnLogs(Vector3 treePosition, int treeType, int skillLevel)
    {
        if (!isInitialized)
        {
            Debug.LogWarning("[LogSpawner] 初期化されていません");
            return;
        }

        // 木の種類バリデーション
        if (treeType < TREE_TYPE_PINE || treeType > TREE_TYPE_ANCIENT)
        {
            Debug.LogWarning($"[LogSpawner] 無効な木の種類: {treeType}。Pine(0)に設定します");
            treeType = TREE_TYPE_PINE;
        }

        // スキルレベルバリデーション
        if (skillLevel < 1 || skillLevel > 10)
        {
            Debug.LogWarning($"[LogSpawner] 無効なスキルレベル: {skillLevel}。1にクランプします");
            skillLevel = Mathf.Clamp(skillLevel, 1, 10);
        }

        // 丸太生成数を計算
        int logCount = CalculateLogCount(treeType, skillLevel);

        Debug.Log($"[LogSpawner] 丸太生成開始: treeType={treeType}, skillLevel={skillLevel}, logCount={logCount}");

        // 丸太を1本ずつ生成
        for (int i = 0; i < logCount; i++)
        {
            SpawnSingleLog(treePosition, i, logCount);
        }

        Debug.Log($"[LogSpawner] 丸太生成完了: {logCount}本生成、アクティブ数={activeLogCount}/{maxActiveLogs}");
    }

    /// <summary>
    /// 丸太生成数の計算（基本数 + スキルボーナス）
    /// </summary>
    private int CalculateLogCount(int treeType, int skillLevel)
    {
        // 基本生成数（ランダム範囲）
        int minCount = minLogCounts[treeType];
        int maxCount = maxLogCounts[treeType];
        int baseCount = Random.Range(minCount, maxCount + 1);  // Range(1, 3)は1〜2を返すため+1

        // スキルボーナス判定
        int bonusCount = 0;
        for (int i = 0; i < bonusSkillLevels.Length; i++)
        {
            if (skillLevel >= bonusSkillLevels[i])
            {
                float roll = Random.value;  // 0.0〜1.0
                if (roll < bonusChances[i])
                {
                    bonusCount++;
                    Debug.Log($"[LogSpawner] スキルボーナス発動！ Lv{bonusSkillLevels[i]}、確率{bonusChances[i] * 100}%");
                }
            }
        }

        return baseCount + bonusCount;
    }

    /// <summary>
    /// 単一の丸太をスポーン
    /// </summary>
    private void SpawnSingleLog(Vector3 centerPosition, int index, int totalCount)
    {
        // オブジェクトプールから空きを取得
        GameObject logObj = GetAvailableLogFromPool();
        if (logObj == null)
        {
            Debug.LogWarning("[LogSpawner] プールに空きがありません。最も古い丸太を削除します");
            logObj = RemoveOldestLog();
        }

        if (logObj == null)
        {
            Debug.LogError("[LogSpawner] 丸太のスポーンに失敗しました");
            return;
        }

        // Function F-42: Calculate Spawn Position and Force
        Vector3 spawnPosition = CalculateSpawnPosition(centerPosition, index, totalCount);

        // 丸太を配置
        logObj.transform.position = spawnPosition;
        logObj.transform.rotation = Random.rotation;  // ランダムな回転
        logObj.SetActive(true);

        // Rigidbodyに初期力を加えて散らばらせる
        Rigidbody rb = logObj.GetComponent<Rigidbody>();
        if (rb != null)
        {
            rb.velocity = Vector3.zero;   // 前回の速度をリセット
            rb.angularVelocity = Vector3.zero;

            // ランダムな方向に力を加える
            Vector3 randomForce = new Vector3(
                Random.Range(-spawnForce, spawnForce),
                Random.Range(0, spawnForce * 0.5f),  // 上方向は弱めに
                Random.Range(-spawnForce, spawnForce)
            );
            rb.AddForce(randomForce, ForceMode.Impulse);
        }

        // プール管理情報を更新
        int poolIndex = System.Array.IndexOf(logPool, logObj);
        if (poolIndex >= 0)
        {
            logSpawnTimes[poolIndex] = Time.time;  // 現在時刻を記録（FIFO用）
        }

        activeLogCount++;
    }

    /// <summary>
    /// Function F-42: Calculate Spawn Position
    /// スポーン位置の計算（1m間隔 + ランダムオフセット）
    /// </summary>
    private Vector3 CalculateSpawnPosition(Vector3 centerPosition, int index, int totalCount)
    {
        // 倒木の中心から1m間隔で配置（直線状）
        float offset = (index - (totalCount - 1) / 2.0f) * logSpacing;
        Vector3 basePosition = centerPosition + new Vector3(offset, 0.5f, 0);  // Y+0.5mで地面から浮かせる

        // ランダムなオフセットを追加して自然に散らばらせる
        Vector3 randomPos = new Vector3(
            Random.Range(-randomOffset, randomOffset),
            Random.Range(0, randomOffset * 0.5f),      // 上方向は控えめに
            Random.Range(-randomOffset, randomOffset)
        );

        return basePosition + randomPos;
    }

    // ========================================
    // オブジェクトプール管理（Function F-41）
    // ========================================

    /// <summary>
    /// プールから空きの丸太を取得
    /// </summary>
    private GameObject GetAvailableLogFromPool()
    {
        for (int i = 0; i < poolSize; i++)
        {
            if (!logPool[i].activeSelf)  // 非アクティブ = 空き
            {
                return logPool[i];
            }
        }

        return null;  // 空きなし
    }

    /// <summary>
    /// 最も古い丸太を削除してプールに戻す（FIFO方式）
    /// </summary>
    private GameObject RemoveOldestLog()
    {
        if (activeLogCount == 0)
        {
            Debug.LogWarning("[LogSpawner] アクティブな丸太がありません");
            return null;
        }

        // 最も古い丸太（最小のspawnTime）を検索
        int oldestIndex = -1;
        float oldestTime = float.MaxValue;

        for (int i = 0; i < poolSize; i++)
        {
            if (logPool[i].activeSelf && logSpawnTimes[i] >= 0)
            {
                if (logSpawnTimes[i] < oldestTime)
                {
                    oldestTime = logSpawnTimes[i];
                    oldestIndex = i;
                }
            }
        }

        if (oldestIndex < 0)
        {
            Debug.LogError("[LogSpawner] 最も古い丸太の検索に失敗しました");
            return null;
        }

        // 古い丸太を非アクティブ化（削除）
        GameObject oldestLog = logPool[oldestIndex];
        oldestLog.SetActive(false);
        logSpawnTimes[oldestIndex] = -1f;  // 未使用状態に戻す
        activeLogCount--;

        Debug.Log($"[LogSpawner] 最も古い丸太を削除: {oldestLog.name}、生成時刻={oldestTime:F2}秒");

        return oldestLog;  // 空きになった丸太を返す
    }

    /// <summary>
    /// 手動で丸太を削除する（納品時などに使用）
    /// </summary>
    public void _RemoveLog(GameObject logObj)
    {
        if (logObj == null) return;

        int index = System.Array.IndexOf(logPool, logObj);
        if (index >= 0 && logPool[index].activeSelf)
        {
            logPool[index].SetActive(false);
            logSpawnTimes[index] = -1f;
            activeLogCount--;
            Debug.Log($"[LogSpawner] 丸太を削除: {logObj.name}");
        }
    }

    // ========================================
    // デバッグ用メソッド
    // ========================================

    /// <summary>
    /// プールの状態を確認（デバッグ用）
    /// </summary>
    public void _DebugPoolStatus()
    {
        int activeCount = 0;
        int inactiveCount = 0;

        for (int i = 0; i < poolSize; i++)
        {
            if (logPool[i].activeSelf)
                activeCount++;
            else
                inactiveCount++;
        }

        Debug.Log($"[LogSpawner] プール状態: アクティブ={activeCount}, 非アクティブ={inactiveCount}, 合計={poolSize}");
    }
}
```

**コード解説**:
1. **オブジェクトプール**: 事前に40個の丸太を生成し、SetActive()で管理（GameObject.Instantiate()の連打を回避）
2. **FIFO削除**: logSpawnTimes配列で生成時刻を記録し、最も古いものから削除
3. **配列実装**: UdonSharpではList<T>が使えないため、GameObject[]とfloat[]で実装
4. **ランダム生成**: Random.Range()で丸太数、Random.value でスキルボーナス判定
5. **物理演算**: AddForce()でランダムな力を加えて自然に散らばる

**確認方法**:
- Unityエディタでスクリプトをコンパイル
- Consoleにエラーがないことを確認
- UdonSharp → Compile All UdonSharp Programs

---

### 手順3: LogSpawnerオブジェクトのシーン配置
**目的**: LogSpawnerをシーンに配置し、設定を行う

**操作手順**:
1. Hierarchyウィンドウで空のGameObjectを作成
2. 名前を`LogSpawner`に変更
3. Inspectorで`Add Component` → `LogSpawner`スクリプトを追加
4. Inspectorで以下を設定:
   - **Log Prefab**: `/Assets/WoodcutterCamp/Prefabs/Log.prefab`をドラッグ
   - **Pool Size**: 40
   - **Max Active Logs**: 30
   - **Log Spacing**: 1.0
   - **Random Offset**: 0.5
   - **Spawn Force**: 2.0

**確認方法**:
- Inspectorで全フィールドが設定されているか確認
- Log Prefabのリンクが紫色（正常）になっているか確認（Missing = ピンク色ならNG）

---

### 手順4: TreeControllerとの連携実装
**目的**: TreeControllerからLogSpawnerを呼び出す

**操作手順**:
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`を開く
2. LogSpawner参照フィールドを追加
3. _OnFallingComplete()メソッドでLogSpawnerを呼び出す

**TreeController.cs への追加コード**:
```csharp
// ========================================
// Inspector設定（既存部分に追加）
// ========================================
[Header("LogSpawner連携")]
public LogSpawner logSpawner;           // LogSpawnerへの参照

// ========================================
// 倒木処理（既存メソッドを修正）
// ========================================

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

    // ★ LogSpawnerへの通知（WI-0012で追加）
    if (logSpawner != null)
    {
        // 木の種類を取得（TreeConfigから）
        int treeType = GetTreeType(treeConfig.treeName);

        // スキルレベルを取得（後でSkillManagerと連携予定、現在は暫定値1）
        int skillLevel = 1;  // TODO: WI-0005でSkillManager連携後に実装

        // 丸太生成
        logSpawner._SpawnLogs(transform.position, treeType, skillLevel);
    }
    else
    {
        Debug.LogWarning("[TreeController] LogSpawnerが設定されていません");
    }

    // リスポーンタイマー開始
    StartRespawnTimer();

    // ネットワーク同期
    RequestSerialization();
}

/// <summary>
/// 木の名前から木の種類（int）を取得
/// </summary>
private int GetTreeType(string treeName)
{
    if (treeName.Contains("Pine")) return 0;
    if (treeName.Contains("Oak")) return 1;
    if (treeName.Contains("Ancient")) return 2;
    return 0;  // デフォルトはPine
}
```

**設定手順**:
1. Hierarchyで各木オブジェクト（Pine/Oak/Ancient）を選択
2. Inspectorの`TreeController`コンポーネントを確認
3. `Log Spawner`フィールドに`LogSpawner`オブジェクトをドラッグして設定
4. すべての木オブジェクトで設定を確認

**確認方法**:
- TreeController.csがエラーなくコンパイルされる
- Inspectorで`Log Spawner`フィールドにリンクが設定されている
- Console に警告「LogSpawnerが設定されていません」が出ないことを確認

---

### 手順5: ClientSimでのローカルテスト
**目的**: 丸太生成とオブジェクトプール管理の基本動作を確認

**操作手順**:
1. Unityエディタで再生ボタン（▶️）をクリック
2. ClientSimで木に近づく
3. 木を攻撃してHP=0にする（斧未実装の場合は、Inspectorから手動でHP=0に設定）
4. 倒木アニメーション完了後、丸太が生成されることを確認
5. 生成された丸太の数を確認（Pine: 1〜2本、Oak: 2〜3本、Ancient: 3〜4本）
6. 丸太がランダムな位置に散らばっていることを確認
7. 丸太をPickup（Eキー）できることを確認

**テストシナリオ1: 丸太生成確認**
- **操作**: Pine（小）を伐採
- **期待結果**: 1〜2本の丸太が生成される
- **確認**: Hierarchyウィンドウで`Log_000`, `Log_001`が表示される

**テストシナリオ2: オブジェクトプール確認**
- **操作**: Pineを連続で15本伐採（30個の丸太生成）
- **期待結果**:
  - 30個目まで正常に生成される
  - 31個目で最も古い丸太が自動削除される
  - Consoleに「最も古い丸太を削除」ログが表示される

**テストシナリオ3: VRC_Pickup確認**
- **操作**: 丸太に近づいてEキー（デスクトップ）またはグリップボタン（VR）
- **期待結果**: 丸太が手に吸着する
- **確認**: 移動しても丸太が追従する、Eキーで離すとDropする

**デバッグログ確認**:
```
[LogSpawner] オブジェクトプール初期化完了: poolSize=40, maxActiveLogs=30
[TreeController] 倒木完了
[LogSpawner] 丸太生成開始: treeType=0, skillLevel=1, logCount=2
[LogSpawner] 丸太生成完了: 2本生成、アクティブ数=2/30
```

**確認項目**:
- [ ] 木の種類に応じた丸太数が生成される
- [ ] 丸太が倒木の中心から1m間隔で配置される
- [ ] 丸太にランダムな力が加わり散らばる
- [ ] 丸太をPickupできる
- [ ] 30個を超えると古い丸太が削除される

---

### 手順6: スキルボーナステスト
**目的**: スキルレベルに応じた丸太ボーナス生成を確認

**操作手順**:
1. TreeController.csの`_OnFallingComplete()`で`skillLevel`を手動変更
   ```csharp
   int skillLevel = 9;  // テスト用にLv9に設定
   ```
2. Pineを10回伐採してログを記録
3. ボーナス発動回数を確認（Lv9 = 30%確率）
4. 統計を取る（10回中3回程度ボーナス発動すればOK）

**期待結果**:
- Lv3: 10%確率でボーナス → 10回中1回程度
- Lv6: 20%確率でボーナス → 10回中2回程度
- Lv9: 30%確率でボーナス → 10回中3回程度

**デバッグログ例**:
```
[LogSpawner] スキルボーナス発動！ Lv9、確率30%
[LogSpawner] 丸太生成完了: 3本生成（基本2本 + ボーナス1本）
```

**確認方法**:
- Consoleのログで「スキルボーナス発動」が確率通り出現するか確認
- 確率のばらつきは正常（10回テストで0〜5回の範囲内なら正常）

---

### 手順7: パフォーマンステスト（Quest最適化確認）
**目的**: Quest 2環境で30個の丸太が同時表示されても60fps維持を確認

**操作手順**:
1. Unityエディタで木を15本伐採し、30個の丸太を同時表示させる
2. Unityエディタの`Stats`ウィンドウでパフォーマンスを確認
   - FPS: 60fps以上
   - Batches: 50以下
   - Tris（三角形数）: 10,000以下
3. Quest 2ビルドテスト（VRChat SDK → Build & Test）
4. Quest 2デバイスで実行し、FPS値をモニタリング

**Quest最適化チェックリスト**:
- [ ] 丸太のポリゴン数: Cylinder（デフォルト24面） → 12面に削減
- [ ] 丸太のマテリアル: Mobile/Diffuseシェーダー使用
- [ ] テクスチャ: 512x512以下、ASTC圧縮
- [ ] Collider: CapsuleCollider（高速）を使用
- [ ] Rigidbody: Sleep Threshold = 0.05（早めにスリープ状態に）
- [ ] オブジェクトプール: プール再利用でInstantiate()回避

**パフォーマンス計測**:
```
Quest 2環境:
- 30個の丸太同時表示: 60fps維持
- メモリ増加量: 10MB以下
- Batches: 32（動的バッチング有効）
```

**確認方法**:
- VRChat内蔵パフォーマンスモニター（メニュー → Performance → Show FPS）
- Unityエディタの`Stats`ウィンドウ（再生中に確認）

---

### 手順8: 統合テスト（TreeController + LogSpawner）
**目的**: 倒木から丸太生成までの一連の流れを確認

**操作手順**:
1. VRChat SDK → Build & Test
2. VRChatクライアントで起動
3. 以下のシナリオをテスト

**テストシナリオ1: Pine伐採**
- **操作**: Pineを伐採（HP=30）
- **期待結果**:
  - 倒木アニメーション後、1〜2本の丸太が生成される
  - 丸太が倒木の中心から1m間隔で配置される
  - 丸太をPickupできる

**テストシナリオ2: Oak伐採**
- **操作**: Oakを伐採（HP=60）
- **期待結果**: 2〜3本の丸太が生成される

**テストシナリオ3: Ancient伐採**
- **操作**: Ancientを伐採（HP=100）
- **期待結果**: 3〜4本の丸太が生成される

**テストシナリオ4: 30個上限テスト**
- **操作**: 15本の木を連続伐採（30個の丸太生成）
- **期待結果**:
  - 30個まで正常に生成される
  - 31個目で最も古い丸太が自動削除される
  - Consoleに「最も古い丸太を削除」ログ表示

**確認項目**:
- [ ] TreeControllerの倒木完了イベントでLogSpawnerが正常に呼び出される
- [ ] 丸太の生成数が木の種類に応じて正しい
- [ ] 丸太のPickupが正常に動作する
- [ ] オブジェクトプールの上限管理が機能する

---

## 9. 完了条件（Doneの条件）

### 必須要件
1. **LogSpawner.cs**が正常にコンパイルされる
2. **Log.prefab**が作成され、VRC_Pickupコンポーネントが設定されている
3. シーンに**LogSpawnerオブジェクト**が配置され、設定が完了している
4. **TreeController**からLogSpawnerが呼び出され、丸太が生成される
5. ClientSimで以下の動作が確認できる:
   - 倒木後に丸太が生成される
   - 丸太数が木の種類に応じて正しい（Pine: 1〜2、Oak: 2〜3、Ancient: 3〜4）
   - 丸太がランダムな位置に散らばる
   - 丸太をPickupできる
   - 30個を超えると古い丸太が削除される
6. **Quest 2環境で60fps維持**（30個の丸太同時表示時）
7. Consoleにエラーログが出力されない

### 品質要件
1. コードがSOLID原則に準拠している（単一責任、依存性逆転）
2. オブジェクトプールがFIFO方式で正しく機能している
3. エラーハンドリングが適切に実装されている（Null参照回避、範囲チェック）
4. デバッグログが適切に出力されている（`[LogSpawner]`プレフィックス）
5. コメントが日本語で記述されている

---

## 10. テスト観点／テストケース

### テスト種別
- **単体テスト**: ClientSimでの丸太生成・プール管理確認
- **統合テスト**: TreeController連携テスト
- **負荷テスト**: 30個同時表示のパフォーマンステスト

### 正常系テストケース

#### TC-001: Pine伐採時の丸太生成
**前提条件**: Pineが倒木完了状態
**操作**: LogSpawner._SpawnLogs()を呼び出し（treeType=0, skillLevel=1）
**期待結果**: 1〜2本の丸太が生成される

#### TC-002: Oak伐採時の丸太生成
**前提条件**: Oakが倒木完了状態
**操作**: LogSpawner._SpawnLogs()を呼び出し（treeType=1, skillLevel=1）
**期待結果**: 2〜3本の丸太が生成される

#### TC-003: Ancient伐採時の丸太生成
**前提条件**: Ancientが倒木完了状態
**操作**: LogSpawner._SpawnLogs()を呼び出し（treeType=2, skillLevel=1）
**期待結果**: 3〜4本の丸太が生成される

#### TC-004: スキルボーナス（Lv3）
**前提条件**: skillLevel=3
**操作**: Pineを10回伐採
**期待結果**: 10回中1回程度ボーナス発動（+1本）

#### TC-005: スキルボーナス（Lv9）
**前提条件**: skillLevel=9
**操作**: Pineを10回伐採
**期待結果**: 10回中3回程度ボーナス発動（+1本）

#### TC-006: オブジェクトプール上限
**前提条件**: 既に30個の丸太がアクティブ
**操作**: 新たに1本丸太を生成
**期待結果**: 最も古い丸太が削除され、新しい丸太が生成される

#### TC-007: VRC_Pickup動作
**前提条件**: 丸太が生成済み
**操作**: 丸太に近づいてEキー（デスクトップ）
**期待結果**: 丸太をPickupできる、再度EキーでDropできる

### 異常系テストケース

#### TC-E001: 無効な木の種類
**前提条件**: treeType=99（無効値）
**操作**: LogSpawner._SpawnLogs()を呼び出し
**期待結果**: treeType=0（Pine）にクランプされ、警告ログ出力

#### TC-E002: 無効なスキルレベル
**前提条件**: skillLevel=-5（無効値）
**操作**: LogSpawner._SpawnLogs()を呼び出し
**期待結果**: skillLevel=1にクランプされ、警告ログ出力

#### TC-E003: Log Prefab未設定
**前提条件**: LogSpawnerのlogPrefabフィールドがnull
**操作**: Start()実行
**期待結果**: エラーログ出力、初期化中断

#### TC-E004: プールサイズ0
**前提条件**: poolSize=0
**操作**: Start()実行
**期待結果**: エラーログ出力、初期化中断

#### TC-E005: maxActiveLogs > poolSize
**前提条件**: poolSize=40, maxActiveLogs=50
**操作**: Start()実行
**期待結果**: maxActiveLogsがpoolSizeに調整される、警告ログ出力

---

## 11. 成果物

### 変更ファイル
1. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/LogSpawner.cs`（新規）
2. `/Assets/WoodcutterCamp/Prefabs/Log.prefab`（新規）
3. `/Assets/WoodcutterCamp/Scripts/Gameplay/Woodcutting/TreeController.cs`（LogSpawner連携追加）
4. シーン内のLogSpawnerオブジェクト（新規配置）
5. シーン内の木オブジェクト（TreeControllerにLogSpawner参照追加）

### 追加テストコード
本作業ではUdon/UdonSharpの制約により自動テストコードは作成しませんが、上記テストケースを手動で実行してください。

### 更新したドキュメント
- 本WI-0012作業指示書（完了報告時に更新）

---

## 12. 備考

### 注意点
1. **オブジェクトプールの重要性**: GameObject.Instantiate()の連打はパフォーマンス低下の原因となるため、必ずプール再利用すること
2. **FIFO削除**: logSpawnTimes配列で生成時刻を記録し、最も古いものから削除することで公平性を保つ
3. **配列実装**: UdonSharpではList<T>が使えないため、固定長配列で実装（poolSize変更時は配列サイズも変更必要）
4. **Random.value**: 0.0〜1.0の浮動小数点を返すため、確率判定に使用
5. **AddForce()**: ForceMode.Impulseで瞬間的な力を加え、丸太を散らばらせる

### 制約事項
- UdonSharpではList<T>、Dictionary<K,V>が使用できないため、配列で実装
- Random.Seed()によるシード固定はできない（マルチプレイヤー環境で各クライアントで異なる乱数になる）
- VRC_PickupのOwnership転送により、丸太の所有権が移動する（ネットワーク帯域への影響は軽微）

### 次の作業との連携
- **WI-0013**: CoinManager実装後、TreeController経由でコイン報酬付与
- **WI-0015**: LogPickup実装で、丸太の運搬・納品機能を追加
- **WI-0005**: SkillManager実装後、TreeControllerでスキルレベルを取得してLogSpawnerに渡す

### パフォーマンス最適化のヒント
1. **丸太のポリゴン削減**: Cylinder 24面 → 12面に削減（LOD0のみ）
2. **Rigidbody Sleep**: Sleep Threshold = 0.05で早めにスリープさせる
3. **Collider最適化**: CapsuleColliderは高速（MeshColliderより軽量）
4. **プール再利用**: Instantiate()を最小限に抑える（初期化時のみ）
5. **テクスチャ圧縮**: ASTC 6x6圧縮でメモリ削減

### 参考資料
- **TDD.md**: Section 5.12（M-12 LogSpawner仕様）
- **FSD.md**: Section 3.1（FNC-001 伐採システム - 丸太生成仕様）
- **VRChat_Tools_API_Reference.md**: Section 2.4.2（VRC_Pickup設定）
- Unity公式ドキュメント: https://docs.unity3d.com/ScriptReference/Rigidbody.html
- UdonSharp制約: https://udonsharp.docs.vrchat.com/

### 推定作業時間
- **合計**: 1.5日（12時間）
  - Log Prefab作成: 1時間
  - LogSpawner.cs実装: 4時間
  - LogSpawnerオブジェクト設定: 0.5時間
  - TreeController連携: 1時間
  - ローカルテスト: 2時間
  - スキルボーナステスト: 1時間
  - パフォーマンステスト: 1.5時間
  - 統合テスト: 1時間

---

**作成者**: VRChat World Development Team
**作成日**: 2025-11-17
**最終更新**: 2025-11-17
**ステータス**: 作業指示確定
**優先度**: 最高
**依存関係**: WI-0001, WI-0002, WI-0007
**関連モジュール**: M-12 LogSpawner, M-10 TreeController
**関連機能**: F-40, F-41, F-42
