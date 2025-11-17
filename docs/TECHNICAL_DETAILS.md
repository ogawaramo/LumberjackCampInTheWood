# VRChat 技術詳細リファレンス

**作成日**: 2025年11月17日  
**対象SDK**: 3.9.0 - 3.9.1-beta.2

---

## A. PlayerData/Persistence API 詳細仕様

### A1. Persistence ライフサイクル

```
Start()
  ↓
OnPlayerRestored()  ← ここで初めてデータ安全アクセス
  ↓
OnPreSerialization() ← 送信前
  ↓
[Network送信]
  ↓
OnPostSerialization() ← 送信後
```

### A2. データ型サポート表 (完全版)

```csharp
// ✓ サポート済み
int, long → VRCPlayerData.SetInt/GetInt
float, double → VRCPlayerData.SetFloat/GetFloat
Vector3 → VRCPlayerData.SetVector/GetVector
Quaternion → VRCPlayerData.SetQuaternion/GetQuaternion
Color → VRCPlayerData.SetColor/GetColor
string → VRCPlayerData.SetString/GetString
byte[] → VRCPlayerData.SetByteArray/GetByteArray

// ❌ 非対応
List<T>, Dictionary<K,V> → PlayerData直接は不可（JSON化して byte[] で対応）
bool → int の 0/1 で代用
```

### A3. 実装パターン: ゲームスコア保存

```csharp
public class GameStatePersistence : UdonSharpBehaviour
{
    [UdonSynced(UdonSyncMode.Manual)]
    public int savedScore;
    
    private int currentScore;
    
    public override void OnPlayerRestored()
    {
        // データ読み込み
        currentScore = VRCPlayerData.GetInt("score", 0);
        savedScore = currentScore;
        Debug.Log($"Game restored with score: {currentScore}");
    }
    
    public void AddPoints(int points)
    {
        currentScore += points;
        
        // UI更新
        _uiText.text = currentScore.ToString();
        
        // スコア閾値で自動保存
        if(currentScore % 100 == 0)
        {
            SaveScore();
        }
    }
    
    private void SaveScore()
    {
        // PlayerData に書き込み
        VRCPlayerData.SetInt("score", currentScore);
        
        // ネットワーク同期も更新
        savedScore = currentScore;
        RequestSerialization();
    }
    
    public override void OnPreSerialization()
    {
        // 送信直前に最新値を確認
        Debug.Log($"Saving score: {savedScore}");
    }
}
```

---

## B. ネットワーク同期 詳細仕様

### B1. RequestSerialization() のメカニズム

```csharp
public class NetworkManager : UdonSharpBehaviour
{
    [UdonSynced(UdonSyncMode.Manual)]
    public int state;
    
    private float lastSerializationTime;
    private const float MIN_SERIALIZATION_INTERVAL = 0.1f;
    
    public void UpdateState(int newState)
    {
        state = newState;
        
        // ✓ 正しい: 最小間隔をチェック
        if(Time.time - lastSerializationTime > MIN_SERIALIZATION_INTERVAL)
        {
            RequestSerialization();
            lastSerializationTime = Time.time;
        }
        // ❌ 誤り: 毎フレーム呼び出し
        // RequestSerialization(); // 実際は 0.1s 間隔で送信されるだけ
    }
    
    public override void OnPreSerialization()
    {
        // Serialization開始直前に呼ばれる
        Debug.Log("About to send state");
    }
    
    public override void OnPostSerialization(SerializationResult result)
    {
        // Serialization完了時に呼ばれる
        if(result.success)
        {
            Debug.Log("Successfully sent state");
        }
        else
        {
            Debug.LogError("Failed to send state");
        }
    }
}
```

### B2. Continuous Sync の自動送信周期

```
VRChat 内部ロジック:
- 20ms (50Hz) での定期チェック
- 変更があれば 次のフレーム で送信
- 変更なければ スキップ

実効レート (stable):
- 送信側: 50-60 Hz
- 受信側: その時点での最新値のみ
- 中間値は スキップ される
```

### B3. 帯域幅計算例

```
Continuous 例:
- Vector3 position: 12 bytes
- float health: 4 bytes
- int score: 4 bytes
→ 計 20 bytes / serialization
→ 50 Hz で送信
→ 1000 bytes/s = 1 KB/s

100プレイヤー World の場合:
→ 100 KB/s 必要
→ レート制限 (4-6 KB/s) に達して rate limited
→ 実効的に順番に送信される (QoS機能)
```

---

## C. Quest最適化 実装チェックリスト

### C1. ワールド最適化

```
□ ポリゴン数計測
  └─ Editor: Stats window で確認
  └─ 目標: 250,000 以下

□ テクスチャ最適化
  └─ フォーマット: ASTC (Quest 推奨)
  └─ 解像度: 512x512 - 1024x1024
  └─ 圧縮率: 50% 目標

□ メッシュ結合 (batching)
  □ Dynamic Batching 有効
  □ Static Batching 活用
  □ Occlusion Culling 設定

□ Light設定
  □ Realtime Lights: 1-2 個のみ
  □ Baked Light: 推奨
  □ Shadow: 無効化を検討
```

