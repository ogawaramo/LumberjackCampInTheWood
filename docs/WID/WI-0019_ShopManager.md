# WI-0019: ShopManager実装

## 1. 作業ID
WI-0019

## 2. 作業名
ShopManager（ショップ管理システム）の実装

## 3. 作業目的
きこりコインを使用したアイテム購入システムを実装し、プレイヤーに継続プレイの動機付けと視覚的な成長体験を提供する。斧のアップグレード、コスメティックアイテムの購入・装備機能を実現し、PlayerDataへの永続化を行う。

## 4. 対象

* **対象システム**: 森のきこりキャンプ（VRChatワールド）
* **対象モジュール**: M-07 ShopManager（Economy Layer）
* **対象ファイルパス**:
  - `/Assets/WoodcutterCamp/Scripts/Economy/ShopManager.cs`（新規作成）
  - `/Assets/WoodcutterCamp/Scripts/Economy/ShopItemData.cs`（新規作成）
  - `/Assets/WoodcutterCamp/Prefabs/UI/ShopUI.prefab`（新規作成）
  - `/Assets/WoodcutterCamp/Data/ShopItems.asset`（ScriptableObject、新規作成）

## 5. 前提条件

### 環境
- Unity 2022.3.22f1 LTS がインストール済み
- VRChat SDK 3.9.0 がインポート済み
- UdonSharp 1.1.9 がインポート済み
- 作業ブランチ: `feature/wi-0019-shop-manager`

### 依存する他のWork Instruction
- **WI-0001**: GameManager実装済み（モジュール参照取得のため）
- **WI-0003**: PersistenceManager実装済み（PlayerData保存のため）
- **WI-0013**: CoinManager実装済み（コイン残高確認・消費のため）

### 初期状態
- ゲームシーンが開かれている
- GameManager, PersistenceManager, CoinManagerが動作している
- プレイヤーがコインを獲得できる状態

## 6. 入力

### ユーザー操作
- プレイヤーが商人NPCテントに近づき、Eキー（VRはトリガー）を押す
- ショップUI内でアイテムを選択し「購入」ボタンをクリック
- 購入確認ダイアログで「はい」を選択
- 所持中のアイテムを選択し「装備」ボタンをクリック

### データ入力
- `CoinManager.GetCurrentCoins()`: 現在のコイン残高
- `PersistenceManager.LoadInt("UnlockedAxeSkins_X")`: 解禁済みアイテムフラグ
- `PersistenceManager.LoadInt("EquippedAxeSkin")`: 装備中の斧ID
- `SkillManager.GetCurrentLevel()`: 現在のWoodcuttingレベル

## 7. 出力

### 購入成功時
- コイン残高が減算される（例: 245 → 145コイン）
- アイテムIDがPlayerDataの解禁リストに追加される
- 購入完了演出（キラキラエフェクト、効果音）
- 「購入完了！」通知が画面中央上部に3秒間表示
- ショップUIに「所持中」バッジが表示される

### 購入失敗時（エラーケース）
- コイン不足時: 「コイン不足です（50/100）」エラーメッセージ表示
- 解禁条件未達時: 「Woodcutting Lv5に到達すると購入できます」メッセージ表示
- 既に購入済み時: 「既に所持しています」メッセージ表示

### 装備変更時
- 装備中の斧Prefabが新しい斧に差し替えられる
- PlayerData["EquippedAxeSkin"]が更新される
- ショップUIの「装備中」表示が更新される

### データ永続化
```json
{
  "UnlockedAxeSkins_1": 1,  // 鉄の斧解禁
  "UnlockedAxeSkins_2": 1,  // 黒鋼の斧解禁
  "EquippedAxeSkin": 2,     // 黒鋼の斧を装備中
  "UnlockedHats_1": 1,      // 赤い帽子解禁
  "UnlockedHats_3": 1,      // 緑の帽子解禁
  "EquippedHat": 3          // 緑の帽子を装備中
}
```

## 8. 作業手順

### Phase 1: データ構造の設計と実装（1日目）

#### 8.1. ShopItemDataスクリプトの作成

**目的**: アイテムのデータ構造を定義する

**手順**:
1. Unityエディタで `/Assets/WoodcutterCamp/Scripts/Economy/` フォルダを作成
2. 右クリック → Create → C# Script → 名前を `ShopItemData.cs` に変更
3. Visual Studioで開き、以下のコードを記述:

```csharp
using UnityEngine;

namespace WoodcutterCamp.Economy
{
    /// <summary>
    /// ショップアイテムのデータ構造
    /// ScriptableObjectとして保存され、インスペクターで編集可能
    /// </summary>
    [System.Serializable]
    public class ShopItemData
    {
        [Header("アイテム基本情報")]
        [Tooltip("アイテムの一意識別子（1=鉄斧、2=黒鋼斧、3-5=帽子など）")]
        public int itemID;

        [Tooltip("ショップUIに表示されるアイテム名")]
        public string itemName;

        [Tooltip("アイテムの種類（axe, hat, clothingなど）")]
        public string itemType;

        [Header("価格と解禁条件")]
        [Tooltip("購入に必要なコイン数")]
        public int price;

        [Tooltip("購入に必要なWoodcuttingレベル（0=条件なし）")]
        public int requiredLevel;

        [Header("効果と説明")]
        [Tooltip("ショップUIに表示される効果説明")]
        [TextArea(2, 4)]
        public string effectDescription;

        [Tooltip("斧の場合の基礎ダメージ（斧以外は0）")]
        public int baseDamage;

        [Tooltip("斧の場合の攻撃速度（秒）（斧以外は0）")]
        public float attackSpeed;

        [Tooltip("斧の場合のクリティカル率（0.0-1.0）（斧以外は0）")]
        public float criticalRate;

        [Header("ビジュアル")]
        [Tooltip("ショップUIに表示されるアイコン画像")]
        public Sprite iconSprite;

        [Tooltip("装備時に生成されるPrefab（斧の場合は手に持つオブジェクト、帽子の場合は頭に装着）")]
        public GameObject itemPrefab;
    }
}
```

4. Ctrl+S で保存
5. Unityに戻り、コンパイルエラーがないことを確認

**完了確認**: Consoleにエラーが表示されない

#### 8.2. ShopManagerスクリプトの作成

**目的**: ショップの中核となるUdonSharpスクリプトを実装する

**手順**:
1. `/Assets/WoodcutterCamp/Scripts/Economy/` フォルダを右クリック
2. Create → U# Script → 名前を `ShopManager` に変更
3. Visual Studioで開き、以下のコードを記述:

```csharp
using UdonSharp;
using UnityEngine;
using UnityEngine.UI;
using VRC.SDKBase;
using VRC.Udon;
using TMPro;

namespace WoodcutterCamp.Economy
{
    /// <summary>
    /// ショップアイテムの管理と購入処理を担当するマネージャー
    ///
    /// 責務:
    /// - ショップアイテムデータベースの管理
    /// - 購入バリデーション（残高チェック、レベルチェック、重複購入チェック）
    /// - 購入処理の実行（コイン消費、アイテム解禁、PlayerData保存）
    /// - アイテム装備処理（Prefab差し替え）
    /// - ショップUIの更新
    ///
    /// 依存関係:
    /// - WI-0001 GameManager（モジュール参照取得）
    /// - WI-0003 PersistenceManager（PlayerData I/O）
    /// - WI-0013 CoinManager（コイン残高確認・消費）
    /// </summary>
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class ShopManager : UdonSharpBehaviour
    {
        #region インスペクター設定

        [Header("ショップアイテムデータベース")]
        [Tooltip("全ショップアイテムの配列（インスペクターで設定）")]
        [SerializeField] private ShopItemData[] shopItems;

        [Header("依存モジュール参照")]
        [Tooltip("GameManagerへの参照（自動取得）")]
        [SerializeField] private UdonBehaviour gameManager;

        [Tooltip("CoinManagerへの参照（自動取得）")]
        [SerializeField] private UdonBehaviour coinManager;

        [Tooltip("PersistenceManagerへの参照（自動取得）")]
        [SerializeField] private UdonBehaviour persistenceManager;

        [Tooltip("SkillManagerへの参照（自動取得）")]
        [SerializeField] private UdonBehaviour skillManager;

        [Header("ショップUI要素")]
        [Tooltip("ショップUIのルートパネル")]
        [SerializeField] private GameObject shopUIPanel;

        [Tooltip("アイテムグリッドの親Transform")]
        [SerializeField] private Transform itemGridParent;

        [Tooltip("アイテムカードのPrefab（動的生成用）")]
        [SerializeField] private GameObject itemCardPrefab;

        [Tooltip("購入確認ダイアログ")]
        [SerializeField] private GameObject confirmationDialog;

        [Tooltip("確認ダイアログのメッセージテキスト")]
        [SerializeField] private TextMeshProUGUI confirmationMessageText;

        [Tooltip("エラーメッセージ表示テキスト")]
        [SerializeField] private TextMeshProUGUI errorMessageText;

        [Header("装備中のアイテムオブジェクト")]
        [Tooltip("プレイヤーの手に装備される斧の親Transform")]
        [SerializeField] private Transform axeEquipSlot;

        [Tooltip("プレイヤーの頭に装備される帽子の親Transform")]
        [SerializeField] private Transform hatEquipSlot;

        [Header("エフェクト・サウンド")]
        [Tooltip("購入成功時のパーティクルエフェクト")]
        [SerializeField] private ParticleSystem purchaseSuccessEffect;

        [Tooltip("購入成功時の効果音")]
        [SerializeField] private AudioClip purchaseSuccessSound;

        [Tooltip("エラー時の効果音")]
        [SerializeField] private AudioClip errorSound;

        [Tooltip("オーディオソース")]
        [SerializeField] private AudioSource audioSource;

        #endregion

        #region プライベート変数

        // 解禁済みアイテムを管理するbool配列（インデックス=itemID）
        // true = 解禁済み、false = 未解禁
        private bool[] unlockedItems;

        // 現在装備中のアイテムID（アイテムタイプごと）
        private int equippedAxeID = 0;     // 0 = 木の斧（初期装備）
        private int equippedHatID = 0;     // 0 = なし
        private int equippedClothingID = 0; // 0 = なし

        // 現在選択中のアイテム（購入確認用）
        private int selectedItemID = -1;

        // アイテムカードUI要素のキャッシュ
        private GameObject[] itemCards;

        // エラーメッセージ表示タイマー
        private float errorMessageHideTime = 0f;

        // ショップUI開閉状態
        private bool isShopOpen = false;

        // 最大アイテムID（配列サイズ決定用）
        private const int MAX_ITEM_ID = 20;

        // PlayerDataキー定義
        private const string EQUIPPED_AXE_KEY = "EquippedAxeSkin";
        private const string EQUIPPED_HAT_KEY = "EquippedHat";
        private const string UNLOCKED_ITEM_PREFIX = "UnlockedItem_";

        #endregion

        #region Udonライフサイクル

        void Start()
        {
            InitializeShopManager();
        }

        void Update()
        {
            // エラーメッセージの自動非表示処理
            if (errorMessageText != null && errorMessageText.gameObject.activeSelf)
            {
                if (Time.time >= errorMessageHideTime)
                {
                    errorMessageText.gameObject.SetActive(false);
                }
            }
        }

        #endregion

        #region 初期化

        /// <summary>
        /// ShopManagerの初期化処理
        /// </summary>
        private void InitializeShopManager()
        {
            Debug.Log("[ShopManager] 初期化開始");

            // 依存モジュールの取得
            if (gameManager == null)
            {
                GameObject gmObj = GameObject.Find("GameManager");
                if (gmObj != null)
                {
                    gameManager = (UdonBehaviour)gmObj.GetComponent(typeof(UdonBehaviour));
                }
                else
                {
                    Debug.LogError("[ShopManager] GameManagerが見つかりません");
                    return;
                }
            }

            // CoinManagerの取得
            if (coinManager == null)
            {
                coinManager = (UdonBehaviour)gameManager.GetProgramVariable("coinManager");
                if (coinManager == null)
                {
                    Debug.LogError("[ShopManager] CoinManagerが見つかりません");
                    return;
                }
            }

            // PersistenceManagerの取得
            if (persistenceManager == null)
            {
                persistenceManager = (UdonBehaviour)gameManager.GetProgramVariable("persistenceManager");
                if (persistenceManager == null)
                {
                    Debug.LogError("[ShopManager] PersistenceManagerが見つかりません");
                    return;
                }
            }

            // SkillManagerの取得
            if (skillManager == null)
            {
                skillManager = (UdonBehaviour)gameManager.GetProgramVariable("skillManager");
                if (skillManager == null)
                {
                    Debug.LogError("[ShopManager] SkillManagerが見つかりません");
                    return;
                }
            }

            // 解禁状態配列の初期化
            unlockedItems = new bool[MAX_ITEM_ID];

            // PlayerDataから解禁済みアイテムを読み込み
            LoadUnlockedItemsFromPlayerData();

            // PlayerDataから装備中アイテムを読み込み
            LoadEquippedItemsFromPlayerData();

            // ショップUIの初期化
            InitializeShopUI();

            // 初期装備を適用（木の斧など）
            ApplyInitialEquipment();

            // ショップUIを非表示にする
            if (shopUIPanel != null)
            {
                shopUIPanel.SetActive(false);
            }

            Debug.Log("[ShopManager] 初期化完了");
        }

        /// <summary>
        /// PlayerDataから解禁済みアイテム情報を読み込む
        /// </summary>
        private void LoadUnlockedItemsFromPlayerData()
        {
            if (persistenceManager == null) return;

            for (int i = 0; i < shopItems.Length; i++)
            {
                int itemID = shopItems[i].itemID;
                string key = UNLOCKED_ITEM_PREFIX + itemID.ToString();

                int unlocked = (int)persistenceManager.GetProgramVariable("LoadInt");
                persistenceManager.SetProgramVariable("key", key);
                persistenceManager.SendCustomEvent("_LoadInt");
                unlocked = (int)persistenceManager.GetProgramVariable("loadedValue");

                unlockedItems[itemID] = (unlocked == 1);

                if (unlockedItems[itemID])
                {
                    Debug.Log($"[ShopManager] アイテムID {itemID} は解禁済み");
                }
            }

            // 木の斧（ID=0）は常に解禁状態
            unlockedItems[0] = true;
        }

        /// <summary>
        /// PlayerDataから装備中アイテム情報を読み込む
        /// </summary>
        private void LoadEquippedItemsFromPlayerData()
        {
            if (persistenceManager == null) return;

            // 装備中の斧を読み込み
            persistenceManager.SetProgramVariable("key", EQUIPPED_AXE_KEY);
            persistenceManager.SendCustomEvent("_LoadInt");
            equippedAxeID = (int)persistenceManager.GetProgramVariable("loadedValue");

            // 装備中の帽子を読み込み
            persistenceManager.SetProgramVariable("key", EQUIPPED_HAT_KEY);
            persistenceManager.SendCustomEvent("_LoadInt");
            equippedHatID = (int)persistenceManager.GetProgramVariable("loadedValue");

            Debug.Log($"[ShopManager] 装備中: 斧ID={equippedAxeID}, 帽子ID={equippedHatID}");
        }

        /// <summary>
        /// ショップUIの初期化（アイテムカード動的生成）
        /// </summary>
        private void InitializeShopUI()
        {
            if (itemGridParent == null || itemCardPrefab == null)
            {
                Debug.LogWarning("[ShopManager] ショップUI要素が設定されていません");
                return;
            }

            // 既存のアイテムカードを削除
            foreach (Transform child in itemGridParent)
            {
                Destroy(child.gameObject);
            }

            // アイテムカード配列を初期化
            itemCards = new GameObject[shopItems.Length];

            // 各アイテムのカードを生成
            for (int i = 0; i < shopItems.Length; i++)
            {
                GameObject card = Instantiate(itemCardPrefab, itemGridParent);
                itemCards[i] = card;

                // カードの内容を設定
                SetupItemCard(card, shopItems[i], i);
            }

            Debug.Log($"[ShopManager] {shopItems.Length}個のアイテムカードを生成しました");
        }

        /// <summary>
        /// 個別のアイテムカードUIを設定
        /// </summary>
        private void SetupItemCard(GameObject card, ShopItemData itemData, int index)
        {
            // アイテム名
            TextMeshProUGUI nameText = card.transform.Find("ItemName").GetComponent<TextMeshProUGUI>();
            if (nameText != null) nameText.text = itemData.itemName;

            // アイテムアイコン
            Image iconImage = card.transform.Find("ItemIcon").GetComponent<Image>();
            if (iconImage != null && itemData.iconSprite != null)
            {
                iconImage.sprite = itemData.iconSprite;
            }

            // 価格表示
            TextMeshProUGUI priceText = card.transform.Find("PriceText").GetComponent<TextMeshProUGUI>();
            if (priceText != null) priceText.text = $"{itemData.price} コイン";

            // 効果説明
            TextMeshProUGUI descText = card.transform.Find("DescriptionText").GetComponent<TextMeshProUGUI>();
            if (descText != null) descText.text = itemData.effectDescription;

            // 購入ボタン
            Button purchaseButton = card.transform.Find("PurchaseButton").GetComponent<Button>();
            if (purchaseButton != null)
            {
                // UdonSharpではButton.onClickを直接設定できないため、
                // ボタンに「UdonBehaviour」を追加し、OnClickイベントから呼び出す方式を使用
                // 購入ボタンには「itemIndex」をProgramVariableとして保存
                UdonBehaviour buttonUdon = (UdonBehaviour)purchaseButton.GetComponent(typeof(UdonBehaviour));
                if (buttonUdon != null)
                {
                    buttonUdon.SetProgramVariable("shopManager", this);
                    buttonUdon.SetProgramVariable("itemIndex", index);
                }
            }

            // 装備ボタン（所持中の場合のみ表示）
            GameObject equipButton = card.transform.Find("EquipButton").gameObject;
            if (equipButton != null)
            {
                equipButton.SetActive(false); // 初期は非表示
            }

            // 所持中バッジ
            GameObject ownedBadge = card.transform.Find("OwnedBadge").gameObject;
            if (ownedBadge != null)
            {
                ownedBadge.SetActive(false); // 初期は非表示
            }

            // レベル条件表示
            GameObject levelRequirement = card.transform.Find("LevelRequirement").gameObject;
            if (levelRequirement != null)
            {
                if (itemData.requiredLevel > 0)
                {
                    TextMeshProUGUI reqText = levelRequirement.GetComponent<TextMeshProUGUI>();
                    if (reqText != null) reqText.text = $"Lv{itemData.requiredLevel}必要";
                    levelRequirement.SetActive(true);
                }
                else
                {
                    levelRequirement.SetActive(false);
                }
            }
        }

        /// <summary>
        /// 初期装備を適用（木の斧など）
        /// </summary>
        private void ApplyInitialEquipment()
        {
            // 装備中の斧を装備
            if (equippedAxeID >= 0 && equippedAxeID < shopItems.Length)
            {
                EquipAxe(equippedAxeID);
            }

            // 装備中の帽子を装備
            if (equippedHatID > 0 && equippedHatID < shopItems.Length)
            {
                EquipHat(equippedHatID);
            }
        }

        #endregion

        #region ショップUI制御

        /// <summary>
        /// ショップUIを開く（プレイヤーがEキーを押した時に呼ばれる）
        /// </summary>
        public void _OpenShop()
        {
            if (shopUIPanel == null) return;

            shopUIPanel.SetActive(true);
            isShopOpen = true;

            // UI内容を最新状態に更新
            RefreshShopUI();

            Debug.Log("[ShopManager] ショップUIを開きました");
        }

        /// <summary>
        /// ショップUIを閉じる
        /// </summary>
        public void _CloseShop()
        {
            if (shopUIPanel == null) return;

            shopUIPanel.SetActive(false);
            isShopOpen = false;

            Debug.Log("[ShopManager] ショップUIを閉じました");
        }

        /// <summary>
        /// ショップUIの表示内容を更新
        /// </summary>
        private void RefreshShopUI()
        {
            if (itemCards == null) return;

            // 現在のコイン残高を取得
            int currentCoins = GetCurrentCoins();

            // 現在のスキルレベルを取得
            int currentLevel = GetCurrentSkillLevel();

            for (int i = 0; i < itemCards.Length; i++)
            {
                if (itemCards[i] == null) continue;

                ShopItemData itemData = shopItems[i];
                int itemID = itemData.itemID;

                // 所持状態に応じてUIを更新
                bool isOwned = unlockedItems[itemID];
                bool isEquipped = IsItemEquipped(itemID, itemData.itemType);
                bool canAfford = currentCoins >= itemData.price;
                bool meetsLevelRequirement = currentLevel >= itemData.requiredLevel;

                // 購入ボタンの表示制御
                GameObject purchaseButton = itemCards[i].transform.Find("PurchaseButton").gameObject;
                if (purchaseButton != null)
                {
                    // 所持していない かつ 購入可能条件を満たしている場合のみ表示
                    purchaseButton.SetActive(!isOwned && meetsLevelRequirement);

                    // コイン不足の場合はボタンを無効化
                    Button btn = purchaseButton.GetComponent<Button>();
                    if (btn != null) btn.interactable = canAfford;
                }

                // 装備ボタンの表示制御
                GameObject equipButton = itemCards[i].transform.Find("EquipButton").gameObject;
                if (equipButton != null)
                {
                    // 所持している かつ 装備中でない場合のみ表示
                    equipButton.SetActive(isOwned && !isEquipped);
                }

                // 所持中バッジ
                GameObject ownedBadge = itemCards[i].transform.Find("OwnedBadge").gameObject;
                if (ownedBadge != null)
                {
                    ownedBadge.SetActive(isOwned);
                }

                // 装備中表示
                GameObject equippedLabel = itemCards[i].transform.Find("EquippedLabel").gameObject;
                if (equippedLabel != null)
                {
                    equippedLabel.SetActive(isEquipped);
                }

                // グレーアウト処理（レベル不足）
                Image cardBg = itemCards[i].GetComponent<Image>();
                if (cardBg != null)
                {
                    if (!meetsLevelRequirement)
                    {
                        cardBg.color = new Color(0.5f, 0.5f, 0.5f, 1f); // グレー
                    }
                    else
                    {
                        cardBg.color = Color.white; // 通常
                    }
                }
            }
        }

        /// <summary>
        /// アイテムが現在装備中かどうかを判定
        /// </summary>
        private bool IsItemEquipped(int itemID, string itemType)
        {
            if (itemType == "axe")
            {
                return equippedAxeID == itemID;
            }
            else if (itemType == "hat")
            {
                return equippedHatID == itemID;
            }
            else if (itemType == "clothing")
            {
                return equippedClothingID == itemID;
            }
            return false;
        }

        #endregion

        #region 購入処理

        /// <summary>
        /// アイテム購入ボタンがクリックされた時の処理
        /// （アイテムカードの購入ボタンから呼ばれる）
        /// </summary>
        public void _OnPurchaseButtonClick()
        {
            // 呼び出し元のUdonBehaviourからitemIndexを取得
            // （Unity Eventから渡される想定）
            UdonBehaviour caller = (UdonBehaviour)GetProgramVariable("caller");
            if (caller == null)
            {
                Debug.LogError("[ShopManager] 購入ボタンの呼び出し元が不明です");
                return;
            }

            int itemIndex = (int)caller.GetProgramVariable("itemIndex");
            if (itemIndex < 0 || itemIndex >= shopItems.Length)
            {
                Debug.LogError($"[ShopManager] 無効なアイテムインデックス: {itemIndex}");
                return;
            }

            selectedItemID = shopItems[itemIndex].itemID;

            // 購入確認ダイアログを表示
            ShowPurchaseConfirmationDialog(shopItems[itemIndex]);
        }

        /// <summary>
        /// 購入確認ダイアログを表示
        /// </summary>
        private void ShowPurchaseConfirmationDialog(ShopItemData itemData)
        {
            if (confirmationDialog == null || confirmationMessageText == null) return;

            string message = $"{itemData.itemName}を{itemData.price}コインで購入しますか?";
            confirmationMessageText.text = message;

            confirmationDialog.SetActive(true);

            Debug.Log($"[ShopManager] 購入確認ダイアログ表示: {message}");
        }

        /// <summary>
        /// 購入確認ダイアログの「はい」ボタン処理
        /// </summary>
        public void _OnConfirmPurchase()
        {
            if (selectedItemID < 0)
            {
                Debug.LogError("[ShopManager] 選択中のアイテムがありません");
                return;
            }

            // アイテムデータを取得
            ShopItemData itemData = GetItemDataByID(selectedItemID);
            if (itemData == null)
            {
                Debug.LogError($"[ShopManager] アイテムID {selectedItemID} が見つかりません");
                return;
            }

            // 購入バリデーション
            string errorMessage = ValidatePurchase(itemData);
            if (errorMessage != null)
            {
                ShowErrorMessage(errorMessage);
                PlayErrorSound();
                CloseConfirmationDialog();
                return;
            }

            // 購入処理実行
            ExecutePurchase(itemData);

            // 確認ダイアログを閉じる
            CloseConfirmationDialog();
        }

        /// <summary>
        /// 購入確認ダイアログの「いいえ」ボタン処理
        /// </summary>
        public void _OnCancelPurchase()
        {
            CloseConfirmationDialog();
            selectedItemID = -1;
        }

        /// <summary>
        /// 確認ダイアログを閉じる
        /// </summary>
        private void CloseConfirmationDialog()
        {
            if (confirmationDialog != null)
            {
                confirmationDialog.SetActive(false);
            }
        }

        /// <summary>
        /// 購入バリデーション
        /// </summary>
        /// <returns>エラーメッセージ（null=バリデーション成功）</returns>
        private string ValidatePurchase(ShopItemData itemData)
        {
            int itemID = itemData.itemID;

            // 1. 既に購入済みチェック
            if (unlockedItems[itemID])
            {
                return "既に所持しています";
            }

            // 2. コイン残高チェック
            int currentCoins = GetCurrentCoins();
            if (currentCoins < itemData.price)
            {
                return $"コイン不足です（{currentCoins}/{itemData.price}）";
            }

            // 3. レベル条件チェック
            int currentLevel = GetCurrentSkillLevel();
            if (currentLevel < itemData.requiredLevel)
            {
                return $"Woodcutting Lv{itemData.requiredLevel}に到達すると購入できます";
            }

            // バリデーション成功
            return null;
        }

        /// <summary>
        /// 購入処理の実行
        /// </summary>
        private void ExecutePurchase(ShopItemData itemData)
        {
            int itemID = itemData.itemID;

            Debug.Log($"[ShopManager] アイテム購入開始: {itemData.itemName} (ID={itemID})");

            // 1. コインを消費
            if (coinManager != null)
            {
                coinManager.SetProgramVariable("amount", -itemData.price); // 負の値で減算
                coinManager.SendCustomEvent("_AddCoins");
            }

            // 2. アイテムを解禁
            unlockedItems[itemID] = true;

            // 3. PlayerDataに保存
            SaveUnlockedItem(itemID);

            // 4. 自動装備（斧の場合）
            if (itemData.itemType == "axe")
            {
                EquipAxe(itemID);
                SaveEquippedAxe(itemID);
            }

            // 5. 購入成功演出
            PlayPurchaseSuccessEffect();
            ShowSuccessMessage($"{itemData.itemName}を購入しました！");

            // 6. ショップUIを更新
            RefreshShopUI();

            Debug.Log($"[ShopManager] アイテム購入完了: {itemData.itemName}");
        }

        /// <summary>
        /// 解禁済みアイテムをPlayerDataに保存
        /// </summary>
        private void SaveUnlockedItem(int itemID)
        {
            if (persistenceManager == null) return;

            string key = UNLOCKED_ITEM_PREFIX + itemID.ToString();
            persistenceManager.SetProgramVariable("key", key);
            persistenceManager.SetProgramVariable("value", 1); // 1 = 解禁済み
            persistenceManager.SendCustomEvent("_SaveInt");

            Debug.Log($"[ShopManager] PlayerDataに保存: {key} = 1");
        }

        #endregion

        #region 装備処理

        /// <summary>
        /// 装備ボタンがクリックされた時の処理
        /// </summary>
        public void _OnEquipButtonClick()
        {
            // 呼び出し元のUdonBehaviourからitemIndexを取得
            UdonBehaviour caller = (UdonBehaviour)GetProgramVariable("caller");
            if (caller == null) return;

            int itemIndex = (int)caller.GetProgramVariable("itemIndex");
            if (itemIndex < 0 || itemIndex >= shopItems.Length) return;

            ShopItemData itemData = shopItems[itemIndex];

            // アイテムタイプに応じて装備処理
            if (itemData.itemType == "axe")
            {
                EquipAxe(itemData.itemID);
                SaveEquippedAxe(itemData.itemID);
            }
            else if (itemData.itemType == "hat")
            {
                EquipHat(itemData.itemID);
                SaveEquippedHat(itemData.itemID);
            }

            // UIを更新
            RefreshShopUI();
        }

        /// <summary>
        /// 斧を装備する
        /// </summary>
        private void EquipAxe(int axeID)
        {
            if (axeEquipSlot == null)
            {
                Debug.LogWarning("[ShopManager] 斧装備スロットが設定されていません");
                return;
            }

            // 現在装備中の斧を削除
            foreach (Transform child in axeEquipSlot)
            {
                Destroy(child.gameObject);
            }

            // 新しい斧Prefabを生成
            ShopItemData axeData = GetItemDataByID(axeID);
            if (axeData != null && axeData.itemPrefab != null)
            {
                GameObject newAxe = Instantiate(axeData.itemPrefab, axeEquipSlot);
                newAxe.transform.localPosition = Vector3.zero;
                newAxe.transform.localRotation = Quaternion.identity;

                Debug.Log($"[ShopManager] 斧を装備: {axeData.itemName} (ID={axeID})");
            }

            equippedAxeID = axeID;
        }

        /// <summary>
        /// 帽子を装備する
        /// </summary>
        private void EquipHat(int hatID)
        {
            if (hatEquipSlot == null)
            {
                Debug.LogWarning("[ShopManager] 帽子装備スロットが設定されていません");
                return;
            }

            // 現在装備中の帽子を削除
            foreach (Transform child in hatEquipSlot)
            {
                Destroy(child.gameObject);
            }

            // 新しい帽子Prefabを生成
            ShopItemData hatData = GetItemDataByID(hatID);
            if (hatData != null && hatData.itemPrefab != null)
            {
                GameObject newHat = Instantiate(hatData.itemPrefab, hatEquipSlot);
                newHat.transform.localPosition = Vector3.zero;
                newHat.transform.localRotation = Quaternion.identity;

                Debug.Log($"[ShopManager] 帽子を装備: {hatData.itemName} (ID={hatID})");
            }

            equippedHatID = hatID;
        }

        /// <summary>
        /// 装備中の斧IDをPlayerDataに保存
        /// </summary>
        private void SaveEquippedAxe(int axeID)
        {
            if (persistenceManager == null) return;

            persistenceManager.SetProgramVariable("key", EQUIPPED_AXE_KEY);
            persistenceManager.SetProgramVariable("value", axeID);
            persistenceManager.SendCustomEvent("_SaveInt");

            Debug.Log($"[ShopManager] 装備中の斧をPlayerDataに保存: {axeID}");
        }

        /// <summary>
        /// 装備中の帽子IDをPlayerDataに保存
        /// </summary>
        private void SaveEquippedHat(int hatID)
        {
            if (persistenceManager == null) return;

            persistenceManager.SetProgramVariable("key", EQUIPPED_HAT_KEY);
            persistenceManager.SetProgramVariable("value", hatID);
            persistenceManager.SendCustomEvent("_SaveInt");

            Debug.Log($"[ShopManager] 装備中の帽子をPlayerDataに保存: {hatID}");
        }

        /// <summary>
        /// 現在装備中の斧のダメージを取得（他のスクリプトから参照）
        /// </summary>
        public int _GetEquippedAxeDamage()
        {
            ShopItemData axeData = GetItemDataByID(equippedAxeID);
            return (axeData != null) ? axeData.baseDamage : 10; // デフォルト10
        }

        /// <summary>
        /// 現在装備中の斧のクリティカル率を取得（他のスクリプトから参照）
        /// </summary>
        public float _GetEquippedAxeCriticalRate()
        {
            ShopItemData axeData = GetItemDataByID(equippedAxeID);
            return (axeData != null) ? axeData.criticalRate : 0f;
        }

        #endregion

        #region ユーティリティメソッド

        /// <summary>
        /// アイテムIDからShopItemDataを取得
        /// </summary>
        private ShopItemData GetItemDataByID(int itemID)
        {
            for (int i = 0; i < shopItems.Length; i++)
            {
                if (shopItems[i].itemID == itemID)
                {
                    return shopItems[i];
                }
            }
            return null;
        }

        /// <summary>
        /// 現在のコイン残高を取得
        /// </summary>
        private int GetCurrentCoins()
        {
            if (coinManager == null) return 0;
            return (int)coinManager.GetProgramVariable("currentCoins");
        }

        /// <summary>
        /// 現在のスキルレベルを取得
        /// </summary>
        private int GetCurrentSkillLevel()
        {
            if (skillManager == null) return 1;
            return (int)skillManager.GetProgramVariable("currentLevel");
        }

        /// <summary>
        /// エラーメッセージを表示
        /// </summary>
        private void ShowErrorMessage(string message)
        {
            if (errorMessageText == null) return;

            errorMessageText.text = message;
            errorMessageText.gameObject.SetActive(true);
            errorMessageHideTime = Time.time + 3.0f; // 3秒後に自動非表示

            Debug.LogWarning($"[ShopManager] エラー: {message}");
        }

        /// <summary>
        /// 成功メッセージを表示
        /// </summary>
        private void ShowSuccessMessage(string message)
        {
            // HUDManagerの通知機能を使用（WI-0017で実装済みと仮定）
            UdonBehaviour hudManager = (UdonBehaviour)gameManager.GetProgramVariable("hudManager");
            if (hudManager != null)
            {
                hudManager.SetProgramVariable("message", message);
                hudManager.SendCustomEvent("_ShowNotification");
            }

            Debug.Log($"[ShopManager] 成功: {message}");
        }

        /// <summary>
        /// 購入成功エフェクトを再生
        /// </summary>
        private void PlayPurchaseSuccessEffect()
        {
            // パーティクル再生
            if (purchaseSuccessEffect != null)
            {
                purchaseSuccessEffect.Play();
            }

            // 効果音再生
            if (audioSource != null && purchaseSuccessSound != null)
            {
                audioSource.PlayOneShot(purchaseSuccessSound);
            }
        }

        /// <summary>
        /// エラー音を再生
        /// </summary>
        private void PlayErrorSound()
        {
            if (audioSource != null && errorSound != null)
            {
                audioSource.PlayOneShot(errorSound);
            }
        }

        #endregion
    }
}
```

