# 簡介
FunC 語言用於在 TON 區塊鏈上編寫智能合約。合約邏輯在 TON 虛擬機(TVM)中執行，TVM 是基於堆疊的 TON 虛擬機。

# 第一部分 基本語法，第一個智能合約 — 資料類型、儲存、函數

	;; 單行註解
	{- 這是多行註解
		{- 這是註解中的註解 -}
	-}
	
	(int) sum(int a, int b) { ;; 這是一個接收兩個整數參數並返回整數結果的函數
	  return a + b;  ;; 所有整數都是有符號的，長度為257位。溢出會拋出異常
	  ;; 表達式必須以分號結尾
	}
	
	() f(int i, cell c, slice s, builder b, cont c, tuple t) {
	  ;; FunC 有 7 種原子類型: 
	  ;; int - 257 位有符號整數，
	  ;; cell - TON 不透明數據結構的基礎，包含最多 1,023 位和最多 4 個對其他 cell 的引用，
	  ;; slice 和 builder - 用於從 cell 讀取和寫入的特殊對象，
	  ;; continuation - 包含可立即執行的 TVM 字節碼的另一種 cell
	  ;; tuple 是最多 255 個組件的有序集合，具有任意值類型，可能是不同的。
	  ;; 張量類型 (A,B, ...) 是一個有序集合，可以像這樣進行賦值: (int, int) a = (3, 5);
	  ;; 張量類型的特殊情況是單位類型 (())。
	  ;; 它表示函數不返回任何值，或者沒有參數。
	}
	
	;; 在執行期間，合約可以讀取本地上下文: 儲存、餘額、時間、網絡配置等。
	;; 合約可以更改其儲存和代碼，還可以向其他合約發送消息。
	;; 讓我們編寫一個計數器智能合約，該合約從傳入消息中獲取一個數字，
	;; 將其添加到已存儲的數字中並將結果存儲在“儲存”中。
	;; 對於處理特殊事件，智能合約有保留的方法:
	;; recv_internal() 處理來自其他智能合約的內部消息
	;; recv_external() 處理來自外部世界（例如用戶）的外部消息。
	() recv_internal(slice in_msg_body) {
	  ;; 在堆疊式的 TVM 中，cell 扮演內存的角色。可以將 cell 轉換為 slice，
	  ;; 然後可以從 slice 中加載數據位和對其他 cell 的引用
	  ;; 通過將它們從 slice 中加載出來。數據位和對其他 cell 的引用可以存儲
	  ;; 到 builder 中，然後將 builder 完成為新的 cell。
	  ;; recv_internal 以帶有傳入消息數據的 slice 作為參數。
	  ;; 與 TON 上的其他所有內容一樣，永久儲存數據以 cell 的形式存儲。
	  ;; 可以通過 get_data() 方法檢索它
	  ;; begin_parse - 將帶有數據的 cell 轉換為可讀的 slice
	  slice ds = get_data().begin_parse(); ;; `.` 是一種語法糖: a.b() 等效於 b(a)
	  ;; load_uint 是 FunC 標準庫中的一個函數；它從 slice 中加載一個無符號的 n 位整數
	  int total = ds~load_uint(64); ;; `~` 是一個“修改”方法:
	  ;; 本質上，它是一種語法糖: `r = a~b(x)` 等效於 (a,r) = b(a,x)
	  
	  ;; 現在讓我們從消息體 slice 中讀取傳入值
	  int n = in_msg_body~load_uint(32);
	  total += n; ;; 整數支持常規的 +-*/ 操作，以及 (+-*/)= 語法糖
	  ;; 為了保持存儲的整數值，我們需要做四件事:
	  ;; 為未來的 cell 創建一個 Builder - begin_cell()
	  ;; 將值寫入 total - store_uint(value, bit_size)
	  ;; 將 Builder 轉換為 Cell - end_cell()
	  ;; 將結果 cell 寫入永久儲存 - set_data()
	  set_data(begin_cell().store_uint(total, 64).end_cell());
	}
	;; FunC 程序本質上是一系列函數聲明/定義和全局變量聲明。
	;; FunC 中的任何函數都符合以下模式:
	;; [<forall declarator>] <return_type> <function_name>(<comma_separated_function_args>) <specifiers>
	;; Specifiers:
	;; impure 指示不應優化函數調用（無論其結果是否使用）
	;; 對於更改智能合約數據或發送消息的方法，這很重要
	;; method_id 允許您按名稱調用 GET 函數
	
	;; 例如，我們可以為上面的合約創建一個 get 方法，以允許外部查看器讀取計數器
	int get_total() method_id {
	  slice ds = get_data().begin_parse();
	  int total = ds~load_uint(64);
	  ;; 注意 (int) 和 int 是相同的，因此函數聲明和返回語句中的括號被省略。
	  return total;
	}
	;; 現在任何觀察者都可以通過 lite-client 或 explorer 讀取 get_total 的值

