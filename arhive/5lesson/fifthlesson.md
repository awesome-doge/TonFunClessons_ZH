# 課程 5：記住地址和識別操作
## 介紹

在這節課中，我們將編寫一個智能合約，該合約可以根據標誌在 The Open Network 測試網中的 FUNC 語言執行不同的操作，使用 [toncli](https://github.com/disintar/toncli) 部署到測試網，並在下一節課中進行測試。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面。

還需要能夠使用 toncli 創建/部署項目，您可以在[第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md)中學習如何操作。

## Op - 識別操作

在考慮本課程中將進行什麼樣的智能合約之前，我建議您研究[關於智能合約訊息正文的建議](https://ton-blockchain.github.io/docs/#/howto/smart-contract-guidelines?id=smart-contract-guidelines)。

為了在智能合約上創建類似於客戶端-伺服器的架構，建議在每個訊息（嚴格來說是訊息正文）的開頭添加一個 `op` 標誌，以識別智能合約應執行的操作。

在本教學中，我們將製作一個智能合約，根據 `op` 執行不同的操作。

## 智能合約

智能合約將記住由管理者設置的地址，並將其傳達給任何請求者，具體功能如下**：
- 當合約接收到來自管理者的 `op` 等於 1 的訊息，後跟一些 `query_id` 和 `MsgAddress` 時，應將接收到的地址存儲在存儲中。
- 當合約接收到來自任何地址的內部訊息，`op` 等於 2，後跟 `query_id` 時，應回覆發送者一個包含以下內容的訊息：
  - `op` 等於 3
  - 相同的 `query_id`
  - 管理者的地址
  - 自上次管理者請求以來記住的地址（如果尚未有管理者請求，則為空地址 `addr_none`）
  - 附加到訊息中的 TON 值減去處理費用。
- 當智能合約接收到任何其他訊息時，應拋出異常。

** 我決定從 [FunC contest1](https://github.com/ton-blockchain/func-contest1) 任務中選取智能合約的點子，因為它們非常適合用來熟悉 TON 智能合約的開發。

## 智能合約結構

### 外部方法

為了讓我們的代理接收訊息，我們將使用外部方法 `recv_internal()`

```func
() recv_internal()  {

}
```

### 外部方法參數

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當 TON 鏈之一的賬戶發生事件時，它會觸發一個交易。

每個交易由最多 5 個階段組成。詳情請參見[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是**計算階段**。具體來說，是初始化時“在棧上”的內容。對於正常訊息觸發的交易，棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以 nanoTons 為單位）
- 傳入訊息的餘額（以 nanoTons 為單位）
- 包含傳入訊息的 Cell
- 傳入訊息正文，類型為 slice
- 函數選擇器（對於 `recv_internal` 是 0）

最終，我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {

}
```

### 方法內部

在方法內部，我們將從函數參數中提取 `op`、`query_id` 和發送者地址 `sender_address`，然後使用條件運算符構建圍繞 `op` 的邏輯。

```func
() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
 ;; 取 op , query_id, 和發送者地址 sender_address

  if (op == 1) {
    ;; 這裡將保存從管理者那裡接收到的地址
  } else {
    if (op == 2) {
      ;; 發送訊息
    } else {
       ;; 這裡將拋出異常
    }
  }
}
```

## 輔助函數

讓我們思考一下函數中可以執行哪些功能？

- 比較地址，以便在 op 等於 1 時檢查請求是否來自管理者。
- 從寄存器 c4 卸載和加載管理者的地址和我們在合約中存儲的地址。
- 從傳入訊息中解析發送者的地址。

### 地址比較

FunC 支持在彙編中定義函數（即 Fift）。這樣做的方法是 - 我們將函數定義為低級 TVM 原語。對於比較函數，它會看起來像這樣：

```func
int equal_slices (slice a, slice b) asm "SDEQ";
```

如您所見，使用了 `asm` 關鍵字。

您可以在 [TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 第 77 頁起查看可能的原語列表。

### 從寄存器 c4 卸載地址

我們將地址存儲在 slices 中，但根據任務，我們必須存儲兩個地址，管理者的地址用於驗證，和管理者將發送的存儲地址。因此，slices 將以元組的形式返回。

為了從 c4 中“獲取”數據，我們需要來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的兩個函數。

即：
`get_data` - 從 c4 寄存器獲取一個 Cell。
`begin_parse` - 將 Cell 轉換為 slice。

讓我們將此值傳遞給變數 ds：

```func
var ds = get_data().begin_parse()
```

使用 `load_msg_addr()` 從訊息中加載地址 - 它從 slice 中加載唯一前綴是有效 MsgAddress 的地址。我們有兩個，所以需要“減去”兩次。

```func
return (ds~load_msg_addr(), ds~load_msg_addr());
```

最終，我們得到以下函數：

```func
(slice, slice) load_data () inline {
  var ds = get_data().begin_parse();
  return (ds~load_msg_addr(), ds~load_msg_addr());
}
```

### Inline

在之前的課程中，我們已經使用了 `inline` 修飾符，這實際上將代碼替換到每個調用函數的地方。在這節課中，我們將從實踐的角度來看為什麼這是必要的。

正如我們從[文檔](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)中知道的，交易費用包括：

- storage_fees - 占用區塊鏈空間的費用。
- in_fwd_fees - 導入訊息的費用（這是處理 `external` 訊息的情況）。
- computation_fees - 執行 TVM 指令的費用。
- action_fees - 與處理動作列表相關的費用（例如發送訊息）。
- out_fwd_fees - 導出傳出訊息的費用。

詳細資料請參見[這裡](https://ton-blockchain.github.io/docs/tvm.pdf)。
`inline` 修飾符本身可以節省 **computation_fee**。

默認情況下，當您有一個 FunC 函數時，它會獲得自己的標識符，存儲在一個單獨的 id->function 字典中，並且當您在程序中的某個地方調用它時，它會在字典中查找該函數，然後跳轉到該函數。

`inline` 修飾符將函數的主體直接放入父函數的代碼中。

因此，如果函數只使用一次或兩次，通常將函數聲明為 `inline` 更便宜，因為跳轉到鏈接比通過字典查找和跳轉便宜得多。

### 加載地址到寄存器 c4

當然，除了卸載，還需要加載。讓我們製作一個函數來保存管理者的地址和管理者將發送的地址：

```func
() save_data (slice manager_address, slice memorized_address) impure inline {
     
}
```

請注意，該函數具有 `impure` 修飾符。我們必須指定 `impure` 修飾符，如果該函數可能修改合約存儲。否則，FunC 編譯器可能會移除此函數調用。

為了“保存”來自 c4 的數據，我們需要來自 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的函數。

即：

`begin_cell()` - 創建一個 Builder 用於未來的 Cell。
`store_slice()` - 在 Builder 中存儲 Slice。
`end_cell()` - 創建一個 Cell。

`set_data()` - 將 Cell 寫入寄存器 c4。

組裝 Cell：

```func
begin_cell().store_slice(manager_address).store_slice(memorized_address).end_cell()
```

加載到 c4：

```func
set_data(begin_cell().store_slice(manager_address).store_slice(memorized_address).end_cell());
```

最終，我們得到以下函數：

```func
() save_data (slice manager_address, slice memorized_address) impure inline {
      set_data(begin_cell().store_slice(manager_address).store_slice(memorized_address).end_cell());
}
```

### 從傳入訊息中解析發送者地址

讓我們聲明一個函數，通過它我們可以從訊息 Cell 中獲取發送者地址。該函數將返回一個 slice，因為我們將使用 `load_msg_addr()` 獲取地址 - 它從 slice 中加載唯一前綴是有效 MsgAddress 並將其返回給 slice。

```func
slice parse_sender_address (cell in_msg_full) inline {

  return sender_address;
}
```

現在，使用我們已經熟悉的 `begin_parse`，我們將 Cell 轉換為 slice。

```func
slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();

  return sender_address;
}
```

我們開始使用 `load_uint`（[FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib)中的一個函數）讀取 Cell，它從 slice 中加載一個無符號的 n 位整數。

在這節課中，我們不會詳細討論標誌，但您可以在[3.1.7 節](https://ton-blockchain.github.io/docs/tblkch.pdf)中閱讀更多內容。

最後，我們獲取地址。

最終，我們得到以下函數：

```func
slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}
```

## 小結

目前我們已有了輔助函數和智能合約的主函數 `recv_internal()` 的主體。

```func
int equal_slices (slice a, slice b) asm "SDEQ";

(slice, slice) load_data () inline {
  var ds = get_data().begin_parse();
  return (ds~load_msg_addr(), ds~load_msg_addr());
}

() save_data (slice manager_address, slice memorized_address) impure inline {
  set_data(begin_cell().store_slice(manager_address).store_slice(memorized_address).end_cell());
}

slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}

() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  ;; 取 op , query_id, 和發送者地址 sender_address

  if (op == 1) {
    ;; 這裡將保存從管理者那裡接收到的地址
  } else {
    if (op == 2) {
      ;; 發送訊息
    } else {
       ;; 這裡將拋出異常
    }
  }
}
```

剩下的就是填充 `recv_internal()`。

## 填充外部方法

### 取 op , query_id, 和 sender_address

從訊息正文中依次減去 op 和 query_id。根據[建議](https://ton-blockchain.github.io/docs/#/howto/smart-contract-guidelines?id=smart-contract-guidelines)，這些是 32 和 64 位的值。

並且使用我們上面編寫的 `parse_sender_address()` 函數獲取發送者地址。

```func
() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  int op = in_msg_body~load_int(32);
  int query_id = in_msg_body~load_uint(64);
  var sender_address = parse_sender_address(in_msg_full);
     
  if (op == 1) {
    ;; 這裡將保存從管理者那裡接收到的地址
  } else {
    if (op == 2) {
      ;; 發送訊息
    } else {
       ;; 這裡將拋出異常
    }
  }
}
```

### 標誌 op == 1

根據任務要求，當標誌為 1 時，我們必須接收管理者地址和保存的地址，檢查發送者地址是否等於管理者地址（只有管理者可以更改地址），並保存存儲在訊息正文中的新地址。

使用我們之前編寫的 `load_data()` 函數從 c4 加載管理者地址 `manager_address` 和保存的地址 `memorized_address)`。

```func
(slice manager_address, slice memorized_address) = load_data();
```

使用 `equal_slices` 函數和一元 `~` 運算符（按位非）檢查地址是否相等，如果地址不相等則拋出異常。

```func
(slice manager_address, slice memorized_address) = load_data();
throw_if(1001, ~ equal_slices(manager_address, sender_address));
```

使用我們已經熟悉的 `load_msg_addr()` 獲取地址，並使用我們之前編寫的 `save_data()` 函數保存地址。

```func
(slice manager_address, slice memorized_address) = load_data();
throw_if(1001, ~ equal_slices(manager_address, sender_address));
slice new_memorized_address = in_msg_body~load_msg_addr();
save_data(manager_address, new_memorized_address);
```

### 標誌 op == 2

根據任務要求，當標誌為 2 時，我們必須發送一個包含以下內容的訊息：
- `op` 等於 3
- 相同的 `query_id`
- 管理者的地址
- 自上次管理者請求以來記住的地址（如果尚未有管理者請求，則為空地址 `addr_none`）
- 附加到訊息中的 TON 值減去處理費用。

在發送訊息之前，讓我們加載存儲在合約中的地址。

```func
(slice manager_address, slice memorized_address) = load_data();
```

完整的訊息結構可以在[這裡 - 訊息佈局](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=message-layout)找到。但通常我們不需要控制每個字段，因此我們可以使用來自[示例](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=sending-messages)的簡短形式：

```func
 var msg = begin_cell()
    .store_uint(0x18, 6)
    .store_slice(addr)
    .store_coins(amount)
    .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
    .store_slice(message_body)
  .end_cell();