### C2. パフォーマンステスト コマンド

```
# Unity Editor での確認
Window > Analysis > Frame Debugger
Window > 2D > Sprite Sorter
Device > Android > Performance Settings

# 実機テスト
adb logcat | grep "FPS\|Performance"
```

### C3. ポリゴン削減テクニック

```csharp
// ❌ 高ポリゴンメッシュ
Mesh mesh = GetComponent<MeshFilter>().mesh;
// vertices: 100,000
// triangles: 50,000

// ✅ LOD system で削減
LODGroup lod = gameObject.AddComponent<LODGroup>();
// LOD0: full detail
// LOD1: 50% reduction
// LOD2: 10% reduction
// Culled: hidden
```

---

## D. UdonSharp コード例集

### D1. PlayerData ローカル値の同期

```csharp
public class GameManager : UdonSharpBehaviour
{
    // Local player data (Persistence)
    [UdonSynced(UdonSyncMode.Manual)]
    public int kills, deaths;
    
    public void OnKill()
    {
        kills++;
        VRCPlayerData.SetInt("kills", kills);
        RequestSerialization();
    }
    
    public override void OnPlayerRestored()
    {
        int savedKills = VRCPlayerData.GetInt("kills", 0);
        kills = savedKills;
    }
}
```

### D2. Continuous での位置同期

```csharp
public class PlayerMovement : UdonSharpBehaviour
{
    [UdonSynced(UdonSyncMode.Continuous)]
    public Vector3 networkPosition;
    
    private Vector3 desiredPosition;
    private float interpolationSpeed = 10f;
    
    void Update()
    {
        // ローカル入力で目標位置を決定
        desiredPosition = GetInputPosition();
        
        // Continuous で他プレイヤーに送信
        networkPosition = desiredPosition;
        
        // 他プレイヤーからの情報で補間
        Vector3 diff = networkPosition - transform.position;
        transform.position += diff * Time.deltaTime * interpolationSpeed;
    }
}
```

### D3. 非Behaviour クラス (1.2.0-beta)

```csharp
// ❌ 1.1.9: 不可
public class MathHelper
{
    public static float Distance(Vector3 a, Vector3 b)
    {
        return Vector3.Distance(a, b);
    }
}

// ✅ 1.2.0-beta: 可能
public class GameStateMachine : UdonSharpBehaviour
{
    private StateHelper helper = new StateHelper();
    
    void Start()
    {
        helper.Initialize(gameObject);
    }
}

// 非Behaviour クラス (1.2.0+のみ)
public class StateHelper
{
    public void Initialize(GameObject obj) { }
}
```

### D4. Dictionary を使った在庫管理 (1.2.0-beta)

```csharp
public class InventorySystem : UdonSharpBehaviour
{
    // 1.2.0-beta: Dictionary サポート
    private Dictionary<string, int> inventory;
    
    void Start()
    {
        inventory = new Dictionary<string, int>();
    }
    
    public void AddItem(string itemName, int count)
    {
        if(inventory.ContainsKey(itemName))
        {
            inventory[itemName] += count;
        }
        else
        {
            inventory[itemName] = count;
        }
    }
    
    public int GetItemCount(string itemName)
    {
        return inventory.ContainsKey(itemName) ? inventory[itemName] : 0;
    }
}
```

---

## E. World PhysBones API (SDK 3.9.1+)

### E1. PhysBone の動的制御

```csharp
public class PhysBoneController : UdonSharpBehaviour
{
    public VRCPhysBone bone;
    
    void Start()
    {
        // PhysBone パラメータの設定
        bone.enabled = true;
        bone.radius = 0.05f;
        bone.spring = 50f;
        bone.damping = 0.9f;
    }
    
    public void SetBoneActive(bool active)
    {
        bone.enabled = active;
    }
    
    public void SetBoneRadius(float radius)
    {
        bone.radius = radius;
        // 他プレイヤーにも反映させるには
        RequestSerialization();
    }
}
```

### E2. Contact イベント

```csharp
public class ContactDetector : UdonSharpBehaviour
{
    public VRCContactReceiver contact;
    
    public override void OnContactEnter(Collider other)
    {
        Debug.Log($"Contact enter: {other.gameObject.name}");
        
        // プレイヤーが接触したか確認
        if(other.CompareTag("Player"))
        {
            OnPlayerContact();
        }
    }
    
    public override void OnContactExit(Collider other)
    {
        Debug.Log($"Contact exit: {other.gameObject.name}");
    }
    
    private void OnPlayerContact()
    {
        // 接触時の処理
        Debug.Log("Player contacted!");
    }
}
```

### E3. Constraints の制御

