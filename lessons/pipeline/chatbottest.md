# Chatbot 智能合約測試

在這篇教程中，我們將為聊天機器人智能合約撰寫測試。主要目的是學習如何在 `@ton-community/sandbox` 中仔細檢查交易，並了解如何在測試網路上進行測試，也就是所謂的鏈上測試。

我們先從常規測試開始。

## 檢查是否存在交易

由於我們使用了上次教程的草稿作為模板，我們已經有了測試框架，打開 `main.spec.ts` 文件並移除所有與 GET 方法相關的內容：

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

        const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(),toNano("0.05"));

        expect(sentMessageResult.transactions).toHaveTransaction({
            from: senderWallet.address,
            to: myContract.address,
            success: true,
        });

    });
});
```    

我們看到目前檢查的是交易是否已發送到我們的智能合約。這是通過 `sentMessageResult.transactions` 對象實現的。讓我們仔細看看它，看看可以基於這個對象進行哪些測試。

如果我們只是將此對象打印到控制台，它會包含大量的原始信息，為了方便，我們將使用 `@ton-community/test-utils` 中的 `flattenTransaction`：

``` ts
import { Cell, Address, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";
import { flattenTransaction } from "@ton-community/test-utils";

describe("msg test", () => {
    it("test", async() => {
        const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

        const blockchain = await Blockchain.create();

        const myContract = blockchain.openContract(
            await MainContract.createFromConfig({}, codeCell)
        );

        const senderWallet = await blockchain.treasury("sender");

        const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(),toNano("0.05"));

        expect(sentMessageResult.transactions).toHaveTransaction({
            from: senderWallet.address,
            to: myContract.address,
            success: true,
        });

        const arr = sentMessageResult.transactions.map(tx => flattenTransaction(tx));
        console.log(arr)
    });
});
```     

你在控制台看到的內容可以用於測試，讓我們檢查聊天機器人發送的消息是否等於 "reply"。

讓我們按照我們在智能合約中組裝消息的方式來組裝消息。

```ts
let reply = beginCell().storeUint(0, 32).storeStringTail("reply").endCell();
``` 

現在，使用消息來檢查是否存在這樣的交易：

```ts
import { Cell, Address, toNano, beginCell } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { Blockchain } from "@ton-community/sandbox";
import { MainContract } from "../wrappers/MainContract";
import { send } from "process";
import "@ton-community/test-utils";
import { flattenTransaction } from "@ton-community/test-utils";

describe("msg test", () => {
    it("test", async() => {
        const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];

        const blockchain = await Blockchain.create();

        const myContract = blockchain.openContract(
            await MainContract.createFromConfig({}, codeCell)
        );

        const senderWallet = await blockchain.treasury("sender");

        const sentMessageResult = await myContract.sendInternalMessage(senderWallet.getSender(),toNano("0.05"));

        expect(sentMessageResult.transactions).toHaveTransaction({
            from: senderWallet.address,
            to: myContract.address,
            success: true,
        });

        //const arr = sentMessageResult.transactions.map(tx => flattenTransaction(tx));

        let reply = beginCell().storeUint(0, 32).storeStringTail("reply").endCell();

        expect(sentMessageResult.transactions).toHaveTransaction({
            body: reply,
            from: myContract.address,
            to: senderWallet.address
        });

    });
});
``` 

運行 `yarn test` 命令查看一切是否正常工作。通過這種方式，我們可以在測試中組裝與智能合約中相同的對象，並檢查交易是否存在。

## 鏈上測試

有時可能需要在測試網路上運行智能合約（當有很多合約時）。讓我們嘗試使用這個範例。

在 `scripts` 文件夾中，我們將創建 `onchain.ts` 文件，為了便於啟動，在 `package.json` 中添加 `"onchain": "ts-node ./scripts/onchain.ts"`：

```json
"scripts": {
    "compile": "ts-node ./scripts/compile.ts",
    "test": "yarn jest",
    "deploy": "yarn compile && ts-node ./scripts/deploy.ts",
    "onchain": "ts-node ./scripts/onchain.ts"
},
```

第一件需要做的是為測試獲取智能合約地址：

```ts
import { Cell, beginCell, contractAddress, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { TonClient } from "ton";

async function onchainScript() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });

    console.log("Address: ", address);
}

onchainScript();
``` 

測試網路的測試將提供一個 QR 碼來通過 Tonkeeper 部署交易，並每 10 秒檢查一次是否在網路上出現了回應。

> 當然，這是為了示例而簡化，重點是展示邏輯。

我們將生成一個 QR 碼，通過它可以通過 Tonkeeper 進行交易。對於我們的示例，重要的是 TON 的數量足夠，這樣不會觸發合約中寫的異常。

```ts
import { Cell, beginCell, contractAddress, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { TonClient } from "ton";
import qs from "qs";
import qrcode from "qrcode-terminal";

async function onchainScript() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });

    console.log("Address: ", address);

    let transactionLink =
        'https://app.tonkeeper.com/transfer/' +
        address.toString({
            testOnly: true,
        }) +
        "?" +
        qs.stringify({
            text: "Sent simple in",
            amount: toNano("0.6").toString(10),
        });

    console.log("Transaction link:", transactionLink);

    qrcode.generate(transactionLink, { small: true }, (qr) => {
        console.log(qr);
    });
}

