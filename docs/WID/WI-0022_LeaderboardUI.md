# 作業指示書 WI-0022: LeaderboardUI実装

## 1. 作業ID
**WI-0022**

## 2. 作業名
LeaderboardUI（ランキングボードUI）の実装

## 3. 作業目的
インスタンス内のプレイヤー伐採ランキングを表示するUIシステムを実装し、プレイヤー間の健全な競争とコミュニティ感の醸成を実現する。TOP10のランキング表示、ローカルプレイヤーのハイライト表示、5秒ごとの自動更新機能を提供する。

## 4. 対象

### 4.1 対象システム
- **プロジェクト名**: 森のきこりキャンプ（Woodcutter Camp）
- **Phase**: Phase 1
- **VRChatワールド**: Quest最適化対応

### 4.2 対象モジュール
- **モジュールID**: M-09 LeaderboardUI
- **レイヤー**: UI Layer
- **依存モジュール**:
  - M-01 GameManager（初期化・依存性注入）
  - M-16 RankingTracker（ランキングデータソース）

### 4.3 対象ファイルパス
```
Assets/
└── WoodcutterCamp/
    └── Scripts/
        └── UI/
            └── LeaderboardUI.cs  # 新規作成
```

### 4.4 Unity Scene設定
```
WoodcutterCamp Scene/
└── [キャンプ中央広場]
    └── LeaderboardBoard (GameObject)
        ├── Canvas (WorldSpace)
        ├── RankingPanel
        │   ├── Header
        │   ├── RankingRowsContainer (10行)
        │   └── Footer
        └── LeaderboardUI (UdonBehaviour)
```

## 5. 前提条件

### 5.1 環境・バージョン
- Unity 2022.3.22f1 LTS
- VRChat SDK 3.9.0
- UdonSharp 1.1.9
- TextMeshPro 3.0.6以上

### 5.2 依存関係
- **必須完了作業**:
  - WI-0001: GameManager Singleton実装済み
  - WI-0021: RankingTracker実装済み（データソース）
- **必須リソース**:
  - TextMeshPro フォント（日本語対応）
  - ランキングアイコンスプライト（🥇🥈🥉またはテクスチャ）

### 5.3 初期状態
- Unityシーンにキャンプ中央広場のエリアが配置済み
- RankingTrackerがプレイヤー統計を収集中
- GameManagerが正常に初期化されている状態

## 6. 入力

### 6.1 データソース（RankingTracker経由）
```csharp
// RankingTrackerから取得するデータ
public struct RankingEntry
{
    public string playerName;        // プレイヤー名（最大20文字）
    public int treesCut;             // 伐採した木の数
    public int totalDamage;          // 累計与ダメージ
    public int coinsEarned;          // 獲得コイン数
    public int playerID;             // プレイヤーID（0-19）
}

// TOP10配列
RankingEntry[] top10Rankings;

// ローカルプレイヤーのランク（TOP10外の場合）
int localPlayerRank;              // 例: 15位
RankingEntry localPlayerEntry;
```

### 6.2 更新トリガー
- **自動更新**: 5秒ごとにRankingTrackerからデータ取得
- **手動更新**: GameManagerからの明示的な更新通知（レアケース）

## 7. 出力

### 7.1 UI表示内容
```
=========================================
        🏆 伐採ランキング 🏆
=========================================
ランク | プレイヤー名      | 伐採 | ダメージ | コイン
---------+------------------+------+----------+-------
🥇 1位   | PlayerABC         | 42本 | 1250     | 245
🥈 2位   | PlayerXYZ         | 38本 | 1140     | 220
🥉 3位   | LongNamePlayer... | 31本 | 930      | 185
  4位   | Player_D          | 28本 | 840      | 168
  5位   | Player_E          | 25本 | 750      | 150
  6位   | [あなた]          | 22本 | 660      | 132 ★
  7位   | Player_G          | 20本 | 600      | 120
  8位   | Player_H          | 18本 | 540      | 108
  9位   | Player_I          | 15本 | 450      | 90
 10位   | Player_J          | 12本 | 360      | 72
=========================================
```

### 7.2 ハイライト表示
- **ローカルプレイヤー行**: 背景色を半透明イエロー（#FFFF0080）に変更
- **ランク1-3位**: ランク番号を特別な色で表示
  - 1位: ゴールド（#FFD700）
  - 2-3位: シルバー（#C0C0C0）
  - 4-7位: ホワイト（#FFFFFF）
  - 8-10位: ブロンズ（#CD7F32）
- **1位のみ**: 王冠アイコン（👑）または王冠スプライトを表示

## 8. 作業手順

### 8.1 Unity Scene準備（Unityエディタ作業）

#### ステップ1: LeaderboardBoardゲームオブジェクト作成
1. Hierarchyで右クリック → `Create Empty`
2. 名前を `LeaderboardBoard` に変更
3. Transform設定:
   ```
   Position: キャンプ中央広場の適切な位置（例: X=0, Y=2, Z=-5）
   Rotation: プレイヤー方向を向くように調整（例: Y=180）
   Scale: (1, 1, 1)
   ```

