## 簡介

在這篇教程中，我們將分析聊天機器人智能合約。我們需要它來了解如何在測試中檢查交易以及進行鏈上測試。

## 關於 TON

TON 是一種[Actor 模型](https://en.wikipedia.org/wiki/Actor_model)，這是一種數學並行計算模型，是 TON 智能合約的基礎。在這個模型中，每個智能合約可以在一個單位時間內接收一條消息、改變自己的狀態，或發送一條或多條消息。

通常，要在 TON 上創建一個完整的應用程序，需要編寫幾個智能合約，這些合約通過消息相互通信。為了讓合約能夠理解當消息到達時需要執行什麼操作，建議使用 `op`。`op` 是一個 32 位的標識符，應該在消息體中傳遞。

因此，在消息內使用條件語句，根據智能合約的 `op` 執行不同的操作。

因此，能夠測試消息是非常重要的，這就是我們今天要做的。

聊天機器人智能合約接收任何內部消息並用帶有回復文本的內部消息回應它。

## 解析合約

##### 標準庫

首先需要[導入標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib)。該庫只是最常見的 TVM（TON 虛擬機）命令的包裝，不是內置的。

```func
#include "imports/stdlib.fc";
```

為了處理內部消息，我們需要 `recv_internal()` 方法

```func
() recv_internal()  {

}
```
	
##### 外部方法參數

這裡有一個合乎邏輯的問題——如何理解函數應該具有哪些參數，以便它能夠接收 TON 網絡上的消息？

根據 [TON 虛擬機 - TVM](https://ton-blockchain.github.io/docs/tvm.pdf) 的文檔，當 TON 鏈中的一個賬戶發生事件時，它會觸發一個交易。

每筆交易最多包含 5 個階段。詳細信息[見這裡](https://ton-blockchain.github.io/docs/#/smart-contracts/tvm_overview?id=transactions-and-phases)。

我們感興趣的是**計算階段**。具體來說，是初始化期間 "在堆棧上" 的內容。對於正常的觸發後交易，堆棧的初始狀態如下：

5 個元素：
- 智能合約餘額（以納噸為單位）
- 收到消息的餘額（以納噸為單位）
- 包含收到消息的 Cell
- 收到消息體，類型為 slice
- 函數選擇器（對於 recv_internal 是 0）

```func
() recv_internal(int balance, int msg_value, cell in_msg_full, slice in_msg_body)  {

}
```
	
但是，並不需要在 `recv_internal()` 中寫出所有參數。通過設置參數，我們告訴智能合約代碼它們中的一些參數。那些代碼不會知道的參數將簡單地位於堆棧底部，從不被觸及。對於我們的智能合約，這是：

```func
	() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

	}
```

##### 處理消息所需的 Gas

我們的智能合約需要使用 Gas 來發送消息，因此我們將檢查消息到達的 msg_value，如果它非常小（小於 0.01 TON），我們將使用 `return()` 結束智能合約的執行。

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

  if (msg_value < 10000000) { ;; 10000000 納噸 == 0.01 TON
	return ();
  }
  
}
```

##### 獲取地址

要發送回復消息，需要獲取發送者的地址。為此，需要解析 `in_msg` cell。

為了獲取地址，我們需要使用 `begin_parse` 將 cell 轉換為 slice：

```func
var cs = in_msg_full.begin_parse();
```

現在我們需要 "減載" 獲得的 slice 到地址。使用 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 中的 `load_uint` 函數，它從 slice 中加載一個無符號的 n 位整數，"減載" 標誌。

```func
var flags = cs~load_uint(4);
```

在本課中，我們不會詳細介紹標誌，但你可以在[這裡](https://ton-blockchain.github.io/docs/tblkch.pdf)閱讀更多。

最後，獲取地址。使用 `load_msg_addr()` - 從 slice 中加載唯一的有效 MsgAddress 前綴。

```func
slice sender_address = cs~load_msg_addr(); 
```
	
代碼：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

  if (msg_value < 10000000) { ;; 10000000 納噸 == 0.01 TON
	return ();
  }
  
  slice cs = in_msg.begin_parse();
  int flags = cs~load_uint(4); 
  slice sender_address = cs~load_msg_addr(); 

}
```

##### 發送消息

現在需要發送一條回復消息。

##### 消息結構

完整的消息結構可以在[這裡 - 消息佈局](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=message-layout)找到。但通常我們不需要控制每個字段，因此可以使用[示例](https://ton-blockchain.github.io/docs/#/smart-contracts/messages?id=sending-messages)中的簡化形式：

```func
 var msg = begin_cell()
	.store_uint(0x18, 6)
	.store_slice(addr)
	.store_coins(amount)
	.store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	.store_slice(message_body)
  .end_cell();
```

如你所見，消息的構建使用了 [FunC 標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib) 的函數。即 Builder 原語的 "包裝器" 函數（部分構建的 cell，這是你可能從第一課記得的）。考慮：

`begin_cell()` - 將創建未來 cell 的 Builder
`end_cell()` - 將創建一個 Cell
`store_uint` - 在 Builder 中存儲 uint
`store_slice` - 在 Builder 中存儲 slice
`store_coins` - 這裡文檔指的是 `store_grams` - 用於存儲 TonCoins。更多細節[這裡](https://ton-blockchain.github.io/docs/#/func/stdlib?id=store_grams)。

##### 消息體

在消息體中，我們放入 `op` 和我們的消息 `reply`，為了放入消息，我們需要做 `slice`。

```func
slice msg_text = "reply";
```

在消息體的建議中，有建議添加 `op`，儘管在這裡不會有任何功能，我們還是會添加它。

為了在智能合約中創建類似的客戶端-服務器架構，建議將每個消息（嚴格來說，是消息體）以某個 `op` 標誌開頭，這將標識智能合約應該執行的操作。

讓我們在消息中設置 `op` 為 0。

現在代碼如下所示：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

  if (msg_value < 10000000) { ;; 10000000 納噸 == 0.01 TON
	return ();
  }
	  
  slice cs = in_msg.begin_parse();
  int flags = cs~load_uint(4); 
  slice sender_address = cs~load_msg_addr(); 

  slice msg_text = "reply"; 

  cell msg = begin_cell()
	  .store_uint(0x18, 6)
	  .store_slice(sender_address)
	  .store_coins(100) 
	  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	  .store_uint(0, 32)
	  .store_slice(msg_text) 
  .end_cell();

	}
```

消息準備好了，現在發送它。

##### 消息發送模式(mode)

要發送消息，使用 [標準庫](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message) 中的 `send_raw_message`。

我們已經收集了 msg 變量，剩下的是弄清楚 `mode`。每個模式的描述在[文檔](https://ton-blockchain.github.io/docs/#/func/stdlib?id=send_raw_message)中。讓我們看看一個示例以更清楚。

假設智能合約餘額為 100 個硬幣，我們收到一個內部消息，帶有 60 個硬幣，並發送一個帶有 10 個硬幣的消息，總費用為 3。

  `mode = 0` - 餘額 (100+60-10 = 150 個硬幣)，發送(10-3 = 7 個硬幣)
  `mode = 1` - 餘額 (100+60-10-3 = 147 個硬幣)，發送(10 個硬幣)
  `mode = 64` - 餘額 (100-10 = 90 個硬幣)，發送 (60+10-3 = 67 個硬幣)
  `mode = 65` - 餘額 (100-10-3=87 個硬幣)，發送 (60+10 = 70 個硬幣)
  `mode = 128` -餘額 (0 個硬幣)，發送 (100+60-3 = 157 個硬幣)

選擇 `mode`，讓我們參考[文檔](https://docs.ton.org/develop/smart-contracts/messages#message-modes)：

- 我們正在發送一條普通消息，所以 mode 是 0。
- 我們將分開支付傳輸的費用，因此 +1。
- 我們還將忽略在操作階段處理此消息期間發生的任何錯誤，因此 +2。

我們得到 `mode` == 3，最終的智能合約：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

  if (msg_value < 10000000) { ;; 10000000 納噸 == 0.01 TON
	return ();
  }
	  
  slice cs = in_msg.begin_parse();
  int flags = cs~load_uint(4); 
  slice sender_address = cs~load_msg_addr(); 

  slice msg_text = "reply"; 

  cell msg = begin_cell()
	  .store_uint(0x18, 6)
	  .store_slice(sender_address)
	  .store_coins(100) 
	  .store_uint(0, 1 + 4 + 4 + 64 + 32 + 1 + 1)
	  .store_uint(0, 32)
	  .store_slice(msg_text) 
  .end_cell();

  send_raw_message(msg, 3);
}
```
## hexBoC

在部署智能合約之前，需要將其編譯為 hexBoС，讓我們從上一個教程中拿取項目。

將 `main.fc` 重命名為 `chatbot.fc`，並將我們的智能合約寫入其中。

由於我們更改了文件名，我們也需要更新 `compile.ts`：
```ts
import * as fs from "fs";
import { readFileSync } from "fs";
import process from "process";
import { Cell } from "ton-core";
import { compileFunc } from "@ton-community/func-js";

async function compileScript() {

	const compileResult = await compileFunc({
		targets: ["./contracts/chatbot.fc"], 
		sources: (path) => readFileSync(path).toString("utf8"),
	});

	if (compileResult.status ==="error") {
		console.log("發生錯誤");
		process.exit(1);
	}

	const hexBoC = 'build/main.compiled.json';

	fs.writeFileSync(
		hexBoC,
		JSON.stringify({
			hex: Cell.fromBoc(Buffer.from(compileResult.codeBoc,"base64"))[0]
				.toBoc()
				.toString("hex"),
		})

	);

	console.log("已編譯，hexBoC:"+hexBoC);

}

compileScript();
```
使用 `yarn compile` 命令編譯智能合約。

你現在有了智能合約的 `hexBoC` 表示。

## 結論

在下一篇教程中，我們將編寫鏈上測試。