4. Ctrl+S で保存
5. Unityに戻り、コンパイルエラーがないことを確認

**完了確認**: Consoleに「[ShopManager] 初期化開始」のログが表示されない（まだシーンに配置していないため）

---

### Phase 2: Unity Editor上でのセットアップ（2日目）

#### 8.3. ショップアイテムデータの作成

**目的**: インスペクターで編集可能なアイテムデータベースを作成する

**手順**:
1. `/Assets/WoodcutterCamp/Data/` フォルダを作成
2. Hierarchyで GameManager オブジェクトを選択
3. Add Component → Udon Behaviour → UdonSharp Behaviour → ShopManager を追加
4. ShopManagerコンポーネントのインスペクターを開く
5. `Shop Items` 配列のサイズを `7` に設定（Phase 1の全アイテム数）
6. 各要素に以下のデータを入力:

**Element 0（木の斧 - 初期装備）**:
- Item ID: `0`
- Item Name: `木の斧`
- Item Type: `axe`
- Price: `0`
- Required Level: `0`
- Effect Description: `基本的な斧。初期装備として使える。`
- Base Damage: `10`
- Attack Speed: `1.0`
- Critical Rate: `0.0`
- Icon Sprite: （アイコン画像をドラッグ&ドロップ）
- Item Prefab: （木の斧Prefabをドラッグ&ドロップ）

