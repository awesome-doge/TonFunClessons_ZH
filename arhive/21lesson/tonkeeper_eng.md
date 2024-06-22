## 簡介

在去中心化應用程式中，使用加密錢包進行授權是一個重要的部分。在本教程中，我們將逐步使用 [tonconnect/sdk](https://github.com/ton-connect/sdk) 為 TON 區塊鏈建立授權。

本教程的目標是建立一個簡單的單頁應用程式（網站），並實現使用 [Tonkeeper](https://tonkeeper.com/) 錢包的授權。為了簡單起見，教程中省略了許多可以優化代碼的部分。

我們也不會深入討論樣式，因為本教程的目標是解析一個方便未來擴展的例子，以適應你的需求。

## 功能需求

在本例中，我們將實現以下功能：
- 授權按鈕，點擊後顯示 QR 碼或連結，通過 Tonkeeper 錢包進行授權
- 當用戶從手機進入應用程式或在錢包內部的瀏覽器中打開時，顯示連結
- 在所有其他情況（桌面設備中打開應用程式）下，顯示 QR 碼
- 顯示用戶登入後的錢包地址
- 授權後在頁首顯示網路類型（測試網或主網）
- 錢包從應用程式斷開連接

效果如下：

1. 點擊授權按鈕

![1 test](./img/1_1.PNG)

2. 在應用程式中掃描 QR 碼

![2 test](./img/1_3.jpg)

3. 我們將看到錢包地址和測試網的標籤

![3 test](./img/1_2.PNG)

此外，本教程還會展示如何獲取集成在 [tonconnect/sdk](https://github.com/ton-connect/sdk) 中的不同錢包列表，以便用戶不僅能使用 Tonkeeper 登錄。

## 安裝 Vite

在開始之前，你必須在系統上安裝 Node 和 npm。我們的第一步是使用 `vite` 命令來創建一個新應用程式。這可以通過 `npm init` 命令來完成，而無需安裝額外的軟體。在你選擇的資料夾中打開終端，運行以下命令：

```bash
npm init vite@latest vite-tonconnect -- --template react-ts
```

接下來進入資料夾：

```bash
cd vite-tonconnect
```

運行以下命令：

```bash
npm install
```

測試我們是否做對了：

```bash
npm run dev
```

你應該會看到類似以下的內容：

![vite test](./img/1_4.PNG)

## 安裝庫

安裝與 TON 區塊鏈交互所需的庫：

```bash
npm i ton ton-core ton-crypto
```

授權需要 `tonconnect` 庫：

```bash
npm i @tonconnect/sdk
```

為了美化界面，你需要以下庫：

```bash
npm i antd sass
```

要使用 TonConnect 登錄，你需要使用 Tonkeeper 錢包，我們將在應用程式中掃描 QR 碼並在錢包中確認。這意味著你需要一個生成 QR 碼的庫：

```bash
npm i react-qr-code
```

在授權時，我們需要讀取異步選擇器的值（以獲取錢包列表），因此我們將安裝 `recoil` 庫：

```bash
npm i recoil
```

如果你現在調用 ton 庫（例如轉換地址並顯示給用戶），你會看到 Buffer is not defined 錯誤。為了解決這個問題，安裝以下庫：

```bash
npm i node-stdlib-browser vite-plugin-node-stdlib-browser
```

並在 Vite 配置文件 `vite.config.ts` 中添加 `nodePolyfills()` 插件。

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import nodePolyfills from 'vite-plugin-node-stdlib-browser'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react(), nodePolyfills()],
})
```

## 撰寫輔助函數

根據功能需求，我們需要一些輔助函數：
- 檢查設備是否為手機
- 檢查設備是否為桌面
- 打開連結以進行授權後的重定向

在 `src` 資料夾中創建 `utils.ts` 文件，並添加 `isMobile()` 和 `isDesktop()` 函數。根據 `window` 的 `innerWidth` 屬性來判斷設備類型：

```ts
export function isMobile(): boolean {
	return window.innerWidth <= 500;
}