#### ステップ2: WorldSpace Canvas作成
1. `LeaderboardBoard` の子オブジェクトとして `Canvas` を追加
   - 右クリック → `UI` → `Canvas`
2. Canvas設定:
   ```
   Render Mode: World Space
   Width: 200
   Height: 300
   Scale: (0.01, 0.01, 0.01)  # 実サイズ2m×3mになるよう調整
   ```
3. Canvas ScalerコンポーネントはWorld Spaceのため不要

#### ステップ3: RankingPanel作成
1. Canvasの子として `Image` オブジェクト追加（Panel背景）
2. 名前を `RankingPanel` に変更
3. RectTransform設定:
   ```
   Anchor: ストレッチ（左上から右下まで）
   Left/Right/Top/Bottom: 各10ピクセルのマージン
   ```
4. Image設定:
   ```
   Color: 半透明黒（R=0, G=0, B=0, A=180）
   Material: None（デフォルト）
   ```

#### ステップ4: Header（ヘッダー）作成
1. RankingPanelの子として `TextMeshProUGUI` 追加
2. 名前を `Header` に変更
3. RectTransform設定:
   ```
   Anchor: Top Center
   Pivot: (0.5, 1)
   Pos Y: -10（上端から10px下）
   Width: 180
   Height: 40
   ```
4. TextMeshPro設定:
   ```
   Text: "🏆 伐採ランキング 🏆"
   Font Size: 24
   Alignment: Center
   Color: ゴールド（#FFD700）
   Font Style: Bold
   ```

#### ステップ5: RankingRowsContainer作成
1. RankingPanelの子として `Empty GameObject` 追加
2. 名前を `RankingRowsContainer` に変更
3. RectTransform設定:
   ```
   Anchor: Top Stretch
   Pivot: (0.5, 1)
   Pos Y: -60（ヘッダー下）
   Height: 200
   ```
4. Vertical Layout Groupコンポーネント追加:
   ```
   Child Alignment: Upper Center
   Spacing: 2
   Child Force Expand: Width=true, Height=false
   ```

#### ステップ6: RankingRow Prefab作成（10行分のテンプレート）
1. RankingRowsContainerの子として `Image` 追加
2. 名前を `RankingRow_Template` に変更
3. RectTransform設定:
   ```
   Height: 18
   Width: 180（親から自動調整）
   ```
4. Image設定:
   ```
   Color: 透明（通常時）
   Material: None
   ```
5. RankingRow_Templateの子として5つのTextMeshProUGUIを追加:

   **A. RankText（ランク番号）**
   ```
   Name: RankText
   Anchor: Left
   Width: 40
   Height: 18
   Font Size: 16
   Alignment: Center
   Text: "1位"
   ```

   **B. PlayerNameText（プレイヤー名）**
   ```
   Name: PlayerNameText
   Anchor: Left（RankTextの右隣）
   Width: 70
   Height: 18
   Font Size: 14
   Alignment: Left
   Text: "PlayerName"
   Overflow: Truncate（省略記号）
   ```

   **C. TreesText（伐採数）**
   ```
   Name: TreesText
   Anchor: Left
   Width: 30
   Height: 18
   Font Size: 14
   Alignment: Right
   Text: "42本"
   ```

   **D. DamageText（ダメージ）**
   ```
   Name: DamageText
   Anchor: Left
   Width: 35
   Height: 18
   Font Size: 12
   Alignment: Right
   Text: "1250"
   Color: グレー（#CCCCCC）
   ```

   **E. CoinsText（コイン）**
   ```
   Name: CoinsText
   Anchor: Right
   Width: 30
   Height: 18
   Font Size: 14
   Alignment: Right
   Text: "245"
   Color: ゴールド（#FFD700）
   ```

6. Prefab化:
   - `RankingRow_Template` をProject Windowの `Assets/WoodcutterCamp/Prefabs/UI/` にドラッグ
   - Prefab名: `RankingRowPrefab`

7. RankingRowsContainer内に`RankingRow_Template`を9回複製して合計10行作成
   - 名前を `RankingRow_0` 〜 `RankingRow_9` に変更

#### ステップ7: Footer（フッター）作成
1. RankingPanelの子として `TextMeshProUGUI` 追加
2. 名前を `Footer` に変更
3. RectTransform設定:
   ```
   Anchor: Bottom Center
   Pivot: (0.5, 0)
   Pos Y: 10（下端から10px上）
   Width: 180
   Height: 30
   ```
4. TextMeshPro設定:
   ```
   Text: "あなたの順位: 15位"
   Font Size: 14
   Alignment: Center
   Color: イエロー（#FFFF00）
   ```

#### ステップ8: LeaderboardUIスクリプトアタッチ
1. `LeaderboardBoard` ゲームオブジェクトを選択
2. `Add Component` → `Udon Sharp Behaviour`
3. スクリプトは後述の8.2で作成するため、一旦空のまま

### 8.2 LeaderboardUI.cs実装（コーディング作業）