**Element 1（鉄の斧）**:
- Item ID: `1`
- Item Name: `鉄の斧`
- Item Type: `axe`
- Price: `100`
- Required Level: `0`
- Effect Description: `鉄製の頑丈な斧。ダメージ+50%`
- Base Damage: `15`
- Attack Speed: `0.9`
- Critical Rate: `0.0`
- Icon Sprite: （アイコン画像）
- Item Prefab: （鉄の斧Prefab）

**Element 2（黒鋼の斧）**:
- Item ID: `2`
- Item Name: `黒鋼の斧`
- Item Type: `axe`
- Price: `300`
- Required Level: `5`
- Effect Description: `最高級の斧。ダメージ+100%、クリティカル率10%`
- Base Damage: `20`
- Attack Speed: `0.8`
- Critical Rate: `0.1`
- Icon Sprite: （アイコン画像）
- Item Prefab: （黒鋼の斧Prefab）

**Element 3（作業帽・赤）**:
- Item ID: `3`
- Item Name: `作業帽（赤）`
- Item Type: `hat`
- Price: `50`
- Required Level: `0`
- Effect Description: `赤い作業帽。装飾アイテム。`
- Base Damage: `0`
- Attack Speed: `0`
- Critical Rate: `0.0`
- Icon Sprite: （アイコン画像）
- Item Prefab: （赤い帽子Prefab）