export function isDesktop(): boolean {
	return window.innerWidth >= 1050;
}
```

要打開連結，使用 `window` 介面的 [open](https://developer.mozilla.org/en-US/docs/Web/API/Window/open) 方法：

```ts
export function openLink(href: string, target = '_self') {
	window.open(href, target, 'noreferrer noopener');
}
```

`noreferrer noopener` 用於安全目的，防止惡意連結攔截新打開的標籤頁，因為 `JavaScript window.opener` 對象允許新打開的標籤頁控制父窗口。

## 連接 TonConnect

首先要做的是建立連接，為此我們在 `src` 資料夾中創建 `connector.ts` 文件。並導入 `TonConnect`：

```ts
import { TonConnect } from '@tonconnect/sdk';
```

要建立連接，需要調用 `new TonConnect()` 並傳遞你的應用程式參數，當用戶使用錢包進行授權時，他會看到你的應用程式的數據，並明白他將連接到哪裡。參數或稱為元數據包含以下字段：

```json
{
  "url": "<app-url>",                        // required
  "name": "<app-name>",                      // required
  "iconUrl": "<app-icon-url>",               // required
  "termsOfUseUrl": "<terms-of-use-url>",     // optional
  "privacyPolicyUrl": "<privacy-policy-url>" // optional
}
```

最佳做法是將元數據清單放在應用程式的根目錄中，但你也可以將其放在 github 上。例如，我會附上來自 github 的元數據清單連結：

```ts
const dappMetadata = {
	manifestUrl: 'https://gist.githubusercontent.com/siandreev/75f1a2ccf2f3b4e2771f6089aeb06d7f/raw/d4986344010ec7a2d1cc8a2a9baa57de37aaccb8/gistfile1.txt',
};
```

形成元數據後，只需調用連接即可，得到如下代碼：

```ts
import { TonConnect } from '@tonconnect/sdk';

const dappMetadata = {
	manifestUrl: 'https://gist.githubusercontent.com/siandreev/75f1a2ccf2f3b4e2771f6089aeb06d7f/raw/d4986344010ec7a2d1cc8a2a9baa57de37aaccb8/gistfile1.txt',
};

export const connector = new TonConnect(dappMetadata);
```

如果用戶之前已連接過錢包，連接器會重新連接。

為了提供方便的用戶體驗，需要處理用戶之前連接過錢包並再次進入你的應用程式的情況。確保在這種情況下立即恢復連接。

為此，我們將使用 [useEffect hook](https://legacy.reactjs.org/docs/hooks-effect.html)，在 hook 中調用 `connector.restoreConnection()`。我們會在 `App.tsx` 文件中這樣做：

```tsx
import React, { useEffect } from 'react';
import reactLogo from './assets/react.svg'
import viteLogo from '/vite.svg'
import './App.css'

import { connector } from '../src/connector';

function App() {
  useEffect(() => {
		connector.restoreConnection();
	}, []);

  return (
	<div>Auth will be here</div>
  )
}

export default App
```

> 我已經導入了樣式，不會詳細討論它們，因為本教程主要講解授權

## 創建自定義 hooks

為了授權按鈕的方便使用，我們將創建幾個自定義 hooks。讓我們為它們創建一個單獨的 hooks 資料夾，裡面包含四個腳本：
- `useTonWallet.ts`
- `useTonWalletConnectionError.ts`
- `useSlicedAddress.ts`
- `useForceUpdate.ts`

逐一講解它們。

#### useTonWallet

為了授權按鈕的運作，需要訂閱連接狀態的變化，我們將使用 `connector.onStatusChange()` 來完成這個操作：

```ts
import { Wallet } from '@tonconnect/sdk';
import { useEffect, useState } from 'react';
import { connector } from '../connector';

export function useTonWallet() {
	const [wallet, setWallet] = useState<Wallet | null>(connector.wallet);

	useEffect(() => connector.onStatusChange(setWallet, console.error), []);

	return wallet;
}
```

#### useTonWalletConnectionError

我們還需要處理授權過程中的錯誤，使用 `connector.onStatusChange()`。我們將單獨處理在 `tonconnect/sdk` 中註冊的 `UserRejectsError` 錯誤，該錯誤在用戶拒絕錢包操作時發生。

```ts
import { UserRejectsError } from '@tonconnect/sdk';
import { useCallback, useEffect } from 'react';
import { connector } from '../connector';

