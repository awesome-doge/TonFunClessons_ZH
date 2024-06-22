# 課程 1：簡單的 FunC 智能合約
## 介紹

在這個教學中，我們將在 The Open Network 測試網上使用 FunC 語言編寫您的第一個智能合約，並使用 [toncli](https://github.com/disintar/toncli) 部署* 到測試網，並使用 Fift 語言進行測試。

> *部署是將智能合約傳送到區塊鏈網絡的過程。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面。

## 智能合約

我們將創建的智能合約應具備以下功能：
- 在其數據中存儲一個整數 *total* - 一個64位無符號數；
- 當接收到內部傳入消息時，合約必須從消息正文中提取一個32位無符號整數，將其加到 *total* 並將結果存儲在合約數據中；
- 智能合約必須提供一個 *get total* 方法，允許返回 *total* 的值；
- 如果傳入消息的正文小於32位，則合約必須拋出異常。

## 使用 toncli 創建項目

在控制台中運行以下命令：

```shell
toncli start wallet
cd wallet
```

Toncli 創建了一個簡單的錢包項目，您可以看到其中有4個文件夾：
- build；
- func；
- fift；
- test；

在這個階段，我們關注的是 `func` 和 `fift` 文件夾，在這裡我們將分別用 FunC 和 Fift 編寫代碼。

### 什麼是 FunC 和 Fift

FunC 是用於編寫 TON 智能合約的高級語言。FunC 程序被編譯為 Fift 彙編代碼，生成對應的 TON 虛擬機（TVM）字節碼（有關 TVM 的更多信息[在此](https://ton-blockchain.github.io/docs/tvm.pdf)）。該字節碼可以用於在區塊鏈上創建智能合約，或在本地 TVM 實例中運行。

有關 FunC 的更多信息可以在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/)找到。

### 準備代碼文件

進入 `func` 文件夾：

```shell
cd func
```

打開 `code.func` 文件，您將看到錢包智能合約，刪除所有代碼，準備開始編寫我們的第一個智能合約。

## 外部方法

TON 網絡上的智能合約有兩個保留方法可以訪問。

首先，`recv_external()` 函數在來自外部世界的請求到達合約時執行，即不來自 TON 內部，例如，我們自己生成消息並通過 lite-client 發送時（有關安裝 [lite-client](https://ton-blockchain.github.io/docs/#/compile?id=lite-client) 的信息）。
其次，`recv_internal()` 函數在 TON 內部執行，例如，當任何合約調用我們的合約時。

> 輕客戶端（lite-client）是連接到完整節點以與區塊鏈交互的軟件。它們幫助用戶訪問和交互區塊鏈，而無需同步整個區塊鏈。

`recv_internal()` 符合我們的需求。

在 `code.fc` 文件中編寫：

```func
() recv_internal(slice in_msg_body) impure {
  ;; 這裡將是代碼
}
```

> FunC 有單行註釋，以 `;;`（雙 `;`）開始。

我們將 `in_msg_body` slice 傳遞給函數並使用 `impure` 關鍵字。

`impure` 是一個關鍵字，表示函數會更改智能合約數據。

例如，如果函數可以修改合約存儲、發送消息或在某些數據無效時拋出異常，則必須指定 `impure`。

重要：如果沒有指定 `impure` 且函數調用的結果未被使用，則 FunC 編譯器可能會刪除該函數調用。

但是為了理解 slice 是什麼，我們來談談 TON 網絡智能合約中的類型。

### FunC 中的 Cell、Slice、Builder、整數類型

在我們的簡單智能合約中，我們將使用以下四種類型：

- Cell - TVM 單元，由 1023 位數據和最多 4 個引用組成；
- Slice - TVM 單元的部分表示，用於解析單元中的數據；
- Builder - 部分構建的單元，包含最多 1023 位數據和最多四個鏈接；可以用於創建新單元；
- Integer - 簽名的 257 位整數。

有關 FunC 類型的更多信息：
[簡要介紹](https://ton-blockchain.github.io/docs/#/smart-contracts/)、
[詳細描述在 2.1 節](https://ton-blockchain.github.io/docs/fiftbase.pdf)

簡單來說，Cell 是密封的單元，Slice 是可以讀取的單元，而 Builder 是用來組裝單元的。

## 將結果 Slice 轉換為整數

要將結果 Slice 轉換為整數，請添加以下代碼：
`int n = in_msg_body~load_uint(32);`

`recv_internal()` 函數現在看起來像這樣：

```func
() recv_internal(slice in_msg_body) impure {
    int n = in_msg_body~load_uint(32);
}
```

`load_uint` 函數來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib)，它從 Slice 加載一個無符號的 n 位整數。

## 持久化智能合約數據

為了將結果變量加到 `total` 並將值存儲在智能合約中，我們來看看 TON 中實現持久數據/存儲的方式。

> 注意：不要與 TON Storage 混淆，此處的存儲是一個方便的比喻。

TVM 虛擬機是基於棧的，所以將數據存儲在合約中使用特定的寄存器，而不是將數據存儲在棧頂是個好習慣。

要存儲永久數據，使用寄存器 c4，數據類型為 Cell。

有關寄存器的更多信息可以在 [c4](https://ton-blockchain.github.io/docs/tvm.pdf) 的 1.3 節找到。

### 從 c4 獲取數據

為了從 c4 獲取數據，我們需要 FunC 標準庫中的兩個函數。

分別是：
`get_data` - 從 c4 寄存器獲取一個 Cell。
`begin_parse` - 將 Cell 轉換為 Slice。

將該值傳遞給 Slice `ds`：

```func
slice ds = get_data().begin_parse();
```

同時我們將該 Slice 轉換為 64 位整數以進行加法運算（使用我們已熟悉的 `load_uint` 函數）。

```func
int total = ds~load_uint(64);
```

現在我們的函數如下：

```func
() recv_internal(slice in_msg_body) impure {
    int n = in_msg_body~load_uint(32);

    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);
}
```

### 累加

使用二進制加法運算 `+` 和賦值 `=` 來進行加法運算：

```func
() recv_internal(slice in_msg_body) impure {
    int n = in_msg_body~load_uint(32);

    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    total += n;
}
```

### 保存值

為了保持一個常數值，我們需要做四件事：

- 創建一個 Builder 用於未來的 Cell；
- 將值寫入其中；
- 從 Builder 創建 Cell；
- 將結果 Cell 寫入寄存器。

我們將再次使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的函數：

```func
set_data(begin_cell().store_uint(total, 64).end_cell());
```

`begin_cell()` - 創建 Builder 用於未來的 Cell。
`store_uint()` - 寫入 `total` 的值。
`end_cell()` - 創建 Cell。
`set_data()` - 將 Cell 寫入寄存器 c4。

結果：

```func
() recv_internal(slice in_msg_body) impure {
    int n = in_msg_body~load_uint(32);

    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    total += n;

    set_data(begin_cell().store_uint(total, 64).end_cell());
}
```

## 拋出異常

我們內部函數需要做的最後一件事是，如果接收的變量不是32位，則拋出異常。

我們將使用 [內置](https://ton-blockchain.github.io/docs/#/func/builtins) 異常。

異常可以由條件原語 `throw_if` 和 `throw_unless` 以及無條件的 `throw` 拋出。

我們使用 `throw_if` 並傳遞任何錯誤代碼。為了獲取位數，我們使用 `slice_bits()`。

```func
throw_if(35, in_msg_body.slice_bits() < 32);
```

順便說一下，在 TON TVM 虛擬機中，有標準異常代碼，我們在測試中確實需要它們。您可以在[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)查看。

插入函數的開頭：

```func
() recv_internal(slice in_msg_body) impure {
    throw_if(35, in_msg_body.slice_bits() < 32);

    int n = in_msg_body~load_uint(32);

    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    total += n;

    set_data(begin_cell().store_uint(total, 64).end_cell());
}
```

## 寫一個 Get 函數

在 FunC 中，任何函數都遵循以下模式：

`[<forall declarator>] <return_type> <function_name>(<comma_separated_function_args>) <specifiers>`

讓我們編寫一個返回整數並具有 method_id 規範的 `get_total()` 函數（稍後會詳細介紹）。

```func
int get_total() method_id {
    ;; 這裡將是代碼
}
```

### Method_id

method_id 規範允許您從 lite-client 或 ton-explorer 調用 GET 函數。

簡單來說，所有的函數都有一個數字標識符，GET 方法的標識符是它們名稱的 crc16 哈希值。

### 從 c4 獲取數據

為了讓函數返回存儲在合約中的 `total`，我們需要從寄存器中獲取數據，我們已經做到過這一點：

```func
int get_total() method_id {
    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    return total;
}
```

## 我們的智能合約的所有代碼

```func
() recv_internal(slice in_msg_body) impure {
    throw_if(35, in_msg_body.slice_bits() < 32);

    int n = in_msg_body~load_uint(32);

    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    total += n;

    set_data(begin_cell().store_uint(total, 64).end_cell());
}
 
int get_total() method_id {
    slice ds = get_data().begin_parse();
    int total = ds~load_uint(64);

    return total;
}
```

## 部署合約到測試網

要部署到測試網，我們將使用命令行界面 [toncli](https://github.com/disintar/toncli/)

```shell
toncli deploy -n testnet
```

### 如果提示 TON 不足該怎麼辦？

您需要從測試水龍頭獲取 TON，這個機器人是 @testgiver_ton_bot。

您可以直接在控制台中看到錢包地址，部署命令後，toncli 會在 INFO 行的右側顯示它：Found existing deploy-wallet

要檢查 TON 是否到達您的測試網錢包，可以使用這個瀏覽器：https://testnet.tonscan.org/

> 重要：這僅是測試網。

## 測試合約

### 調用 recv_internal()

要調用 `recv_internal()`，您需要在 TON 網絡中發送一個消息。使用 [toncli send](https://github.com/disintar/toncli/blob/master/docs/advanced/send_fift_internal.md)

讓我們編寫一個小的 Fift 腳本，將 32 位消息發送到我們的合約。

### 消息腳本

為此，在 `fift` 文件夾中創建一個 `try.fif` 文件，並在其中寫入以下代碼：

```fift
"Asm.fif" include

<b
    11 32 u, // 數字
b>
```

`"Asm.fif" include` - 用於將消息編譯為字節碼。

現在分析消息：

`<b b>` - 創建 Builder 單元，更多詳細信息請參見[5.2 節](https://ton-blockchain.github.io/docs/fiftbase.pdf)。

`10 32 u` - 放置 32 位無符號整數 10。

` // 數字` - 單行註釋。

### 部署生成的消息

在命令行中：

```shell
toncli send -n testnet -a 0.03 --address "您的合約地址" --body ./fift/try.fif
```

現在測試 GET 函數：

```shell
toncli get get_total
```

您應該會得到以下結果：

![toncli get send](./img/tonclisendget.png)

## 恭喜您完成了

### 練習

如您所見，我們尚未測試異常，請修改消息，使智能合約拋出異常。