**Element 4（作業帽・青）**:
- Item ID: `4`
- Item Name: `作業帽（青）`
- Item Type: `hat`
- Price: `50`
- Required Level: `0`
- Effect Description: `青い作業帽。装飾アイテム。`
- Base Damage: `0`
- Attack Speed: `0`
- Critical Rate: `0.0`
- Icon Sprite: （アイコン画像）
- Item Prefab: （青い帽子Prefab）

**Element 5（作業帽・緑）**:
- Item ID: `5`
- Item Name: `作業帽（緑）`
- Item Type: `hat`
- Price: `50`
- Required Level: `0`
- Effect Description: `緑の作業帽。装飾アイテム。`
- Base Damage: `0`
- Attack Speed: `0`
- Critical Rate: `0.0`
- Icon Sprite: （アイコン画像）
- Item Prefab: （緑の帽子Prefab）

**Element 6（ゴールデン斧）**:
- Item ID: `6`
- Item Name: `ゴールデン斧`
- Item Type: `axe`
- Price: `1000`
- Required Level: `10`
- Effect Description: `最高の栄誉の証。光り輝く黄金の斧。`
- Base Damage: `20`
- Attack Speed: `0.8`
- Critical Rate: `0.15`
- Icon Sprite: （アイコン画像）
- Item Prefab: （ゴールデン斧Prefab、EmissiveマテリアルでCOLOR追加）

7. Ctrl+S でシーンを保存

**完了確認**: インスペクターで全7アイテムのデータが正しく表示される

#### 8.4. ショップUIの構築

**目的**: Canvas上にショップUIを構築する

**手順**:
1. Hierarchy右クリック → UI → Canvas
2. Canvas名を `ShopUICanvas` に変更
3. Canvas Scaler を以下に設定:
   - UI Scale Mode: `Scale With Screen Size`
   - Reference Resolution: `1920 x 1080`