export function useTonWalletConnectionError(callback: () => void) {
	const errorsHandler = useCallback(
		(error: unknown) => {
			if (typeof error === 'object' && error instanceof UserRejectsError) {
				callback();
			}
		},
		[callback],
	);

	const emptyCallback = useCallback(() => {}, []);

	useEffect(() => connector.onStatusChange(emptyCallback, errorsHandler), [emptyCallback, errorsHandler]);
}
```

#### useSlicedAddress

一個方便的 UX 模式是授權後顯示用戶的錢包地址。在 TON Connect 中，地址以 `0:<hex>` 格式（raw 格式）傳輸，用戶會覺得顯示成友好格式比較方便，因此我們要做一個處理器。關於 TON 地址格式的詳細信息可以在 [這裡](https://docs.ton.org/learn/overviews/addresses) 查看。

```ts
import { CHAIN } from '@tonconnect/sdk';
import { useMemo } from 'react';
import { Address } from 'ton';

export function useSlicedAddress(address: string | null | undefined, chain?: CHAIN) {
	return useMemo(() => {
		if (!address) {
			return '';
		}

		const userFriendlyAddress = Address.parseRaw(address).toString({ testOnly: chain === CHAIN.TESTNET });

		return userFriendlyAddress.slice(0, 4) + '...' + userFriendlyAddress.slice(-3);

	}, [address]);
}
```

> 這裡也考慮到了錢包可能在測試網中的情況

#### useForceUpdate

在大多數情況下，React 會自動處理組件重新渲染。當 props 或 state 更新時，組件會重新渲染。然而，我們的組件（授權按鈕）依賴於第三方用戶確認授權動作，因此強制刷新組件是很重要的，因為 React 可能無法檢測到變化。

```ts
import { useState } from 'react';

export function useForceUpdate() {
	const [_, setValue] = useState(0);
	return () => setValue((value) => value + 1);
}
```

## 獲取錢包列表

儘管在本例中我們只使用 Tonkeeper 錢包，但 TONConnect 可以向用戶提供選擇的錢包列表。為此，我們需要獲取它們。

在 `src` 資料夾中創建一個 `state` 資料夾，並在其中創建一個 `wallets-list.ts` 文件。為了從連接中獲取錢包列表，我們使用 `connector.getWallets()`。在本教程中，我們將使用 `recoil` 來管理狀態。

讓我們創建一個選擇器：

```ts
import { isWalletInfoInjected } from '@tonconnect/sdk';
import { selector } from 'recoil';
import { connector } from '../../src/connector';

export const walletsListQuery = selector({
	key: 'walletsList',
	get: async () => {
		const walletsList = await connector.getWallets();
	},
});
```

在使用去中心化應用程式時，可能會遇到這種情況：我們在錢包的瀏覽器中打開一個提議，當然在這種情況下提供錢包列表是沒有意義的，最好直接給出合適的錢包。為了應對這種情況，在 `tonconnect/sdk` 中有 `isWalletInfoInjected`，它可以讓我們立即獲取所需的錢包：

```ts
import { isWalletInfoInjected } from '@tonconnect/sdk';
import { selector } from 'recoil';
import { connector } from '../../src/connector';