#### ステップ1: ファイル作成とクラス構造定義
```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon;
using TMPro; // TextMeshPro使用

namespace WoodcutterCamp.UI
{
    /// <summary>
    /// ランキングボードUI管理クラス
    /// TOP10のプレイヤーランキングを表示し、5秒ごとに自動更新する
    /// </summary>
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)] // UIのみ、同期不要
    public class LeaderboardUI : UdonSharpBehaviour
    {
        #region Serialized Fields（Inspector設定項目）

        [Header("依存モジュール")]
        [Tooltip("GameManagerへの参照（初期化時に自動取得）")]
        [SerializeField] private UdonSharpBehaviour gameManagerBehaviour;

        [Tooltip("RankingTrackerへの参照（GameManager経由で取得）")]
        [SerializeField] private UdonSharpBehaviour rankingTrackerBehaviour;

        [Header("UI要素")]
        [Tooltip("ランキング行のTextMeshProUGUI配列（10行）")]
        [SerializeField] private TextMeshProUGUI[] rankTexts;       // ランク番号用
        [SerializeField] private TextMeshProUGUI[] playerNameTexts; // プレイヤー名用
        [SerializeField] private TextMeshProUGUI[] treesTexts;      // 伐採数用
        [SerializeField] private TextMeshProUGUI[] damageTexts;     // ダメージ用
        [SerializeField] private TextMeshProUGUI[] coinsTexts;      // コイン用
        [SerializeField] private Image[] rowBackgrounds;            // 行の背景Image

        [Tooltip("フッター（ローカルプレイヤー順位表示用）")]
        [SerializeField] private TextMeshProUGUI footerText;

        [Tooltip("Canvas全体への参照（距離チェック用）")]
        [SerializeField] private Canvas leaderboardCanvas;

        [Header("表示設定")]
        [Tooltip("自動更新間隔（秒）")]
        [SerializeField] private float updateInterval = 5.0f;

        [Tooltip("Canvas無効化する距離（メートル）")]
        [SerializeField] private float disableDistance = 20.0f;

        [Tooltip("プレイヤー名の最大文字数")]
        [SerializeField] private int maxNameLength = 20;

        [Header("色設定")]
        [Tooltip("ランクティア色")]
        [SerializeField] private Color goldColor = new Color(1.0f, 0.84f, 0.0f, 1.0f);      // #FFD700
        [SerializeField] private Color silverColor = new Color(0.75f, 0.75f, 0.75f, 1.0f);  // #C0C0C0
        [SerializeField] private Color bronzeColor = new Color(0.8f, 0.5f, 0.2f, 1.0f);     // #CD7F32
        [SerializeField] private Color normalColor = new Color(1.0f, 1.0f, 1.0f, 1.0f);     // #FFFFFF

        [Tooltip("ローカルプレイヤーハイライト色")]
        [SerializeField] private Color localPlayerHighlight = new Color(1.0f, 1.0f, 0.0f, 0.5f); // #FFFF0080

        #endregion

        #region Private Fields（内部変数）

        private VRCPlayerApi localPlayer;
        private float nextUpdateTime = 0f;
        private bool isInitialized = false;

        // キャッシュしたランキングデータ（更新差分チェック用）
        private string[] cachedPlayerNames = new string[10];
        private int[] cachedTreesCut = new int[10];
        private int cachedLocalPlayerRank = -1;

        // RankingTrackerから取得する定数
        private const int MAX_RANKING_DISPLAY = 10;

        #endregion

        #region Unity Lifecycle

        /// <summary>
        /// 初期化処理
        /// 1秒遅延でLate Joiner対応
        /// </summary>
        void Start()
        {
            // Late Joiner対応のため1秒遅延
            SendCustomEventDelayedSeconds(nameof(_Initialize), 1.0f);
        }

        /// <summary>
        /// 距離チェックとCanvas有効化制御
        /// </summary>
        void Update()
        {
            if (!isInitialized) return;

            // ローカルプレイヤーとの距離をチェック
            if (localPlayer != null && leaderboardCanvas != null)
            {
                float distance = Vector3.Distance(
                    localPlayer.GetPosition(),
                    transform.position
                );

                // 距離に応じてCanvas有効/無効切り替え（パフォーマンス最適化）
                bool shouldBeEnabled = distance <= disableDistance;
                if (leaderboardCanvas.enabled != shouldBeEnabled)
                {
                    leaderboardCanvas.enabled = shouldBeEnabled;
                }
            }

            // 自動更新タイミングチェック
            if (Time.time >= nextUpdateTime)
            {
                _UpdateLeaderboard();
                nextUpdateTime = Time.time + updateInterval;
            }
        }

        #endregion

        #region Public Methods（Udon CustomEvent）

        /// <summary>
        /// 初期化（Udon CustomEvent）
        /// GameManagerからモジュール参照を取得し、UI要素を初期化する
        /// </summary>
        public void _Initialize()
        {
            Debug.Log("[LeaderboardUI] 初期化開始");

            // ローカルプレイヤー取得
            localPlayer = Networking.LocalPlayer;
            if (localPlayer == null)
            {
                Debug.LogError("[LeaderboardUI] ローカルプレイヤーが取得できません");
                return;
            }

            // GameManagerから依存モジュール取得
            if (gameManagerBehaviour != null)
            {
                // GameManager経由でRankingTrackerを取得
                // 実装例: gameManagerBehaviour.GetComponent<RankingTracker>()
                // ※実際のGameManager実装に応じて調整
                rankingTrackerBehaviour = (UdonSharpBehaviour)gameManagerBehaviour.GetProgramVariable("rankingTracker");

                if (rankingTrackerBehaviour == null)
                {
                    Debug.LogError("[LeaderboardUI] RankingTrackerの取得に失敗しました");
                    return;
                }
            }
            else
            {
                Debug.LogError("[LeaderboardUI] GameManagerが設定されていません");
                return;
            }

            // UI要素の検証
            if (!ValidateUIElements())
            {
                Debug.LogError("[LeaderboardUI] UI要素が不足しています");
                return;
            }

            // 初期表示
            _UpdateLeaderboard();

            // 次回更新時刻設定
            nextUpdateTime = Time.time + updateInterval;

            isInitialized = true;
            Debug.Log("[LeaderboardUI] 初期化完了");
        }

        /// <summary>
        /// ランキング表示更新（Udon CustomEvent）
        /// RankingTrackerから最新データを取得し、UIを更新する
        /// </summary>
        public void _UpdateLeaderboard()
        {
            if (!isInitialized) return;

            Debug.Log("[LeaderboardUI] ランキング更新開始");

            // RankingTrackerからTOP10データを取得
            string[] playerNames = (string[])rankingTrackerBehaviour.GetProgramVariable("topPlayerNames");
            int[] treesCut = (int[])rankingTrackerBehaviour.GetProgramVariable("topTreesCut");
            int[] totalDamage = (int[])rankingTrackerBehaviour.GetProgramVariable("topTotalDamage");
            int[] coinsEarned = (int[])rankingTrackerBehaviour.GetProgramVariable("topCoinsEarned");
            int[] playerIDs = (int[])rankingTrackerBehaviour.GetProgramVariable("topPlayerIDs");

            if (playerNames == null || treesCut == null)
            {
                Debug.LogWarning("[LeaderboardUI] ランキングデータが取得できません");
                return;
            }

            // ローカルプレイヤーのランク取得（TOP10外の場合のみ）
            int localPlayerRank = (int)rankingTrackerBehaviour.GetProgramVariable("localPlayerRank");

            // 各行を更新（差分更新で最適化）
            for (int i = 0; i < MAX_RANKING_DISPLAY; i++)
            {
                if (i < playerNames.Length && !string.IsNullOrEmpty(playerNames[i]))
                {
                    // データが変更された行のみ更新
                    bool needsUpdate = (cachedPlayerNames[i] != playerNames[i]) ||
                                       (cachedTreesCut[i] != treesCut[i]);

                    if (needsUpdate)
                    {
                        UpdateRankingRow(i, playerNames[i], treesCut[i], totalDamage[i], coinsEarned[i], playerIDs[i]);
                        cachedPlayerNames[i] = playerNames[i];
                        cachedTreesCut[i] = treesCut[i];
                    }
                }
                else
                {
                    // データがない行は空表示
                    ClearRankingRow(i);
                    cachedPlayerNames[i] = null;
                    cachedTreesCut[i] = 0;
                }
            }

            // ローカルプレイヤーハイライト更新
            if (localPlayerRank != cachedLocalPlayerRank)
            {
                HighlightLocalPlayer(localPlayerRank, playerNames, playerIDs);
                cachedLocalPlayerRank = localPlayerRank;
            }

            // フッター更新（TOP10外の場合）
            UpdateFooter(localPlayerRank);

            Debug.Log("[LeaderboardUI] ランキング更新完了");
        }

        #endregion

        #region Private Methods（内部処理）

        /// <summary>
        /// UI要素の検証
        /// 必要なUI要素がすべて設定されているか確認
        /// </summary>
        private bool ValidateUIElements()
        {
            if (rankTexts == null || rankTexts.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] rankTexts配列が正しく設定されていません");
                return false;
            }

            if (playerNameTexts == null || playerNameTexts.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] playerNameTexts配列が正しく設定されていません");
                return false;
            }

            if (treesTexts == null || treesTexts.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] treesTexts配列が正しく設定されていません");
                return false;
            }

            if (damageTexts == null || damageTexts.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] damageTexts配列が正しく設定されていません");
                return false;
            }

            if (coinsTexts == null || coinsTexts.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] coinsTexts配列が正しく設定されていません");
                return false;
            }

            if (rowBackgrounds == null || rowBackgrounds.Length != MAX_RANKING_DISPLAY)
            {
                Debug.LogError("[LeaderboardUI] rowBackgrounds配列が正しく設定されていません");
                return false;
            }

            if (footerText == null)
            {
                Debug.LogError("[LeaderboardUI] footerTextが設定されていません");
                return false;
            }

            if (leaderboardCanvas == null)
            {
                Debug.LogError("[LeaderboardUI] leaderboardCanvasが設定されていません");
                return false;
            }

            return true;
        }

        /// <summary>
        /// ランキング行の更新
        /// </summary>
        /// <param name="index">行インデックス（0-9）</param>
        /// <param name="playerName">プレイヤー名</param>
        /// <param name="trees">伐採数</param>
        /// <param name="damage">累計ダメージ</param>
        /// <param name="coins">獲得コイン</param>
        /// <param name="playerID">プレイヤーID</param>
        private void UpdateRankingRow(int index, string playerName, int trees, int damage, int coins, int playerID)
        {
            int rank = index + 1; // 1位から表示

            // ランク番号とティア色設定
            string rankText;
            Color rankColor;

            if (rank == 1)
            {
                rankText = "👑 1位"; // 王冠アイコン付き
                rankColor = goldColor;
            }
            else if (rank <= 3)
            {
                rankText = rank + "位";
                rankColor = rank == 2 ? silverColor : bronzeColor;
            }
            else if (rank <= 7)
            {
                rankText = rank + "位";
                rankColor = normalColor;
            }
            else
            {
                rankText = rank + "位";
                rankColor = bronzeColor;
            }

            rankTexts[index].text = rankText;
            rankTexts[index].color = rankColor;

            // プレイヤー名（最大文字数で切り捨て）
            string displayName = TruncatePlayerName(playerName, maxNameLength);
            playerNameTexts[index].text = displayName;

            // 伐採数
            treesTexts[index].text = trees + "本";

            // ダメージ
            damageTexts[index].text = damage.ToString();

            // コイン
            coinsTexts[index].text = coins.ToString();

            // 背景色をデフォルトに戻す（ハイライトがリセットされる場合に備え）
            rowBackgrounds[index].color = Color.clear;
        }

        /// <summary>
        /// ランキング行をクリア
        /// </summary>
        /// <param name="index">行インデックス</param>
        private void ClearRankingRow(int index)
        {
            rankTexts[index].text = "";
            playerNameTexts[index].text = "";
            treesTexts[index].text = "";
            damageTexts[index].text = "";
            coinsTexts[index].text = "";
            rowBackgrounds[index].color = Color.clear;
        }

        /// <summary>
        /// プレイヤー名の切り捨て
        /// </summary>
        /// <param name="name">元の名前</param>
        /// <param name="maxLength">最大文字数</param>
        /// <returns>切り捨てられた名前</returns>
        private string TruncatePlayerName(string name, int maxLength)
        {
            if (string.IsNullOrEmpty(name))
                return "---";

            if (name.Length <= maxLength)
                return name;

            return name.Substring(0, maxLength - 3) + "...";
        }

        /// <summary>
        /// ローカルプレイヤーのハイライト表示
        /// </summary>
        /// <param name="localRank">ローカルプレイヤーのランク（-1=圏外）</param>
        /// <param name="playerNames">TOP10プレイヤー名配列</param>
        /// <param name="playerIDs">TOP10プレイヤーID配列</param>
        private void HighlightLocalPlayer(int localRank, string[] playerNames, int[] playerIDs)
        {
            if (localPlayer == null) return;

            // すべての背景色をリセット
            for (int i = 0; i < MAX_RANKING_DISPLAY; i++)
            {
                rowBackgrounds[i].color = Color.clear;
            }

            // ローカルプレイヤーがTOP10内にいる場合
            if (localRank >= 1 && localRank <= MAX_RANKING_DISPLAY)
            {
                int index = localRank - 1;

                // プレイヤーIDで一致確認（念のため）
                int localPlayerId = localPlayer.playerId;
                if (playerIDs != null && index < playerIDs.Length && playerIDs[index] == localPlayerId)
                {
                    rowBackgrounds[index].color = localPlayerHighlight;

                    // プレイヤー名に[あなた]マーク追加
                    string currentText = playerNameTexts[index].text;
                    if (!currentText.Contains("[あなた]"))
                    {
                        playerNameTexts[index].text = "[あなた] " + currentText;
                    }

                    // フォントをBoldに変更
                    playerNameTexts[index].fontStyle = FontStyles.Bold;

                    Debug.Log($"[LeaderboardUI] ローカルプレイヤーをハイライト（{localRank}位）");
                }
            }
        }

        /// <summary>
        /// フッター更新（TOP10外の場合の順位表示）
        /// </summary>
        /// <param name="localRank">ローカルプレイヤーのランク</param>
        private void UpdateFooter(int localRank)
        {
            if (footerText == null) return;

            if (localRank > MAX_RANKING_DISPLAY)
            {
                // TOP10圏外の場合は順位を表示
                footerText.text = $"あなたの順位: {localRank}位";
                footerText.gameObject.SetActive(true);
            }
            else if (localRank >= 1 && localRank <= MAX_RANKING_DISPLAY)
            {
                // TOP10圏内の場合はフッター非表示
                footerText.gameObject.SetActive(false);
            }
            else
            {
                // ランク未確定（伐採未実施など）
                footerText.text = "順位: --位（伐採を開始してください）";
                footerText.gameObject.SetActive(true);
            }
        }

        #endregion
    }
}
```