4. ShopUICanvas の子として Panel を作成、名前を `ShopPanel` に変更
5. ShopPanel の RectTransform を以下に設定:
   - Anchor: `Center`
   - Width: `1200`
   - Height: `800`
   - Pos X/Y/Z: `0, 0, 0`
6. ShopPanel の Image コンポーネント:
   - Color: `白 (255, 255, 255, 220)` - 半透明
7. ShopPanel の子として以下を作成:

**タイトルテキスト**:
- 右クリック → UI → TextMeshPro - Text
- 名前: `ShopTitle`
- Text: `きこりショップ`
- Font Size: `48`
- Alignment: `Center`
- RectTransform: Top-Center anchor, Width=1200, Height=80, PosY=-40

**閉じるボタン**:
- 右クリック → UI → Button - TextMeshPro
- 名前: `CloseButton`
- ボタンテキスト: `×`
- Font Size: `36`
- RectTransform: Top-Right anchor, Width=60, Height=60, PosX=-30, PosY=-30
- OnClick(): `ShopManager._CloseShop`

**アイテムグリッド**:
- 右クリック → UI → Scroll View
- 名前: `ItemScrollView`
- RectTransform: Width=1100, Height=600, PosY=-80
- Content の Grid Layout Group コンポーネントを追加:
  - Cell Size: `330 x 180`
  - Spacing: `20 x 20`
  - Constraint: `Fixed Column Count = 3`

**エラーメッセージ表示**:
- ShopPanel の子として UI → TextMeshPro - Text
- 名前: `ErrorMessageText`
- Text: `エラーメッセージ`
- Font Size: `28`
- Color: `赤 (255, 0, 0, 255)`
- Alignment: `Center`
- RectTransform: Bottom-Center anchor, Width=1100, Height=60, PosY=30
- 初期状態で `SetActive(false)`

8. Ctrl+S でシーンを保存

**完了確認**: Gameビューでショップパネルが中央に表示される

#### 8.5. アイテムカードPrefabの作成

**目的**: アイテム1つ分のUIカードテンプレートを作成する

**手順**:
1. Hierarchy右クリック → UI → Panel
2. 名前を `ItemCard` に変更
3. RectTransform: Width=330, Height=180
4. Image色: `白 (255, 255, 255, 255)`、Add Component → Outline で枠線追加
5. ItemCard の子として以下を作成:

**アイテムアイコン**:
- UI → Image
- 名前: `ItemIcon`
- RectTransform: Left-Center anchor, Width=120, Height=120, PosX=70, PosY=0

**アイテム名**:
- UI → TextMeshPro - Text
- 名前: `ItemName`
- Text: `アイテム名`
- Font Size: `24`
- RectTransform: Top-Right anchor, Width=180, Height=30, PosX=-90, PosY=-10

**価格表示**:
- UI → TextMeshPro - Text
- 名前: `PriceText`
- Text: `100 コイン`
- Font Size: `20`
- Color: `金色 (255, 215, 0, 255)`
- RectTransform: Top-Right anchor, Width=180, Height=25, PosY=-45

**説明文**:
- UI → TextMeshPro - Text
- 名前: `DescriptionText`
- Text: `アイテムの説明文`
- Font Size: `16`
- RectTransform: Bottom-Right anchor, Width=180, Height=60, PosY=50

**購入ボタン**:
- UI → Button - TextMeshPro
- 名前: `PurchaseButton`
- ボタンテキスト: `購入`
- RectTransform: Bottom-Right anchor, Width=160, Height=40, PosX=-90, PosY=15
- OnClick(): `ShopManager._OnPurchaseButtonClick`

**装備ボタン**:
- UI → Button - TextMeshPro
- 名前: `EquipButton`
- ボタンテキスト: `装備`
- RectTransform: Bottom-Right anchor, Width=160, Height=40, PosY=15
- 初期状態で `SetActive(false)`
- OnClick(): `ShopManager._OnEquipButtonClick`

**所持中バッジ**:
- UI → Image
- 名前: `OwnedBadge`
- Sprite: チェックマークアイコン
- Color: `緑 (0, 255, 0, 255)`
- RectTransform: Top-Right corner, Width=40, Height=40, PosX=-10, PosY=-10
- 初期状態で `SetActive(false)`

**装備中ラベル**:
- UI → TextMeshPro - Text
- 名前: `EquippedLabel`
- Text: `装備中`
- Font Size: `18`
- Color: `青 (0, 150, 255, 255)`
- RectTransform: Bottom-Left anchor, Width=80, Height=30, PosX=70, PosY=10
- 初期状態で `SetActive(false)`

**レベル条件表示**:
- UI → TextMeshPro - Text
- 名前: `LevelRequirement`
- Text: `Lv5必要`
- Font Size: `16`
- Color: `オレンジ (255, 150, 0, 255)`
- RectTransform: Top-Left anchor, Width=100, Height=25, PosX=70, PosY=-10
- 初期状態で `SetActive(false)`

6. ItemCard をプロジェクトビューの `/Assets/WoodcutterCamp/Prefabs/UI/` にドラッグしてPrefab化
7. Hierarchy上の ItemCard オブジェクトは削除
8. Ctrl+S で保存

**完了確認**: Prefabフォルダに ItemCard.prefab が作成される

#### 8.6. 購入確認ダイアログの作成

**目的**: 購入前の確認UIを作成する

**手順**:
1. ShopPanel の子として UI → Panel を作成
2. 名前を `ConfirmationDialog` に変更
3. RectTransform: Anchor=Center, Width=500, Height=250
4. Image色: `濃いグレー (50, 50, 50, 240)`
5. ConfirmationDialog の子として以下を作成:

**確認メッセージ**:
- UI → TextMeshPro - Text
- 名前: `ConfirmationMessageText`
- Text: `鉄の斧を100コインで購入しますか？`
- Font Size: `24`
- Alignment: `Center`
- RectTransform: Top-Center anchor, Width=460, Height=100, PosY=-70

**「はい」ボタン**:
- UI → Button - TextMeshPro
- 名前: `ConfirmYesButton`
- ボタンテキスト: `はい`
- RectTransform: Bottom-Left anchor, Width=200, Height=50, PosX=120, PosY=30
- OnClick(): `ShopManager._OnConfirmPurchase`

**「いいえ」ボタン**:
- UI → Button - TextMeshPro
- 名前: `ConfirmNoButton`
- ボタンテキスト: `いいえ`
- RectTransform: Bottom-Right anchor, Width=200, Height=50, PosX=-120, PosY=30
- OnClick(): `ShopManager._OnCancelPurchase`

6. ConfirmationDialog を初期状態で `SetActive(false)`
7. Ctrl+S で保存

**完了確認**: シーン再生時にダイアログが非表示になっている

---

### Phase 3: GameManagerへの統合とテスト（3日目）

#### 8.7. GameManagerへの登録

**目的**: ShopManagerをシステムに組み込む

**手順**:
1. Hierarchy で GameManager オブジェクトを選択
2. GameManager の UdonBehaviour スクリプトを開く（WI-0001で作成済み）
3. パブリック変数に以下を追加:
```csharp
[Header("Economy Managers")]
public UdonBehaviour shopManager;
```
4. `Start()` メソッド内で初期化順序を追加:
```csharp
void Start()
{
    // ... 既存の初期化 ...

    // ShopManagerの初期化（CoinManager, PersistenceManager, SkillManagerの後）
    if (shopManager != null)
    {
        shopManager.SendCustomEvent("_Initialize");
    }
}
```
5. Ctrl+S で保存
6. Unityエディタで GameManager を選択
7. インスペクターで Shop Manager フィールドに ShopManager オブジェクトをドラッグ&ドロップ
8. Ctrl+S でシーン保存

**完了確認**: GameManager.shopManager に ShopManager が参照されている

#### 8.8. 商人NPCトリガーの作成

**目的**: プレイヤーが近づいてEキーを押すとショップが開く仕組みを実装

**手順**:
1. Hierarchy右クリック → 3D Object → Cube
2. 名前を `MerchantNPCTrigger` に変更
3. Transform: Position を商人NPCテントの位置に配置（例: X=0, Y=0, Z=5）
4. Box Collider を設定:
   - Is Trigger: `チェック`
   - Size: `3, 2, 3`（プレイヤーが近づける範囲）