export const walletsListQuery = selector({
	key: 'walletsList',
	get: async () => {
		const walletsList = await connector.getWallets();

		const embeddedWallet = walletsList.filter(isWalletInfoInjected).find((wallet) => wallet.embedded);

		return {
			walletsList,
			embeddedWallet,
		};
	},
});
```

## 授權按鈕組件

最後，我們來看授權按鈕本身。導入我們之前寫的腳本，還有佈局所需的一切：

```ts
import { DownOutlined } from '@ant-design/icons';
import { Button, Dropdown, Menu, Modal, notification, Space } from 'antd';
import React, { useCallback, useEffect, useState } from 'react';
import QRCode from 'react-qr-code';
import { useRecoilValueLoadable } from 'recoil';
import { addReturnStrategy, connector } from '../../../src/connector';
import { useForceUpdate } from '../../../src/hooks/useForceUpdate';
import { useSlicedAddress } from '../../../src/hooks/useSlicedAddress';
import { useTonWallet } from '../../../src/hooks/useTonWallet';
import { useTonWalletConnectionError } from '../../../src/hooks/useTonWalletConnectionError';
import { walletsListQuery } from '../../../src/state/wallets-list';
import { isDesktop, isMobile, openLink } from '../../../src/utils';
import './style.scss';
```

你可能注意到，在之前的步驟中，沒有提到從應用程式中斷開錢包（disconnect）。根據我們的示例邏輯，授權後，用戶會看到已連接的錢包地址，點擊按鈕會出現下拉選單，選擇斷開連接的選項。使用 `antd` 庫中的 `Menu` 來實現這一點：

```ts
const menu = (
	<Menu
		onClick={() => connector.disconnect()}
		items={[
			{
				label: 'Disconnect',
				key: '1',
			},
		]}
	/>
);
```

現在到了組件本身，實際上授權只是獲取一個授權連結並顯示給用戶（連結或 QR 碼），所以我們最重要的 hook 是處理連結：

```ts
export function AuthButton() {
	const [modalUniversalLink, setModalUniversalLink] = useState('');

	return (
		<>
			<div className="auth-button">
			</div>
		</>
	);
}
```

這個 hook 允許你設置授權連結的值。如果用戶未授權，顯示一個按鈕開始授權過程；如果已授權，顯示一個地址按鈕，允許斷開錢包。為此，需要知道錢包是否已連接，我們之前寫的 hook `useTonWallet()` 會幫助我們。此外，我們的組件接收變更信息來自第三方，這意味著我們需要強制刷新組件：

```ts
export function AuthButton() {
	const [modalUniversalLink, setModalUniversalLink] = useState('');
	const forceUpdate = useForceUpdate();
	const wallet = useTonWallet();
	
	return (
		<>
			<div className="auth-button">
				{wallet ? (
				<Dropdown overlay={menu}>
					<Button shape="round" type="primary">
						<Space>
							{address}
							<DownOutlined />
						</Space>
					</Button>
				</Dropdown>
			) : (
				<Button shape="round" type="primary" onClick={handleButtonClick}>
					Connect Wallet
				</Button>
			)}
			</div>
		</>
	);
}
```

這裡我們會處理錯誤並獲取錢包列表，同時立即轉換地址：

```ts
export function AuthButton() {
	const [modalUniversalLink, setModalUniversalLink] = useState('');
	const forceUpdate = useForceUpdate();
	const wallet = useTonWallet();
	const onConnectErrorCallback = useCallback(() => {
		setModalUniversalLink('');
		notification.error({
			message: 'Connection was rejected',
			description: 'Please approve connection to the dApp in your wallet.',
		});
	}, []);
	useTonWalletConnectionError(onConnectErrorCallback);

	const walletsList = useRecoilValueLoadable(walletsListQuery);

	const address = useSlicedAddress(wallet?.account.address, wallet?.account.chain);

	return (
		<>
			<div className="auth-button">
				{wallet ? (
					<Dropdown overlay={menu}>
						<Button shape="round" type="primary">
							<Space>
								{address}
								<DownOutlined />
							</Space>
						</Button>
					</Dropdown>
				) : (
					<Button shape="round" type="primary" onClick={handleButtonClick}>
						Connect Wallet
					</Button>
				)}
			</div>
		</>
	);
}
```

讓我們處理按鈕點擊事件，首先檢查錢包列表是否加載，如果沒有，我們會等待：

```ts
const handleButtonClick = useCallback(async () => {
	if (!(walletsList.state === 'hasValue')) {
		setTimeout(handleButtonClick, 200);
	}
}, [walletsList]);
```

獲取錢包列表時，我們會檢查應用程式是否在錢包內部的瀏覽器中打開，如果不是桌面設備且應用程式在錢包內部打開，我們會使用該錢包進行連接：

```ts
const handleButtonClick = useCallback(async () => {
	if (!(walletsList.state === 'hasValue')) {
		setTimeout(handleButtonClick, 200);
	}

	if (!isDesktop() && walletsList.contents.embeddedWallet) {
		connector.connect({ jsBridgeKey: walletsList.contents.embeddedWallet.jsBridgeKey });
		return;
	}
}, [walletsList]);
```

現在是獲取授權連結的時候了，從錢包列表中獲取第一個（即 Tonkeeper），使用 `connector.connect()` 獲取連結：

```ts
const handleButtonClick = useCallback(async () => {
	if (!(walletsList.state === 'hasValue')) {
		setTimeout(handleButtonClick, 200);
	}

	if (!isDesktop() && walletsList.contents.embeddedWallet) {
		connector.connect({ jsBridgeKey: walletsList.contents.embeddedWallet.jsBridgeKey });
		return;
	}

	const tonkeeperConnectionSource = {
		universalLink: walletsList.contents.walletsList[0].universalLink,
		bridgeUrl: walletsList.contents.walletsList[0].bridgeUrl,
	};

	const universalLink = connector.connect(tonkeeperConnectionSource);
}, [walletsList]);
```

現在我們處理移動設備，不方便顯示 QR 碼，所以我們會直接打開連結，其他情況下，我們會在 QR 碼下顯示授權連結。之前寫的輔助函數會幫助我們：

```ts
const handleButtonClick = useCallback(async () => {
	// Use loading screen/UI instead (while wallets list is loading)
	if (!(walletsList.state === 'hasValue')) {
		setTimeout(handleButtonClick, 200);
	}

	if (!isDesktop() && walletsList.contents.embeddedWallet) {
		connector.connect({ jsBridgeKey: walletsList.contents.embeddedWallet.jsBridgeKey });
		return;
	}

	const tonkeeperConnectionSource = {
		universalLink: walletsList.contents.walletsList[0].universalLink,
		bridgeUrl: walletsList.contents.walletsList[0].bridgeUrl,
	};

	const universalLink = connector.connect(tonkeeperConnectionSource);

	if (isMobile()) {
		openLink(addReturnStrategy(universalLink, 'none'), '_blank');
	} else {
		setModalUniversalLink(universalLink);
	}
}, [walletsList]);
```

最後添加模態窗口，這是最終的 `AuthButton.tsx` 組件：

```tsx
import { DownOutlined } from '@ant-design/icons';
import { Button, Dropdown, Menu, Modal, notification, Space } from 'antd';
import React, { useCallback, useEffect, useState } from 'react';
import QRCode from 'react-qr-code';
import { useRecoilValueLoadable } from 'recoil';
import { addReturnStrategy, connector } from '../../../src/connector';
import { useForceUpdate } from '../../../src/hooks/useForceUpdate';
import { useSlicedAddress } from '../../../src/hooks/useSlicedAddress';
import { useTonWallet } from '../../../src/hooks/useTonWallet';
import { useTonWalletConnectionError } from '../../../src/hooks/useTonWalletConnectionError';
import { walletsListQuery } from '../../../src/state/wallets-list';
import { isDesktop, isMobile, openLink } from '../../../src/utils';
import './style.scss';

