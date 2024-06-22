# 第八課：針對 Hashmap 的智能合約進行 FunC 測試
## 簡介

在這個教程中，我們將為第七課中在 The Open Network 測試網上使用 FUNC 語言創建的智能合約編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 來執行它們。

## 需求

要完成這個教程，你需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面並完成前面的教程。

## 重要提示

以下內容描述的是舊版測試。目前的 toncli 測試適用於開發版本的 func/fift，具體說明請見 [這裡](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新版測試的教程請見 [這裡](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/11lesson/11lesson.md)。新版測試的發布並不意味著舊版測試沒有意義——它們很好地傳達了邏輯，因此通過這一課程仍然是有價值的。需要注意的是，使用 `toncli run_tests` 時可以使用 `--old` 標誌來運行舊版測試。

## Hashmap 智能合約的測試

針對第七課中的智能合約，我們將編寫以下測試：
- test_example()
- get_stored_value()
- get_not_stored_value()
- wrong_op()
- bad_query()
- remove_outdated()
- get_stored_value_after_remove()
- remove_outdated2()
- get_stored_value_after_remove2()
- get_not_stored_value2()
- remove_outdated3()

> 重要提示：這一課中有很多測試，我們不會詳細解析每一個，而是重點關注最重要的細節。因此，我建議你在進行這一課之前先完成前面的課程。

## 在 toncli 下的 FunC 測試結構

在 toncli 下，每個 FunC 測試都需要編寫兩個函數。第一個函數將確定數據（在 TON 中更準確地說是狀態），我們將把這些數據傳送給第二個函數進行測試。

每個測試函數都必須指定一個 method_id。測試函數的 method_id 應從 0 開始。

##### 數據函數

數據函數不接受參數，但必須返回：
- 函數選擇器 - 被測合約中調用函數的 ID；
- tuple - 我們將傳遞給執行測試的函數的值；
- c4 cell - 控制寄存器 c4 中的“永久數據”；
- c7 tuple - 控制寄存器 c7 中的“臨時數據”；
- gas limit integer - gas 限制（要了解 gas 的概念，我建議你先閱讀 [Ethereum](https://ethereum.org/en/developers/docs/gas/) 的相關內容）；

> Gas 衡量在網絡上執行某些操作所需的計算努力量

有關 c4 和 c7 寄存器的更多信息請見 [這裡](https://ton-blockchain.github.io/docs/tvm.pdf) 第 1.3.1 節

##### 測試函數

測試函數必須接受以下參數：

- exit code - 虛擬機的返回代碼，這樣我們可以了解是否有錯誤
- c4 cell - 控制寄存器 c4 中的“永久數據”
- tuple - 我們從數據函數傳遞的值
- c5 cell - 用於檢查傳出消息
- gas - 使用的 gas

[TVM 返回代碼](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_exit_codes)

## 測試合約觸發並為下一次測試準備數據

讓我們編寫第一個測試 `test_example` 並解析其代碼。

##### 數據函數

我們從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_example_data() method_id(0) {
	int function_selector = 0;

	slice message_body = begin_cell()
	  .store_uint(1, 32) ;; add key
	  .store_uint(12345, 64) ;; query id
	  .store_uint(787788, 256) ;; key
	  .store_uint(1000, 64) ;; valid until
	  .store_uint(12345, 128) ;; 128-bit value
	  .end_cell().begin_parse();

	cell message = begin_cell()
			.store_uint(0x18, 6)
			.store_uint(0, 2) 
			.store_grams(0)
			.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
			.store_slice(message_body)
			.end_cell();

	tuple stack = unsafe_tuple([12345, 100, message, message_body]);

	cell data = begin_cell().end_cell();

	return [function_selector, stack, data, get_c7_now(100), null()];
}
```

## 解析

`int function_selector = 0;`

由於我們調用的是 `recv_internal()`，因此我們將值設置為 0，為什麼是 0？Fift（即我們在其中編譯 FunC 腳本）有預定義的標識符，即：
- `main` 和 `recv_internal` 的 ID 為 0
- `recv_external` 的 ID 為 -1
- `run_ticktock` 的 ID 為 -2

接下來，我們根據 [第七課的任務](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/7lesson/seventhlesson.md) 收集消息正文。

```func
slice message_body = begin_cell()
  .store_uint(1, 32) ;; op
  .store_uint(12345, 64) ;; query id
  .store_uint(787788, 256) ;; key
  .store_uint(1000, 64) ;; valid until
  .store_uint(12345, 128) ;; 128-bit value
  .end_cell().begin_parse();
```

註釋為每個值進行了描述。接下來我們收集消息 cell：

```func
cell message = begin_cell()
        .store_uint(0x18, 6)
        .store_uint(0, 2)
        .store_grams(0)
        .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_slice(message_body)
        .end_cell();
```

消息必須發送到智能合約的地址。為此，我們將使用 `addr_none`（即 `.store_uint(0, 2)`），因為根據 [SENDRAWMSG 文檔](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)，智能合約的當前地址將自動替換為它。

接下來是數據函數的標準部分：

```func
tuple stack = unsafe_tuple([12345, 100, message, message_body]);

cell data = begin_cell().end_cell();

return [function_selector, stack, data, get_c7_now(100), null()];
```

除了 `c7` tuple - "臨時數據" 在 `c7` 控制寄存器中，之前我們並不關心 `c7` 中的內容，只是使用 `get_c7()`，即當前狀態的 `c7`。但在這個教程中，我們需要操作 `c7` 中的數據，因此我們需要為測試編寫一個輔助函數。

## 輔助函數

代碼如下：

```func
tuple get_c7_now(int now) inline method_id {
	return unsafe_tuple([unsafe_tuple([
		0x076ef1ea,           ;; magic
		0,                    ;; actions
		0,                    ;; msgs_sent
		now,                  ;; unixtime
		1,                    ;; block_lt
		1,                    ;; trans_lt
		239,                  ;; randseed
		unsafe_tuple([1000000000, null()]),  ;; balance_remaining
		null(),               ;; myself
		get_config()          ;; global_config
	])]);
}
```

在這個智能合約中，我們需要操縱智能合約中的時間，我們將通過更改 `c7` 寄存器中的數據來實現這一點。要了解應該將哪種 tuple 格式“放入” `c7`，我們參考文檔，即 [TON 描述第 4.4.10 節](https://ton-blockchain.github.io/docs/tblkch.pdf)。

我們不會詳細討論每個參數；我試圖用代碼中的註釋簡要傳達要點。

##### 測試函數

代碼如下：

```func
_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
	throw_if(100, exit_code != 0);
}
```

## 解析

`throw_if(100, exit_code != 0);`

我們檢查返回代碼，如果返回代碼不為零，則函數將拋出異常。
0 - 成功執行智能合約的標準返回代碼。

就是這樣，第一個測試只是放置數據，以便我們可以在後續測試中檢查智能合約的運行。

## 測試獲取存儲值

讓我們編寫一個 `get_stored_value` 測試，該測試將取出我們在 `test_example` 中放置的值並解析其代碼。

##### 數據函數

我們從數據函數開始：

```func
[int, tuple, cell, tuple, int] get_stored_value_data() method_id(2) {
	int function_selector = 127977;

	int key = 787788;

	tuple stack = unsafe_tuple([key]);

	return [function_selector, stack, get_prev_c4(), get_c7(), null()];
}
```

## 解析

```func
int function_selector = 127977;
```

要了解 GET 函數的 ID 是什麼，需要進入編譯的智能合約並查看分配給該函數的 ID。進入 build 文件夾，打開 `contract.fif`，找到帶有 `get_key` 的行

```fift
127977 DECLMETHOD get_key
```

有必要將鍵傳遞給 `get_key` 函數，傳遞我們在上一個測試中設置的鍵，即 `787788`

```func
int key = 787788;

tuple stack = unsafe_tuple([key]);
```

最後返回數據：

```func
return [function_selector, stack, get_prev_c4(), get_c7(), null()];
```

如你所見，我們使用 `get_c7()` 將當前狀態放入 `c7` 中，並在 gas limit integer 中放置 `null()`。然後有趣的是 `c4` 寄存器的情況，我們需要放置上一個測試中的 cell，這無法使用標準的 FunC 庫完成，但在 `toncli` 中這一點已經考慮到了：
在 [toncli 測試描述](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests.md) 中，有 `get_prev_c4` / `get_prev_c5` 函數允許你從先前的測試中獲取 c4/c5 cell。

##### 測試函數

代碼如下：

```func
_ get_stored_value(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
	throw_if(100, exit_code != 0);

	var valid_until = first(stack);
	throw_if(102, valid_until != 1000);
	var value = second(stack);
	throw_if(101, value~load_uint(128) != 12345);
}
```

## 解析

`throw_if(100, exit_code != 0);`

我們檢查返回代碼，如果返回代碼不為零，則函數將拋出異常。
0 - 成功執行智能合約的標準返回代碼。

提醒你一下，`tuple` 變量是我們從數據函數傳遞的（堆棧）值。我們將使用 [數據類型原語](https://ton-blockchain.github.io/docs/#/func/stdlib?id=other-tuple-primitives) `tuple` - `first` 和 `second` 來解析它。

```func
var valid_until = first(stack);
throw_if(102, valid_until != 1000);
var value = second(stack);
throw_if(101, value~load_uint(128) != 12345);
```

我們檢查 `valid_until` 的值和我們傳遞的 `128-bit value`。如果值不同則拋出異常。

## 測試在接收到的鍵沒有條目的情況下引發異常

讓我們編寫 `get_not_stored_value()` 測試並解析其代碼。

##### 數據函數

我們從數據函數開始：

```func
[int, tuple, cell, tuple, int] get_not_stored_value_data() method_id(4) {
	int function_selector = 127977;

	int key = 787789; 

	tuple stack = unsafe_tuple([key]);

	return [function_selector, stack, get_prev_c4(), get_c7(), null()];
}
```

## 解析

數據函數的唯一不同是鍵，我們取一個不在合約存儲中的鍵。

```func
int key = 787789;
```

##### 測試函數

代碼如下：

```func
_ get_not_stored_value(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(5) {
	throw_if(100, exit_code == 0);
}
```

## 解析

我們檢查返回代碼是否為 0，即如果成功完成，則拋出異常。

```func
throw_if(100, exit_code == 0);
```

就是這樣。

## 檢查 op = 2 如果消息包含除了 op 和 query_id 之外的內容是否引發異常

讓我們編寫 `bad_query()` 測試並解析其代碼。

##### 數據函數

我們從數據函數開始：

```func
[int, tuple, cell, tuple, int] bad_query_data() method_id(8) {
   int function_selector = 0;

   slice message_body = begin_cell()
	 .store_uint(2, 32) ;; remove old
	 .store_uint(12345, 64) ;; query id
	 .store_uint(12345, 128) ;; 128-bit value
	 .end_cell().begin_parse();

   cell message = begin_cell()
		   .store_uint(0x18, 6)
		   .store_uint(0, 2) ;; should be contract address
		   .store_grams(0)
		   .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		   .store_slice(message_body)
		   .end_cell();

   tuple stack = unsafe_tuple([12345, 100, message, message_body]);

   return [function_selector, stack, get_prev_c4(), get_c7(), null()];
}
```

## 解析

這個數據函數的主要部分是消息正文，除了 `op` 和 `query_id` 之外，還添加了垃圾數據 `store_uint(12345, 128)`。你需要這樣做來檢查合約中的以下代碼：

```func
if (op == 2) {
	in_msg_body.end_parse();
}
```

因此，進入這裡時合約將拋出異常。

##### 測試函數

代碼如下：

```func
_ bad_query(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(9) {
	throw_if(100, exit_code == 0);
}
```

## 解析

如你所見，我們只是檢查合約是否會拋出異常。

## 測試刪除數據，但現在 < 所有鍵

讓我們編寫 `remove_outdated()` 測試並解析其代碼。

##### 數據函數

我們從數據函數開始：

```func
[int, tuple, cell, tuple, int] remove_outdated_data() method_id(10) {
   int function_selector = 0;

   slice message_body = begin_cell()
	 .store_uint(2, 32) ;; remove old
	 .store_uint(12345, 64) ;; query id
	 .end_cell().begin_parse();

   cell message = begin_cell()
		   .store_uint(0x18, 6)
		   .store_uint(0, 2)
		   .store_grams(0)
		   .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		   .store_slice(message_body)
		   .end_cell();

   tuple stack = unsafe_tuple([12345, 100, message, message_body]);

   return [function_selector, stack, get_prev_c4(), get_c7_now(1000), null()];
}
```

## 解析

`int function_selector = 0;`

由於我們調用的是 `recv_internal()`，因此我們將值設置為 0，為什麼是 0？Fift（即我們在其中編譯 FunC 腳本）有預定義的標識符，即：
- `main` 和 `recv_internal` 的 ID 為 0
- `recv_external` 的 ID 為 -1
- `run_ticktock` 的 ID 為 -2

接下來，我們根據 [第七課的任務](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/7lesson/seventhlesson.md) 收集消息正文。

```func
slice message_body = begin_cell()
 .store_uint(2, 32) ;; remove old
 .store_uint(12345, 64) ;; query id
 .end_cell().begin_parse();
```

我們還收集消息 cell：

```func
cell message = begin_cell()
	   .store_uint(0x18, 6)
	   .store_uint(0, 2)
	   .store_grams(0)
	   .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	   .store_slice(message_body)
	   .end_cell();
```

接下來是數據函數的標準部分：

```func
tuple stack = unsafe_tuple([12345, 100, message, message_body]);

return [function_selector, stack, get_prev_c4(), get_c7_now(1000), null()];
```

我們還將 `1000` 放入 `c7` 以檢查在合約中 `now` < `valid_until` 時的刪除情況。

##### 測試函數

代碼如下：

```func
_ remove_outdated(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(11) {
	throw_if(100, exit_code != 0);
}
```

## 解析

當 `now` < `valid_until` 時，合約應該正常運行，因此：

`throw_if(100, exit_code != 0);`

我們檢查返回代碼，如果返回代碼不為零，則函數將拋出異常。
0 - 成功執行智能合約的標準返回代碼。

是否刪除值將在下一個測試中檢查。

## 測試現在 < 所有鍵時未刪除的值

在這個 `get_stored_value_after_remove()` 測試中，所有內容與 `get_stored_value()` 測試完全相同，因此我不再詳述，只給出代碼：

##### `get_stored_value_after_remove()` 代碼 

```func
[int, tuple, cell, tuple, int] get_stored_value_after_remove_data() method_id(12) {
	int function_selector = 127977;

	int key = 787788;

	tuple stack = unsafe_tuple([key]);

	return [function_selector, stack, get_prev_c4(), get_c7(), null()];
}


_ get_stored_value_after_remove(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(13) {
	throw_if(100, exit_code != 0);

	var valid_until = first(stack);
	throw_if(102, valid_until != 1000);
	var value = second(stack);
	throw_if(101, value~load_uint(128) != 12345);
}
```

## 測試刪除過期數據

現在讓我們通過設置 `get_c7_now(now)` 為 `1001` 來刪除過期數據。這樣做：測試函數 `remove_outdated2()` 使用 `op` = 2 和 `now` = 1001 將刪除過期數據，而 `get_stored_value_after_remove2()` 將檢查鍵 `787788` 的數據是否已刪除並且 `get_key()` 函數將返回異常。結果如下：

```func
[int, tuple, cell, tuple, int] remove_outdated2_data() method_id(14) {
   int function_selector = 0;

   slice message_body = begin_cell()
	 .store_uint(2, 32) ;; remove old
	 .store_uint(12345, 64) ;; query id
	 .end_cell().begin_parse();

   cell message = begin_cell()
		   .store_uint(0x18, 6)
		   .store_uint(0, 2) ;; should be contract address
		   .store_grams(0)
		   .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		   .store_slice(message_body)
		   .end_cell();

   tuple stack = unsafe_tuple([12345, 100, message, message_body]);

   return [function_selector, stack, get_prev_c4(), get_c7_now(1001), null()];
}


_ remove_outdated2(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(15) {
	throw_if(100, exit_code != 0);
}


[int, tuple, cell, tuple, int] get_stored_value_after_remove2_data() method_id(16) {
	int function_selector = 127977;

	int key = 787788;

	tuple stack = unsafe_tuple([key]);

	return [function_selector, stack, get_prev_c4(), get_c7(), null()];
}


_ get_stored_value_after_remove2(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(17) {
	throw_if(100, exit_code == 0);
}
```

## 測試另一個鍵

刪除過期數據後，我們將嘗試使用另一個鍵獲取數據，`get_key` 函數應該也會拋出異常。

```func
[int, tuple, cell, tuple, int] get_not_stored_value2_data() method_id(18) {
	;; Funtion to run (recv_internal)
	int function_selector = 127977;

	int key = 787789; ;; random key

	;; int balance, int msg_value, cell in_msg_full, slice in_msg_body
	tuple stack = unsafe_tuple([key]);

	cell data = begin_cell().end_cell();

	return [function_selector, stack, data, get_c7(), null()];
}


_ get_not_stored_value2(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(19) {
	throw_if(100, exit_code == 0);
}
```

如你所見，我們使用鍵 `787789`，該鍵不存在，並檢查函數是否會拋出異常。

## 再次運行刪除已刪除的數據

最後一個測試，當數據已刪除時再次運行刪除操作

```func
[int, tuple, cell, tuple, int] remove_outdated3_data() method_id(20) {
   int function_selector = 0;

   slice message_body = begin_cell()
	 .store_uint(2, 32) ;; remove old
	 .store_uint(12345, 64) ;; query id
	 .end_cell().begin_parse();

   cell message = begin_cell()
		   .store_uint(0x18, 6)
		   .store_uint(0, 2) 
		   .store_grams(0)
		   .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
		   .store_slice(message_body)
		   .end_cell();

   tuple stack = unsafe_tuple([12345, 100, message, message_body]);

   return [function_selector, stack, begin_cell().end_cell(), get_c7_now(1000), null()];
}


_ remove_outdated3(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(21) {
	throw_if(100, exit_code != 0);
}
```

## 結論

特別感謝那些捐款支持這個項目的人，這給了我們很大的動力，幫助我們更快地發布課程。如果你想幫助這個項目（更快地發布課程、將其翻譯成英文等），在 [主頁底部](https://github.com/romanovichim/TonFunClessons_ru) 有捐款地址。