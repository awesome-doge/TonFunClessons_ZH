# 課程 3：代理智能合約
## 介紹

在這個教學中，我們將在 The Open Network 測試網中使用 FunC 語言編寫一個代理（將所有消息轉發給其擁有者）智能合約，並使用 [toncli](https://github.com/disintar/toncli) 部署到測試網，並在下一課中進行測試。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面。

還需要能夠使用 toncli 創建/部署項目，您可以在[第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md)中學習如何操作。

## 智能合約

我們將創建的智能合約應具備以下功能**：
- 將進入合約的所有消息轉發給擁有者；
- 在轉發時，發送者的地址應該首先出現，然後是消息正文；
- 附加到消息中的 Toncoin 數量應等於傳入消息的數量減去處理費用（計算和消息轉發費用）；
- 擁有者的地址存儲在智能合約的存儲中；
- 當擁有者向合約發送消息時，不應進行轉發。

** 我決定從 [FunC contest1](https://github.com/ton-blockchain/func-contest1) 任務中選取智能合約的點子，因為它們非常適合用來熟悉 TON 智能合約的開發。

## 外部方法

為了讓我們的代理接收消息，我們將使用外部方法 `recv_internal()`

```func
() recv_internal() {

}
```

### 外部方法參數
這裡有一個邏輯問題 - 如何理解函數應該有什麼參數，以便它可以接收 TON 網絡上的消息？

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當 TON 鏈之一的賬戶發生事件時，它會觸發一個交易。

每個交易由最多 5 個階段組成。詳情請參見[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是**計算階段**。具體來說，是初始化時“在棧上”的內容。對於正常消息觸發的交易，棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以 nanoTons 為單位）
- 傳入消息的餘額（以 nanoTons 為單位）
- 包含傳入消息的 Cell
- 傳入消息正文，類型為 slice
- 函數選擇器（對於 `recv_internal` 是 0）

最終，我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {

}
```

## 發送者地址

根據任務要求，我們需要獲取發送者的地址。我們將從 `in_msg_full` 中的消息 Cell 獲取地址。我們將代碼移到一個單獨的函數中。

```func
() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  slice sender_address = parse_sender_address(in_msg_full);
}
```

### 編寫函數

讓我們編寫 `parse_sender_address` 函數的代碼，從消息 Cell 中提取發送者地址並解析它：

```func
slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}
```

如您所見，該函數有一個 `inline` 修飾符，其代碼實際上會在每個調用函數的地方被替換。

為了獲取地址，我們需要使用 `begin_parse` 將 Cell 轉換為 slice：

```func
var cs = in_msg_full.begin_parse();
```

現在我們需要“減去”得到的 slice 來獲取地址。使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的 `load_uint` 函數從 slice 中加載一個無符號的 n 位整數，並“減去”標誌。

```func
var flags = cs~load_uint(4);
```

在這一課中，我們不會詳細討論標誌，但您可以在[3.1.7 節](https://ton-blockchain.github.io/docs/tblkch.pdf)中閱讀更多內容。

最後，使用 `load_msg_addr()` 加載唯一前綴為有效 MsgAddress 的地址。

```func
slice sender_address = cs~load_msg_addr();
return sender_address;
```

## 接收者地址

我們將從 [c4](https://ton-blockchain.github.io/docs/tvm.pdf) 中獲取地址，我們在之前的課程中已經討論過。

我們將使用：
`get_data` - 從 c4 寄存器獲取一個 Cell。
`begin_parse` - 將 Cell 轉換為 slice。
`load_msg_addr()` - 加載唯一前綴為有效 MsgAddress 的地址。

最終，我們得到以下函數：

```func
slice load_data () inline {
  var ds = get_data().begin_parse();
  return ds~load_msg_addr();
}
```

只需調用它即可：

```func
slice load_data () inline {
  var ds = get_data().begin_parse();
  return ds~load_msg_addr();
}

slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}

() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  slice sender_address = parse_sender_address(in_msg_full);
  slice owner_address = load_data();
}
```

## 檢查地址是否相等的條件

根據任務要求，如果合約擁有者訪問代理智能合約，代理不應該轉發消息。因此，有必要比較兩個地址。

### 比較函數

FunC 支持在彙編中定義函數（即 Fift）。這樣做的方法是 - 我們將函數定義為低級 TVM 原語。對於比較函數，它會看起來像這樣：

```func
int equal_slices (slice a, slice b) asm "SDEQ";
```

如您所見，使用了 `asm` 關鍵字。

您可以在 [TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 第 77 頁起查看可能的原語列表。

### 一元運算符

我們將在 `if` 中使用我們的 `equal_slices` 函數：

```func
() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  slice sender_address = parse_sender_address(in_msg_full);
  slice owner_address = load_data();

  if  equal_slices(sender_address, owner_address) {

   }
}
```

但該函數將檢查相等性，如何檢查不等性？這裡可以使用一元運算符 `~`，它是按位非。現在我們的代碼看起來像這樣：

```func
int equal_slices (slice a, slice b) asm "SDEQ";

slice load_data () inline {
  var ds = get_data().begin_parse();
  return ds~load_msg_addr();
}

slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}

() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  slice sender_address = parse_sender_address(in_msg_full);
  slice owner_address = load_data();

  if ~ equal_slices(sender_address, owner_address) {

   }
}
```

現在只剩下發送消息。

## 發送消息

現在我們需要根據任務要求填充條件運算符的主體，即發送傳入的消息。

### 消息結構

完整的消息結構可以在[這裡 - 消息佈局](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=message-layout)找到。但通常我們不需要控制每個字段，因此我們可以使用來自[示例](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=sending-messages)的簡短形式：

```func
 var msg = begin_cell()
    .store_uint(0x18, 6)
    .store_slice(addr)
    .store_coins(amount)
    .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
    .store_slice(message_body)
  .end_cell();
```

如您所見，構建消息使用了 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的函數。具體來說，是 Builder 原語的“封裝”函數（部分構建的 Cell，您可能還記得第一課）。考慮以下函數：

 `begin_cell()` - 創建一個 Builder 用於未來的 Cell
 `end_cell()` - 創建一個 Cell
 `store_uint` - 在 Builder 中存儲 uint
 `store_slice` - 在 Builder 中存儲 slice
 `store_coins` - 文檔中實際上是 `store_grams` - 用於存儲 TonCoins。詳情請參見[這裡](https://ton-blockchain.github.io/docs/#/func/stdlib?id=store_grams)。

另外，考慮 `store_ref`，這將在發送地址時需要。

 `store_ref` - 在 Builder 中存儲一個 Cell 引用
 現在我們有了所有必要的信息，讓我們組裝消息。

### 最後一步 - 傳入消息的主體

要在消息中發送 `recv_internal` 中接收到的消息主體，我們將組裝 Cell，並在消息中使用 `store_ref` 將其引用。

```func
  if ~ equal_slices(sender_address, owner_address) {
    cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();
   }
```

### 組裝消息

根據問題的條件，我們必須發送地址和消息主體。因此，將 `.store_slice(message_body)` 更改為 `.store_slice(sender_address)`，並將 `.store_ref(msg_body_cell)` 添加到 msg 變量中。我們得到：

```func
  if ~ equal_slices(sender_address, owner_address) {
    cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

    var msg = begin_cell()
          .store_uint(0x10, 6)
          .store_slice(owner_address)
          .store_grams(0)
          .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
          .store_slice(sender_address)
          .store_ref(msg_body_cell)
          .end_cell();
   }
```

只剩下發送我們的消息。

### 消息發送模式

要發送消息，使用 [標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message) 中的 `send_raw_message`。

我們已經組裝了 msg 變量，只剩下弄清 `mode`。每個模式在[文檔](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)中都有描述。讓我們看一個例子來更清楚地理解。

假設智能合約餘額為 100 coins，並且我們接收到一個包含 60 coins 的內部消息，並發送一個包含 10 coins 的消息，總費用為 3 coins。

 `mode = 0` - 餘額 (100+60-10 = 150 coins)，發送(10-3 = 7 coins)
 `mode = 1` - 餘額 (100+60-10-3 = 147 coins)，發送(10 coins)
 `mode = 64` - 餘額 (100-10 = 90 coins)，發送 (60+10-3 = 67 coins)
 `mode = 65` - 餘額 (100-10-3=87 coins)，發送 (60+10 = 70 coins)
 `mode = 128` - 餘額 (0 coins)，發送 (100+60-3 = 157 coins)
 
 上述的 mode 1 和 65 是 mode' = mode + 1。

由於問題要求附加到消息中的 Toncoin 數量必須等於傳入消息的數量減去處理相關的費用。`mode = 64` 配合 `.store_grams(0)` 將適合我們。該示例將導致以下結果：

假設智能合約餘額為 100 coins，並且我們接收到一個包含 60 coins 的內部消息，並發送一個包含 0 coins 的消息（因為 `.store_grams(0)`），總費用為 3 coins。

 `mode = 64` - 餘額 (100 = 100 coins)，發送 (60-3 = 57 coins)
 
 所以我們的條件語句將看起來像這樣：

```func
   if ~ equal_slices(sender_address, owner_address) {
    cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

    var msg = begin_cell()
          .store_uint(0x10, 6)
          .store_slice(owner_address)
          .store_grams(0)
          .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
          .store_slice(sender_address)
          .store_ref(msg_body_cell)
          .end_cell();
     send_raw_message(msg, 64);
   }
```

完整的智能合約代碼如下：

```func
int equal_slices (slice a, slice b) asm "SDEQ";

slice load_data () inline {
  var ds = get_data().begin_parse();
  return ds~load_msg_addr();
}

slice parse_sender_address (cell in_msg_full) inline {
  var cs = in_msg_full.begin_parse();
  var flags = cs~load_uint(4);
  slice sender_address = cs~load_msg_addr();
  return sender_address;
}

() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
  slice sender_address = parse_sender_address(in_msg_full);
  slice owner_address = load_data();

  if ~ equal_slices(sender_address, owner_address) {
    cell msg_body_cell = begin_cell().store_slice(in_msg_body).end_cell();

    var msg = begin_cell()
          .store_uint(0x10, 6)
          .store_slice(owner_address)
          .store_grams(0)
          .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
          .store_slice(sender_address)
          .store_ref(msg_body_cell)
          .end_cell();
     send_raw_message(msg, 64);
   }
}
```

## 結論

由於消息和我們的代理功能是“內部的”，因此無法通過 `toncli` “調用” 合約 - 它在 TON 內部處理消息。那麼如何正確開發這樣的合約呢？答案來自[測試](https://en.wikipedia.org/wiki/Test-driven_development)。我們將在下一課中編寫測試。