const menu = (
	<Menu
		onClick={() => connector.disconnect()}
		items={[
			{
				label: 'Disconnect',
				key: '1',
			},
		]}
	/>
);

export function AuthButton() {
	const [modalUniversalLink, setModalUniversalLink] = useState('');
	const forceUpdate = useForceUpdate();
	const wallet = useTonWallet();
	const onConnectErrorCallback = useCallback(() => {
		setModalUniversalLink('');
		notification.error({
			message: 'Connection was rejected',
			description: 'Please approve connection to the dApp in your wallet.',
		});
	}, []);
	useTonWalletConnectionError(onConnectErrorCallback);

	const walletsList = useRecoilValueLoadable(walletsListQuery);

	const address = useSlicedAddress(wallet?.account.address, wallet?.account.chain);

	useEffect(() => {
		if (modalUniversalLink && wallet) {
			setModalUniversalLink('');
		}
	}, [modalUniversalLink, wallet]);

	const handleButtonClick = useCallback(async () => {
		if (!(walletsList.state === 'hasValue')) {
			setTimeout(handleButtonClick, 200);
		}

		if (!isDesktop() && walletsList.contents.embeddedWallet) {
			connector.connect({ jsBridgeKey: walletsList.contents.embeddedWallet.jsBridgeKey });
			return;
		}

		const tonkeeperConnectionSource = {
			universalLink: walletsList.contents.walletsList[0].universalLink,
			bridgeUrl: walletsList.contents.walletsList[0].bridgeUrl,
		};

		const universalLink = connector.connect(tonkeeperConnectionSource);

		if (isMobile()) {
			openLink(addReturnStrategy(universalLink, 'none'), '_blank');
		} else {
			setModalUniversalLink(universalLink);
		}
	}, [walletsList]);

	return (
		<>
			<div className="auth-button">
				{wallet ? (
					<Dropdown overlay={menu}>
						<Button shape="round" type="primary">
							<Space>
								{address}
								<DownOutlined />
							</Space>
						</Button>
					</Dropdown>
				) : (
					<Button shape="round" type="primary" onClick={handleButtonClick}>
						Connect Wallet
					</Button>
				)}
			</div>
			<Modal
				title="Connect to Tonkeeper"
				open={!!modalUniversalLink}
				onOk={() => setModalUniversalLink('')}
				onCancel={() => setModalUniversalLink('')}
			>
				<QRCode
					size={256}
					style={{ height: '260px', maxWidth: '100%', width: '100%' }}
					value={modalUniversalLink}
					viewBox={`0 0 256 256`}
				/>
			</Modal>
		</>
	);
}
```

## 應用程式標題組件

為了方便使用去中心化應用程式，顯示進行操作的是測試網還是主網非常重要。我們將在應用程式名稱旁邊顯示用戶授權後所使用的錢包所在的網路類型。為此，讓我們創建一個單獨的組件 `AppTitle.tsx`。

```ts
import { CHAIN } from '@tonconnect/sdk';
import React, { useEffect, useRef, useState } from 'react';
import { useTonWallet } from '../../../src/hooks/useTonWallet';
import './style.scss';