### 8.3 Unity Inspector設定（エディタ作業）

#### ステップ1: LeaderboardUIスクリプトの参照設定
1. `LeaderboardBoard` ゲームオブジェクトを選択
2. Inspectorで `LeaderboardUI (Script)` コンポーネントを確認
3. 以下の参照を設定:

**依存モジュール**:
```
Game Manager Behaviour: GameManagerオブジェクトをドラッグ
Ranking Tracker Behaviour: （初期化時に自動取得されるため空でOK）
```

**UI要素配列（10個ずつ）**:
```
Rank Texts (Size: 10):
  Element 0: RankingRow_0/RankText
  Element 1: RankingRow_1/RankText
  ...
  Element 9: RankingRow_9/RankText

Player Name Texts (Size: 10):
  Element 0: RankingRow_0/PlayerNameText
  ...（同様に設定）

Trees Texts (Size: 10):
  Element 0: RankingRow_0/TreesText
  ...（同様に設定）

Damage Texts (Size: 10):
  Element 0: RankingRow_0/DamageText
  ...（同様に設定）

Coins Texts (Size: 10):
  Element 0: RankingRow_0/CoinsText
  ...（同様に設定）

Row Backgrounds (Size: 10):
  Element 0: RankingRow_0（Imageコンポーネント）
  ...（同様に設定）
```

