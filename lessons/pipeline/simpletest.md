# 智能合約部署管道 Part2 - 測試我們的智能合約

## 簡介

在[第一部分](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/pipeline/simplesmartcontract.md)中，我們創建了一個專案並撰寫了一個簡單的智能合約，現在是進行測試的時候了。

## 開始進行測試

為了進行測試，我們需要一個測試框架，我們將使用 [jest](https://jestjs.io)。我們還需要模擬區塊鏈的運作，為此我們將使用 [ton-community/sandbox](https://github.com/ton-community/sandbox)。安裝以下依賴：

```bash
yarn add @ton-community/sandbox jest ts-jest @types/jest ton --dev
```

要使用 jest 框架，我們需要一個配置文件。創建一個名為 `jest.config.js` 的文件，並添加以下內容：
```js
/** @type {import('ts-jest').JestConfigWithTsJest} */
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
};
```

創建一個用於測試的資料夾，命名為 `tests`。在該資料夾內創建一個文件 `main.spec.ts`。

讓我們運行一個簡單的測試，檢查我們是否安裝正確。將以下代碼添加到 `main.spec.ts` 文件：
```ts
describe("test tests", () => {
	it("test of test", async() => {});
});
```

使用 `yarn jest` 命令運行測試，你應該會看到測試通過。為了方便運行測試，我們將修改 `package.json` 文件。

```json
{
  "name": "third",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "devDependencies": {
	"@swc/core": "^1.3.59",
	"@ton-community/func-js": "^0.6.2",
	"@ton-community/sandbox": "^0.11.0",
	"@types/jest": "^29.5.1",
	"@types/node": "^20.2.1",
	"jest": "^29.5.0",
	"ton": "^13.5.0",
	"ton-core": "^0.49.1",
	"ton-crypto": "^3.2.0",
	"ts-jest": "^29.1.0",
	"ts-node": "^10.9.1",
	"typescript": "^5.0.4"
  },
  "scripts": {
	"compile": "ts-node ./scripts/compile.ts",
	"test": "yarn jest"
  }
}
```

現在我們將編譯好的合約和 `ton-core` 中的 `Cell` 導入到 `main.spec.ts` 文件中，以便打開合約：
```ts
import { Cell } from "ton-core";
import { hex } from "../build/main.compiled.json";

describe("test tests", () => {
	it("test of test", async() => {});
});
```

在測試中獲取包含代碼的 Cell：
```ts
import { Cell } from "ton-core";
import { hex } from "../build/main.compiled.json";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
	});
});
```

接下來使用 `@ton-community/sandbox`，首先使用本地版本的區塊鏈。
```ts
import { Cell } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();
	});
});
```

為了方便與合約交互，使用包裝器。最簡單的包裝器描述了合約的部署（即初始數據及其方法或與之交互）。

創建一個 `wrappers` 資料夾，並在其中創建一個 `MainContract.ts` 包裝器，然後在其中導入合約類型和 `ton-core`：
```ts
import { Contract } from "ton-core";
```

通過實現 `Contract` 來創建一個我們的合約類：
```ts
import { Contract } from "ton-core";

export class MainContract implements Contract {

}
```

在創建類對象時調用構造函數，撰寫構造函數並導入必要的類型——address 和 cell。
```ts
import { Address, Cell, Contract } from "ton-core";

export class MainContract implements Contract {
	constructor(
		readonly address: Address,
		readonly init?: { code: Cell, data: Cell }
	){}
}
```

為了理解為什麼構造函數是這樣的，建議從[這裡](https://docs.ton.org/develop/howto/step-by-step)開始了解。

現在需要知道最重要的是，`data` 是合約初始化時將存儲在 c4 寄存器中的數據。

為了方便，我們將從配置中獲取合約的數據，因此我們將創建一個靜態類來實現這一點。
```ts
import { Address, beginCell, Cell, Contract, contractAddress } from "ton-core";

export class MainContract implements Contract {
	constructor(
		readonly address: Address,
		readonly init?: { code: Cell, data: Cell }
	){}

	static createFromConfig(config: any, code: Cell, workchain = 0) {
		const data = beginCell().endCell();
		const init = { code, data };
		const address = contractAddress(workchain, init);

		return new MainContract(address, init);
	}
}
```

為了部署智能合約，你需要智能合約的代碼及其初始數據，這些都將放在配置中，以方便測試和部署。

回到 `main.spec.ts` 文件。現在我們有了代碼和包裝器，讓我們使用 `sandbox` 中的 `openContract` 函數來通過配置打開合約。
```ts
import { Cell, Address } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);
	});
});
```

配置現在是空的，我們之後會回到它。我們還將從 `ton-core` 導入 `Address`，這在測試中會用到。為了測試合約，我們需要一個能夠發送消息的實體，在 `sandbox` 中這是 `treasury`。
```ts
import { Cell, Address } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");
	});
});
```

所以我們需要為測試發送內部消息。因此，有必要修改我們的包裝器。讓我們在 `MainContract.ts` 中添加 `sendInternalMessage`。
```ts
import { Address, beginCell, Cell, Contract, contractAddress, ContractProvider, Sender, SendMode } from "ton-core";

export class MainContract implements Contract {
	constructor(
		readonly address: Address,
		readonly init?: { code: Cell, data: Cell }
	){}

	static createFromConfig(config: any, code: Cell, workchain = 0) {
		const data = beginCell().endCell();
		const init = { code, data };
		const address = contractAddress(workchain, init);

		return new MainContract(address, init);
	}

	async sendInternalMessage(
		provider: ContractProvider,
		sender: Sender,
		value: bigint,
	) {
		await provider.internal(sender, {
			value,
			sendMode: SendMode.PAY_GAS_SEPARATELY,
			body: beginCell().endCell(),
		});
	}
}
```

回到 `main.spec.ts` 測試文件，使用我們剛在包裝器中撰寫的方法：
```ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");

		myContract.sendInternalMessage(senderWallet.getSender(), toNano("0.05"));
	});
});
```

在包裝器中，你可以看到需要發送的 TON 值是 bigint 類型，因此測試本身使用了方便的 `toNano` 函數，將人類可讀的數字轉換為 `bigInt`。為了檢查發送消息是否正確，我們需要調用 `getMethod`，像發送消息一樣，首先需要處理包裝器，將其添加到 `MainContract.ts`：
```ts
import { Address, beginCell, Cell, Contract, contractAddress, ContractProvider, Sender, SendMode } from "ton-core";

export class MainContract implements Contract {
	constructor(
		readonly address: Address,
		readonly init?: { code: Cell, data: Cell }
	){}

	static createFromConfig(config: any, code: Cell, workchain = 0) {
		const data = beginCell().endCell();
		const init = { code, data };
		const address = contractAddress(workchain, init);

		return new MainContract(address, init);
	}

	async sendInternalMessage(
		provider: ContractProvider,
		sender: Sender,
		value: bigint,
	) {
		await provider.internal(sender, {
			value,
			sendMode: SendMode.PAY_GAS_SEPARATELY,
			body: beginCell().endCell(),
		});
	}

	async getData(provider: ContractProvider) {
		const { stack } = await provider.get("get_sender", []);
		return {
			recent_sender: stack.readAddress(),
			number: stack.readNumber(),
		};
	}
}
```

終於，我們完成了測試的所有準備步驟，現在我們可以進行測試了。為了方便，我們將安裝 `test-utils`。這個庫將使我們能夠為 Jest 測試框架使用自定義匹配器。
```bash
yarn add @ton-community/test-utils
```

將實用程序導入到測試文件中，並將發送消息的結果傳遞給變量。
```ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");

		const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(), toNano("0.05"));
	});
});
```

我們將添加第一個測試，檢查我們的消息是否通過了交易。
```ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");

		const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(), toNano("0.05"));

		expect(sentMessageResult.transactions).toHaveTransaction({
			from: senderWallet.address,
			to: myContract.address,
			success: true,
		});
	});
});
```

接下來，我們調用 get 方法並檢查返回的地址是否正確。
```ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");

		const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(), toNano("0.05"));

		expect(sentMessageResult.transactions).toHaveTransaction({
			from: senderWallet.address,
			to: myContract.address,
			success: true,
		});

		const getData = await myContract.getData();

		expect(getData.recent_sender.toString()).toBe(senderWallet.address.toString());
	});
});
```

在控制台中輸入 `yarn test` 來運行測試。如果一切正確，你應該會看到：

```
Pass
Test Suites: 1 passed, 1 total
Tests:       1 passed, 1 total
```

最後檢查我們存儲的數字，我們將用 `toEqual()` 來檢查：
```ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";

describe("test tests", () => {
	it("test of test", async() => {
		const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

		const blockchain = await Blockchain.create();

		const myContract = blockchain.openContract(
			await MainContract.createFromConfig({}, codeCell)
		);

		const senderWallet = await blockchain.treasury("sender");

		const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(), toNano("0.05"));

		expect(sentMessageResult.transactions).toHaveTransaction({
			from: senderWallet.address,
			to: myContract.address,
			success: true,
		});

		const getData = await myContract.getData();

		expect(getData.recent_sender.toString()).toBe(senderWallet.address.toString());
		expect(getData.number).toEqual(1); 
	});
});
```

## 結論

測試已通過，我們需要將合約部署到網絡中。在下一個教程中，我們將製作一個方便的部署系統。