```

完整的訊息分析在[第三課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/3lesson/thirdlesson.md)中有詳細描述。

根據條件發送訊息：

```func
(slice manager_address, slice memorized_address) = load_data();
var msg = begin_cell()
        .store_uint(0x10, 6)
        .store_slice(sender_address)
        .store_grams(0)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_uint(3, 32)
        .store_uint(query_id, 64)
        .store_slice(manager_address)
        .store_slice(memorized_address)
      .end_cell();
send_raw_message(msg, 64);
```

### 異常

這裡只需使用來自[內建 FunC 模組](https://ton-blockchain.github.io/docs/#/func/builtins?id=throwing-exceptions)的 `throw`。

```func
throw(3);
```

## 完整的智能合約代碼

```func
int equal_slices (slice a, slice b) asm "SDEQ";

(slice, slice) load_data () inline {
  var ds = get_data().begin_parse();
  return (ds~load_msg_addr(), ds~load_msg_addr());
}

() save_data (slice manager_address, slice memorized_address) impure inline {
  set_data(begin_cell().store_slice(manager_address).store_slice(memorized_address).end_cell());
}

slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}

() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  int op = in_msg_body~load_int(32);
  int query_id = in_msg_body~load_uint(64);
  var sender_address = parse_sender_address(in_msg_full);

  if (op == 1) {
    (slice manager_address, slice memorized_address) = load_data();
    throw_if(1001, ~ equal_slices(manager_address, sender_address));
    slice new_memorized_address = in_msg_body~load_msg_addr();
    save_data(manager_address, new_memorized_address);
  } else {
    if (op == 2) {
      (slice manager_address, slice memorized_address) = load_data();
      var msg = begin_cell()
              .store_uint(0x10, 6)
              .store_slice(sender_address)
              .store_grams(0)
              .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
              .store_uint(3, 32)
              .store_uint(query_id, 64)
              .store_slice(manager_address)
              .store_slice(memorized_address)
            .end_cell();
      send_raw_message(msg, 64);
    } else {
      throw(3); 
    }
  }
}
```

## 結論

我們將在下一課中編寫測試。此外，我特別感謝那些捐贈 TON 支持該項目的人，這非常激勵人心並有助於更快地發布課程。