**その他のUI要素**:
```
Footer Text: Footer（TextMeshProUGUI）
Leaderboard Canvas: Canvas（Canvasコンポーネント）
```

**表示設定**:
```
Update Interval: 5.0
Disable Distance: 20.0
Max Name Length: 20
```

**色設定**:
```
Gold Color: #FFD700（R=1.0, G=0.84, B=0.0, A=1.0）
Silver Color: #C0C0C0（R=0.75, G=0.75, B=0.75, A=1.0）
Bronze Color: #CD7F32（R=0.8, G=0.5, B=0.2, A=1.0）
Normal Color: #FFFFFF（R=1.0, G=1.0, B=1.0, A=1.0）
Local Player Highlight: #FFFF0080（R=1.0, G=1.0, B=0.0, A=0.5）
```

### 8.4 UdonSharpコンパイル

#### ステップ1: コンパイル実行
1. Unity上部メニュー → `UdonSharp` → `Compile All UdonSharp Programs`
2. Consoleでエラーがないことを確認
3. エラーがある場合:
   - エラーメッセージを確認し、該当行を修正
   - `TMPro`の名前空間インポートを確認
   - `UdonSharpBehaviour`の継承を確認

#### ステップ2: Udon Behaviourへの変換確認
1. `LeaderboardBoard` を選択
2. InspectorでコンポーネントがUdon Behaviourに変換されていることを確認
3. Program Sourceが `LeaderboardUI` になっていることを確認