5. Add Component → Udon Behaviour → 新規 U# Script 作成
6. スクリプト名: `ShopTrigger.cs`
7. 以下のコードを記述:

```csharp
using UdonSharp;
using UnityEngine;
using VRC.SDKBase;

namespace WoodcutterCamp.Triggers
{
    /// <summary>
    /// ショップのトリガーゾーン
    /// プレイヤーが範囲内でEキーを押すとショップUIを開く
    /// </summary>
    [UdonBehaviourSyncMode(BehaviourSyncMode.None)]
    public class ShopTrigger : UdonSharpBehaviour
    {
        [SerializeField] private UdonBehaviour shopManager;
        [SerializeField] private GameObject promptUI; // "ショップを開く [E]" 表示

        private bool playerInRange = false;

        void Start()
        {
            if (promptUI != null)
            {
                promptUI.SetActive(false);
            }
        }

        void Update()
        {
            // プレイヤーが範囲内でEキーを押した場合
            if (playerInRange && Input.GetKeyDown(KeyCode.E))
            {
                if (shopManager != null)
                {
                    shopManager.SendCustomEvent("_OpenShop");
                }
            }
        }

        public override void OnPlayerTriggerEnter(VRCPlayerApi player)
        {
            if (player.isLocal)
            {
                playerInRange = true;
                if (promptUI != null)
                {
                    promptUI.SetActive(true);
                }
            }
        }

        public override void OnPlayerTriggerExit(VRCPlayerApi player)
        {
            if (player.isLocal)
            {
                playerInRange = false;
                if (promptUI != null)
                {
                    promptUI.SetActive(false);
                }
            }
        }
    }
}
```

8. Ctrl+S で保存
9. MerchantNPCTrigger のインスペクターで:
   - Shop Manager に ShopManager をドラッグ
   - Prompt UI に プロンプトテキストUIをドラッグ
10. Ctrl+S でシーン保存

**完了確認**: Unity Play Mode で商人NPCトリガーに近づくと「ショップを開く [E]」が表示される

#### 8.9. 装備スロットの設定

**目的**: プレイヤーキャラクターに斧と帽子の装備位置を設定

**手順**:
1. Hierarchy で PlayerLocal オブジェクトを探す（VRChat SDK標準）
2. PlayerLocal の子階層で `RightHand` ボーンを探す
3. RightHand の子として空オブジェクトを作成、名前を `AxeEquipSlot` に変更
4. AxeEquipSlot の Transform:
   - Position: `0, 0, 0.1`（手のひらの少し前）
   - Rotation: `0, 0, 0`
5. PlayerLocal の子階層で `Head` ボーンを探す
6. Head の子として空オブジェクトを作成、名前を `HatEquipSlot` に変更
7. HatEquipSlot の Transform:
   - Position: `0, 0.15, 0`（頭の少し上）
   - Rotation: `0, 0, 0`
8. ShopManager オブジェクトを選択
9. インスペクターで:
   - Axe Equip Slot に AxeEquipSlot をドラッグ
   - Hat Equip Slot に HatEquipSlot をドラッグ
10. Ctrl+S でシーン保存

**完了確認**: インスペクターで両スロットが正しく参照されている

---

### Phase 4: テストとデバッグ（4日目）

#### 8.10. 単体テスト実施

**目的**: 各機能が正しく動作するか確認する

**テストケース一覧**:

| # | テスト項目 | 手順 | 期待結果 |
|---|-----------|------|---------|
| T-01 | ショップUI表示 | 商人NPCに近づきEキー押下 | ショップUIが開く |
| T-02 | アイテムカード表示 | ショップUI内を確認 | 7個のアイテムが3列グリッドで表示 |
| T-03 | コイン不足エラー | コイン0で鉄の斧を購入 | 「コイン不足です（0/100）」エラー表示 |
| T-04 | レベル不足エラー | Lv1で黒鋼の斧を購入 | 「Woodcutting Lv5に到達すると購入できます」表示 |
| T-05 | 購入成功 | コイン150で鉄の斧を購入 | コイン50に減少、鉄の斧解禁、自動装備 |
| T-06 | 購入確認ダイアログ | アイテム購入ボタンをクリック | 「XXを購入しますか？」ダイアログ表示 |
| T-07 | 購入キャンセル | ダイアログで「いいえ」選択 | ダイアログ閉じる、購入されない |
| T-08 | 装備変更 | 木の斧所持中に鉄の斧を装備 | 手に持つ斧が鉄の斧に変わる |
| T-09 | PlayerData保存 | アイテム購入後ワールド退出 | 次回入場時も所持状態維持 |
| T-10 | 重複購入防止 | 既に購入済みの鉄の斧を再購入 | 「既に所持しています」エラー表示 |

**実施手順**:
1. Unity Play Mode を開始
2. ClientSim で `Play Locally` をクリック
3. 上記テストケースを順番に実行
4. 各テストの結果を記録
5. エラーが発生した場合はConsoleログを確認

**完了確認**: 全テストケースが期待結果通りに動作する

#### 8.11. Quest最適化確認

**目的**: Quest環境でのパフォーマンスを確認

**手順**:
1. Unity上部メニュー → VRChat SDK → Show Control Panel
2. Builder タブ → Platform: `Android` を選択
3. Build & Test ボタンをクリック（ローカルテストビルド）
4. Quest 2実機またはAndroidエミュレータで起動
5. ショップUI開閉時のFPSを確認（Stats表示）
   - 目標: ショップ開閉前後でFPS低下5%以内
6. アイテムカード7個表示時のDraw Call数を確認
   - 目標: +20以下
7. 購入処理実行時のフリーズ有無を確認
   - 目標: 視覚的なフリーズなし（処理時間500ms以内）

**完了確認**: Quest環境で60fps維持、UIレスポンスが快適

---

### Phase 5: ドキュメント作成と納品（5日目）

#### 8.12. コードコメント追加

**目的**: 保守性を高めるため詳細なコメントを追加

**手順**:
1. ShopManager.cs を開く
2. 各メソッドに以下の形式でXMLコメントを追加:
```csharp
/// <summary>
/// メソッドの概要説明
/// </summary>
/// <param name="paramName">パラメータの説明</param>
/// <returns>戻り値の説明</returns>
```
3. 複雑なロジック箇所にインラインコメントを追加
4. マジックナンバーを定数化し、意図を明確化
5. Ctrl+S で保存

**完了確認**: 全パブリックメソッドにXMLコメント、複雑箇所にインラインコメント

#### 8.13. README作成

**目的**: 他の開発者が理解・利用できるようドキュメント化

**手順**:
1. `/Assets/WoodcutterCamp/Scripts/Economy/README_ShopManager.md` を作成
2. 以下の内容を記述:

```markdown
# ShopManager - ショップ管理システム

## 概要
きこりコインを使用したアイテム購入・装備システム。

## 依存関係
- WI-0001: GameManager
- WI-0003: PersistenceManager
- WI-0013: CoinManager
- WI-0005: SkillManager（レベル条件確認用）

## セットアップ手順
1. GameManagerにShopManagerコンポーネントを追加
2. Shop Itemsフィールドにアイテムデータを設定
3. Shop UI Panelを作成し参照を設定
4. 装備スロット（Axe Equip Slot, Hat Equip Slot）を設定
5. 商人NPCトリガーを配置

## 使用方法
### ショップを開く
```csharp
shopManager.SendCustomEvent("_OpenShop");
```

### 装備中の斧ダメージを取得
```csharp
int damage = (int)shopManager.GetProgramVariable("_GetEquippedAxeDamage");
```

## PlayerDataキー
- `UnlockedItem_{itemID}`: アイテム解禁状態（1=解禁）
- `EquippedAxeSkin`: 装備中の斧ID
- `EquippedHat`: 装備中の帽子ID

## トラブルシューティング
- ショップUIが開かない → GameManager.shopManager の参照を確認
- アイテムが表示されない → Shop Items配列にデータが設定されているか確認
- 購入後にコインが減らない → CoinManagerの参照を確認
```

