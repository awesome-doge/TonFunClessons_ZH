# 課程 2：為智能合約編寫 FunC 測試
## 介紹

在這個教學中，我們將為在第一課中使用 FunC 語言編寫的智能合約編寫測試，並使用 [toncli](https://github.com/disintar/toncli) 在 The Open Network 測試網上執行這些測試。

## 必要條件

完成此教學，您需要安裝 [toncli](https://github.com/disintar/toncli/blob/master/INSTALLATION.md) 命令行界面，並完成[第一課](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/1lesson/firstlesson.md)。

## 重要提示

以下內容描述了舊版本的測試。目前可用於 func/fift 的開發版本的新 toncli 測試，相關說明[在此](https://github.com/disintar/toncli/blob/master/docs/advanced/func_tests_new.md)，新測試的課程[在此](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/11lesson/11lesson.md)。新測試的發布並不意味著舊測試的課程毫無意義 - 它們能很好地傳達邏輯，因此順利完成課程也是有意義的。還要注意，使用 `toncli run_tests` 時，可以使用 `--old` 標誌來運行舊測試。

## 第一個智能合約的測試

對於我們的第一個智能合約，我們將編寫以下測試：

- test_example - 調用 `recv_internal()` 並傳遞數字 10
- test_get_total - 測試 `get` 方法
- test_exception - 檢查添加不符合位條件的數字

## toncli 下的 FunC 測試結構

對於每個 toncli 下的 FunC 測試，我們將編寫兩個函數。第一個函數將確定我們將發送給第二個函數進行測試的數據（在 TON 中，稱之為狀態可能更準確，但我希望數據是一個更易理解的比喻）。

每個測試函數必須指定一個 `method_id`。`method_id` 測試函數應從 0 開始。

### 創建測試文件

讓我們在之前課程的代碼中，於 *tests* 文件夾中創建一個 *example.func* 文件，並在其中編寫我們的測試。

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

## 測試 `recv_internal()` 調用

讓我們編寫第一個測試 `test_example` 並分析其代碼。

### 數據函數

從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_example_data() method_id(0) {
    int function_selector = 0;

    cell message = begin_cell()     
            .store_uint(10, 32)          
            .end_cell();

    tuple stack = unsafe_tuple([message.begin_parse()]); 

    cell data = begin_cell()             
        .store_uint(0, 64)              
        .end_cell();

    return [function_selector, stack, data, get_c7(), null()];
}
```

### 分析

`int function_selector = 0;`

由於我們在調用 `recv_internal()`，我們將值設置為 0，為什麼是 0？Fift（即我們編譯 FunC 腳本所用的語言）有預定義的標識符，即：
- `main` 和 `recv_internal` 的 id = 0
- `recv_external` 的 id = -1
- `run_ticktock` 的 id = -2

```func
    cell message = begin_cell()     
            .store_uint(10, 32)          
            .end_cell();
```

在 message 單元中，我們寫入無符號的 32 位整數 10。

`tuple stack = unsafe_tuple([message.begin_parse()]);`

`tuple` 是另一個 FunC 數據類型。
Tuple - 有序的任意值集合，類型為棧值。

使用 `begin_parse()` 將 *message* 單元轉換為 *slice*，並使用 `unsafe_tuple()` 函數將其寫入 *tuple*。

```func
    cell data = begin_cell()             
        .store_uint(0, 64)              
        .end_cell();
```

在控制寄存器 c4 中放置 64 位的 0。

剩下的就是返回數據：

`return [function_selector, stack, data, get_c7(), null()];`

如您所見，我們使用 `get_c7()` 將 c7 的當前狀態放入 c7，而在 gas 限制整數中放入 `null()`。

### 測試函數

代碼：

```func
_ test_example(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(1) {
    throw_if(100, exit_code != 0);

    var ds = data.begin_parse();

    throw_if(101, ds~load_uint(64) != 10); 
    throw_if(102, gas > 1000000); 
}
```

### 分析

`throw_if(100, exit_code != 0);`

我們檢查返回代碼，如果返回代碼不等於零，則函數將拋出異常。
0 是智能合約成功執行的標準返回代碼。

```func
    var ds = data.begin_parse();

    throw_if(101, ds~load_uint(64) != 10); 
```

我們檢查我們發送的數字是否等於 10，即我們發送了 32 位的數字 10，而智能合約執行後，64 位的 10 被寫入控制寄存器 c4。

具體來說，如果不是 10，我們會創建一個異常。

`throw_if(102, gas > 1000000);`

儘管我們在第一課解決的問題中沒有對 gas 的使用設置限制，但在智能合約的測試中，不僅要檢查執行邏輯，還要確保邏輯不會導致非常高的 gas 消耗，否則合約在主網上將無法運行。

## 測試 Get 函數調用

讓我們編寫 `test_get_total` 測試並分析其代碼。

### 數據函數

從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_get_total_data() method_id(2) {
    int function_selector = 128253; 
    
    tuple stack = unsafe_tuple([]); 

    cell data = begin_cell()            
        .store_uint(10, 64)              
        .end_cell();

    return [function_selector, stack, data, get_c7(), null()];
}
```

### 分析

`int function_selector = 128253;`

要了解 GET 函數的 id，您需要打開編譯的智能合約，查看分配給該函數的 id。進入 `build` 文件夾，打開 `contract.fif`，找到 `get_total` 的那一行：

`128253 DECLMETHOD get_total`

對於 `get_total` 函數，我們不需要傳遞任何參數，因此我們只需聲明一個空的元組：

`tuple stack = unsafe_tuple([]);`

並在 c4 中寫入 10 以便驗證。

```func
cell data = begin_cell()            
    .store_uint(10, 64)              
    .end_cell();
```

### 測試函數

代碼：

```func
_ test_get_total(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(3) {
    throw_if(103, exit_code != 0); 
    int counter = first(stack); 
    throw_if(104, counter != 10); 
}
```

### 分析

`throw_if(103, exit_code != 0);`

檢查返回代碼。

```func
    int counter = first(stack); 
    throw_if(104, counter != 10); 
```

在我們的測試中，我們需要確保我們傳遞的值 10 是棧的頂部，因此我們使用標準庫 [stdlib.fc](https://ton-blockchain.github.io/docs/#/func/stdlib?id=first) 中的 `first` 函數提取元組的第一個值。

## 測試異常

讓我們編寫 `test_exception` 測試並分析其代碼。

### 數據函數

從數據函數開始：

```func
[int, tuple, cell, tuple, int] test_exception_data() method_id(4) {
    int function_selector = 0;

    cell message = begin_cell()     
            .store_uint(30, 31)           
            .end_cell();

    tuple stack = unsafe_tuple([message.begin_parse()]);

    cell data = begin_cell()            
        .store_uint(0, 64)               
        .end_cell();

    return [function_selector, stack, data, get_c7(), null()];
}
```

### 分析

如我們所見，與我們的第一個函數的區別最小，即我們放入元組中的值為 30 的 31 位。

```func
    cell message = begin_cell()     
            .store_uint(30, 31)           
            .end_cell();
```

但在測試函數中，差異會更明顯。

### 測試函數

```func
_ test_exception(int exit_code, cell data, tuple stack, cell actions, int gas) method_id(5) {
    throw_if(100, exit_code == 0);
}
```

與其他測試函數不同，我們期望如果智能合約成功執行，則拋出異常。

## 運行測試

為了讓 toncli "理解" 測試的位置，需要在 `project.yaml` 中添加信息。

```yaml
contract:
  data: fift/data.fif
  func:
    - func/code.func
  tests:
    - tests/example.func
```

現在我們使用命令運行測試：

```shell
toncli run_tests
```

> 如果您使用的是 func 的開發版本，則需要添加 `--old` 標誌，即 `toncli run_tests --old`

應該會得到以下結果：

![toncli get send](./img/run_tests.png)

P.S 如果您有任何問題，建議在[這裡](https://t.me/ton_learn)提問。