### 8.5 テスト準備

#### ステップ1: RankingTrackerのモックデータ設定
テスト用にRankingTrackerにダミーデータを設定する:

```csharp
// RankingTracker側のテストコード例（実際のWI-0021実装に含める）
public void _SetTestData()
{
    topPlayerNames = new string[]
    {
        "TestPlayer1", "TestPlayer2", "TestPlayer3",
        "TestPlayer4", "TestPlayer5", "TestPlayer6",
        "TestPlayer7", "TestPlayer8", "TestPlayer9",
        "TestPlayer10"
    };
    topTreesCut = new int[] { 42, 38, 31, 28, 25, 22, 20, 18, 15, 12 };
    topTotalDamage = new int[] { 1250, 1140, 930, 840, 750, 660, 600, 540, 450, 360 };
    topCoinsEarned = new int[] { 245, 220, 185, 168, 150, 132, 120, 108, 90, 72 };
    topPlayerIDs = new int[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
}
```

#### ステップ2: ClientSimでローカルテスト
1. Unity上部メニュー → `VRChat SDK` → `Show Control Panel`
2. `Builder` タブ → `Build & Test`
3. ClientSimが起動したら、以下を確認:
   - ランキングボードが正しく表示されるか
   - 5秒ごとに自動更新されるか（Consoleログを確認）
   - プレイヤーとの距離に応じてCanvas有効化/無効化されるか

## 9. 完了条件（Doneの定義）

### 9.1 実装完了チェックリスト
- [ ] LeaderboardUI.csが正しくコンパイルされ、エラーがない
- [ ] Unity SceneにLeaderboardBoardが配置され、すべてのUI要素が設定されている
- [ ] Inspector上ですべての参照が正しく設定されている
- [ ] Udon Behaviourへの変換が完了している
- [ ] ClientSimでローカルテスト実行時、エラーログが出ない

### 9.2 機能完了チェックリスト
- [ ] TOP10のランキングが正しく表示される
- [ ] プレイヤー名が20文字を超える場合、"..."で省略される
- [ ] ランクに応じて色分けされる（1位=ゴールド、2-3位=シルバー、4-7位=ホワイト、8-10位=ブロンズ）
- [ ] 1位に王冠アイコン（👑）が表示される
- [ ] ローカルプレイヤーの行が黄色背景でハイライトされる
- [ ] ローカルプレイヤーの名前に[あなた]マークが付く
- [ ] ローカルプレイヤーの名前が太字になる
- [ ] TOP10圏外の場合、フッターに順位が表示される
- [ ] TOP10圏内の場合、フッターが非表示になる
- [ ] 5秒ごとに自動更新される
- [ ] 20m以上離れるとCanvasが無効化される（パフォーマンス最適化）