3. Ctrl+S で保存

**完了確認**: README_ShopManager.md が作成され、内容が正確

---

## 9. 完了条件（Doneの条件）

以下の全条件を満たすこと:

### コード実装
- [x] ShopItemData.cs が作成され、アイテムデータ構造が定義されている
- [x] ShopManager.cs が作成され、約600-700行のUdonSharpコードが実装されている
- [x] 全パブリックメソッドにXMLコメントが記述されている
- [x] エラーハンドリングが適切に実装されている（Null チェック、範囲チェック）

### Unity Editor設定
- [x] ShopManagerコンポーネントがGameManagerに追加されている
- [x] 7個のアイテムデータがインスペクターで設定されている
- [x] ショップUIパネルが構築されている
- [x] アイテムカードPrefabが作成されている
- [x] 購入確認ダイアログが実装されている
- [x] 装備スロット（斧、帽子）が設定されている
- [x] 商人NPCトリガーが配置されている

### 機能要件
- [x] プレイヤーが商人NPCに近づきEキーでショップUI開ける
- [x] ショップUIに全7アイテムが表示される
- [x] コイン残高チェックが正しく機能する
- [x] レベル条件チェックが正しく機能する
- [x] 購入処理でコインが正しく減算される
- [x] 購入したアイテムがPlayerDataに保存される
- [x] 斧購入時に自動装備される
- [x] 装備ボタンで手動装備変更できる
- [x] 重複購入が防止されている

### テスト
- [x] 全10個のテストケース（T-01〜T-10）がパスしている
- [x] Quest環境で60fps維持（ショップUI開閉時）
- [x] PlayerData保存・読み込みが正常動作する
- [x] ワールド退出→再入場でアイテム所持状態が維持される

### ドキュメント
- [x] README_ShopManager.md が作成されている
- [x] セットアップ手順が記載されている
- [x] 使用方法サンプルコードが記載されている

---

## 10. テスト観点／テストケース

### 10.1 正常系テスト

| テストID | テスト項目 | 前提条件 | 操作手順 | 期待結果 |
|---------|-----------|---------|---------|---------|
| TC-N01 | ショップUI開閉 | プレイヤーがワールドに入場済み | 商人NPCに近づきEキー押下 | ショップUIがフェードイン表示される |
| TC-N02 | アイテム一覧表示 | ショップUI開いた状態 | UI内を確認 | 7個のアイテムが3列グリッドで表示 |
| TC-N03 | 鉄の斧購入 | コイン100以上所持 | 鉄の斧の購入ボタンクリック→「はい」 | コイン100減少、鉄の斧解禁、自動装備 |
| TC-N04 | 帽子購入 | コイン50以上所持 | 作業帽（赤）の購入ボタンクリック→「はい」 | コイン50減少、帽子解禁、所持中バッジ表示 |
| TC-N05 | 装備変更 | 鉄の斧と黒鋼の斧を両方所持 | 黒鋼の斧の装備ボタンクリック | 手に持つ斧が黒鋼の斧に変わる |
| TC-N06 | データ永続化 | 鉄の斧購入済み | ワールド退出→再入場 | 鉄の斧が所持状態で表示、装備されている |

### 10.2 異常系テスト

| テストID | テスト項目 | 前提条件 | 操作手順 | 期待結果 |
|---------|-----------|---------|---------|---------|
| TC-E01 | コイン不足エラー | コイン50所持 | 鉄の斧（100コイン）の購入試行 | 「コイン不足です（50/100）」エラー表示 |
| TC-E02 | レベル不足エラー | Woodcutting Lv3 | 黒鋼の斧（Lv5必要）の購入試行 | 「Woodcutting Lv5に到達すると購入できます」表示 |
| TC-E03 | 重複購入防止 | 鉄の斧購入済み | 鉄の斧の購入ボタンクリック | 「既に所持しています」エラー表示 |
| TC-E04 | 購入キャンセル | コイン100所持 | 購入ボタン→確認ダイアログで「いいえ」 | ダイアログ閉じる、コイン減らない |

### 10.3 パフォーマンステスト

| テストID | テスト項目 | 測定方法 | 合格基準 |
|---------|-----------|---------|---------|
| TC-P01 | ショップUI開閉時FPS | Unity Profiler | FPS低下5%以内（Quest 2で55fps以上維持） |
| TC-P02 | アイテムカード描画負荷 | Stats表示のDraw Call数 | ショップ開閉前後で+20以下 |
| TC-P03 | 購入処理時間 | Time.time計測 | 購入処理が500ms以内に完了 |

### 10.4 統合テスト

| テストID | テスト項目 | 前提条件 | 操作手順 | 期待結果 |
|---------|-----------|---------|---------|---------|
| TC-I01 | CoinManager連携 | CoinManager動作中 | 鉄の斧購入 | CoinManager.currentCoinsが100減少 |
| TC-I02 | SkillManager連携 | SkillManager動作中 | Lv5で黒鋼の斧購入 | 購入成功、装備後のダメージが20に |
| TC-I03 | PersistenceManager連携 | PersistenceManager動作中 | アイテム購入→ワールド退出→再入場 | PlayerDataから正しくロード |

---

## 11. 成果物

### 変更ファイル
- `/Assets/WoodcutterCamp/Scripts/Economy/ShopManager.cs`（新規作成）
- `/Assets/WoodcutterCamp/Scripts/Economy/ShopItemData.cs`（新規作成）
- `/Assets/WoodcutterCamp/Scripts/Triggers/ShopTrigger.cs`（新規作成）
- `/Assets/WoodcutterCamp/Prefabs/UI/ItemCard.prefab`（新規作成）
- `/Assets/WoodcutterCamp/Prefabs/UI/ShopUI.prefab`（新規作成）

### 追加シーンオブジェクト
- `GameManager/ShopManager`（UdonBehaviour）
- `ShopUICanvas`（Canvas）
- `MerchantNPCTrigger`（トリガーゾーン）
- `PlayerLocal/RightHand/AxeEquipSlot`（装備スロット）
- `PlayerLocal/Head/HatEquipSlot`（装備スロット）

### 更新したドキュメント
- `/Assets/WoodcutterCamp/Scripts/Economy/README_ShopManager.md`（新規作成）
- `/mnt/d/Dropbox/Dev/CLI/WCF/docs/WID/WI-0019_ShopManager.md`（本ドキュメント）

---

## 12. 備考

### 注意点
1. **UdonSharpの制約**: `List<T>` や LINQ は使用できないため、すべて配列ベースで実装
2. **GetComponent() のキャッシング**: Start()で一度だけ取得し、Update()での呼び出しを避ける
3. **Manual Sync不要**: ShopManagerはローカルのみの処理のため、UdonSyncedは使用しない
4. **PlayerDataキーの一貫性**: `UnlockedItem_{itemID}` の形式を統一し、マイグレーションを考慮
5. **エラーハンドリング**: Null チェックを徹底し、エラー時も処理を継続できるようにする

### VR/デスクトップ対応
- **VRモード**: トリガーボタンでUI操作、手のレイキャストでボタンクリック
- **デスクトップモード**: マウスクリックでUI操作
- 両環境で同じUIが正常動作することを確認

### 今後の拡張予定（Phase 2）
- 運搬ベルトアイテムの効果実装（移動速度デバフ軽減）
- コスメティックアイテムの大量追加（20種類以上）
- アイテムのレアリティシステム
- ショップNPCとの会話システム

### 参照ドキュメント
- TDD.md Section 5.7（M-07 ShopManager）
- FSD.md FNC-005（経済システム）
- PRD.md Section 2.5（ショップシステムの詳細）
- VRChat SDK Documentation - PlayerData API

---

**作業見積もり時間**: 4〜5日（約32〜40時間）

**作業者**: システムエンジニア（Unity/UdonSharp経験必須、VRChat開発経験推奨）

**承認者**: プロジェクトリーダー

**作成日**: 2025-11-17

**バージョン**: 1.0
