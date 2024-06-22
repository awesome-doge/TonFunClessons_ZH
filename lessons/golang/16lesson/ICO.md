## 簡介

## 什麼是 ICO

ICO，即初始代幣發行（Initial Coin Offering），是指任何項目或公司發行其自有代幣（加密貨幣），以籌集投資的行為。

### 為什麼需要 ICO

通過 ICO，項目可以獲得所需的資金，以進行開發、擴展和增長。通常，在進行 ICO 時，預期這些代幣的價值會隨時間上升。「優質」項目通常會在其路線圖中設計各種機制，防止代幣價格急劇下跌，進而避免更大的價格波動。

如果你對 ICO 的盈利情況感興趣，可以查看 [這裡](https://icodrops.com/ico-stats/) 的 ICO 項目 ROI 統計數據。

> 你可以通過 USD ROI 來篩選頂尖項目。

### 重要提示：風險

談到 ICO，不能忽視風險。購買代幣時，實際上是購買區塊鏈中的記錄，其價值完全由代幣發行項目提供。從技術角度來看，進行 ICO 的智能合約可能會被黑客攻擊，或初始設計中就存在後門，使智能合約所有者可以改變 ICO 的條款。此外，任何項目都有可能變成詐騙，即使其初衷並非如此。

## 智能合約概述

在本教程中，我們將使用 Jetton 標準示例中的智能合約，即 `jetton-minter-ICO.fc` 主合約 [代碼在這裡](https://github.com/ton-blockchain/token-contract/tree/main/ft)。

與我們在第九課中詳細分析的主合約相比，這個 ICO 智能合約的主要區別在於 `recv_internal()` 中包含的機制：

```func
if (in_msg_body.slice_empty?()) { ;; 用 Toncoin 購買 jettons

    int amount = 10000000; ;; mint 消息的數量
    int buy_amount = msg_value - amount;
    throw_unless(76, buy_amount > 0);

    int jetton_amount = buy_amount; ;; 1 jetton = 1 toncoin; 這裡可乘以價格

    var master_msg = begin_cell()
        .store_uint(op::internal_transfer(), 32)
        .store_uint(0, 64) ;; quert_id
        .store_coins(jetton_amount)
        .store_slice(my_address()) ;; from_address
        .store_slice(sender_address) ;; response_address
        .store_coins(0) ;; no forward_amount
        .store_uint(0, 1) ;; forward_payload in this slice, not separate cell
        .end_cell();

    mint_tokens(sender_address, jetton_wallet_code, amount, master_msg);
    save_data(total_supply + jetton_amount, admin_address, content, jetton_wallet_code);
    return ();
}
```

如你所見，Toncoin 兌換代幣的操作是通過發送空消息體的消息來實現的。因此，在本課中我們將完成以下步驟：
- 創建兩個錢包：一個用於部署主合約，另一個用於發送空消息以獲得代幣
- 部署 `jetton-minter-ICO.fc`
- 從第二個錢包發送空消息體及一些 Toncoin 以兌換代幣
- 驗證代幣餘額變化

## 部署 ICO 智能合約到測試網

> 如果你已經完成了前面的課程並記得內容，可以直接跳到部署合約部分

### 錢包

首先，需要在 TON 中創建兩個錢包 w1 和 w2，其中一個將用作智能合約的「管理員地址」，另一個將用於在測試網中用測試 TON 兌換 Jetton。（有關如何創建錢包的教程在 [這裡](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/14lesson/wallet_eng.md)）

`SeedPhrase.go` 代碼：

```go
package main

import (
	"context"
	"log"
	"fmt"

	"github.com/xssnick/tonutils-go/liteclient"
	"github.com/xssnick/tonutils-go/ton"
	"github.com/xssnick/tonutils-go/ton/wallet"
)

func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/testnet-global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}
	api := ton.NewAPIClient(client)

	seed1 := wallet.NewSeed()
	fmt.Println("Seed phrase one:")
	fmt.Println(seed1)

	w1, err := wallet.FromSeed(api, seed1, wallet.V3)
	if err != nil {
		log.Fatalln("FromSeed err:", err.Error())
		return
	}
	fmt.Println("Address one:")
	fmt.Println(w1.Address())

	seed2 := wallet.NewSeed()
	fmt.Println("Seed phrase two:")
	fmt.Println(seed2)

	w2, err := wallet.FromSeed(api, seed2, wallet.V3)
	if err != nil {
		log.Fatalln("FromSeed err:", err.Error())
		return
	}
	fmt.Println("Address two:")
	fmt.Println(w2.Address())

	block, err := api.CurrentMasterchainInfo(context.Background())
	if err != nil {
		log.Fatalln("CurrentMasterchainInfo err:", err.Error())
		return
	}

	balance1, err := w1.GetBalance(context.Background(), block)
	if err != nil {
		log.Fatalln("GetBalance err:", err.Error())
		return
	}
	fmt.Println("Balance one:")
	fmt.Println(balance1)

	balance2, err := w2.GetBalance(context.Background(), block)
	if err != nil {
		log.Fatalln("GetBalance err:", err.Error())
		return
	}
	fmt.Println("Balance two:")
	fmt.Println(balance2)
}
```

將種子短語保存到某個地方，並通過測試網上的機器人向兩個地址發送測試 Toncoin：https://t.me/testgiver_ton_bot

稍後檢查資金是否到達：https://testnet.tonscan.org/

> 由於第二個錢包需要一些時間來補充資金，因此需要稍作等待。

我們將使用我們編寫的函數來管理錢包，類似於之前的課程。我們將使用這些函數來查詢餘額。

`WalletFunC.go` 代碼：

```go
package main

import (
	"context"
	"log"
	"fmt"
	"strings"

	"github.com/xssnick/tonutils-go/liteclient"
	"github.com/xssnick/tonutils-go/ton"
	"github.com/xssnick/tonutils-go/ton/wallet"
)

func main() {
	client := liteclient.NewConnectionPool()
	configUrl := "https://ton-blockchain.github.io/testnet-global.config.json"

	err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
	if err != nil {
		panic(err)
	}
	api := ton.NewAPIClient(client)

	w1 := getWallet1(api)
	w2 := getWallet2(api)

	fmt.Println(w1.Address())
	fmt.Println(w2.Address())

	block, err := api.CurrentMasterchainInfo(context.Background())
	if err != nil {
		log.Fatalln("CurrentMasterchainInfo err:", err.Error())
		return
	}

	balance1, err := w1.GetBalance(context.Background(), block)
	if err != nil {
		log.Fatalln("GetBalance1 err:", err.Error())
		return
	}

	fmt.Println(balance1)

	balance2, err := w2.GetBalance(context.Background(), block)
	if err != nil {
		log.Fatalln("GetBalance2 err:", err.Error())
		return
	}

	fmt.Println(balance2)
}

func getWallet1(api *ton.APIClient) *wallet.Wallet {
	words := strings.Split("你的種子短語 1", " ")
	w, err := wallet.FromSeed(api, words, wallet.V3)
	if err != nil {
		panic(err)
	}
	return w
}

func getWallet2(api *ton.APIClient) *wallet.Wallet {
	words := strings.Split("你的種子短語 2", " ")
	w, err := wallet.FromSeed(api, words, wallet.V3)
	if err != nil {
		panic(err)
	}
	return w
}
```

> 當然，你可以寫一個函數並傳遞參數，但這樣做是為了便於理解代碼。

### 部署合約

#### 創建合約的 hexBoc 表示

在 `tonutils-go` 庫中，可以將智能合約以 hexBoc 形式部署。Boc 是智能合約的序列化形式（bag-of-cells）。要將智能合約從 func 轉換為 hexBoc 形式，首先需要將其編譯為 fift，然後使用一個獨立的 fift 腳本來獲取 hexBoc。這可以使用熟悉的 `toncli` 來完成。具體步驟如下。

##### 構建 jetton-minter-ICO 和 jetton-wallet 代碼

我們將從[示例](https://github.com/ton-blockchain/token-contract/tree/main/ft)中獲取 func 代碼，我們需要 `jetton-minter-ICO.fc` 和 `jetton-minter.fc`，以及一些輔助代碼：
- `jetton-utils.fc`
- `op-codes.fc`
- `params.fc`

> 為方便起見，我已將兩個合併代碼（所有文件）放在一起，見 `code-amalgama.func` 和 `codewallet-amalgama.func`。

##### 獲取 fift

使用 `toncli func build` 將 func 代碼轉換為 fift。

> 在代碼中，生成的文件是 `contract` 和 `contractwallet`。

##### 打印 hexBoc

現在使用一個腳本將代碼轉換為 hexBOC 格式：

```fift
#!/usr/bin/fift -s
"TonUtil.fif" include
"Asm.fif" include

."first contract:" cr

"first.fif" include
2 boc+>B dup Bx. cr cr
```

我不會詳細介紹 fift，這超出了本課範圍，我只想指出：
- boc+>B - 序列化為 boc 格式
- cr - 在字符串中顯示值

> 你可以使用熟悉的 `toncli` 運行腳本，即 `toncli fift run`，或按[此處](https://ton-blockchain.github.io/docs/#/compile?id=fift)描述的方法運行。

示例腳本在 `print-hex.fif` 文件中。

結果：
- `jetton-minter-ICO.fc` 的 hexBoc: B5EE9C7241020B010001F5000114FF00F4A413F4BCF2C80B0102016202030202CD040502037A60090A03F7D00E8698180B8D8492F81F07D201876A2687D007D206A6A1812E38047221AC1044C4B4028B350906100797026381041080BC6A28CE4658FE59F917D017C14678B13678B10FD0165806493081B2044780382502189E428027D012C678B666664F6AA701B02698FE99FC00AA9185D718141083DEECBEF09DD71812F83C0607080093F7C142201B82A1009AA0A01E428027D012C678B00E78B666491646580897A007A00658064907C80383A6465816503E5FFE4E83BC00C646582AC678B28027D0109E5B589666664B8FD80400606C215131C705F2E04902FA40FA00D43020D08060D721FA00302710345042F007A05023C85004FA0258CF16CCCCC9ED5400FC01FA00FA40F82854120970542013541403C85004FA0258CF1601CF16CCC922C8CB0112F400F400CB00C9F9007074C8CB02CA07CBFFC9D05006C705F2E04A13A1034145C85004FA0258CF16CCCCC9ED5401FA403020D70B01C3008E1F8210D53276DB708010C8CB055003CF1622FA0212CB6ACB1FCB3FC98042FB00915BE20008840FF2F0007DADBCF6A2687D007D206A6A183618FC1400B82A1009AA0A01E428027D012C678B00E78B666491646580897A007A00658064FC80383A6465816503E5FFE4E840001FAF16F6A2687D007D206A6A183FAA9040CA85A166 
- `jetton-minter.fc` 的 hexBoc: B5EE9C7241021201000330000114FF00F4A413F4BCF2C80B0102016202030202CC0405001BA0F605DA89A1F401F481F481A8610201D40607020148080900BB0831C02497C138007434C0C05C6C2544D7C0FC02F83E903E900C7E800C5C75C87E800C7E800C00B4C7E08403E29FA954882EA54C4D167C0238208405E3514654882EA58C511100FC02780D60841657C1EF2EA4D67C02B817C12103FCBC2000113E910C1C2EBCB853600201200A0B020120101101F100F4CFFE803E90087C007B51343E803E903E90350C144DA8548AB1C17CB8B04A30BFFCB8B0950D109C150804D50500F214013E809633C58073C5B33248B232C044BD003D0032C032483E401C1D3232C0B281F2FFF274013E903D010C7E800835D270803CB8B11DE0063232C1540233C59C3E8085F2DAC4F3200C03F73B51343E803E903E90350C0234CFFE80145468017E903E9014D6F1C1551CDB5C150804D50500F214013E809633C58073C5B33248B232C044BD003D0032C0327E401C1D3232C0B281F2FFF274140371C1472C7CB8B0C2BE80146A2860822625A020822625A004AD822860822625A028062849F8C3C975C2C070C008E00D0E0F00AE8210178D4519C8CB1F19CB3F5007FA0222CF165006CF1625FA025003CF16C95005CC2391729171E25008A813A08208989680AA008208989680A0A014BCF2E2C504C98040FB001023C85004FA0258CF1601CF16CCC9ED5400705279A018A182107362D09CC8CB1F5230CB3F58FA025007CF165007CF16C9718010C8CB0524CF165006FA0215CB6A14CCC971FB0010241023000E10491038375F040076C200B08E218210D53276DB708010C8CB055008CF165004FA0216CB6A12CB1F12CB3FC972FB0093356C21E203C85004FA0258CF1601CF16CCC9ED5400DB3B51343E803E903E90350C01F4CFFE803E900C145468549271C17CB8B049F0BFFCB8B0A0822625A02A8005A805AF3CB8B0E0841EF765F7B232C7C572CFD400FE8088B3C58073C5B25C60063232C14933C59C3E80B2DAB33260103EC01004F214013E809633C58073C5B3327B55200083200835C87B51343E803E903E90350C0134C7E08405E3514654882EA0841EF765F784EE84AC7CB8B174CFCC7E800C04E81408F214013E809633C58073C5B3327B55209FB23AB6

#### 為 Jetton 準備數據

除了 hexBoc，我們還需要 jetton-minter-ICO 存儲合約的數據。讓我們來看看標準中需要的數據：

```func
;; storage scheme
;; storage#_ total_supply:Coins admin_address:MsgAddress content:^Cell jetton_wallet_code:^Cell = Storage;
```

為方便起見，我們來看看將數據保存到 `c4` 寄存器的函數：

```func
() save_data(int total_supply, slice admin_address, cell content, cell jetton_wallet_code) impure inline {
    set_data(begin_cell()
        .store_coins(total_supply)
        .store_slice(admin_address)
        .store_ref(content)
        .store_ref(jetton_wallet_code)
        .end_cell()
    );
}
```

內容可以參考標準中的描述[這裡](https://github.com/ton-blockchain/TIPs/issues/64)。由於這是一個測試範例，我們不會收集所有數據，只會放置一個鏈接，以便在課程中使用。

```go
func getContractData(OwnerAddr *address.Address) *cell.Cell {
    // storage scheme
    // storage#_ total_supply:Coins admin_address:MsgAddress content:^Cell jetton_wallet_code:^Cell = Storage;

    uri := "https://github.com/romanovichim/TonFunClessons_ru"
    jettonContentCell := cell.BeginCell().MustStoreStringSnake(uri).EndCell()

    contentRef := cell.BeginCell().
        MustStoreRef(jettonContentCell).
        EndCell()

    return data
}
```

在準備好鏈接後，我們將組合數據 cell，並將以下內容放入：
- 總代幣供應量 MustStoreUInt(10000000, 64)
- 管理員錢包地址 MustStoreAddr(OwnerAddr)
- jettonContentCell 內容 cell
- 合約錢包代碼 MustStoreRef(getJettonWalletCode())

```go
func getContractData(OwnerAddr *address.Address) *cell.Cell {
    // storage scheme
    // storage#_ total_supply:Coins admin_address:MsgAddress content:^Cell jetton_wallet_code:^Cell = Storage;

    uri := "https://github.com/romanovichim/TonFunClessons_ru"
    jettonContentCell := cell.BeginCell().MustStoreStringSnake(uri).EndCell()

    contentRef := cell.BeginCell().
        MustStoreRef(jettonContentCell).
        EndCell()

    data := cell.BeginCell().MustStoreUInt(10000000, 64).
        MustStoreAddr(OwnerAddr).
        MustStoreRef(contentRef).
        MustStoreRef(getJettonWalletCode()).
        EndCell()

    return data
}
```

#### 部署

總體來說，部署腳本與我們部署 NFT 集合的課程中的腳本相似。我們有一個 `getContractData` 函數來準備數據，兩個從 hexboc 合約和錢包主合約中提取函數，然後從主函數中部署 ICO 合約：

```go
func main() {
    // 連接到主網 lite 服務器
    client := liteclient.NewConnectionPool()
    configUrl := "https://ton-blockchain.github.io/testnet-global.config.json"

    err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
    if err != nil {
        panic(err)
    }
    api := ton.NewAPIClient(client)
    w := getWallet(api)

    msgBody := cell.BeginCell().EndCell()

    fmt.Println("部署 Jetton ICO 合約到主網...")
    addr, err := w.DeployContract(context.Background(), tlb.MustFromTON("0.02"),
        msgBody, getJettonMasterCode(), getContractData(w.Address()), true)
    if err != nil {
        panic(err)
    }

    fmt.Println("已部署合約地址:", addr.String())
}
```

完整的腳本在 `DeployJettonMinter.go` 文件中。

### 調用智能合約

部署完智能合約後，剩下的就是調用它並用 Toncoin 兌換我們的代幣。為此，需要發送一個空消息體和一些 Toncoin。我們將使用在課程開始時準備的第二個錢包。

`ICO.go` 代碼：

```go
func main() {
    client := liteclient.NewConnectionPool()
    // 連接到測試網 lite 服務器
    err := client.AddConnectionsFromConfigUrl(context.Background(), "https://ton-blockchain.github.io/testnet-global.config.json")
    if err != nil {
        panic(err)
    }

    // 初始化 ton api lite 連接包裝器
    api := ton.NewAPIClient(client)

    // 賬戶的種子短語，你可以使用任何錢包或 wallet.NewSeed() 方法生成它們
    words := strings.Split("你的種子短語", " ")

    w, err := wallet.FromSeed(api, words, wallet.V3)
    if err != nil {
        log.Fatalln("FromSeed err:", err.Error())
        return
    }

    log.Println("錢包地址:", w.Address())

    block, err := api.CurrentMasterchainInfo(context.Background())
    if err != nil {
        log.Fatalln("CurrentMasterchainInfo err:", err.Error())
        return
    }

    balance, err := w.GetBalance(context.Background(), block)
    if err != nil {
        log.Fatalln("GetBalance err:", err.Error())
        return
    }

    if balance.NanoTON().Uint64() >= 100000000 {
        // ICO 地址 
        addr := address.MustParseAddr("EQD_yyEbNQeWbWfnOIowqNilB8wwbCg6nLxHDP3Rbey1eA72")

        fmt.Println("發送消息")
        err = w.Send(context.Background(), &wallet.Message{
            Mode: 3,
            InternalMessage: &tlb.InternalMessage{
                IHRDisabled: true,
                Bounce:      true,
                DstAddr:     addr,
                Amount:      tlb.MustFromTON("1"),
                Body:        cell.BeginCell().EndCell(),
            },
        }, true)
        if err != nil {
            fmt.Println(err)
        }

        // 更新鏈信息
        block, err = api.CurrentMasterchainInfo(context.Background())
        if err != nil {
            log.Fatalln("CurrentMasterchainInfo err:", err.Error())
            return
        }

        balance, err = w.GetBalance(context.Background(), block)
        if err != nil {
            log.Fatalln("GetBalance err:", err.Error())
            return
        }

        log.Println("交易已發送，餘額剩餘:", balance.TON())
        return
    }

    log.Println("餘額不足:", balance.TON())
}
```

如果成功，我們可以在 https://testnet.tonscan.org/ 上看到如下圖所示的畫面：

![tnscn](./img/tnscn.PNG)

我們的消息和返回消息通知。

### 檢查結果

讓我們從我們發送 Toncoin 的錢包中獲取代幣餘額。

`JettonBalance.go` 代碼：

```go
package main

import (
    "context"
    "github.com/xssnick/tonutils-go/address"
    _ "github.com/xssnick/tonutils-go/tlb"
    "github.com/xssnick/tonutils-go/ton/jetton"
    _ "github.com/xssnick/tonutils-go/ton/nft"
    _ "github.com/xssnick/tonutils-go/ton/wallet"
    "log"
    _ "strings"

    "github.com/xssnick/tonutils-go/liteclient"
    "github.com/xssnick/tonutils-go/ton"
)

func main() {
    client := liteclient.NewConnectionPool()
    // 連接到測試網 lite 服務器
    err := client.AddConnectionsFromConfigUrl(context.Background(), "https://ton-blockchain.github.io/testnet-global.config.json")
    if err != nil {
        panic(err)
    }

    // 初始化 ton api lite 連接包裝器
    api := ton.NewAPIClient(client)

    // jetton 合約地址
    contract := address.MustParseAddr("EQD_yyEbNQeWbWfnOIowqNilB8wwbCg6nLxHDP3Rbey1eA72")
    master := jetton.NewJettonMasterClient(api, contract)

    // 獲取賬戶的 jetton 錢包
    ownerAddr := address.MustParseAddr("EQAIz6DspthuIkUaBZaeH7THhe7LSOXmQImH2eT97KI2Dl4z")
    tokenWallet, err := master.GetJettonWallet(context.Background(), ownerAddr)
    if err != nil {
        log.Fatal(err)
    }

    tokenBalance, err := tokenWallet.GetBalance(context.Background())
    if err != nil {
        log.Fatal(err)
    }

    log.Println("代幣餘額:", tokenBalance.String())
}
```

如果成功，我們可以看到如下圖所示的畫面：

![cli](./img/wg.PNG)

> 由於有費用，再加上合約需要發送一條消息回來，因此獲得的代幣數量會少於我們發送的 Toncoin 數量。

## 練習

在 tonutils-go 庫中，有一些便捷的方法可以將代幣從一個錢包轉移到另一個錢包，嘗試使用它們將代幣從錢包 `w2` 轉移到 `w1`。

## 總結

代幣提供了許多機會，但也伴隨著相應的風險。