### 9.3 Quest最適化チェックリスト
- [ ] TextMeshProUGUIを使用（legacy Textは不使用）
- [ ] Canvas Rendererが1つのみ（Draw Call最小化）
- [ ] 差分更新により、変更された行のみ更新される
- [ ] 距離チェックでCanvas有効化制御が機能する
- [ ] Update()内の処理が軽量（GetComponent呼び出しなし）

## 10. テスト観点・テストケース

### 10.1 テスト種別
- **ユニットテスト**: 各メソッドの単体動作確認（手動）
- **統合テスト**: RankingTracker連携動作確認
- **パフォーマンステスト**: Quest 2環境で60fps維持確認

### 10.2 正常系テストケース

#### TC-001: 基本表示確認
**前提条件**:
- RankingTrackerに10人分のランキングデータが存在
- ローカルプレイヤーがTOP10内（6位）

**実行手順**:
1. ClientSimでワールドに入場
2. キャンプ中央のランキングボードに近づく

**期待結果**:
- TOP10が正しく表示される
- 6位の行が黄色背景でハイライトされる
- プレイヤー名の前に[あなた]が付く
- フッターが非表示

**合格基準**: すべての期待結果が満たされる

---

#### TC-002: TOP10圏外表示確認
**前提条件**:
- ローカルプレイヤーが15位（TOP10外）

**実行手順**:
1. ClientSimでワールドに入場
2. ランキングボードを確認

**期待結果**:
- TOP10に自分の名前がない
- フッターに「あなたの順位: 15位」と表示される
- ハイライトされている行がない

**合格基準**: すべての期待結果が満たされる

---

#### TC-003: 自動更新確認
**前提条件**:
- ランキングボード表示中
- RankingTrackerのデータが動的に変化

**実行手順**:
1. ClientSimでワールドに入場
2. 5秒間待機
3. Consoleログを確認

**期待結果**:
- 5秒ごとに「[LeaderboardUI] ランキング更新開始」ログが出力される
- ランキング表示が最新データに更新される

**合格基準**: 5秒間隔で自動更新が実行される

---

#### TC-004: プレイヤー名切り捨て確認
**前提条件**:
- ランキングに21文字以上のプレイヤー名が存在

**実行手順**:
1. RankingTrackerに長い名前（例: "VeryLongPlayerNameTestCase"）を設定
2. ランキング表示を確認

**期待結果**:
- プレイヤー名が17文字+"..."で表示される
- 例: "VeryLongPlayerNa..."

**合格基準**: 20文字以内に切り捨てられる

---

#### TC-005: 色分け確認
**前提条件**:
- TOP10が全員埋まっている

**実行手順**:
1. ランキングボードの各行の色を確認

**期待結果**:
- 1位: ゴールド（#FFD700）+ 王冠アイコン
- 2位: シルバー（#C0C0C0）
- 3位: ブロンズ（#CD7F32）
- 4-7位: ホワイト（#FFFFFF）
- 8-10位: ブロンズ（#CD7F32）

**合格基準**: すべての色が正しく適用される

---

#### TC-006: 距離チェック確認
**前提条件**:
- ランキングボードから20m以上離れた位置

**実行手順**:
1. キャンプから遠い森エリアに移動
2. Hierarchyで `LeaderboardBoard/Canvas` の有効状態を確認

**期待結果**:
- Canvas.enabled が false になる
- Draw Callが減少する

**合格基準**: 20m以上離れるとCanvasが無効化される

### 10.3 異常系テストケース

#### TC-101: データなし時の表示
**前提条件**:
- RankingTrackerのデータが空（プレイヤー0人）

**実行手順**:
1. RankingTrackerのデータをクリア
2. ランキングボード表示を確認

**期待結果**:
- すべての行が空白表示
- エラーログが出ない
- フッターに「順位: --位（伐採を開始してください）」と表示

**合格基準**: クラッシュせず、空白表示される

---

#### TC-102: RankingTracker未設定
**前提条件**:
- Inspector上でRankingTrackerの参照が null

**実行手順**:
1. ClientSimでワールドに入場
2. Consoleログを確認

**期待結果**:
- 初期化時にエラーログ出力: "[LeaderboardUI] RankingTrackerの取得に失敗しました"
- ランキングが表示されない
- クラッシュしない

**合格基準**: エラーログが出力され、クラッシュしない

---

#### TC-103: UI要素配列サイズ不一致
**前提条件**:
- rankTexts配列のサイズが9（10未満）

**実行手順**:
1. ClientSimでワールドに入場

**期待結果**:
- 初期化時にエラーログ: "[LeaderboardUI] rankTexts配列が正しく設定されていません"
- isInitialized が false のまま
- クラッシュしない

