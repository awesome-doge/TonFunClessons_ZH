# 使用 GO 創建錢包並部署合約

## 簡介

在 tondev 聊天中，經常會有關於如何使用流行的編程語言與 TON 進行交互的問題，特別是關於與 NFT 集合和合約交互的問題。因此，我決定為 [ton_learn](https://t.me/ton_learn) 製作兩節課，讓讀者能夠輕鬆地在 TON 區塊鏈上操作智能合約。

任務目標：
- 在本課中，我們將創建一個錢包，並了解如何部署和操作第一課中的合約。
- 在下一課中，我們將部署 NFT 集合，並調用 Get 方法。

為了使用腳本與 TON 進行交互，我們將使用 GO 庫 [tonutils-go](https://github.com/xssnick/tonutils-go)。這個庫在高層級和低層級之間取得了很好的平衡，既可以編寫簡單的腳本，又不會限制我們與 TON 區塊鏈交互的各種可能性。

即使您不熟悉 GO，我相信這節課和腳本對您來說也是清晰易懂的。不過，為了保險起見，在課程的最後還有一些材料鏈接，可以讓您快速熟悉 GO。

> 此外，該庫有很好的文檔和示例。

## 創建錢包

我們需要一個錢包來在 TON 網絡內發送消息（即接收 `recv_internal()` 的消息）。本質上，錢包是一個能夠接收外部消息（即 `recv_external()` 的消息）並發送內部消息的智能合約。因此，在進入部署智能合約之前，我們首先創建一個錢包。

### 連接到網絡

TON 網絡中的錢包是一個智能合約，為了將智能合約部署到測試或主網絡，我們需要連接到網絡，這需要網絡的配置文件：
- [testnet 配置](https://ton-blockchain.github.io/testnet-global.config.json)
- [mainnet 配置](https://ton-blockchain.github.io/global.config.json)（主網）

我們將通過 light servers 與網絡交互。

> 輕客戶端（lite-client）是一種軟件，通過連接到完整節點來與區塊鏈交互。它們幫助用戶在不需要同步整個區塊鏈的情況下訪問和交互區塊鏈。

所以讓我們連接到網絡：

```go
client := liteclient.NewConnectionPool()

configUrl := "https://ton-blockchain.github.io/testnet-global.config.json"

err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
if err != nil {
    panic(err)
}
api := ton.NewAPIClient(client)
```

我們獲取了 light server API。

> 如果您查看配置文件，您會看到其中有一些 light servers，庫會選擇其中一個進行成功連接。

### 种子短語

要生成錢包，我們需要一對公鑰/私鑰（通過種子短語獲取）和對應於可用錢包版本之一的 [InitialAccountWallet](https://github.com/ton-blockchain/ton/blob/master/tl/generate/scheme/tonlib_api.tl#L60) 結構。

> 種子短語是一組用於生成密鑰的單詞。

讓我們使用 `wallet.NewSeed()` 生成種子短語並打印出來，以便我們可以復製並在未來使用該錢包。

```go
seed := wallet.NewSeed()
fmt.Println("Seed phrase:")
fmt.Println(seed)
```

這個短語應該被保存以便在未來使用錢包。

我們生成錢包並顯示地址。

```go
w, err := wallet.FromSeed(api, seed, wallet.V3)
if err != nil {
    log.Fatalln("FromSeed err:", err.Error())
    return
}

fmt.Println(w.Address())
```

您可以在[這裡](https://github.com/toncenter/tonweb/blob/master/src/contract/wallet/WalletSources.md) 閱讀更多有關不同錢包版本的信息。

### “激活”錢包

根據 [文檔](https://ton-blockchain.github.io/docs/#/payment-processing/overview?id=deploying-wallet)，需要向接收地址發送 Toncoin。測試網有一個機器人 https://t.me/testgiver_ton_bot 可用於此。在主網中，您可以參考官方 [頁面](https://ton-blockchain.github.io/buy-toncoin)。

### 獲取餘額

我們的錢包已準備就緒，為了獲取餘額，我們需要獲取當前網絡的信息（即當前區塊）。

```go
block, err := api.CurrentMasterchainInfo(context.Background())
if err != nil {
    log.Fatalln("CurrentMasterchainInfo err:", err.Error())
    return
}
```

然後從區塊中獲取餘額：

```go
balance, err := w.GetBalance(context.Background(), block)
if err != nil {
    log.Fatalln("GetBalance err:", err.Error())
    return
}

fmt.Println(balance)
```

最終 `createwallet.go` 的代碼：

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

    seed := wallet.NewSeed()
    fmt.Println("Seed phrase:")
    fmt.Println(seed)

    w, err := wallet.FromSeed(api, seed, wallet.V3)
    if err != nil {
        log.Fatalln("FromSeed err:", err.Error())
        return
    }

    fmt.Println(w.Address())

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

    fmt.Println(balance)
}
```

在繼續之前，我們將根據種子短語生成錢包的功能移到一個單獨的函數中。

### 錢包函數

由於我們已經有了種子短語，因此不再需要生成它，只需組裝錢包即可。

```go
func getWallet(api *ton.APIClient) *wallet.Wallet {
    words := strings.Split("write your seed phrase here", " ")
    w, err := wallet.FromSeed(api, words, wallet.V3)
    if err != nil {
        panic(err)
    }
    return w
}
```

使用類似函數生成錢包的示例在單獨的文件 `walletfunc.go` 中。

## 部署智能合約

### hexBoc 智能合約

現在我們有了一個擁有 Toncoin 餘額的錢包，我們可以部署智能合約。在 `tonutils-go` 庫中，您可以以 hexBoc 形式部署智能合約。Boc 是智能合約的序列化形式（bag-of-cells）。

將智能合約轉換為類似形式的最簡單方法是使用 fift 腳本。讓我們從第一個智能合約中獲取 fift 代碼並編寫一個腳本，將其轉換為 hexBoc。

```fift
"Asm.fif" include
// automatically generated from `C:\Users\7272~1\AppData\Local\toncli\toncli\func-libs\stdlib-tests.func` `C:\Users\7272~1\Documents\chain\firsttest\wallet\func\code.func` 
PROGRAM{
  DECLPROC recv_internal
  128253 DECLMETHOD get_total
  recv_internal PROC:<{
    //  in_msg_body
    DUP //  in_msg_body in_msg_body
    SBITS //  in_msg_body _2
    32 LESSINT //  in_msg_body _4
    35 THROWIF
    32 LDU //  _24 _23
    DROP //  n
    c4 PUSH //  n _11
    CTOS //  n ds
    64 LDU //  n _26 _25
    DROP //  n total
    SWAP //  total n
    ADD //  total
    NEWC //  total _18
    64 STU //  _20
    ENDC //  _21
    c4 POP
  }>
  get_total PROC:<{
    // 
    c4 PUSH //  _1
    CTOS //  ds
    64 LDU //  _8 _7
    DROP //  total
  }>
}END>c
```

> 如果您學習了第一課，那麼 Fift 合約代碼位於 fift 文件夾中

現在編寫腳本將代碼轉換為 hexBOC 格式：

```fift
#!/usr/bin/fift -s
"TonUtil.fif" include
"Asm.fif" include

."first contract:" cr

"first.fif" include
2 boc+>B dup Bx. cr cr
```

我不會詳細介紹 fift，這超出了本課範圍，我只會指出：
- boc+>B - 序列化為 boc 格式
- cr - 顯示值為字符串

> 您可以使用熟悉的 toncli 運行腳本，即 `toncli fift run`，或者按照[這裡](https://ton-blockchain.github.io/docs/#/compile?id=fift)的描述運行。

示例腳本在 `print-hex.fif` 文件中。

最終我們會得到：

`B5EE9C72410104010038000114FF00F4A413F4BCF2C80B0102016202030032D020D749C120F263D31F30ED44D0D33F3001A0C8CB3FC9ED540011A1E9FBDA89A1A67E693`

### 部署合約

我們使用擁有錢包的 `walletfunc.go` 作為部署合約腳本的基礎。首先添加 `getContractCode()` 函數，將之前獲取的 hexBOC 轉換為字節：

```go
func getContractCode() *cell.Cell {
    var hexBOC = "B5EE9C72410104010038000114FF00F4A413F4BCF2C80B0102016202030032D020D749C120F263D31F30ED44D0D33F3001A0C8CB3FC9ED540011A1E9FBDA89A1A67E61A6614973"
    codeCellBytes, _ := hex.DecodeString(hexBOC)

    codeCell, err := cell.FromBOC(codeCellBytes)
    if err != nil {
        panic(err)
    }

    return codeCell
}
```

### 智能合約部署過程

要部署智能合約，我們需要形成一個 `StateInit`。`StateInit` 是智能合約代碼和數據的組合。智能合約數據是我們想放入 `c4` 寄存器中的內容，通常是智能合約所有者的地址，以便管理它。您可以在第9和第10課中看到示例，NFT 集合或 Jetton 的所有者存儲在 `c4` 中。在我們的示例中，我們可以放入0或任何數字，主要是64位，確保其為64位，以便合約邏輯正確運行。我們為數據創建一個單獨的函數：

```go
func getContractData() *cell.Cell {
    data := cell.BeginCell().MustStoreUInt(2, 64).EndCell()

    return data
}
```

他們的 `StateInit` 通過哈希計算智能合約的地址。

需要向接收的地址發送消息，並且不要忘記需要少量的 TON，因為智能合約必須有正餘額才能支付數據存儲和處理的費用。

您還需要準備一些消息體，但根據情況可以為空。

在 `tonutils-go` 中，所有這些邏輯都在 `DeployContract` 函數內，調用它在我們的情況下如下所示：

```go
msgBody := cell.BeginCell().MustStoreUInt(0, 64).EndCell()

fmt.Println("Deploying NFT collection contract to net...")
addr, err := w.DeployContract(context.Background(), tlb.MustFromTON("0.02"),
    msgBody, getContractCode(), getContractData(), true)
if err != nil {
    panic(err)
}

fmt.Println("Deployed contract addr:", addr.String())
```

`true` 參數指定是否“等待”消息已發送的確認。

> 需要注意的是，由於我們獲取地址是哈希值，因此無法使用相同數據兩次部署相同的合約，消息會直接發送到現有合約。

最終 `deploycontract.go` 的代碼：

```go
package main

import (
    "context"
    "log"
    "fmt"
    "encoding/hex"
    "strings"

    "github.com/xssnick/tonutils-go/liteclient"
    "github.com/xssnick/tonutils-go/ton"
    "github.com/xssnick/tonutils-go/ton/wallet"
    "github.com/xssnick/tonutils-go/tlb"
    "github.com/xssnick/tonutils-go/tvm/cell"
)

func main() {
    client := liteclient.NewConnectionPool()

    configUrl := "https://ton-blockchain.github.io/testnet-global.config.json"

    err := client.AddConnectionsFromConfigUrl(context.Background(), configUrl)
    if err != nil {
        panic(err)
    }
    api := ton.NewAPIClient(client)

    w := getWallet(api)

    fmt.Println(w.Address())

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

    fmt.Println(balance)

    msgBody := cell.BeginCell().MustStoreUInt(0, 64).EndCell()

    fmt.Println("Deploying NFT collection contract to net...")
    addr, err := w.DeployContract(context.Background(), tlb.MustFromTON("0.02"),
        msgBody, getContractCode(), getContractData(), true)
    if err != nil {
        panic(err)
    }

    fmt.Println("Deployed contract addr:", addr.String())
}

func getWallet(api *ton.APIClient) *wallet.Wallet {
    words := strings.Split("write your seed phrase here", " ")
    w, err := wallet.FromSeed(api, words, wallet.V3)
    if err != nil {
        panic(err)
    }
    return w
}

func getContractCode() *cell.Cell {
    var hexBOC = "B5EE9C72410104010038000114FF00F4A413F4BCF2C80B0102016202030032D020D749C120F263D31F30ED44D0D33F3001A0C8CB3FC9ED540011A1E9FBDA89A1A67E61A6614973"
    codeCellBytes, _ := hex.DecodeString(hexBOC)

    codeCell, err := cell.FromBOC(codeCellBytes)
    if err != nil {
        panic(err)
    }

    return codeCell
}

func getContractData() *cell.Cell {
    data := cell.BeginCell().MustStoreUInt(2, 64).EndCell()

    return data
}
```

## 發送消息

現在我們來測試我們的智能合約，即發送消息，合約應該將其添加到寄存器 `c4` 中的數字並保存結果。我們使用錢包的 `walletfunc.go` 作為基礎，並添加發送消息的代碼：

```go
fmt.Println("Let's send message")
err = w.Send(context.Background(), &wallet.Message{
    Mode: 3,
    InternalMessage: &tlb.InternalMessage{
        IHRDisabled: true,
        Bounce:      true,
        DstAddr:     address.MustParseAddr("your contract address"),
        Amount:      tlb.MustFromTON("0.05"),
        Body:        cell.BeginCell().MustStoreUInt(11, 32).EndCell(),
    },
}, true)
if err != nil {
    fmt.Println(err)
}
```

消息模式與之前一樣）它在第3課中有詳細討論。我們從我們的錢包發送消息。

## 調用 GET 方法

現在需要檢查智能合約中的值是否已被加總。為此，`tonutils-go` 中有 `RunGetMethod()` 方法，您需要傳遞當前區塊、智能合約地址、方法和參數。

```go
fmt.Println("Get Method")
addr := address.MustParseAddr("your contract address")

// 調用 get 方法
res, err := api.RunGetMethod(context.Background(), block, addr, "get_total")
if err != nil {
    // 如果合約退出代碼不為0，這也將被視為錯誤
    panic(err)
}

fmt.Println(res)
```

需要注意的是，如果發送消息和調用 Get 合約方法連續進行，數據可能沒有時間在區塊鏈中更新，您可能會獲取到舊值。因此，在發送消息和 Get 方法之間，我們需要獲取新區塊。並且[time.Sleep](https://www.geeksforgeeks.org/time-sleep-function-in-golang-with-examples/)。或者我們可以註釋掉消息發送，單獨調用 get 方法）。

> 在 TON 中，區塊每5秒更新一次。

示例代碼在 `sendandget.go` 文件中。

## 結論

在下一課中，我們將部署 NFT 集合。此外，我還想指出，tonutil-go 在他們的頁面上有捐贈地址。

## GO 附錄

我收集了一些鏈接，這些鏈接將加快您對本課程腳本的理解。

### 安裝 GO

https://go.dev/

### GO 的 Hello world

https://gobyexample.com/hello-world

### 15分鐘學習語法

https://learnxinyminutes.com/docs/go/

### 錯誤：無需模塊

https://codesource.io/how-to-install-github-packages-in-golang/

### 什麼是 context

https://gobyexample.com/context