# 第二部分 消息
演員模型是並發計算模型，是 TON 智能合約的核心。每個智能合約一次只能處理一個消息，更改自己的狀態或發送一個或多個消息。消息的處理在一個事務中進行，即它不能被中斷。對於一個合約的消息是依次處理的。因此，每個事務的執行是本地的，可以在區塊鏈層面上進行並行處理，這允許按需通過水平擴展提供吞吐量和托管無限數量的用戶和交易。

	;; 對於正常的內部消息觸發的事務，在將控制傳遞給 recv_internal 之前，TVM 將以下元素放在堆疊上。
	;;;; 智能合約餘額（以 nanoTons 為單位）
	;;;; 傳入消息餘額（以 nanoTons 為單位）
	;;;; 帶有傳入消息的 cell
	;;;; 傳入消息體，slice 類型
	;; 而 recv_internal 只能使用所需數量的字段（例如上面的例子中的 1 或下面的 4）
	
	;; 讓我們深入研究消息發送
	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  ;; 
	  ;; 每個消息都有一個嚴格的佈局，因此通過解析它，我們可以獲得發送者的地址
	  ;; 首先，我們需要讀取一些技術標誌，然後使用 FunC 標準庫中的 load_msg_addr 函數獲取地址 - ()
	  var cs = in_msg_full.begin_parse();
	  var flags = cs~load_uint(4);
	  slice sender_address = cs~load_msg_addr();
	  ;; 如果我們想發送一條消息，我們首先需要構造它
	  throw(101);    ;; 無條件拋出代碼為 101 的異常
	}
# 第三部分 流程控制: 條件語句和循環; 字典

	;; FunC 當然支持 if 語句
	;;;; 常規的 if-else
	if (flag) {
	  ;; 做某事();
	}
	else {
	  ;; 做其他事();
	}
	;; if 語句通常用作智能合約的操作標識符，例如:
	() recv_internal (int balance, int msg_value, cell in_msg_full, slice in_msg_body) {
	  int op = in_msg_body~load_int(32);
	  if (op == 1) {…}
	}
	;; 循環
	;; FunC 支持 repeat、while 和 do { ... } until 循環。不支持 for 循環。
	;; repeat
	int x = 1;
	repeat(10) {
	  x *= 2;
	}
	;; x = 1024
	;; while
	int x = 2;
	while (x < 100) {
	  x = x * x;
	}
	;; x = 256
	;; until 循環
	int x = 0;
	do {
	  x += 3;
	} until (x % 17 == 0);
	;; x = 51
	;; 在實踐中，TON 智能合約中的循環通常用於處理字典，或者在 TON 中也稱為哈希映射
	;; 哈希映射是一種由樹表示的數據結構。哈希映射將鍵映射到任意類型的值，以便可以快速查找和修改。
	;; udict_get_next? 函數與循環結合使用，可以遍歷字典
	int key = -1;
	do {
	  (key, slice cs, int f) = dic.udict_get_next?(256, key);
	} until (~ f);
	;; udict_get_next? - 計算字典 dict 中大於某個給定值的最小鍵 k，並返回 k、關聯的值和表示成功的標誌。如果字典為空，則返回 (null, null, 0)。

# 第四部分 函數
	;; 最有用的函數是 slice 讀取器和 builder 寫入器原始函數、儲存處理程序和發送消息
	;; slice begin_parse(cell c) - 將 cell 轉換為 slice
	;; (slice, int) load_int(slice s, int len) - 從 slice 中加載一個有符號的 len 位整數。
	;; (slice, int) load_uint(slice s, int len) - 從 slice 中加載一個無符號的 len 位整數。
	;; (slice, slice) load_bits(slice s, int len) - 從 slice 中加載前 0 ≤ len ≤ 1023 位到一個獨立的 slice 中。
	;; (slice, cell) load_ref(slice s) - 從 slice 中加載引用 cell。
	;; builder begin_cell() - 創建一個新的空 builder。
	;; cell end_cell(builder b) - 將 builder 轉換為普通的 cell。
	;; builder store_int(builder b, int x, int len) - 將一個有符號的 len 位整數 x 存儲到 b 中，其中 0 ≤ len ≤ 257。
	;; builder store_uint(builder b, int x, int len) - 將一個無符號的 len 位整數 x 存儲到 b 中，其中 0 ≤ len ≤ 256。
	;; builder store_slice(builder b, slice s) - 將 slice s 存儲到 builder b 中。
	;; builder store_ref(builder b, cell c) - 將對 cell c 的引用存儲到 builder b 中。
	;; cell get_data() - 返回持久合約儲存 cell。
	;; () set_data(cell c) - 將 cell c 設置為持久合約數據。
	
	;; () send_raw_message(cell msg, int mode) - 將消息 msg 以 mode 放入發送隊列。請注意，消息將在整個事務成功執行後發送
	
	;; 所有標準函數的詳細描述可以在文檔中找到 https://ton-blockchain.github.io/docs/#/func/stdlib
  
# 進一步閱讀

	{- 
		我對 TON 生態系統非常感興趣，所以我寫了一些關於開發智能合約的課程: https://github.com/romanovichim/TonFunClessons_Eng
		我還有一個頻道，我在其中介紹 TON 的技術方面: https://t.me/ton_learn

	-}  


