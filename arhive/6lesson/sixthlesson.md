# 課程 6：包含 `op` 和 `query_id` 的智能合約測試
## 介紹

在這節課中，我們將為第五課中在 The Open Network 測試網中使用 FUNC 語言創建的智能合約編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 執行這些測試。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面，並完成[第五課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/5lesson/fifthlesson.md)。

## 重要事項

以下描述的是舊版本的測試。新版本的 toncli 測試，目前適用於 func/fift 的開發版本，說明請參見[這裡](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新測試課程請參見[這裡](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/11lesson/11lesson.md)。新測試的發布並不意味著舊課程無效——它們很好地傳達了邏輯，因此通過這些課程仍然有價值。請注意，使用 `toncli run_tests` 時，可以使用 `--old` 標誌來運行舊測試。

## 第五課的任務

為方便起見，這裡回顧一下我們在第五課中所做的事情。智能合約將記住由管理者設置的地址，並將其傳達給任何請求者，具體功能如下**：
- 當合約接收到來自管理者的 `op` 等於 1 的訊息，後跟一些 `query_id` 和 `MsgAddress` 時，應將接收到的地址存儲在存儲中。
- 當合約接收到來自任何地址的內部訊息，`op` 等於 2，後跟 `query_id` 時，應回覆發送者一個包含以下內容的訊息：
  - `op` 等於 3
  - 相同的 `query_id`
  - 管理者的地址
  - 自上次管理者請求以來記住的地址（如果尚未有管理者請求，則為空地址 `addr_none`）
  - 附加到訊息中的 TON 值減去處理費用。
- 當智能合約接收到任何其他訊息時，應拋出異常。

## 包含 `op` 和 `query_id` 的智能合約測試

我們將為智能合約編寫以下測試：

- `test_example()` 測試保存地址功能，當 `op` 等於 1 時
- `only_manager_can_change()` 測試只有管理者可以更改智能合約中的地址
- `query()` 測試當 `op` 等於 2 時合約的工作
- `query_op3()` 測試異常情況

## toncli 中的 FunC 測試結構

提醒您，每個 FunC 測試在 toncli 中需要編寫兩個函數。第一個函數將確定數據（從 TON 的角度來看，更準確地說是狀態，但希望數據是一個更容易理解的類比），這些數據將發送給第二個函數進行測試。

每個測試函數必須指定一個 `method_id`。`method_id` 測試函數應從 0 開始。

### 資料函數

資料函數不接受任何參數，但必須返回：
- 函數選擇器 - 測試合約中被調用函數的 ID；
- 元組 - 我們將傳遞給執行測試的函數的值；
- c4 cell - 控制寄存器 c4 中的“永久數據”；
- c7 元組 - 控制寄存器 c7 中的“臨時數據”；
- gas 限制整數 - gas 限制（要了解 gas 的概念，我建議先閱讀[以太坊](https://ethereum.org/en/developers/docs/gas/)的相關資料）；

> Gas 衡量執行網絡上某些操作所需的計算工作量。

有關寄存器 c4 和 c7 的更多信息請參見[這裡](https://ton-blockchain.github.io/docs/tvm.pdf) 的 1.3.1 節。

### 測試函數

測試函數必須接受以下參數：

- 退出代碼 - 虛擬機的返回代碼，以便我們了解是否存在錯誤
- c4 cell - 控制寄存器 c4 中的“永久數據”
- 元組 - 我們從資料函數傳遞的值
- c5 cell - 用於檢查傳出的訊息
- gas - 使用的 gas

[TVM 返回代碼](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)

## 測試保存地址功能，當 `op` 等於 1

讓我們編寫第一個測試 `test_example()` 並分析其代碼。該測試將檢查合約是否保存了管理者的地址以及管理者傳遞給合約的地址。

### 資料函數

讓我們從資料函數開始：

```func
[int, tuple, cell, tuple, int] test_example_data() method_id(0) {

	int function_selector = 0;

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice message_body = begin_cell().store_uint(1, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();

	cell message = begin_cell()
			.store_uint(0x6, 4)
			.store_slice(manager_address.begin_parse())
			.store_uint(0, 2) ;; 應該是合約地址
			.store_grams(100)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

	cell data = begin_cell().store_slice(manager_address.begin_parse()).store_uint(0, 2).end_cell();

	return [function_selector, stack, data, get_c7(), null()];
}
```

## 解析

在第一個測試中，我們要檢查智能合約在 `op` 等於 1 時的操作。相應地，我們將從合約管理者那裡發送一條 `op` 等於 1 的訊息並存儲一些地址。為此，在資料函數中我們需要：

- 管理者地址 `manager_address`
- 要存儲在合約中的地址 `stored_address`
- `op` 等於 1 的訊息正文
- 相應的訊息 `message`
- 用於檢查的 c4 中的管理者地址 `data`

讓我們開始解析：

`int function_selector = 0;`

由於我們調用了 `recv_internal()`，因此我們分配值 0，為什麼是 0？Fift（即我們在其中編譯 FunC 腳本）有預定義的標識符，即：
- `main` 和 `recv_internal` 的 id = 0
- `recv_external` 的 id = -1
- `run_ticktock` 的 id = -2

讓我們收集兩個必要的地址，設為 1 和 3：

```func
	cell manager_address = begin_cell().
		store_uint(1, 2).
		store_uint(5, 9).
		store_uint(1, 5).
		end_cell();
		
	cell stored_address = begin_cell().
		store_uint(1, 2)
		.store_uint(5, 9)
		.store_uint(3, 5)
		.end_cell();
```

我們按照 [TL-B 方案](https://github.com/tonblockchain/ton/blob/master/crypto/block/block.tlb) 收集地址，具體在第 100 行開始描述地址。例如 `manager_address`：

`.store_uint(1, 2)` - 0x01 外部地址；

`.store_uint(5, 9)` - len 等於 5；

`.store_uint(1, 5)` - 地址為 1；

現在讓我們組裝訊息正文的 slice，它將包含：
- 32 位的 op `store_uint(1, 32)`
- 64 位的 query_id `store_uint(12345, 64)`
- 存儲地址 `store_slice(stored_address.begin_parse())`

由於我們在正文中存儲了 slice，並且我們使用 cell 設置地址，因此我們將使用 `begin_parse()`（將 cell 轉換為 slice）。

要組裝訊息正文，我們將使用：

`begin_cell()` - 創建一個 Builder 用於未來的 cell
`end_cell()` - 創建一個 cell

它看起來像這樣：

```func
slice message_body = begin_cell().store_uint(1, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();
```

現在只需組裝訊息本身，但如何將訊息發送到智能合約的地址。為此，我們將使用 `addr_none`，因為根據 [SENDRAWMSG 文件](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)，“addr_none” 將自動替換為當前的智能合約地址。我們得到：

```func
cell message = begin_cell()
        .store_uint(0x6, 4)
        .store_slice(sender_address.begin_parse()) 
        .store_uint(0, 2) 
        .store_grams(100)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_slice(message_body)
        .end_cell();
```

現在讓我們收集將傳遞給執行測試的函數的值，即 `int balance`, `int msg_value`, `cell in_msg_full`, `slice in_msg_body`：

```func
tuple stack = unsafe_tuple([12345, 100, message, message_body]);
```

我們還將為寄存器 `c4` 收集一個 cell，使用我們已經熟悉的函數將管理者地址和 `addr_none` 放入其中。

```func
cell data = begin_cell().store_slice(manager_address.begin_parse()).store_uint(0, 2).end_cell();
```

當然，返回所需的值。

```func
return [function_selector, stack, data, get_c7(), null()];
```

如您所見，在 c7 中，我們使用 `get_c7()` 放置了 c7 的當前狀態，而在 gas 限制整數中，我們放置了 `null()`。

### 測試函數

代碼：

```func
_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
	throw_if(100, exit_code != 0);

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice stored = data.begin_parse();
	throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
	throw_if(102, ~ equal_slices(stored~load_msg_addr(), stored_address.begin_parse()));
	stored.end_parse();
}
```

## 解析

`throw_if(100, exit_code != 0);`

我們檢查返回代碼，如果返回代碼不為零，該函數將拋出異常。0 - 智能合約成功執行的標準返回代碼。

接下來，我們將收集兩個地址，類似於我們在資料函數中收集的地址，以便進行比較。

```func
cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();
```

現在讓我們獲取一些東西來進行比較，我們有一個 `data` cell，因此使用 `begin_parse` - 我們將 cell 轉換為 slice。

```func
slice stored = data.begin_parse();

stored.end_parse();
```

最後，我們在讀取後檢查 slice 是否為空。重要的是要注意，`end_parse()` 如果 slice 不為空將拋出異常，這在測試中非常方便。

我們將使用 `load_msg_addr()` 從 `stored` 中讀取地址。使用我們從上一課中取來的 `equal_slices` 函數進行地址比較。

```func
slice stored = data.begin_parse();
throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
throw_if(102, ~ equal_slices(stored~load_msg_addr(), stored_address.begin_parse()));
stored.end_parse();
```

## 測試當 `op` 等於 1 時，只有管理者可以更改智能合約中的地址

讓我們編寫 `only_manager_can_change()` 測試並分析其代碼。

### 資料函數

讓我們從資料函數開始：

```func
[int, tuple, cell, tuple, int] only_manager_can_change_data() method_id(2) {
	int function_selector = 0;

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice message_body = begin_cell().store_uint(1, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();

	cell message = begin_cell()
			.store_uint(0x6, 4)
			.store_slice(sender_address.begin_parse()) 
			.store_uint(0, 2) 
			.store_grams(100)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

	cell data = begin_cell().store_slice(manager_address.begin_parse()).store_uint(0, 2).end_cell();

	return [function_selector, stack, data, get_c7(), null()];
}
```

## 解析

如您所見，代碼幾乎與 `test_example()` 相同。除了：
- 添加了一個額外的地址 `sender_address`
- 在訊息中，管理者地址 `manager_address` 更改為發送者地址 `sender_address`

我們將收集發送者地址，如同其他所有地址一樣：

```func
cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
```

並將訊息中的管理者地址 `manager_address` 更改為發送者地址 `sender_address`。

```func
cell message = begin_cell()
        .store_uint(0x6, 4)
        .store_slice(sender_address.begin_parse()) 
        .store_uint(0, 2) 
        .store_grams(100)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_slice(message_body)
        .end_cell();
```

### 測試函數

代碼：

```func
_ only_manager_can_change(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
	throw_if(100, exit_code == 0); 

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice stored = data.begin_parse();
	throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
	throw_if(102, stored~load_uint(2) != 0);
	stored.end_parse();
}
```

## 解析

再次，我們檢查返回代碼，如果返回代碼不為零，該函數將拋出異常。

`throw_if(100, exit_code != 0);`

0 - 智能合約成功執行的標準返回代碼。

我們收集管理者地址 `manager_address;` 和存儲地址 `stored_address`，與資料函數中的相同，以便進行檢查。

```func
cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();
```

現在讓我們獲取一些東西來進行比較，我們有一個 `data` cell，因此使用 `begin_parse` - 我們將 cell 轉換為 slice。

```func
slice stored = data.begin_parse();

stored.end_parse();
```

最後，我們在讀取後檢查 slice 是否為空。重要的是要注意，`end_parse()` 如果 slice 不為空將拋出異常，這在測試中非常方便。

我們將使用 `load_msg_addr()` 從 `stored` 中讀取地址。使用我們從上一課中取來的 `equal_slices` 函數進行地址比較。

```func
slice stored = data.begin_parse();
throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));

stored.end_parse();
```

另外使用 `~load_uint(2)` 將存儲地址與 `addr_none` 進行比較

`load_uint` - 從 slice 加載 n 位無符號整數

我們得到：

```func
slice stored = data.begin_parse();
throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
throw_if(102, stored~load_uint(2) != 0);
stored.end_parse();
```

## 測試智能合約在 `op` 等於 2 時的操作

讓我們編寫 `query()` 測試並分析其代碼。當 `op` 等於 2 時，我們必須發送一條具有特定正文的訊息。

### 資料函數

讓我們從資料函數開始：

```func
[int, tuple, cell, tuple, int] query_data() method_id(4) {
	int function_selector = 0;

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice message_body = begin_cell().store_uint(2, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();

	cell message = begin_cell()
			.store_uint(0x6, 4)
			.store_slice(sender_address.begin_parse()) 
			.store_uint(0, 2)
			.store_grams(100)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

	cell data = begin_cell().store_slice(manager_address.begin_parse()).store_slice(stored_address.begin_parse()).end_cell();

	return [function_selector, stack, data, get_c7(), null()];
}
```

## 解析

資料函數與本課中之前的資料函數相差不大。

`int function_selector = 0;`

檢查預定義的函數編號 `recv_internal`。

我們收集三個地址：

- 管理者地址 `manager_address`
- 發送者地址 `sender_address`
- 要存儲在合約中的地址 `stored_address`

```func
cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();
```

現在讓我們組裝訊息正文的 slice（重要：現在 `op` 等於 2），它將包含：
- 32 位的 op `store_uint(2, 32)`
- 64 位的 query_id `store_uint(12345, 64)`
- 存儲地址 `store_slice(stored_address.begin_parse())`

由於我們在正文中存儲了 slice，並且我們使用 cell 設置地址，因此我們將使用 `begin_parse()`（將 cell 轉換為 slice）。

要組裝訊息正文，我們將使用：

`begin_cell()` - 創建一個 Builder 用於未來的 cell
`end_cell()` - 創建一個 cell

它看起來像這樣：

```func
slice message_body = begin_cell().store_uint(2, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();
```

現在是訊息的時間：

```func
cell message = begin_cell()
		.store_uint(0x6, 4)
		.store_slice(sender_address.begin_parse()) 
		.store_uint(0, 2)
		.store_grams(100)
		.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		.store_slice(message_body)
		.end_cell();
```
發送者是 `sender_address`，接收者是合約的地址，通過發送 `addr_none`。當前的智能合約地址將自動替換。

現在讓我們收集將傳遞給執行測試的函數的值，即 `int balance`, `int msg_value`, `cell in_msg_full`, `slice in_msg_body`：

```func
tuple stack = unsafe_tuple([12345, 100, message, message_body]);
```

我們還將為寄存器 `c4` 收集一個 cell，使用我們已經熟悉的函數將管理者地址和存儲地址放入其中。

```func
cell data = begin_cell().store_slice(manager_address.begin_parse()).store_slice(stored_address.begin_parse()).end_cell();
```

當然，返回所需的值。

```func
return [function_selector, stack, data, get_c7(), null()];
```

如您所見，在 c7 中，我們使用 `get_c7()` 放置了 c7 的當前狀態，而在 gas 限制整數中，我們放置了 `null()`。

### 測試函數

代碼：

```func
_ query(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(5) {
	throw_if(100, exit_code != 0); 

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice stored = data.begin_parse();
	throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
	throw_if(102, ~ equal_slices(stored~load_msg_addr(), stored_address.begin_parse()));
	stored.end_parse();

	slice all_actions = actions.begin_parse();
	all_actions~load_ref();
	slice msg = all_actions~load_ref().begin_parse();

	throw_if(103, msg~load_uint(6) != 0x10);

	slice send_to_address = msg~load_msg_addr();

	throw_if(104, ~ equal_slices(sender_address.begin_parse(), send_to_address));
	throw_if(105, msg~load_grams() != 0);
	throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);

	throw_if(107, msg~load_uint(32) != 3);
	throw_if(108, msg~load_uint(64) != 12345);
	throw_if(109, ~ equal_slices(manager_address.begin_parse(), msg~load_msg_addr()));
	throw_if(110, ~ equal_slices(stored_address.begin_parse(), msg~load_msg_addr()));

	msg.end_parse();
}
```

## 解析

開頭類似於我們已經解析的內容，三個地址 (`manager_address`,`sender_address`,`stored_address`) 與我們在資料函數中收集的相同，使用 `equal_slices()` 進行比較。

```func
cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

slice stored = data.begin_parse();
throw_if(101, ~ equal_slices(stored~load_msg_addr(), manager_address.begin_parse()));
throw_if(102, ~ equal_slices(stored~load_msg_addr(), stored_address.begin_parse()));
stored.end_parse();
```

然後我們移動到訊息。傳出的訊息寫入寄存器 c5。我們從 `actions` cell 中獲取它們到 `stored` slice 中。

```func
slice all_actions = actions.begin_parse();
```

現在讓我們回憶一下根據 [文件](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=result-of-tvm-execution) 在 c5 中存儲數據的方式。

那裡存儲了一個包含兩個 cell 參考的列表，分別是最後一個操作中的兩個 cell 參考和前一個操作中的一個 cell 參考。（在教學的結尾，有一段代碼顯示如何完全解析 `actions`，希望這對您有所幫助）

因此，首先我們“卸載”第一個鏈接並取第二個，其中包含我們的訊息：

```func
all_actions~load_ref();
slice msg = all_actions~load_ref().begin_parse();
```

cell 立即加載到 `msg` slice 中。讓我們檢查標誌：

```func
throw_if(103, msg~load_uint(6) != 0x10);
```

從訊息中加載發送者地址使用 `load_msg_addr()` - 從 slice 加載唯一前綴，即有效的 MsgAddress，並檢查它們是否等於我們之前指定的地址。不要忘記將 cell 轉換為 slice。

```func
slice send_to_address = msg~load_msg_addr();

throw_if(104, ~ equal_slices(sender_address.begin_parse(), send_to_address));
```

使用 `load_grams()` 和 `load_uint()` 從 [標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=load_grams) 檢查訊息中的 TON 數量是否不等於 0 以及其他可以在 [訊息模式](https://ton-blockchain.github.io/docs/#/smart-contracts/messages) 中查看的服務字段，從訊息中讀取它們。

```func
throw_if(105, msg~load_grams() != 0);
throw_if(106, msg~load_uint(1 + 4 + 4 + 64 + 32 + 1 + 1) != 0);
```

我們開始檢查訊息正文，從 `op` 和 `query_id` 開始：

```func
throw_if(107, msg~load_uint(32) != 3);
throw_if(108, msg~load_uint(64) != 12345);
```

接下來，取它們的地址訊息正文並使用 `equal_slices` 進行比較。由於該函數將檢查相等性，為測試不相等性，我們使用一元運算符 ` ~`，它是位運算符非。

```func
throw_if(109, ~ equal_slices(manager_address.begin_parse(), msg~load_msg_addr()));
throw_if(110, ~ equal_slices(stored_address.begin_parse(), msg~load_msg_addr()));
```

最後，在讀取後，我們檢查 slice 是否為空，無論是整個訊息還是我們從中取值的訊息正文。重要的是要注意，`end_parse()` 如果 slice 不為空將拋出異常，這在測試中非常方便。

```func
msg.end_parse();
```

## 測試智能合約在異常情況下的操作

讓我們編寫 `query_op3` 測試並分析其代碼。根據分配——當智能合約接收到任何其他訊息時，應拋出異常。

### 資料函數

讓我們從資料函數開始：

```func
[int, tuple, cell, tuple, int] query_op3_data() method_id(6) {
	int function_selector = 0;

	cell manager_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(1, 5).end_cell();
	cell sender_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(2, 5).end_cell();
	cell stored_address = begin_cell().store_uint(1, 2).store_uint(5, 9).store_uint(3, 5).end_cell();

	slice message_body = begin_cell().store_uint(3, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();

	cell message = begin_cell()
			.store_uint(0x6, 4)
			.store_slice(sender_address.begin_parse()) 
			.store_uint(0, 2) 
			.store_grams(100)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

	cell data = begin_cell().store_slice(manager_address.begin_parse()).store_slice(stored_address.begin_parse()).end_cell();

	return [function_selector, stack, data, get_c7(), null()];
}
```

## 解析

這個資料函數幾乎完全等同於我們在上一段中編寫的函數，除了 `op` 值一個細節，以便我們可以檢查 `op` 是否不等於 2 或 1 時會發生什麼。

```func
slice message_body = begin_cell().store_uint(3, 32).store_uint(12345, 64).store_slice(stored_address.begin_parse()).end_cell().begin_parse();
```

### 測試函數

代碼：

```func
_ query_op3(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(7) {
	throw_if(100, exit_code == 0);
}
```

## 解析

我們檢查，如果返回代碼為 0，即成功完成，則拋出異常。

```func
throw_if(100, exit_code == 0);
```

就這麼簡單。

## 結論

特別感謝那些捐款支持本項目的人，這非常有激勵作用，並有助於更快地發布課程。如果您想幫助這個項目（更快地發布課程，將所有內容翻譯成英文等），在[主頁底部](https://github.com/romanovichim/TonFunClessons_ru)有捐款地址。

### 附加內容

一個“解析動作”函數的例子：

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