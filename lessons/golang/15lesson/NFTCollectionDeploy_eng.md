# 部署 NFT 集合

## 簡介

在本教程中，我們將使用 tonutils-go 庫來部署 NFT 集合。為了全面了解如何部署 NFT 集合，我們將首先對現有集合進行查詢，了解其存儲的信息，然後我們將創建自己的 NFT 集合（這是一個測試集合，沒有實際意義）。

在開始本教程之前，建議先查看上一課程，以了解如何創建錢包和部署合約。

## 獲取集合和單個元素的信息

獲取集合信息涉及對智能合約進行 GET 請求。在本課程中，我們將考慮從遵循標準的智能合約中獲取信息。關於 NFT 標準的課程可以參考[這裡](https://github.com/romanovichim/TonFunClessons_ru/blob/main/10lesson/tenthlesson.md)。標準本身可以在[這裡](https://github.com/ton-blockchain/TIPs/issues/62)找到。

### 根據 NFT 集合標準可以獲取哪些信息

符合標準的集合智能合約必須實現 `get_collection_data()` 方法，該方法將返回集合所有者的地址、集合的內容以及當前集合中的 NFT 數量。該方法如下：

```func
(int, cell, slice) get_collection_data() method_id {
  var (owner_address, next_item_index, content, _, _) = load_data();
  slice cs = content.begin_parse();
  return (next_item_index, cs~load_ref(), owner_address);
}
```

> load_data() 從 c4 寄存器中卸載數據

如果我們只是對合約執行請求，我們將不得不“解析”slice 和其他與類型相關的不愉快事情。在 `tonutils-go` 中，有一個 `GetCollectionData` 函數，將幫助我們簡化這個過程，我們將在接下來使用它。

例如，我們從某個市場上選擇一個集合，檢查我們獲取的信息是否與市場上的信息一致。

我將在本教程中使用的集合地址為：

```
EQAA1yvDaDwEK5vHGOXRdtS2MbOVd1-TNy01L1S_t2HF4oLu
```

根據市場上的信息，該地址的集合中有 13333 個元素，我們來驗證一下。

### 使用 GO 獲取 NFT 集合信息

連接到主網上的 lightservers：

```go
func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)
}
```

> 該集合也存在於測試網中，因此如果您使用測試網配置，也會有效。

獲取集合地址並使用 `GetCollectionData` 函數調用 `get_collection_data()` 方法，將數據轉換為可讀格式：

```go
func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)

	nftColAddr := address.MustParseAddr("EQAA1yvDaDwEK5vHGOXRdtS2MbOVd1-TNy01L1S_t2HF4oLu")

	// 獲取我們的 NFT 集合的信息
	collection := nft.NewCollectionClient(api, nftColAddr)
	collectionData, err := collection.GetCollectionData(context.Background())
	if err != nil {
		panic(err)
	}

	fmt.Println("Collection addr      :", nftColAddr)
	fmt.Println("    content          :", collectionData.Content.(*nft.ContentOffchain).URI)
	fmt.Println("    owner            :", collectionData.OwnerAddress.String())
	fmt.Println("    minted items num :", collectionData.NextItemIndex)
}
```

輸出應如下所示：

```
Collection addr      : EQAA1yvDaDwEK5vHGOXRdtS2MbOVd1-TNy01L1S_t2HF4oLu
    content          : http://nft.animalsredlist.com/nfts/collection.json
    owner            : EQANKN8ZnM0OzYOENTkOEg7VVgFog5fBWdCtqQro1MRmU5_2
    minted items num : 13333
```

可以看到信息是對應的，集合中有 13333 個元素。

完整的 `nftcoldata.go` 代碼：

```go
package main

import (
	"context"
	"fmt"

	"github.com/xssnick/tonutils-go/address"
	"github.com/xssnick/tonutils-go/liteclient"
	"github.com/xssnick/tonutils-go/ton"
	"github.com/xssnick/tonutils-go/ton/nft"
)

func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)

	nftColAddr := address.MustParseAddr("EQAA1yvDaDwEK5vHGOXRdtS2MbOVd1-TNy01L1S_t2HF4oLu")

	// 獲取我們的 NFT 集合的信息
	collection := nft.NewCollectionClient(api, nftColAddr)
	collectionData, err := collection.GetCollectionData(context.Background())
	if err != nil {
		panic(err)
	}

	fmt.Println("Collection addr      :", nftColAddr)
	fmt.Println("    content          :", collectionData.Content.(*nft.ContentOffchain).URI)
	fmt.Println("    owner            :", collectionData.OwnerAddress.String())
	fmt.Println("    minted items num :", collectionData.NextItemIndex)
}
```

## 單個 NFT 元素可以獲取哪些信息

假設我們想獲取集合元素的地址、其內容（例如圖片的鏈接）。看起來很簡單，我們只需要調用 Get 方法並獲取信息。但根據[NFT 標準](https://github.com/ton-blockchain/TIPs/issues/62)，我們無法獲取完整的鏈接，只能獲取部分內容，即所謂的單個元素內容。

要獲取完整內容（地址），需要：
- 通過元素的 get 方法 `get_nft_data()` 獲取元素索引和個人內容，以及初始化標誌
- 檢查元素是否已初始化（更多詳情參見第10課，其中討論了 NFT 標準）
- 如果元素已初始化，則通過集合的 get 方法 `get_nft_content(int index, cell individual_content)` 獲取單個元素的完整內容（完整地址）

### 使用 GO 獲取 NFT 元素信息

我將在下面使用的元素地址：

```
UQBzmkmGYAw3qNEQYddY-FjWRPJRjg7Vv2B1Dns3FrERcaRH
```

我們嘗試獲取此 NFT 元素的信息。

建立與 lightservers 的連接：

```go
func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)
}
```

調用元素的 get 方法 `get_nft_data()` 並將接收到的信息輸出到控制台：

```go
func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)

	nftAddr := address.MustParseAddr("UQBzmkmGYAw3qNEQYddY-FjWRPJRjg7Vv2B1Dns3FrERcaRH")
	item := nft.NewItemClient(api, nftAddr)

	nftData, err := item.GetNFTData(context.Background())
	if err != nil {
		panic(err)
	}

	fmt.Println("NFT addr         :", nftAddr.String())
	fmt.Println("    initialized  :", nftData.Initialized)
	fmt.Println("    owner        :", nftData.OwnerAddress.String())
	fmt.Println("    index        :", nftData.Index)
}
```

除了我們顯示的信息外，我們還有關於集合的信息，可以使用以下代碼獲取：

```go
// 獲取我們的 NFT 集合的信息
collection := nft.NewCollectionClient(api, nftData.CollectionAddress)
```

檢查元素是否已初始化，並調用集合的 get 方法 `get_nft_content(int index, cell individual_content)` 獲取元素的鏈接。

```go
// 獲取我們的 NFT 集合的信息
collection := nft.NewCollectionClient(api, nftData.CollectionAddress)

if nftData.Initialized {
	// 使用集合方法獲取完整的 NFT 內容 URL，將基礎 URL 與 NFT 的數據合併
	nftContent, err := collection.GetNFTContent(context.Background(), nftData.Index, nftData.Content)
	if err != nil {
		panic(err)
	}
	fmt.Println("    part content :", nftData.Content.(*nft.ContentOffchain).URI)
	fmt.Println("    full content :", nftContent.(*nft.ContentOffchain).URI)
} else {
	fmt.Println("    empty content")
}
```

最終的 `nftitemdata.go` 代碼：

```go
package main

import (
	"context"
	"fmt"

	"github.com/xssnick/tonutils-go/address"
	"github.com/xssnick/tonutils-go/liteclient"
	"github.com/xssnick/tonutils-go/ton"
	"github.com/xssnick/tonutils-go/ton/nft"
)

func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}

	api := ton.NewAPIClient(client)

	nftAddr := address.MustParseAddr("UQBzmkmGYAw3qNEQYddY-FjWRPJRjg7Vv2B1Dns3FrERcaRH")
	item := nft.NewItemClient(api, nftAddr)

	nftData, err := item.GetNFTData(context.Background())
	if err != nil {
		panic(err)
	}

	fmt.Println("NFT addr         :", nftAddr.String())
	fmt.Println("    initialized  :", nftData.Initialized)
	fmt.Println("    owner        :", nftData.OwnerAddress.String())
	fmt.Println("    index        :", nftData.Index)

	// 獲取我們的 NFT 集合的信息
	collection := nft.NewCollectionClient(api, nftData.CollectionAddress)

	if nftData.Initialized {
		// 使用集合方法獲取完整的 NFT 內容 URL，將基礎 URL 與 NFT 的數據合併
		nftContent, err := collection.GetNFTContent(context.Background(), nftData.Index, nftData.Content)
		if err != nil {
			panic(err)
		}
		fmt.Println("    part content :", nftData.Content.(*nft.ContentOffchain).URI)
		fmt.Println("    full content :", nftContent.(*nft.ContentOffchain).URI)
	} else {
		fmt.Println("    empty content")
	}
}
```

結果應該是如下元素：https://nft.animalsredlist.com/nfts/11030.json

## 部署智能合約集合

在我們了解了如何查看其他集合和元素的信息後，我們將嘗試在測試網上部署我們的集合和元素。在繼續之前，我建議先瀏覽上一課，因為我不會詳細介紹如何創建錢包、創建 hexBOC 合約形式和在測試網上部署合約。

讓我們分析一下部署集合所需的內容。首先，我們需要合約的 hexBOC 表示，其次是 c4 寄存器的初始數據。

我們從第二步開始，根據標準，我們將確定需要放入 c4 的數據。查看 [collection contract](https://github.com/ton-blockchain/token-contract/blob/main/nft/nft-collection.fc) 的數據加載函數非常方便。

```func
(slice, int, cell, cell, cell) load_data() inline {
  var ds = get_data().begin_parse();
  return 
	(ds~load_msg_addr(), ;; owner_address
	 ds~load_uint(64), ;; next_item_index
	 ds~load_ref(), ;; content
	 ds~load_ref(), ;; nft_item_code
	 ds~load_ref()  ;; royalty_params
	 );
}
```

讓集合所有者的地址是我們將用於部署的錢包地址，因此我們將地址作為參數傳遞給函數：

```go
func getContractData(collectionOwnerAddr, royaltyAddr *address.Address) *cell.Cell {
	// implementation
}
```

同時需要傳遞版權地址，我們將其傳遞到版權參數中。在本示例中，我們不設置任何版權值，因此我們將傳遞零。（您可以在[這裡](https://github.com/ton-blockchain/TEPs/blob/afb3b967db3cf693f1b667f771150056d53944d5/text/0066-nft-royalty-standard.md)閱讀有關版權參數的更多信息）

```go
func getContractData(collectionOwnerAddr, royaltyAddr *address.Address) *cell.Cell {
	royalty := cell.BeginCell().
		MustStoreUInt(0, 16).
		MustStoreUInt(0, 16).
		MustStoreAddr(royaltyAddr).
		EndCell()
}
```

現在我們來收集內容部分，它分為兩個單元 `collection_content` 和 `common_content`，根據標準：

```go
func getContractData(collectionOwnerAddr, royaltyAddr *address.Address) *cell.Cell {
	royalty := cell.BeginCell().
		MustStoreUInt(0, 16).
		MustStoreUInt(0, 16).
		MustStoreAddr(royaltyAddr).
		EndCell()

	collectionContent := nft.ContentOffchain{URI: "https://tonutils.com"}
	collectionContentCell, _ := collectionContent.ContentCell()

	commonContent := nft.ContentOffchain{URI: "https://tonutils.com/nft/"}
	commonContentCell, _ := commonContent.ContentCell()

	contentRef := cell.BeginCell().
		MustStoreRef(collectionContentCell).
		MustStoreRef(commonContentCell).
		EndCell()
}
```

索引將為零，代碼我們將創建一個單獨的 `getNFTItemCode()` 函數，它將存儲單個元素的合約代碼 hexBOC 格式。最終，我們得到：

```go
func getContractData(collectionOwnerAddr, royaltyAddr *address.Address) *cell.Cell {
	royalty := cell.BeginCell().
		MustStoreUInt(0, 16).
		MustStoreUInt(0, 16).
		MustStoreAddr(royaltyAddr).
		EndCell()

	collectionContent := nft.ContentOffchain{URI: "https://tonutils.com"}
	collectionContentCell, _ := collectionContent.ContentCell()

	commonContent := nft.ContentOffchain{URI: "https://tonutils.com/nft/"}
	commonContentCell, _ := commonContent.ContentCell()

	contentRef := cell.BeginCell().
		MustStoreRef(collectionContentCell).
		MustStoreRef(commonContentCell).
		EndCell()

	data := cell.BeginCell().MustStoreAddr(collectionOwnerAddr).
		MustStoreUInt(0, 64).
		MustStoreRef(contentRef).
		MustStoreRef(getNFTItemCode()).
		MustStoreRef(royalty).
		EndCell()

	return data
}
```

只剩下部署合約：

```go
addr, err := w.DeployContract(context.Background(), tlb.MustFromTON("0.02"),
	msgBody, getNFTCollectionCode(), getContractData(w.Address(), nil), true)
if err != nil {
	panic(err)
}
```

完整代碼可在[這裡](https://github.com/xssnick/tonutils-go/blob/master/example/deploy-nft-collection/main.go)找到。

## 鑄造元素到集合

將元素添加到集合稱為鑄造。如果查看 [collection contract example](https://github.com/ton-blockchain/token-contract/blob/main/nft/nft-collection.fc) 可以看到，要鑄造新的 NFT 元素，需要發送內部消息。

因此：
- 調用集合的 get 方法 `get_collection_data()` 以獲取我們需要的鑄造索引
- 調用集合的 get 方法 `get_nft_address_by_index(int index)` 以獲取 NFT 元素的地址
- 構建有效負載（元素索引、錢包地址、小量 TON、內容）
- 將消息發送到集合的智能合約地址，附上我們的有效負載

首先連接到 light servers：

```go
func main() {
	client := liteclient.NewConnectionPool()

	// 連接到主網 lite server
	err := client.AddConnection(context.Background(), "135.181.140.212:13206", "K0t3+IWLOXHYMvMcrGZDPs+pn58a17LFbnXoQkKc2xw=")
	if err != nil {
		panic(err)
	}

	// 初始化 ton api lite 連接包裝器
	api := ton.NewAPIClient(client)
}
```

“收集”錢包，調用 `get_collection_data()` 獲取索引：

```go
func main() {
	client := liteclient.NewConnectionPool()

	// 連接到主網 lite server
	err := client.AddConnection(context.Background(), "135.181.140.212:13206", "K0t3+IWLOXHYMvMcrGZDPs+pn58a17LFbnXoQkKc2xw=")
	if err != nil {
		panic(err)
	}

	// 初始化 ton api lite 連接包裝器
	api := ton.NewAPIClient(client)
	w := getWallet(api)

	collectionAddr := address.MustParseAddr("EQCSrRIKVEBaRd8aQfsOaNq3C4FVZGY5Oka55A5oFMVEs0lY")
	collection := nft.NewCollectionClient(api, collectionAddr)

	collectionData, err := collection.GetCollectionData(context.Background())
	if err != nil {
		panic(err)
	}
}
```

> 重要的是要使用我們在部署集合合約時放入 c4 的地址，否則在鑄造時會發生錯誤，因為合約中有檢查可以鑄造的地址（看起來像這樣：`throw_unless(401, equal_slices(sender_address, owner_address));`）。

現在調用集合的 get 方法 `get_nft_address_by_index(int index)` 以獲取元素的 NFT 地址並準備有效負載：

```go
func main() {
	client := liteclient.NewConnectionPool()

	// 連接到主網 lite server
	err := client.AddConnection(context.Background(), "135.181.140.212:13206", "K0t3+IWLOXHYMvMcrGZDPs+pn58a17LFbnXoQkKc2xw=")
	if err != nil {
		panic(err)
	}

	// 初始化 ton api lite 連接包裝器
	api := ton.NewAPIClient(client)
	w := getWallet(api)

	collectionAddr := address.MustParseAddr("EQCSrRIKVEBaRd8aQfsOaNq3C4FVZGY5Oka55A5oFMVEs0lY")
	collection := nft.NewCollectionClient(api, collectionAddr)

	collectionData, err := collection.GetCollectionData(context.Background())
	if err != nil {
		panic(err)
	}

	nftAddr, err := collection.GetNFTAddressByIndex(context.Background(), collectionData.NextItemIndex)
	if err != nil {
		panic(err)
	}

	mintData, err := collection.BuildMintPayload(collectionData.NextItemIndex, w.Address(), tlb.MustFromTON("0.01"), &nft.ContentOffchain{
		URI: fmt.Sprint(collectionData.NextItemIndex) + ".json",
	})
	if err != nil {
		panic(err)
	}
}
```

只剩下將消息從錢包發送到集合的智能合約並顯示我們元素的信息（通過調用 get 方法 `get_nft_data()` 檢查一切是否正確 - 查看是否收到正確的信息）。

```go
func main() {
	client := liteclient.NewConnectionPool()

	// 連接到主網 lite server
	err := client.AddConnection(context.Background(), "135.181.140.212:13206", "K0t3+IWLOXHYMvMcrGZDPs+pn58a17LFbnXoQkKc2xw=")
	if err != nil {
		panic(err)
	}

	// 初始化 ton api lite 連接包裝器
	api := ton.NewAPIClient(client)
	w := getWallet(api)

	collectionAddr := address.MustParseAddr("EQCSrRIKVEBaRd8aQfsOaNq3C4FVZGY5Oka55A5oFMVEs0lY")
	collection := nft.NewCollectionClient(api, collectionAddr)

	collectionData, err := collection.GetCollectionData(context.Background())
	if err != nil {
		panic(err)
	}

	nftAddr, err := collection.GetNFTAddressByIndex(context.Background(), collectionData.NextItemIndex)
	if err != nil {
		panic(err)
	}

	mintData, err := collection.BuildMintPayload(collectionData.NextItemIndex, w.Address(), tlb.MustFromTON("0.01"), &nft.ContentOffchain{
		URI: fmt.Sprint(collectionData.NextItemIndex) + ".json",
	})
	if err != nil {
		panic(err)
	}

	fmt.Println("Minting NFT...")
	mint := wallet.SimpleMessage(collectionAddr, tlb.MustFromTON("0.025"), mintData)

	err = w.Send(context.Background(), mint, true)
	if err != nil {
		panic(err)
	}

	fmt.Println("Minted NFT:", nftAddr.String(), 0)

	newData, err := nft.NewItemClient(api, nftAddr).GetNFTData(context.Background())
	if err != nil {
		panic(err)
	}

	fmt.Println("Minted NFT addr: ", nftAddr.String())
	fmt.Println("NFT Owner:", newData.OwnerAddress.String())
}
```

完整代碼可在[這裡](https://github.com/xssnick/tonutils-go/blob/master/example/nft-mint/main.go)找到。

## 練習

在測試網上部署您的集合並創建一個 NFT 元素，然後嘗試使用課程開頭的腳本獲取集合和元素的信息。

## 結論

我會在[這裡](https://t.me/ton_learn)發布新課程，在[主頁](https://github.com/romanovichim/TonFunClessons_ru)上有捐款地址，如果您希望幫助發布新課程。特別感謝 https://github.com/xssnick/tonutils-go 的開發者，他們做了出色的工作。