**合格基準**: エラーログが出力され、初期化が中断される

### 10.4 パフォーマンステスト

#### TC-201: Quest 2フレームレート確認
**前提条件**:
- Quest 2実機ビルド
- インスタンス内に10人のプレイヤーがいる（シミュレート）

**実行手順**:
1. Quest 2でワールドに入場
2. ランキングボード前に立つ
3. 10秒間FPSを計測（VRChat標準のパフォーマンス表示）

**期待結果**:
- 平均FPS: 60fps以上
- 最低FPS: 55fps以上

**合格基準**: 平均60fps維持

---

#### TC-202: Draw Call確認
**前提条件**:
- Unity Profilerを有効化

**実行手順**:
1. Play Modeでワールドを実行
2. Profilerで Rendering > Draw Calls を確認

**期待結果**:
- LeaderboardUI単体のDraw Call: 1〜2回
- Canvas有効時と無効時で差分が確認できる

**合格基準**: Draw Callが2回以内

## 11. 成果物

### 11.1 変更ファイル
```
Assets/WoodcutterCamp/Scripts/UI/LeaderboardUI.cs  # 新規作成（約400-500行）
```

### 11.2 追加Unityアセット
```
Assets/WoodcutterCamp/Prefabs/UI/RankingRowPrefab.prefab  # 新規作成
```

### 11.3 Sceneファイル
```
Assets/WoodcutterCamp/Scenes/WoodcutterCamp.unity  # 変更（LeaderboardBoard追加）
```

### 11.4 ドキュメント
```
docs/WID/WI-0022_LeaderboardUI.md  # 本ドキュメント
```

## 12. 備考

### 12.1 重要な注意点

#### パフォーマンス最適化
- **差分更新**: `cachedPlayerNames`と`cachedTreesCut`を使い、変更があった行のみ更新する
- **距離チェック**: 20m以上離れた場合はCanvasを無効化し、描画コストを削減
- **GetComponent呼び出し**: Start()時にすべてキャッシュし、Update()では呼ばない
- **TextMeshProUGUI使用**: legacy Textより高速でQuest最適化済み

#### UdonSharp制約
- **配列の扱い**: UdonSharpでは動的配列操作に制限があるため、固定サイズ配列（10個）を使用
- **CustomEventの命名**: すべてのPublicメソッドは`_`プレフィックスを付ける（Udon呼び出し規約）
- **デバッグログ**: 開発時は`Debug.Log`で動作確認し、本番ビルド前に削除またはコメントアウト

#### RankingTrackerとの連携
- RankingTracker（WI-0021）が提供するデータ構造に依存
- RankingTrackerの実装が完了するまで、モックデータでテスト
- 以下の変数名がRankingTrackerに存在する前提:
  ```csharp
  public string[] topPlayerNames;
  public int[] topTreesCut;
  public int[] topTotalDamage;
  public int[] topCoinsEarned;
  public int[] topPlayerIDs;
  public int localPlayerRank;
  ```

### 12.2 トラブルシューティング

#### 問題: ランキングが表示されない
**原因**:
- RankingTrackerが初期化されていない
- UI要素の参照が正しく設定されていない

**対処法**:
1. Consoleで初期化エラーログを確認
2. Inspector上の参照をすべて再設定
3. GameManagerがRankingTrackerを正しく初期化しているか確認

---

#### 問題: プレイヤー名が文字化けする
**原因**:
- TextMeshProのフォントが日本語に対応していない

**対処法**:
1. TextMeshPro Font Asset Creatorで日本語フォント生成
2. すべてのTextMeshProUGUIにフォント適用
3. フォントのFallbackを設定

---

#### 問題: FPSが低下する
**原因**:
- Canvas距離チェックが機能していない
- Update()が重い処理を実行している

**対処法**:
1. Profilerで原因を特定
2. `disableDistance`を調整（デフォルト20m）
3. 差分更新が正しく動作しているか確認

### 12.3 将来の拡張予定
- Phase 2: 週間/月間ランキング対応
- Phase 2: アニメーション追加（順位変動時のハイライト）
- Phase 2: ランキングのソート切り替え（伐採数/コイン/ダメージ）
- Phase 3: グローバルランキング（全インスタンス横断）

### 12.4 参考資料
- VRChat SDK Documentation: https://creators.vrchat.com/
- UdonSharp Documentation: https://udonsharp.docs.vrchat.com/
- TextMeshPro Manual: https://docs.unity3d.com/Packages/com.unity.textmeshpro@3.0/manual/
- TDD.md Section 5.9（M-09 LeaderboardUI詳細）
- FSD.md FNC-006（協力・ソーシャル機能）

---

## 改訂履歴

| バージョン | 日付 | 変更内容 | 作成者 |
|-----------|------|---------|--------|
| 1.0 | 2025-11-17 | 初版作成 | VRChat開発チーム |

---

**承認**
- [ ] システムエンジニアレビュー完了
- [ ] テスト完了確認
- [ ] Quest最適化確認完了
- [ ] 次作業（WI-0023）への引き継ぎ完了
