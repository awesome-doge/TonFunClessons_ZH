# 智能合約部署管道 Part3 - 方便部署到測試網路

## 簡介

在前兩部分中，我們分析了[編譯](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/pipeline/simplesmartcontract.md)和[測試](https://github.com/romanovichim/TonFunClessons_Eng/blob/main/lessons/pipeline/simpletest.md)。在這篇教程中，我們將方便地部署合約到測試網路，為此我們將生成一個部署連結，並將其以 QR 碼的形式展示。我們會使用錢包掃描 QR 碼，然後進行部署。

可能會有一個問題，什麼是部署連結？要部署智能合約，需要發送一個包含合約資料的消息。我們會以以下方式處理，生成一個 Tonkeeper 錢包的[深層連結](https://github.com/tonkeeper/wallet-api)，將所有部署資料傳遞到這個深層連結，然後可以生成 QR 碼，掃描後錢包將部署合約，並將所有信息傳遞到區塊鏈上。

## 部署腳本

讓我們為部署到測試網路創建一個單獨的文件 `deploy.ts`。

### StateInit

[StateInit](https://docs.ton.org/develop/data-formats/msg-tlb#stateinit-tl-b) 用於向合約傳遞初始數據，並在合約部署時使用。

在頂層，無論是部署到測試網路還是主網，我們都需要向網路發送 `StateInit` 消息。`StateInit` 包含一個代碼單元格和一個數據單元格，智能合約地址大致等於：`hash(初始代碼, 初始狀態)`。

讓我們收集 `StateInit` 並發送部署消息。

在 `deploy.ts` 文件中，創建 `deployContract()` 函數：

```ts
async function deployContract() {
}

deployContract()
```

從 `ton-core` 中導入必要的附加功能，以及我們的智能合約的 `hex` 格式：

```ts
import { Cell, StateInit, beginCell, contractAddress, storeStateInit, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";

async function deployContract() {
}

deployContract()
```

我們將從合約的 `hex` 表示中獲取代碼單元格，初始數據單元格為空。現在我們有了形成 `StateInit` 的數據。

```ts
import { Cell, StateInit, beginCell, contractAddress, storeStateInit, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";

async function deployContract() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();   

    const stateInit: StateInit = {
        code: codeCell,
        data: dataCell,
    };
}

deployContract()
```

現在我們需要構建 `StateInit`，首先創建 `Builder`，我們使用 `ton-core` 中的輔助函數 `storeStateInit`：

```ts
import { Cell, StateInit, beginCell, contractAddress, storeStateInit, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";

async function deployContract() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();   

    const stateInit: StateInit = {
        code: codeCell,
        data: dataCell,
    };
    
    const stateInitBuilder = beginCell();
    storeStateInit(stateInit)(stateInitBuilder);
    const stateInitCell = stateInitBuilder.endCell();
}

deployContract()
```

如前所述，可以從源數據中獲取地址，讓我們使用 `contractAddress` 來完成這個操作：

```ts
import { Cell, StateInit, beginCell, contractAddress, storeStateInit, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";

async function deployContract() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();   

    const stateInit: StateInit = {
        code: codeCell,
        data: dataCell,
    };

    const stateInitBuilder = beginCell();
    storeStateInit(stateInit)(stateInitBuilder);
    const stateInitCell = stateInitBuilder.endCell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });
}

deployContract()
```

### QR 碼

開始生成一個用於部署合約的字符串：

安裝方便字符串生成的庫

```bash
yarn add qs @types/qs --dev
```

以及生成 QR 碼的庫：

```bash
yarn add qrcode-terminal @types/qrcode-terminal --dev
```

要發送消息，我們需要使用 deeplink [/transfer](https://github.com/tonkeeper/wallet-api#payment-urls)：

```
ton://transfer/<address>
ton://transfer/<address>?amount=<nanocoins>
ton://transfer/<address>?text=<url-encoded-utf8-text>
ton://transfer/<address>?bin=<url-encoded-base64-boc>
ton://transfer/<address>?bin=<url-encoded-base64-boc>&init=<url-encoded-base64-boc>

https://app.tonkeeper.com/transfer/<address>
https://app.tonkeeper.com/transfer/<address>?amount=<nanocoins>
https://app.tonkeeper.com/transfer/<address>?text=<url-encoded-utf8-text>
https://app.tonkeeper.com/transfer/<address>?bin=<url-encoded-base64-boc>
https://app.tonkeeper.com/transfer/<address>?bin=<url-encoded-base64-boc>&init=<url-encoded-base64-boc>
```

使用 qs 庫，我們將生成以下字符串：

```ts
let deployLink =
    'https://app.tonkeeper.com/transfer/' +
    address.toString({
        testOnly: true,
    }) +
    "?" +
    qs.stringify({
        text: "Deploy contract by QR",
        amount: toNano("0.1").toString(10),
        init: stateInitCell.toBoc({idx: false}).toString("base64"),
    });
```

現在生成 QR 碼：

```ts
qrcode.generate(deployLink, {small: true }, (qr) => {
    console.log(qr);
});
```

最後，生成測試網路中區塊鏈瀏覽器的連結，方便我們在部署後查看合約是否已準備好。

最終代碼：

```ts
import { Cell, StateInit, beginCell, contractAddress, storeStateInit, toNano } from "ton-core";
import { hex } from "../build/main.compiled.json";
import qs from "qs";
import qrcode from "qrcode-terminal";

async function deployContract() {
    const codeCell = Cell.fromBoc(Buffer.from(hex,"hex"))[0];
    const dataCell = new Cell();   

    const stateInit: StateInit = {
        code: codeCell,
        data: dataCell,
    };

    const stateInitBuilder = beginCell();
    storeStateInit(stateInit)(stateInitBuilder);
    const stateInitCell = stateInitBuilder.endCell();

    const address = contractAddress(0, {
        code: codeCell,
        data: dataCell,
    });

    let deployLink =
        'https://app.tonkeeper.com/transfer/' +
        address.toString({
            testOnly: true,
        }) +
        "?" +
        qs.stringify({
            text: "Deploy contract by QR",
            amount: toNano("0.1").toString(10),
            init: stateInitCell.toBoc({idx: false}).toString("base64"),
        });

    qrcode.generate(deployLink, {small: true }, (qr) => {
        console.log(qr);
    });

    let scanAddr = 
        'https://testnet.tonscan.org/address/' +
        address.toString({
            testOnly: true,
        });

    console.log(scanAddr);
}

deployContract();
```

在 `package.json` 文件中添加 `"deploy": "yarn compile && ts-node ./scripts/deploy.ts"` 到 scripts：

```json
{
  "name": "test_folder",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "devDependencies": {
    "@swc/core": "^1.3.63",
    "@ton-community/func-js": "^0.6.2",
    "@ton-community/sandbox": "^0.11.0",
    "@types/jest": "^29.5.2",
    "@types/node": "^20.3.1",
    "@types/qrcode-terminal": "^0.12.0",
    "@types/qs": "^6.9.7",
    "jest": "^29.5.0",
    "qrcode-terminal": "^0.12.0",
    "qs": "^6.11.2",
    "ton": "^13.5.0",
    "ton-core": "^0.49.1",
    "ton-crypto": "^3.2.0",
    "ts-jest": "^29.1.0",
    "ts-node": "^10.9.1",
    "typescript": "^5.1.3"
  },
  "scripts": {
    "compile": "ts-node ./scripts/compile.ts",
    "test": "yarn jest",
    "deploy": "yarn compile && ts-node ./scripts/deploy.ts"
  },
  "dependencies": {
    "@ton-community/test-utils": "^0.2.0"
  }
}
```

剩下的步驟：

1. 在控制台輸入命令 `yarn deploy`
2. 在切換到測試網路的 Tonkeeper 中掃描 QR 碼
3. 確認交易
4. 在區塊鏈瀏覽器中查看合約是否已部署

## 結論

感謝您的關注，在接下來的兩個教程中，我們將重點關注消息的編寫，如何在編寫測試時進行構建，以及如何為測試網路編寫測試。