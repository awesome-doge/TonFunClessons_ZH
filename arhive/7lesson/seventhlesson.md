# 第七課：Hashmap 或字典

## 簡介

在這一課中，我們將編寫一個智能合約，該合約可以在 The Open Network 測試網絡中使用 FUNC 語言進行各種 Hashmap（字典）操作，並使用 [toncli](https://github.com/disintar/toncli) 將其部署到測試網絡中，並在下一課中進行測試。

## 需求

要完成這個教程，你需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面。並且能夠使用 toncli 創建/部署項目，你可以在[第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md)中學習如何操作。

## Hashmap 或字典

Hashmap 是一種由樹表示的數據結構。Hashmap 將鍵映射到任意類型的值，以便可以快速查找和修改。詳細信息見 [3.3 節](https://ton-blockchain.github.io/docs/tvm.pdf)。在 FunC 中，hashmaps 是 [由一個 cell 表示的](https://ton-blockchain.github.io/docs/#/func/stdlib?id=dictionaries-primitives)。

## 智能合約

智能合約的任務是添加和刪除數據以及 Hashmap 存儲的鍵/值，具體功能如下：
- 當智能合約收到結構如下的消息時，合約必須將一個新鍵/值記錄添加到其數據中：
  - 32 位無符號 `op` 等於 1
  - 64 位無符號 `query_id`
  - 256 位無符號鍵
  - 64 位 `valid_until` Unix 時間
  - 剩餘的 slice 值
- 刪除過期數據的消息結構如下：
  - 32 位無符號 `op` 等於 2
  - 64 位無符號 `query_id`
  接收到這樣的消息時，合約必須從其數據中刪除所有過期的記錄（`valid_until` < `now()`）。並檢查消息中除了 32 位無符號 `op` 和 64 位無符號 `query_id` 外沒有其他數據。
- 對於所有其他內部消息，應拋出錯誤。
- 必須實現 Get 方法 `get_key`，該方法接受一個 256 位無符號鍵，必須返回該鍵的 `valid_until` 和數據 slice 值。如果沒有該鍵的記錄，應拋出錯誤。
- 重要！假設合約從空存儲開始。

**我決定從 [FunC contest1](https://github.com/ton-blockchain/func-contest1) 任務中借鑒智能合約的想法，因為它們非常適合熟悉 TON 智能合約的開發。

## 外部方法

### 外部方法結構

為了讓我們的代理接收消息，我們將使用外部方法 `recv_internal()`。

```func
() recv_internal()  {

}
```

### 外部方法參數

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當 TON 鏈中的一個賬戶發生事件時，它會觸發一個交易。每個交易包含最多 5 個階段。詳細信息見 [此處](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們關心的是 **計算階段**。具體來說，是初始化期間在堆棧上的內容。對於正常的消息觸發交易，堆棧的初始狀態如下：

5 個元素：
- 智能合約餘額（nanoTons）
- 傳入消息餘額（nanotons）
- 包含傳入消息的 cell
- 傳入消息正文，slice 類型
- 方法選擇器（對於 recv_internal 為 0）

結果，我們得到以下代碼：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {

}
```

### 從消息正文中獲取數據

根據條件，根據 `op`，合約應以不同方式工作。因此，我們從消息正文中提取 `op` 和 `query_id`。

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	int op = in_msg_body~load_uint(32);
	int query_id = in_msg_body~load_uint(64);
}
```

你可以在 [第五課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/5lesson/fifthlesson.md) 中了解更多關於 `op` 和 `query_id` 的信息。

我們還使用條件運算符圍繞 `op` 構建邏輯。

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	int op = in_msg_body~load_uint(32);
	int query_id = in_msg_body~load_uint(64);
	
	if (op == 1) {
		;; 這裡將添加新值
	}
	if (op == 2) {
		;; 這裡將刪除
	}
}
```

根據任務，所有其他內部消息應拋出錯誤，因此讓我們在條件語句後添加異常。

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	int op = in_msg_body~load_uint(32);
	int query_id = in_msg_body~load_uint(64);
	
	if (op == 1) {
		;; 這裡將添加新值
	}
	if (op == 2) {
		;; 這裡將刪除
	}
	throw(1001);
}
```

現在我們需要從 `c4` 寄存器中獲取數據。為了從 c4 中獲取數據，我們需要 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的兩個函數。

即：
`get_data` - 從 c4 寄存器中獲取 cell。
`begin_parse` - 將 cell 轉換為 slice

將此值傳遞給 slice ds

```func
cell data = get_data();
slice ds = data.begin_parse();
```

重要的是要考慮到任務中的註釋，即合約將從空的 `c4` 開始。因此，為了從 `c4` 中獲取這些變量，我們使用條件運算符，語法如下：

```func
<condition> ? <consequence> : <alternative>
```

它將看起來像這樣：

```func
cell dic = ds.slice_bits() == 0 ? new_dict() : data;
```

這裡使用的 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=dictionaries-primitives) 函數有：

- `slice_bits()` - 返回 slice 中的數據位數，檢查 c4 是否為空
- `new_dict()` - 創建一個空字典，實際上是一個 null 值。null() 的一種特殊情況。

合約的整體框架如下：

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	int op = in_msg_body~load_uint(32);
	int query_id = in_msg_body~load_uint(64);
	
	cell data = get_data();
	slice ds = data.begin_parse();
	cell dic = ds.slice_bits() == 0 ? new_dict() : data;
	if (op == 1) {
		;; 這裡將添加新值
	}
	if (op == 2) {
		;; 這裡將刪除
	}
	throw(1001);
}
```

### op = 1

當 `op` 等於 1 時，我們將值添加到 hashmap 中。根據任務要求，我們需要：
- 從消息正文中獲取鍵
- 使用鍵和消息正文將值設置到 hashmap（字典）中
- 保存 hashmap（字典）
- 結束函數執行，確保我們不會落入 recv_internal() 結尾聲明的異常中

#### 獲取鍵

這裡一切如前，使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的 `load_uint` 函數從 slice 中加載無符號的 n 位整數。

```func
if (op == 1) {
	int key = in_msg_body~load_uint(256);
}
```

#### 操作 hashmap

要添加數據，我們使用 `dict_set`，該函數將與鍵索引鍵 n 位深度關聯的值設置為字典中的 slice，並返回生成的字典。

```func
if (op == 1) {
	int key = in_msg_body~load_uint(256);
	dic~udict_set(256, key, in_msg_body);
}
```

#### 保存字典

使用 `set_data()` 函數 - 將包含 hashmap 的 cell 寫入 c4 寄存器。

```func
if (op == 1) {
	int key = in_msg_body~load_uint(256);
	dic~udict_set(256, key, in_msg_body);
	set_data(dic);
}
```

#### 結束函數執行

這裡很簡單，`return` 操作符將幫助我們。

```func
if (op == 1) {
	int key = in_msg_body~load_uint(256);
	dic~udict_set(256, key, in_msg_body);
	set_data(dic);
	return();
}
```

### op = 2

在這裡，我們的任務是從數據中刪除所有過期的記錄（`valid_until` < `now()`）。為了“遍歷” hashmap，我們將使用循環。FunC 有三種 [循環](https://ton-blockchain.github.io/docs/#/func/statements?id=loops)：`repeat`、`until`、`while`。

由於我們已經從 `op` 和 `query_id` 中提取出來，這裡我們使用 `end_parse()` 檢查 slice in_msg_body 中是否沒有內容。

`end_parse()` - 檢查 slice 是否為空。如果不是，則拋出異常。

```func
if (op == 2) {
	in_msg_body.end_parse();
}
```

對於我們的情況，我們使用 `until` 循環。

```func
if (op == 2) {
	do {

	} until();
}
```

為了檢查每一步中的 `valid_until` < `now()` 條件，我們需要獲取我們的 hashmap 的某些最小鍵。為此，[FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=dict_set) 有一個 `udict_get_next?` 函數。

`udict_get_next?` - 計算字典中大於某個給定值的最小鍵 k，並返回 k、關聯的值和表示成功的標誌。如果字典為空，則返回 (null, null, 0)。

因此，我們在循環之前設置一個值，從中獲取最小鍵，並在循環中使用表示成功的標誌。

```func
if (op == 2) {
	int key = -1;
	do {
		(key, slice cs, int f) = dic.udict_get_next?(256, key);

	} until(~f);
}
```

現在，使用條件運算符，我們將檢查 `valid_until` < `now()` 的條件。從 `slice cs` 中減去 `valid_until` 的值。

```func
if (op == 2) {
	int key = -1;
	do {
		(key, slice cs, int f) = dic.udict_get_next?(256, key);
		if (f) {
			int valid_until = cs~load_uint(64);
			if (valid_until < now()) {
				;; 這裡將刪除
			}
		}
	} until(~f);
}
```

我們將使用 `udict_delete?` 從 hashmap 中刪除。

`udict_delete?` 刪除字典中鍵 k 的索引。如果鍵存在，則返回修改後的字典（hashmap）和成功標誌 -1。否則返回原始字典和 0。

我們得到：

```func
if (op == 2) {
	int key = -1;
	do {
		(key, slice cs, int f) = dic.udict_get_next?(256, key);
		if (f) {
			int valid_until = cs~load_uint(64);
			if (valid_until < now()) {
				dic~udict_delete?(256, key);
			}
		}
	} until(~f);
}
```

#### 保存字典

使用 `dict_empty?` 我們檢查在循環中操作後 hashmap 是否變為空。

如果有值，我們將 hashmap 保存到 c4 中。如果沒有，則使用 `begin_cell().end_cell()` 函數組合將空 cell 放入 c4。

```func
if (dic.dict_empty?()) {
	set_data(begin_cell().end_cell());
} else {
	set_data(dic);
}
```

#### 結束函數執行

這裡很簡單，`return` 操作符將幫助我們。最終代碼 `op`=2

```func
if (op == 2) {
	int key = -1;
	do {
		(key, slice cs, int f) = dic.udict_get_next?(256, key);
		if (f) {
			int valid_until = cs~load_uint(64);
			if (valid_until < now()) {
				dic~udict_delete?(256, key);
			}
		}
	} until(~f);

	if (dic.dict_empty?()) {
		set_data(begin_cell().end_cell());
	} else {
		set_data(dic);
	}

	return();
}
```

## Get 函數

`get_key` 方法應該返回 `valid_until` 和該鍵的數據 slice。根據任務要求，我們需要：

- 從 c4 中獲取數據
- 通過鍵查找數據
- 如果沒有數據則返回錯誤
- 減去 `valid_until`
- 返回數據

#### 從 c4 中獲取數據

為了加載數據，我們將編寫一個單獨的 load_data() 函數，該函數將檢查是否有數據並返回一個空的 `new_dict()` 字典或來自 c4 的數據。我們將使用 `slice_bits()` 檢查，該函數返回 slice 中的數據位數。

```func
cell load_data() {
	cell data = get_data();
	slice ds = data.begin_parse();
	if (ds.slice_bits() == 0) {
		return new_dict();
	} else {
		return data;
	}
}
```

現在讓我們在 get 方法中調用該函數。

```func
(int, slice) get_key(int key) method_id {
	cell dic = load_data();
}
```

#### 通過鍵查找數據

要通過鍵查找數據，使用 `udict_get?` 函數。

`udict_get?` - 查找字典中鍵的索引。如果成功，返回找到的值作為 slice，以及表示成功的 -1 標誌。失敗時返回 (null, 0)。

我們得到：

```func
(int, slice) get_key(int key) method_id {
	cell dic = load_data();
	(slice payload, int success) = dic.udict_get?(256, key);
}
```

#### 如果沒有數據則返回錯誤

`udict_get?` 函數返回我們放入 success 中的便利標誌。
使用 `throw_unless` 我們將返回異常。

```func
(int, slice) get_key(int key) method_id {
	cell dic = load_data();
	(slice payload, int success) = dic.udict_get?(256, key);
	throw_unless(98, success);
}
```

#### 減去 valid_until 並返回數據

這裡很簡單，從 `payload` 變量中減去 `valid_until` 並返回兩個變量。

```func
(int, slice) get_key(int key) method_id {
	cell dic = load_data();
	(slice payload, int success) = dic.udict_get?(256, key);
	throw_unless(98, success);

	int valid_until = payload~load_uint(64);
	return (valid_until, payload);
}
```

## 結論

特別感謝那些捐款支持本項目的人，這非常有激勵作用，並有助於更快地發布課程。如果您想幫助這個項目（更快地發布課程，將所有內容翻譯成英文等），在[主頁底部](https://github.com/romanovichim/TonFunClessons_ru)有捐款地址。