onchainScript();
``` 

為了從測試網路獲取數據，我們需要一些數據來源。數據可以通過 ADNL 從 Liteservers 獲取，但我們將在後續教程中介紹 ADNL。在這篇教程中，我們將使用 TON Center API。

```ts
const API_URL = "https://testnet.toncenter.com/api/v2";
``` 

我們將通過 Http 客戶端 [axios](https://axios-http.com/ru/docs/intro) 來發送請求，安裝：`yarn add axios`。

在 Toncenter 方法中，我們需要帶有 limit 參數 1 的 getTransactions，即我們將獲取最後一個交易。讓我們寫兩個輔助函數來請求信息：

```ts
// axios http client // yarn add axios
async function getData(url: string): Promise<any> {
    try {
        const config: AxiosRequestConfig = {
            url: url,
            method: "get",
        };
        const response: AxiosResponse = await axios(config);
        return response.data.result;
    } catch (error) {
        console.error(error);
        throw error;
    }
}

async function getTransactions(address: String) {
    var transactions;
    try {
        transactions = await getData(
            `${API_URL}/getTransactions?address=${address}&limit=1`
        );
    } catch (e) {
        console.error(e);
    }
    return transactions;
}
``` 

現在我們需要一個函數來間隔調用 API，為此有一個方便的方法 [SetInterval](https://developer.mozilla.org/docs/Web/API/setInterval)：

```ts
import { Cell, beginCell, contractAddress, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { TonClient } from "ton";
import qs from "qs";
import qrcode from "qrcode-terminal";
import axios, { AxiosRequestConfig, AxiosResponse } from "axios";

const API_URL = "https://testnet.toncenter.com/api/v2";

// axios http client // yarn add axios
async function getData(url: string): Promise<any> {
    try {
        const config: AxiosRequestConfig = {
            url: url,
            method: "get",
        };
        const response: AxiosResponse = await axios(config);
        return response.data.result;
    } catch (error) {
        console.error(error);
        throw error;
    }
}

async function getTransactions(address: String) {
    var transactions;
    try {
        transactions = await getData(
            `${API_URL}/getTransactions?address=${address}&limit=1`
        );
    } catch (e) {
        console.error(e);
    }
    return transactions;
}

async function onchainScript() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });

    console.log("Address: ", address);

    let transactionLink =
        'https://app.tonkeeper.com/transfer/' +
        address.toString({
            testOnly: true,
        }) +
        "?" +
        qs.stringify({
            text: "Sent simple in",
            amount: toNano("0.6").toString(10),
        });

    console.log("Transaction link:", transactionLink);

    qrcode.generate(transactionLink, { small: true }, (qr) => {
        console.log(qr);
    });

    setInterval(async () => {
        const txes = await getTransactions(address.toString());
        if (txes[0].in_msg.source === "EQCj2gVRdFS0qOZnUFXdMliONgSANYXfQUDMsjd8fbTW-RuC") {
            console.log("Last tx: " + new Date(txes[0].utime * 1000));
            console.log("IN from: " + txes[0].in_msg.source + " with msg: " + txes[0].in_msg.message);
            console.log("OUT from: " + txes[0].out_msgs[0].source + " with msg: " + txes[0].out_msgs[0].message);
        }
    }, 10000);
}

onchainScript();
```

這裡需要注意的是 API 返回的是交易而不是消息，所以我們需要檢查 IN 是否收到了我們錢包的地址（這裡我只是硬編碼了它）和消息（我們放在 QR 下），並輸出 OUT 中第一條消息的消息。我們還顯示日期，結果如下：

```ts
import { Cell, beginCell, contractAddress, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import { TonClient } from "ton";
import qs from "qs";
import qrcode from "qrcode-terminal";
import axios, { AxiosRequestConfig, AxiosResponse } from "axios";

const API_URL = "https://testnet.toncenter.com/api/v2";

// axios http client // yarn add axios
async function getData(url: string): Promise<any> {
    try {
        const config: AxiosRequestConfig = {
            url: url,
            method: "get",
        };
        const response: AxiosResponse = await axios(config);
        return response.data.result;
    } catch (error) {
        console.error(error);
        throw error;
    }
}

async function getTransactions(address: String) {
    var transactions;
    try {
        transactions = await getData(
            `${API_URL}/getTransactions?address=${address}&limit=1`
        );
    } catch (e) {
        console.error(e);
    }
    return transactions;
}

async function onchainScript() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });

    console.log("Address: ", address);

    let transactionLink =
        'https://app.tonkeeper.com/transfer/' +
        address.toString({
            testOnly: true,
        }) +
        "?" +
        qs.stringify({
            text: "Sent simple in",
            amount: toNano("0.6").toString(10),
        });

    console.log("Transaction link:", transactionLink);

    qrcode.generate(transactionLink, { small: true }, (qr) => {
        console.log(qr);
    });

    setInterval(async () => {
        const txes = await getTransactions(address.toString());
        if (txes[0].in_msg.source === "EQCj2gVRdFS0qOZnUFXdMliONgSANYXfQUDMsjd8fbTW-RuC") {
            console.log("Last tx: " + new Date(txes[0].utime * 1000));
            console.log("IN from: " + txes[0].in_msg.source + " with msg: " + txes[0].in_msg.message);
            console.log("OUT from: " + txes[0].out_msgs[0].source + " with msg: " + txes[0].out_msgs[0].message);
        }
    }, 10000);
}

onchainScript();
```

運行 `yarn onchain` 命令，掃描 QR 碼，發送交易並等待我們的交易到達。

## 結論

希望你喜歡這個系列的教程。如果你喜歡這個倉庫，請給個星星支持一下。