# 第十一課 使用 toncli 為智能合約編寫新式 func 測試

## 簡介

在 toncli 中出現了新的測試方式，本教程將演示如何使用 [toncli](https://github.com/disintar/toncli) 為 FUNC 語言編寫的智能合約編寫新式測試並執行這些測試。

## 需求

完成本教程之前，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面，並完成 [第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md) 和 [第二課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/2lesson/secondlesson.md)。

> 本教程編寫於 2022/08/23，目前新測試僅在 func/fift 的 dev 版本中可用。如何安裝 dev 版本，請參閱 [新測試的安裝說明](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)。

## 我們將測試什麼

在本課中，我們將測試與第一課相似的智能合約。因此，在進行測試之前，讓我們先了解一下合約本身。

我們將創建的智能合約應具有以下功能：

- 在其數據中存儲一個整數 total，即一個 64 位無符號數字；
- 當接收到內部傳入消息時，合約應從消息體中提取一個 32 位無符號整數，將其加到 total 中並存儲在合約數據中；
- 智能合約應具有一個 get_total 方法，用於返回 total 的值。

智能合約代碼：

```func
() recv_internal(slice in_msg) impure {
  int n = in_msg~load_uint(32);

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

> 如果你對這個智能合約有不理解的地方，建議先學習第一課。

## 開始測試

在舊的測試中，我們需要編寫兩個函數：一個數據函數和一個測試函數，並且還需要指定函數的 ID，以便測試能夠理解測試的內容。新的測試中沒有這樣的邏輯。

在新測試中，通過兩個函數來調用智能合約的方法進行測試：
- `invoke_method`，假設不會拋出異常；
- `invoke_method_expect_fail`，假設會拋出異常。

這些特殊函數將在測試函數內調用，測試函數可以返回任意數量的值，所有這些值將在運行測試時顯示在報告中。

每個測試函數的名稱必須以 `__test` 開頭。這樣我們可以區分哪些函數是測試函數，哪些只是輔助函數。

讓我們通過示例來看看它是如何工作的。

### 測試發送消息和檢查金額

讓我們將測試函數命名為 `__test_example()`，它將返回消耗的 gas 數量，因此它的返回類型是 `int`。

```func
int __test_example() {
}
```

由於我們在本課中將編寫大量測試，因此我們經常需要將 `c4` 寄存器歸零，因此我們將創建一個輔助函數來將 `c4` 設置為零。（它的名稱不會包含 `__test`）。

我們將使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的函數來完成這個任務。

`set_data(begin_cell().store_uint(total, 64).end_cell());`

`begin_cell()` - 創建一個用於未來 cell 的建構器；
`store_uint()` - 寫入 total 的值；
`end_cell()` - 創建 cell；
`set_data()` - 將 cell 寫入寄存器 c4。

得到以下代碼：

```func
() set_default_initial_data() impure {
  set_data(begin_cell().store_uint(0, 64).end_cell());
}
```

`impure` 是一個關鍵字，表示該函數會改變智能合約的數據。

> 如果沒有指定 `impure` 並且函數調用的結果未被使用，則 FunC 編譯器可能會移除該函數調用。

讓我們在測試函數中調用這個輔助函數：

```func
int __test_example() {
	set_default_initial_data();
}
```

新測試的一個方便之處是，在一個測試函數中可以調用多個智能合約方法。在我們的測試中，我們將調用 `recv_internal()` 方法和 Get 方法，因此我們將通過消息來遞增 `c4` 的值，並立即檢查該值是否已更改為發送的值。

要調用 `recv_internal()` 方法，我們需要創建一個帶有消息的 cell。

```func
int __test_example() {
	set_default_initial_data();

	cell message = begin_cell().store_uint(10, 32).end_cell();
}
```

現在我們可以調用方法了，我們將使用 `invoke_method`。`invoke_method` 接受兩個參數：方法名稱和參數（將傳遞給方法）作為元組。兩個值會返回：使用的 gas 量和方法返回的值（作為元組）。如果方法調用引發異常，`invoke_method` 也會引發異常，測試將失敗。

在第一次調用中，參數將是 `recv_internal` 和一個使用 `begin_parse()` 轉換為 slice 的消息元組。

`var (int gas_used1, _) = invoke_method(recv_internal, [message.begin_parse()]);`

為了記錄，我們將使用的 gas 量存儲在 `int gas_used1` 中。

在第二次調用中，參數將是 Get 方法 `get_total()` 和一個空元組。

`var (int gas_used2, stack) = invoke_method(get_total, []);`

為了報告，我們還將使用的 gas 量存儲在 `int gas_used2` 中，另外方法返回的值，以便稍後檢查一切是否正常。

得到以下代碼：

```func
int __test_example() {
	set_default_initial_data();

	cell message = begin_cell().store_uint(10, 32).end_cell();
	var (int gas_used1, _) = invoke_method(recv_internal, [message.begin_parse()]);
	var (int gas_used2, stack) = invoke_method(get_total, []);
}
```

讓我們檢查 invoke_method 在第二次調用中返回的值是否等於我們發送的值。如果不是，則引發異常。最後，我們將返回消耗的 gas 量。

```func
int __test_example() {
	set_default_initial_data();

	cell message = begin_cell().store_uint(10, 32).end_cell();
	var (int gas_used1, _) = invoke_method(recv_internal, [message.begin_parse()]);
	var (int gas_used2, stack) = invoke_method(get_total, []);
	[int total] = stack;
	throw_if(101, total != 10);
	return gas_used1 + gas_used2;
}
```

這就是整個測試，非常方便。

### c4 中的數據和新測試

默認情況下，"持久" 數據不會從前一個測試函數中複製。

讓我們通過下一個測試來驗證這一點。

##### 在 c4 中沒有數據的情況下調用 get 方法

由於我們預期該方法會引發異常，因此我們將使用 `invoke_method_expect_fail`。它接受兩個參數，與 `invoke_method` 類似，但僅返回方法使用的 gas 量。

```func
int __test_no_initial_data_should_fail() {
	int gas_used = invoke_method_expect_fail(get_total, []);
	return gas_used;
}
```

##### 在 c4 中使用來自先前測試的數據調用 get 方法

現在，為了展示如何在不同測試函數中使用 c4 中的數據，調用第一個測試（它將為 c4 提供數據），並使用專用函數 `get_prev_c4()`。

我們調用第一個測試 `__test_example()`：

```func
int __test_set_data() {
  return __test_example();
}
```

現在我們只需復制第一個測試，將發送一個數為 10 的消息替換為 `set_data(get_prev_c4());`

```func
int __test_data_from_prev_test() {
	set_data(get_prev_c4());
	var (int gas_used, stack) = invoke_method(get_total, []);
	var [total] = stack;
	throw_if(102, total != 10);
	return gas_used;
}
```

> 當然，我們可以使用 `get_prev_c4()` 和 `get_prev_c5()` 分別從前一個測試函數中獲取 c4 和 c5 的數據，但寫輔助函數（如我們在最開始所做的那樣）並在不同的測試函數中使用它們是一個好的做法。

### 調用方法異常和測試函數異常

重要的是要注意，由 invoke 方法引發的異常不會在內部影響測試套件。讓我們檢查變量的值是否在其聲明之後 `invoke_method_expect_fail` 引發異常時沒有改變。

這個測試的框架：

```func
int __test_throw_doesnt_corrupt_stack() {
	int check_stack_is_not_corrupted = 123;
}
```

如您所見，我們將數字 123 推到堆棧上。現在讓我們調用智能合約的 Get 方法，假設這將導致異常，因為 c4 中沒有數據。

```func
int __test_throw_doesnt_corrupt_stack() {
	int check_stack_is_not_corrupted = 123;
	int gas_used = invoke_method_expect_fail(get_total, []);
}
```

最後，我們檢查該值是否未改變。

```func
int __test_throw_doesnt_corrupt_stack() {
	int check_stack_is_not_corrupted = 123;
	int gas_used = invoke_method_expect_fail(get_total, []);

	throw_if(102, check_stack_is_not_corrupted != 123);
	return gas_used;
}
```

### 測試類型

讓我們看看如何在新測試中測試類型。為此，我們將編寫一個輔助函數，它將存儲一個數字和一個包含兩個數字的 cell。

> 是的，您可以使用 invoke 方法調用輔助函數)

第一個數字將立即確定，第二個數字將傳遞給函數。

```func
(int, cell) build_test_cell(int x, int y) {
  return (12345, begin_cell().store_uint(x, 64).store_uint(y, 64).end_cell());
}
```

讓我們構建測試函數的框架，並立即調用輔助函數：

```func
int __test_not_integer_return_types() {
  var (int gas_used, stack) = invoke_method(build_test_cell, [100, 500]);
}
```

我們獲取值，即數字和 cell `[int res, cell c] = stack;`，立即檢查數字的值 `throw_if(102, res != 12345);`。得到：

```func
int __test_not_integer_return_types() {
  var (int gas_used, stack) = invoke_method(build_test_cell, [100, 500]);
  [int res, cell c] = stack;
  throw_if(102, res != 12345);
}
```

讓我們使用 `begin_parse` 將 cell 轉換為 slice。然後通過減去這些值來檢查值。我們將使用 `load_uint` 函數來減去這些值，這是一個 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的函數，它從 slice 中加載一個無符號的 n 位整數。

```func
int __test_not_integer_return_types() {
  var (int gas_used, stack) = invoke_method(build_test_cell, [100, 500]);
  [int res, cell c] = stack;
  throw_if(102, res != 12345);
  slice s = c.begin_parse();
  throw_if(103, s~load_uint(64) != 100);
  throw_if(104, s~load_uint(64) != 500);
  return gas_used;
}
```

最後，我們返回使用的 gas。

### Gas 流測試示例

由於 invoke 方法返回 gas 消耗量，這使得在新測試中檢查 gas 消耗非常方便。讓我們通過調用一個空的輔助函數來看看它是如何工作的。

空方法：

```func
() empty_method() inline method_id {
}
```

調用一個空方法大約需要 600 單位的 gas。測試框架：

```func
int __test_empty_method_gas_consumption() method_id {
}
```

通過 `invoke_method` 調用該方法，並檢查 gas 消耗不低於 400，但不超過 700。

```func
int __test_empty_method_gas_consumption() method_id {
  var (int gas_used, _) = invoke_method(empty_method, []);
  throw_if(101, gas_used < 500);
  throw_if(102, gas_used > 700);
  return gas_used;
}
```

### 返回多個值

讓我們回到我們的智能合約，利用新測試可以返回多個值的特性。要返回多個值，您需要將它們放在一個元組中。讓我們嘗試向智能合約發送三個相同的消息。

函數框架：

```func
[int, int, int] __test_can_return_complex_type_from_test() {
}
```

我們將使用輔助函數 `set_default_initial_data()` 來設置 "起始狀態"，這是我們為第一個測試編寫的。我們還將收集消息 cell。

```func
[int, int, int] __test_can_return_complex_type_from_test() {
  set_default_initial_data();

  cell message = begin_cell().store_uint(10, 32).end_cell();
}
```

唯一剩下的就是發送消息三次，並將返回的 gas 值封裝成一個元組。

```func
[int, int, int] __test_can_return_complex_type_from_test() {
  set_default_initial_data();

  cell message = begin_cell().store_uint(10, 32).end_cell();

  (int gas_used1, _) = invoke_method(recv_internal, [message.begin_parse()]);
  (int gas_used2, _) = invoke_method(recv_internal, [message.begin_parse()]);
  (int gas_used3, _) = invoke_method(recv_internal, [message.begin_parse()]);

  return [gas_used1, gas_used2, gas_used3];
}
```

### 測試鏈

新測試允許通過保持它們返回的數據在堆棧上創建測試鏈。

> 即，測試函數執行完畢後，數據不會保留在 c4 中，但您可以將數據放在堆棧上。

讓我們將三個數字放在堆棧上作為一個測試函數。

```func
(int, int, int) __test_can_return_more_than_one_stack_entry() {
  return (1, 2, 3);
}
```

而另一個函數檢查堆棧上是否有值。為此，我們需要一個輔助函數，該函數將返回當前的堆棧深度。

讓我提醒您，FunC 支持用彙編語言定義函數（即 Fift）。這發生如下 - 我們將函數定義為低級 TVM 原語。對於返回當前堆棧深度的函數，它看起來像這樣：

```func
int stack_depth() asm "DEPTH";
```

如您所見，使用了 `asm` 關鍵字。

您可以在 [TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 中第 77 頁後面查看可能的原語列表。

因此，讓我們檢查堆棧深度不為零。

```func
int __test_check_stack_depth_after_prev_test() {
  int depth = stack_depth();
  throw_if(100, depth != 0);
  return 0;
}
```

感謝這樣的機制，可以實現複雜的測試鏈。

## 結論

`toncli` 中的新測試大大簡化了 TON 網絡中智能合約的測試，這將使您能夠快速且高質量地開發 TON 網絡的應用程序。我在 [這裡](https://t.me/ton_learn) 撰寫有關 TON 網絡技術組件的課程和文章，歡迎您的訂閱。