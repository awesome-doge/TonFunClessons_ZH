# 課程 4：為代理智能合約編寫 FunC 測試
## 介紹

在這個教學中，我們將為在第三課中使用 FunC 語言編寫的智能合約在 The Open Network 測試網上編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 執行這些測試。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面，並完成[第三課](https://github.com/romanovichim/TonFunClessons_ru/blob/main/3lesson/thirdlesson.md)。

## 重要提示

以下內容描述了舊版本的測試。目前可用於 func/fift 開發版本的新 toncli 測試，相關說明[在此](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新測試的課程[在此](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/11lesson/11lesson.md)。新測試的發布並不意味著舊測試的課程毫無意義 - 它們能很好地傳達邏輯，因此順利完成課程也是有意義的。還要注意，使用 `toncli run_tests` 時，可以使用 `--old` 標誌來運行舊測試。

## 代理智能合約的測試

對於我們的代理智能合約，我們將編寫以下測試：

- test_same_addr() 測試當擁有者向合約發送消息時，不應進行轉發
- test_example_data() 測試其餘條件[第三課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/3lesson/thirdlesson.md)

## toncli 下的 FunC 測試結構

讓我提醒您，對於每個 toncli 下的 FunC 測試，您需要編寫兩個函數。第一個函數將確定我們將發送給第二個函數進行測試的數據（在 TON 中，稱之為狀態可能更準確，但我希望數據是一個更易理解的比喻）。

每個測試函數必須指定一個 method_id。method_id 測試函數應從 0 開始。

### 數據函數

數據函數不接受參數，但必須返回：
- 函數選擇器 - 被測合約中調用函數的 id；
- 元組 - 我們將傳遞給執行測試的函數的值（棧）；
- c4 單元 - 控制寄存器 c4 中的“永久數據”；
- c7 元組 - 控制寄存器 c7 中的“臨時數據”；
- gas 限制整數 - gas 限制（為了解 gas 的概念，我建議先閱讀 [Ethereum](https://ethereum.org/en/developers/docs/gas/) 的相關資料）。

> 簡單來說，gas 測量的是在網絡上執行某些操作所需的計算努力量。詳細資料請參見[這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/fees)。更詳細的資料在[附錄 A](https://ton-blockchain.github.io/docs/tvm.pdf)。

> 棧 - 根據 LIFO 原則（英語 last in - first out，“後進先出”）組織的元素列表。棧的詳細說明可參見 [Wikipedia](https://zh.wikipedia.org/wiki/堆疊)。

更多關於 c4 和 c7 寄存器的信息請參見[這裡](https://ton-blockchain.github.io/docs/tvm.pdf) 的 1.3.1 節。

### 測試函數

測試函數必須接受以下參數：

- exit code - 虛擬機的返回代碼，這樣我們可以理解是否存在錯誤；
- c4 單元 - 控制寄存器 c4 中的“永久數據”；
- 元組 - 我們從數據函數傳遞的值（棧）；
- c5 單元 - 檢查發出的消息；
- gas - 被使用的 gas。

[TVM 返回代碼](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)

## 開始編寫測試

對於本教學中的測試，我們需要一個比較助手函數。我們將使用 `asm` 關鍵字將其定義為低級原語：

```func
int equal_slices (slice a, slice b) asm "SDEQ";
```

## 測試合約代理調用

讓我們編寫第一個測試 `test_example_data()` 並分析其代碼。

### 數據函數

從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_example_data() method_id(0) {
    int function_selector = 0;

    cell my_address = begin_cell()
                .store_uint(1, 2)
                .store_uint(5, 9) 
                .store_uint(7, 5)
                .end_cell();

    cell their_address = begin_cell()
                .store_uint(1, 2)
                .store_uint(5, 9) 
                .store_uint(8, 5) 
                .end_cell();

    slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();

    cell message = begin_cell()
            .store_uint(0x6, 4)
            .store_slice(their_address.begin_parse()) 
            .store_slice(their_address.begin_parse()) 
            .store_grams(100)
            .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
            .store_slice(message_body)
            .end_cell();

    tuple stack = unsafe_tuple([12345, 100, message, message_body]);

    return [function_selector, stack, my_address, get_c7(), null()];
}
```

### 分析

`int function_selector = 0;`

由於我們在調用 `recv_internal()`，我們將值設置為 0，為什麼是 0？Fift（即我們編譯 FunC 腳本所用的語言）有預定義的標識符，即：
- `main` 和 `recv_internal` 的 id = 0
- `recv_external` 的 id = -1
- `run_ticktock` 的 id = -2

為了檢查發送，我們需要發送消息的地址，讓在此範例中我們的地址為 `my_address`，他們的地址為 `their_address`。問題是地址應該如何看待，考慮到它需要與 FunC 類型一致。讓我們參考 [TL-B schema](https://github.com/ton-blockchain/ton/blob/master/crypto/block/block.tlb)，具體到第 100 行，地址描述從這裡開始。

```func
cell my_address = begin_cell()
            .store_uint(1, 2)
            .store_uint(5, 9) 
            .store_uint(7, 5)
            .end_cell();
```

`.store_uint(1, 2)` - 0x01 外部地址；
`.store_uint(5, 9)` - 長度等於 5；
`.store_uint(7, 5)` - 讓我們的地址為 7；

為了理解 TL-B 和這部分具體內容，我建議您學習 [MTProto](https://core.telegram.org/mtproto/TL) 。

我們還會再收集一個地址，讓它為 8。

```func
cell their_address = begin_cell()
            .store_uint(1, 2)
            .store_uint(5, 9) 
            .store_uint(8, 5) 
            .end_cell();
```

為了組裝消息，剩下的是組裝消息主體的 slice，放入數字 12345：

```func
slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();
```

現在剩下的就是組裝消息本身：

```func
cell message = begin_cell()
        .store_uint(0x6, 4)
        .store_slice(their_address.begin_parse()) 
        .store_slice(their_address.begin_parse()) 
        .store_grams(100)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_slice(message_body)
        .end_cell();
```

我注意到我們在 cells 中收集了地址，因此要在消息中存儲它們，需要使用 `begin_parse()`，這將把 cell 轉換為 slice。

您可能已經注意到，發送者和接收者是相同的地址，這樣做是為了簡化測試，不產生大量地址，因為根據條件，當擁有者向合約發送消息時，不應進行轉發。

現在讓我提醒您函數應返回什麼：
- 函數選擇器 - 被測合約中調用函數的 id；
- 元組 - 我們將傳遞給執行測試的函數的值（棧）；
- c4 單元 - 控制寄存器 c4 中的“永久數據”；
- c7 元組 - 控制寄存器 c7 中的“臨時數據”；
- gas 限制整數

如您所見，我們只需要組裝元組並返回數據。根據我們合約的 `recv_internal()` 簽名，我們將以下值放入：

```func
tuple stack = unsafe_tuple([12345, 100, message, message_body]);
```

我注意到我們將返回 `my_address`，這是為了檢查地址匹配的條件。

```func
return [function_selector, stack, my_address, get_c7(), null()];
```

如您所見，我們使用 `get_c7()` 將 c7 的當前狀態放入 c7，而在 gas 限制整數中放入 `null()`。

### 測試函數

代碼：

```func
_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
    throw_if(100, exit_code != 0);

    slice actions = actions.begin_parse();
    throw_if(101, actions~load_uint(32) != 0x0ec3c86d); 


    throw_if(102, ~ slice_empty?(actions~load_ref().begin_parse())); 

    slice msg = actions~load_ref().begin_parse();
    throw_if(103, msg~load_uint(6) != 0x10);

    slice send_to_address = msg~load_msg_addr();
    slice expected_my_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(7, 5).end_cell().begin_parse();

    throw_if(104, ~ equal_slices(expected_my_address, send_to_address));
    throw_if(105, msg~load_grams() != 0);
    throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);

    slice sender_address = msg~load_msg_addr();
    slice expected_sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(8, 5).end_cell().begin_parse();
    throw_if(107, ~ equal_slices(sender_address, expected_sender_address));

    slice fwd_msg = msg~load_ref().begin_parse();

    throw_if(108, fwd_msg~load_uint(32) != 12345);
    fwd_msg.end_parse();

    msg.end_parse();
}
```

### 分析

`throw_if(100, exit_code != 0);`

檢查返回代碼，如果返回代碼不等於零，則函數將拋出異常。
0 是智能合約成功執行的標準返回代碼。

```func
slice actions = actions.begin_parse();
throw_if(101, actions~load_uint(32) != 0x0ec3c86d);
```

發出的消息被寫入 c5 寄存器，因此我們從中卸載一個 32 位值（`load_uint` 是標準 FunC 庫中的一個函數，它從 slice 中加載一個無符號的 n 位整數）並在不等於 0x0ec3c86d 時給出錯誤，即沒有發送消息。可以從 [TL-B schema line 371](https://github.com/ton-blockchain/ton/blob/d01bcee5d429237340c7a72c4b0ad55ada01fcc3/crypto/block/block.tlb) 中取數字 0x0ec3c86d，並確保 `send_raw_message` 使用 ` action_send_msg`，請參見標準庫[第 764 行](https://github.com/ton-blockchain/ton/blob/24dc184a2ea67f9c47042b4104bbb4d82289fac1/crypto/vm/tonops.cpp) 。

在繼續之前，我們需要了解如何從[文檔](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=result-of-tvm-execution)中存儲 c5 中的數據。c5 存儲兩個 cell 引用，其中包含列表中的最後一個操作和一個引用包含前一個操作的 cell。
更多關於如何從操作中完全獲取數據的詳細信息將在下面的代碼中描述。現在主要的是我們將從 c5 中卸載第一個鏈接並立即檢查它是否為空，以便隨後可以獲取包含消息的 cell。

```func
throw_if(102, ~ slice_empty?(actions~load_ref().begin_parse()));
```

我們使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=slice_empty) 的 `slice_empty?` 進行檢查。

我們需要從 "actions" cell 中獲取消息的 slice，使用 `load_ref()` 獲取包含消息的 cell 引用，並使用 `begin_parse()` 將其轉換為 slice。

```func
slice msg = actions~load_ref().begin_parse();
```

繼續：

```func
slice send_to_address = msg~load_msg_addr();
slice expected_my_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(7, 5).end_cell().begin_parse();
throw_if(104, ~ equal_slices(expected_my_address, send_to_address));
```

開始讀取消息。通過使用 `load_msg_addr()` 加載消息中的地址來檢查接收者的地址 - 它從 slice 中加載唯一前綴是有效 MsgAddress 的地址。

在 slice `expected_my_address` 中，我們放入我們在確定數據的函數中收集的相同地址。

並且，我們將使用之前聲明的 `equal_slices()` 檢查它們的不匹配。由於函數將檢查相等性，為了檢查不等性，我們使用一元運算符 `~`，它是按位非。位操作在 [Wikipedia](https://zh.wikipedia.org/wiki/按位操作) 上有詳細描述。

```func
throw_if(105, msg~load_grams() != 0);
throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);
```

使用 [標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=load_grams) 中的 `load_grams()` 和 `load_uint()` 檢查消息中的 Ton 數量是否不等於 0 和其他可以在[消息模式](https://ton-blockchain.github.io/docs/#/smart-contracts/messages)中查看的服務字段，從消息中讀取它們。

```func
slice sender_address = msg~load_msg_addr();
slice expected_sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(8, 5).end_cell().begin_parse();
throw_if(107, ~ equal_slices(sender_address, expected_sender_address));
```

繼續讀取消息，我們檢查發送者的地址，就像之前檢查接收者地址一樣。

```func
slice fwd_msg = msg~load_ref().begin_parse();
throw_if(108, fwd_msg~load_uint(32) != 12345);
```

剩下的是檢查消息主體中的值。首先，使用 `load_ref()` 從消息中加載 cell 引用，並使用 `begin_parse()` 將其轉換為 slice。然後使用標準 FunC 庫中的 `load_uint`（它從 slice 中加載一個無符號的 n 位整數）加載 32 位值，並將其與我們的值 12345 進行比較。

```func
fwd_msg.end_parse();
msg.end_parse();
```

最後，我們在讀取後檢查 slice 是否為空，包括整個消息和從中獲取值的消息主體。需要注意的是，`end_parse()` 會在 slice 不為空時拋出異常，這在測試中非常方便。

## 測試相同地址

根據[第三課](https://github.com/romanovichim/TonFunClessons_ru/blob/main/3lesson/thirdlesson.md)中的任務，當擁有者向合約發送消息時，不應進行轉發，讓我們測試這一點。

### 數據函數

從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_same_addr_data() method_id(2) {
    int function_selector = 0;

    cell my_address = begin_cell()
                            .store_uint(1, 2) 
                            .store_uint(5, 9)
                            .store_uint(7, 5)
                            .end_cell();

    slice message_body = begin_cell().store_uint(12345, 32).end_cell().begin_parse();

    cell message = begin_cell()
            .store_uint(0x6, 4)
            .store_slice(my_address.begin_parse()) 
            .store_slice(my_address.begin_parse())
            .store_grams(100)
            .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
            .store_slice(message_body)
            .end_cell();

    tuple stack = unsafe_tuple([12345, 100, message, message_body]);

    return [function_selector, stack, my_address, get_c7(), null()];
}
```

### 分析

數據函數與前一個數據函數幾乎沒有區別，唯一的區別是只有一個地址，因為我們正在測試如果從我們的地址向智能合約代理發送消息會發生什麼。同樣，我們將發送給自己，以節省我們撰寫測試的時間。

### 測試函數

代碼：

```func
_ test_same_addr(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
    throw_if(100, exit_code != 0);

    throw_if(102, ~ slice_empty?(actions.begin_parse())); 

}
```

我們再次檢查返回代碼，如果返回代碼不等於零，則函數將拋出異常。

`throw_if(100, exit_code != 0);`

0 - 智能合約成功執行的標準返回代碼。

`throw_if(102, ~ slice_empty?(actions.begin_parse()));`

由於代理合約不應發送消息，我們只需檢查 slice 是否為空，使用 `slice_empty?`，關於該函數的更多信息請參見[這裡](https://ton-blockchain.github.io/docs/#/func/stdlib?id=slice_empty) 。

## 練習

如您所見，我們尚未測試使用 `send_raw_message;` 發送消息的模式。

### 提示

一個 "消息解析" 函數示例：

```func
(int, cell) extract_single_message(cell actions) impure inline method_id {
    ;; ---------------- Parse actions list
    ;; prev:^(OutList n)
    ;; #0ec3c86d
    ;; mode:(## 8)
    ;; out_msg:^(MessageRelaxed Any)
    ;; = OutList (n + 1);
    slice cs = actions.begin_parse();
    throw_unless(1010, cs.slice_refs() == 2);
    
    cell prev_actions = cs~load_ref();
    throw_unless(1011, prev_actions.cell_empty?());
    
    int action_type = cs~load_uint(32);
    throw_unless(1013, action_type == 0x0ec3c86d);
    
    int msg_mode = cs~load_uint(8);
    throw_unless(1015, msg_mode == 64); 
    
    cell msg = cs~load_ref();
    throw_unless(1017, cs.slice_empty?());
    
    return (msg_mode, msg);
}
```