const chainNames = {
	[CHAIN.MAINNET]: 'mainnet',
	[CHAIN.TESTNET]: 'testnet',
};

export function AppTitle() {
	const wallet = useTonWallet();

	return (
		<>
			<div className="dapp-title" >
				<span className="dapp-title__text">My Dapp</span>
				{wallet && <span className="dapp-title__badge">{chainNames[wallet.account.chain]}</span>}
			</div>
		</>
	);
}
```

如你所見，這裡非常簡單，我們使用我們的 hook 獲取當前錢包，從錢包中獲取網路編號，並使用 `tonconnect/sdk` 中內建的 `CHAIN` 目錄獲取網路名稱。

## 將組件添加到頁面

在 `App.tsx` 文件中，添加我們撰寫的組件到頁首：

```ts
import React, { useEffect } from 'react';
import { AppTitle } from '../src/components/AppTitle/AppTitle';
import { AuthButton } from '../src/components/AuthButton/AuthButton';
import { connector } from '../src/connector';
import 'antd/dist/reset.css';
import './app.scss';

function App() {
  useEffect(() => {
		connector.restoreConnection();
	}, []);

  return (
	<div className="app">
	  <header>
		<AppTitle />
		<AuthButton />
	  </header>
	  <main>
	  </main>
	</div>    
  )
}

export default App
```

此外，在 `main.tsx` 文件中將應用程式包裝在 `RecoilRoot` 中，因為我們使用了選擇器。

```ts
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App'
import './index.scss';
import { RecoilRoot } from 'recoil';

ReactDOM.createRoot(document.getElementById('root') as HTMLElement).render(
  <RecoilRoot>
	<React.StrictMode>
		<App />
	</React.StrictMode>
  </RecoilRoot>,
)
```

就這樣。在第二部分中，我們將研究如何使用已連接的錢包發送交易，並擴展我們的示例。

## 結論

在 github 上有示例。我在 [這裡](https://t.me/ton_learn) 發表類似的文章。感謝你的閱讀。