```csharp
public class ConstraintController : UdonSharpBehaviour
{
    public VRCConstraintBase constraint;
    
    void Start()
    {
        // Constraint の有効/無効
        constraint.enabled = true;
    }
    
    public void ToggleConstraint()
    {
        constraint.enabled = !constraint.enabled;
    }
}
```

---

## F. パフォーマンス計測

### F1. Udon 実行時間計測

```csharp
public class PerformanceMonitor : UdonSharpBehaviour
{
    private float updateDeltaTime;
    
    void Update()
    {
        float startTime = Time.realtimeSinceStartup;
        
        // 計測対象コード
        HeavyCalculation();
        
        float endTime = Time.realtimeSinceStartup;
        updateDeltaTime = endTime - startTime;
        
        Debug.Log($"Update took {updateDeltaTime * 1000} ms");
    }
    
    private void HeavyCalculation()
    {
        // 処理
    }
}
```

### F2. ネットワーク帯域幅計測

```csharp
public class NetworkMonitor : UdonSharpBehaviour
{
    private int lastSentBytes;
    
    void Update()
    {
        // VRChat Debug GUI で確認
        // --enable-debug-gui オプションで有効化
        // ↓
        // Network Stats パネルで帯域幅表示
    }
}
```

---

## G. デバッグテクニック

### G1. Debug GUI 起動

```bash
# Windows
"C:\Program Files (x86)\Steam\steamapps\common\VRChat\VRChat.exe" --enable-debug-gui

# Linux/Proton
PROTON_DEBUG=1 proton run VRChat.exe --enable-debug-gui
```

### G2. ログ出力

```csharp
public class Debugger : UdonSharpBehaviour
{
    void Update()
    {
        // Editor + BuildでConsole出力
        Debug.Log("Standard log");
        Debug.LogWarning("Warning");
        Debug.LogError("Error");
        
        // Quest環境ではファイル出力推奨
        // Persistent data path:
        // /sdcard/Android/data/com.vrchat.core/files/
    }
}
```

### G3. Network State 確認

```csharp
public class NetworkDebug : UdonSharpBehaviour
{
    void Update()
    {
        VRCPlayerApi player = Networking.LocalPlayer;
        
        Debug.Log($"IsOwner: {Networking.IsOwner(gameObject)}");
        Debug.Log($"IsMaster: {player.isMaster}");
        Debug.Log($"Ping: {player.GetNetworkLatency()} ms");
    }
}
```

---

## H. よくある落とし穴と対応策

### H1. Late Joiner がデータを受け取らない

**原因**: OnDeserialization() が呼ばれない

```csharp
// ❌ 悪い例
void Start()
{
    int value = VRCPlayerData.GetInt("score"); // まだ読み込まれていない
}

// ✓ 正しい
public override void OnPlayerRestored()
{
    int value = VRCPlayerData.GetInt("score"); // 安全
}
```

### H2. 帯域幅リミットにひっかかる

**原因**: Manual Sync で大量データを毎フレーム送信

```csharp
// ❌ 悪い: 毎フレーム大量データ送信
for(int i = 0; i < 10000; i++)
{
    syncData[i] = someValue;
}
RequestSerialization(); // 毎フレーム → Rate limited!

// ✓ 正しい: 変更があるときのみ
if(dataChanged)
{
    // 必要なデータのみ更新
    syncData[changedIndex] = newValue;
    RequestSerialization();
}
```

### H3. 複数UdonBehaviour での同期競合

**仕様**: 同じGameObject上の複数Behaviourは最も制限的なモード準用

```csharp
// Object1: Continuous
[UdonSynced(UdonSyncMode.Continuous)]
public Vector3 pos;

// Object2: Manual (同じObject上)
[UdonSynced(UdonSyncMode.Manual)]
public int state;

// → 両方ともManual扱いになる！
// → 200 bytes制限は両方に適用される
```

**対応**: Player Objects で分離

```csharp
// PlayerObject 1: Continuous
// PlayerObject 2: Manual
// → 独立した制限を適用
```

---

## I. Soba への移行ガイド (未リリース)

### I1. 予想される C# 互換性

```csharp
// Soba (予想)
public class GameLogic
{
    public List<int> scores = new List<int>();
    
    public Dictionary<string, Player> players = 
        new Dictionary<string, Player>();
    
    public void ProcessGame()
    {
        // フル C# 対応予定
        foreach(var player in players.Values)
        {
            player.Update();
        }
    }
}
```

### I2. 移行チェックリスト (準備)

```
現在 (2025):
□ UdonSharp 1.1.9 で堅牢なコード作成
□ Soba アナウンス監視
□ 非推奨API の利用を最小化

Soba Early Access 開始時:
□ 試験的なプロジェクトで評価
□ UdonSharp との共存確認
□ ドキュメント読破

Soba 正式リリース時:
□ 段階的な既存World移行
□ パフォーマンステスト
```

---

**詳細リファレンス完了** - TDD.md 作成時の技術仕様として参照してください

