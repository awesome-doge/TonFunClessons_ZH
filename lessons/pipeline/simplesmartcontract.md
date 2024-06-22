# 智能合約部署管道 Part1 - 撰寫智能合約

## 簡介

在 TON 區塊鏈中使用智能合約的現代工具是 [blueprint](https://github.com/ton-org/blueprint/)，它允許你快速創建專案結構並立即開始便利的開發。在我的 [教程](https://github.com/romanovichim/TonFunClessons_Eng) 中，智能合約開發語言 FunC 使用了 blueprint。

要成功使用 blueprint，你需要能夠使用其各種組件，因此在這系列教程中，我們將涵蓋以下內容：

- 創建專案、簡單的智能合約，並使用 https://github.com/ton-community/func-js 進行編譯
- 使用 https://github.com/ton-org/sandbox 測試智能合約
- 使部署到測試網路變得方便：生成 QR 碼，在錢包中確認
- TON 是一個 actor 模型——智能合約通過消息相互通信——我們將撰寫一個回應消息的聊天機器人智能合約
- 測試聊天機器人智能合約並學習如何測試發送消息的智能合約

讓我們從創建一個簡單的智能合約並編譯它開始。

## 專案初始化

為你的專案創建一個資料夾並進入其中。

```bash
// Windows 範例
mkdir test_folder
cd test_folder
```

在這個教程中，我們將使用 `yarn` 套件管理器
```bash
yarn init
```

初始化 `yarn` 並回答控制台中的問題，因為這是一個測試專案。完成後，資料夾中應該有一個 package.json 文件。

現在讓我們添加 TypeScript 和必要的庫，將它們安裝為開發依賴：
```bash
yarn add typescript ts-node @types/node @swc/core --dev
```

創建一個 `tsconfig.json` 文件，用於專案編譯配置。添加以下內容：

```json
{
	"compilerOptions": {
		"target" : "es2020",
		"module" : "commonjs",
		"esModuleInterop" : true,
		"forceConsistentCasingInFileNames": true,
		"strict" : true,
		"skipLibCheck" : true,
		"resolveJsonModule" : true
	},
	"ts-node": {
		"transpileOnly" : true,
		"transpile" : "ts-node/transpilers/swc"
	}
}
```

在這個教程中，我們不會深入解釋每行配置的含義，因為這個教程是關於智能合約的。現在讓我們安裝 TON 相關的庫：
```bash
yarn add ton-core ton-crypto @ton-community/func-js  --dev
```

現在讓我們在 FunC 中創建一個智能合約。在 `contracts` 資料夾中創建 `main.fc` 文件，並添加最小代碼：

```func
() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

} 
```

`recv_internal` 在智能合約收到入站內部消息時被調用。當 [TVM 初始化](https://docs.ton.org/learn/tvm-instructions/tvm-overview#initialization-of-tvm) 時，堆疊中會有一些變量，通過在 recv_internal 中設置參數，我們告訴智能合約代碼這些變量。

現在讓我們撰寫一個腳本來編譯我們的智能合約模板。在 `scripts` 資料夾中創建 `compile.ts` 文件。

為了使這個腳本可用，我們需要將它作為參數註冊到包管理器中，即在 `package.json` 文件中，如下所示：

```json
{
  "name": "test_folder",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "devDependencies": {
	"@swc/core": "^1.3.63",
	"@ton-community/func-js": "^0.6.2",
	"@types/node": "^20.3.1",
	"ton-core": "^0.49.1",
	"ton-crypto": "^3.2.0",
	"ts-node": "^10.9.1",
	"typescript": "^5.1.3"
  },
  "scripts": {
	  "compile" : "ts-node ./scripts/compile.ts"
  }
}
```

現在讓我們撰寫 `compile.ts` 文件中的編譯腳本。在這裡需要注意的是，編譯結果將是 [bag of Cell](https://docs.ton.org/develop/data-formats/cell-boc) 格式的 base64 編碼字符串表示。我們需要將這個結果保存到某個地方，因此讓我們創建一個 `build` 資料夾。

最後我們來到編譯文件，首先使用 `compileFunc` 函數編譯我們的代碼：

```ts
import * as fs from "fs";
import { readFileSync } from "fs";
import process from "process";
import { Cell } from "ton-core";
import { compileFunc } from "@ton-community/func-js";

async function compileScript() {

	const compileResult = await compileFunc({
		targets: ["./contracts/main.fc"], 
		sources: (path) => readFileSync(path).toString("utf8"),
	});

	if (compileResult.status ==="error") {
		console.log("Error happend");
		process.exit(1);
	}

}
compileScript();
```

將生成的 hexBoC 寫入資料夾：

```ts
import * as fs from "fs";
import { readFileSync } from "fs";
import process from "process";
import { Cell } from "ton-core";
import { compileFunc } from "@ton-community/func-js";

async function compileScript() {

	const compileResult = await compileFunc({
		targets: ["./contracts/main.fc"], 
		sources: (path) => readFileSync(path).toString("utf8"),
	});

	if (compileResult.status ==="error") {
		console.log("Error happend");
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
}

compileScript();
```

為方便起見，可以使用 `console.log()` 混合代碼，這樣在編譯時可以清楚地知道哪些部分工作了，哪些沒有。例如，你可以在末尾添加：

```ts
console.log("Compiled, hexBoC:"+hexBoC);
```

這將輸出生成的 hexBoC。

## 編寫合約

為了創建合約，我們需要標準的 FunC 函數庫。在 `contracts` 資料夾中創建一個 `imports` 資料夾，並添加[這個](https://github.com/ton-blockchain/ton/blob/master/crypto/smartcont/stdlib.fc)文件。

現在進入 `main.fc` 文件並導入這個庫，現在文件如下所示：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {

} 
```

讓我們簡要介紹合約，詳細的分析和 FunC 教程請參考[這裡](https://github.com/romanovichim/TonFunClessons_Eng/)。

我們將撰寫的智能合約將存儲內部消息的發送者地址，並在智能合約中存儲數字 1。它還將實現 Get 方法，當被調用時，將返回合約中最後一個消息發送者的地址和數字 1。

一個內部消息來到我們的函數，我們將首先從中獲取服務標誌，然後是發送者的地址，並將其保存：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {
	slice cs = in_msg.begin_parse();
	int flags = cs~load_uint(4);
	slice sender_address = cs~load_msg_addr();

} 
```

將地址和數字 1 保存到合約中，即寫入寄存器 `c4`：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {
	slice cs = in_msg.begin_parse();
	int flags = cs~load_uint(4);
	slice sender_address = cs~load_msg_addr();

	set_data(begin_cell().store_slice(sender_address).store_uint(1,32).end_cell());
} 
```

現在編寫 Get 方法，該方法將返回一個地址和一個數字，從 `(slice,int)` 開始：

```func
(slice,int) get_sender() method_id {

}
```

在方法中獲取寄存器中的數據並返回給用戶：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {
	slice cs = in_msg.begin_parse();
	int flags = cs~load_uint(4);
	slice sender_address = cs~load_msg_addr();

	set_data(begin_cell().store_slice(sender_address).store_uint(1,32).end_cell());
} 

(slice,int) get_sender() method_id {
	slice ds = get_data().begin_parse();
	return (ds~load_msg_addr(),ds~load_uint(32));
}
```

最終合約：

```func
#include "imports/stdlib.fc";

() recv_internal(int msg_value, cell in_msg, slice in_msg_body) impure {
	slice cs = in_msg.begin_parse();
	int flags = cs~load_uint(4);
	slice sender_address = cs~load_msg_addr();

	set_data(begin_cell().store_slice(sender_address).store_uint(1,32).end_cell());
} 

(slice,int) get_sender() method_id {
	slice ds = get_data().begin_parse();
	return (ds~load_msg_addr(),ds~load_uint(32));
}
```

使用 `yarn compile` 命令開始編譯，並在 `build` 資料夾中獲取 `main.compiled.json` 文件：

```json
{"hex":"b5ee9c72410104010035000114ff00f4a413f4bcf2c80b0102016203020015a1418bda89a1f481a63e610028d03031d0d30331fa403071c858cf16cb1fc9ed5474696b07"}
```

## 結論

下一步是為這個智能合約撰寫[測試](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/pipeline/simpletest.md)，感謝你的關注。