# 智能合約抽獎教學

在這篇教學中，我們將分析一個智能合約抽獎系統。這個智能合約具有以下優點：
- 正確使用隨機數。更多關於TON中的隨機數技術細節，請參考[這裡](https://docs.ton.org/develop/smart-contracts/guidelines/random-number-generation)
- 具備重要的合約餘額管理機制，可應用於您的智能合約中
- 項目的結構和分組方便，值得借鑑

## 隨機數問題

隨機數生成在很多不同的項目中可能會需要。在 FunC 文檔中有 `random()` 函數，但在真實項目中不能使用！其結果很容易被預測，除非應用一些額外的技巧。

為了使隨機數生成不可預測，可以將當前的邏輯時間添加到種子中，使不同的交易擁有不同的種子和結果。

您可以使用 `randomize_lt()` 來實現，例如：

```func
randomize_lt();
int x = random(); ;; 用戶無法預測這個數字
```

也可以使用 `randomize()`，傳遞幾個參數，包括邏輯時間。

```func
() randomize(int x) impure asm "ADDRAND";
```

在本文中，我們將介紹使用 `randomize()` 的抽獎合約。

## 高層概述

### 如何運作？

合約只接受1 TON的消息（其他金額將被退回）。智能合約生成0到10000之間的隨機數來決定是否中獎。

如果用戶中獎，其獎金將扣除我們10%的佣金。合約餘額的其他部分保持不變。獲獎者將在交易評論中收到中獎消息。

### 中獎概率

- 0.1% 中頭獎（合約餘額的一半）[0; 10)
- 9.9% 中5 TON獎金 [10; 1000)
- 10% 中2 TON獎金 [1000; 2000)

如果沒有中獎，則不會發生任何事情。[2000; 9999]

合約代碼 - https://github.com/Vudi/new-year-ruffle/tree/main

### 智能合約結構

智能合約是一個 `recv_internal` 內部消息處理器，當消息體為空時啟動抽獎/彩票，或者執行以下 `op` 條件之一：
- `op == op::add_balance()` 增加餘額，以防合約中的資金用完
- `op == op::maintain()` 允許從合約發送具有不同模式的內部消息，即允許管理智能合約的餘額，也可以允許銷毀合約（消息模式為 `128 + 32`）
- `op == op::withdraw()` 允許提取部分資金——積累的佣金

## 智能合約風格

我們正在考察的智能合約具有良好的風格：
- 智能合約被巧妙地分成多個文件，使其非常易於閱讀
- 與存儲（主要是 `c4` 寄存器）相結合的全局變量改善了代碼的可讀性，使智能合約更加易於理解

`op` 指令和常用的1 TON變量被移至單獨的 `const.func` 文件：

```func
int op::maintain() asm "1001 PUSHINT";
int op::withdraw() asm "1002 PUSHINT";
int op::add_balance() asm "1003 PUSHINT";

int exit::invalid_bet() asm "2001 PUSHINT";

int 1ton() asm "1000000000 PUSHINT";
```

`admin.func` 文件包含管理命令，例如 `adm::maintain`，允許以任何模式從智能合約發送消息，從而管理智能合約的餘額：

```func
() adm::maintain(slice in_msg_body) impure inline_ref {
    int mode = in_msg_body~load_uint(8);
    send_raw_message(in_msg_body~load_ref(), mode);    
}
```

以及 `adm::withdraw()`，允許方便地提取部分資金：

```func
() adm::withdraw() impure inline_ref {
    cell body = begin_cell()
        .store_uint(0, 32)
        .store_slice(msg::commission_withdraw())
        .end_cell();

    cell msg = begin_cell()
        .store_uint(0x18, 6)
        .store_slice(db::admin_addr)
        .store_coins(db::service_balance)
        .store_uint(1, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_ref(body)
        .end_cell();

    db::service_balance = 0;
    send_raw_message(msg, 0);
}
```

智能合約在輸和贏時發送消息，這些消息放在單獨的 `msg.func` 文件中，注意它們的類型是 `slice`：

```func
slice msg::commission_withdraw()    asm "<b 124 word Withdraw commission| $, b> <s PUSHSLICE";
slice msg::jackpot()                asm "<b 124 word Congrats! You have won jackpot!| $, b> <s PUSHSLICE";
slice msg::x2()                     asm "<b 124 word Congrats! You have won x2!| $, b> <s PUSHSLICE";
slice msg::x5()                     asm "<b 124 word Congrats! You have won x5!| $, b> <s PUSHSLICE";
```

`game.func` 包含抽獎/彩票的邏輯，我們稍後將詳細討論這個文件的代碼。智能合約提供了一個Get方法，返回來自智能合約 `c4` 寄存器的信息。這個方法存儲在 `get-methods.func` 文件中：

```func
(int, int, (int, int), int, int) get_info() method_id {
    init_data();
    return (db::available_balance, db::service_balance, parse_std_addr(db::admin_addr), db::last_number, db::hash);
}
```

最後，`storage.func` 文件中的存儲操作。需要注意的是，數據不會永久存儲在 `c4` 寄存器中，而是先存儲在全局變量中，然後在某些邏輯代碼結束時，通過 `pack_data()` 函數存儲到寄存器中：

```func
global int init?;

global int db::available_balance;
global int db::service_balance;
global slice db::admin_addr;
global int db::last_number;
global int db::hash;


() init_data() impure {
    ifnot(null?(init?)) {
        throw(0x123);
    }

    slice ds = get_data().begin_parse();

    db::available_balance = ds~load_coins();
    db::service_balance = ds~load_coins();
    db::admin_addr = ds~load_msg_addr();
    db::last_number = ds~load_uint(64);
    db::hash = slice_empty?(ds) ? 0 : ds~load_uint(256);

    init? = true;
}

() pack_data() impure {
    set_data(
        begin_cell()
            .store_coins(db::available_balance)
            .store_coins(db::service_balance)
            .store_slice(db::admin_addr)
            .store_uint(db::last_number, 64)
            .store_uint(db::hash, 256)
        .end_cell()
    );
}
```

## 分析 recv_internal()

`main.fc` 文件開始於導入上述文件：

```func
#include "lib/stdlib.func";
#include "struct/const.func";
#include "struct/storage.func";
#include "struct/msg.func";
#include "struct/game.func";
#include "struct/admin.func";
#include "struct/get-methods.func";


() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {

}
```

我們獲取消息並使用 [slice_hash](https://docs.ton.org/develop/func/stdlib#slice_hash) 函數創建一個哈希，稍後將用於隨機數。同時，任何消息都以標誌開頭，讓我們進行一些檢查：

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int hash = slice_hash(cs); 
    throw_if(0, cs~load_uint(4) & 1);

}
```

使用 `storage.func` 文件中的輔助函數初始化數據，並從消息中獲取地址：

```func
() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int hash = slice_hash(cs); 
    throw_if(0, cs~load_uint(4) & 1);

    init_data();

    slice sender_addr = cs~load_msg_addr();

}
```

接下來，我們將進行 `op` 檢查，但為了方便使用智能合約，主要功能在沒有 `op` 的情況下使用，因此用戶只需發送空消息到合約即可啟動抽獎。為了實現這樣的功能，我們只需檢查剩餘消息，若為空則啟動遊戲。

```func
#include "lib/stdlib.func";
#include "struct/const.func";
#include "struct/storage.func";
#include "struct/msg.func";
#include "struct/game.func";
#include "struct/admin.func";
#include "struct/get-methods.func";


() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int hash = slice_hash(cs); 
    throw_if(0, cs~load_uint(4) & 1);

    init_data();

    slice sender_addr = cs~load_msg_addr();

    if (in_msg_body.slice_empty?()) {
        game::start(sender_addr, msg_value, hash);
        pack_data();
        throw(0);
    }
}
```

在遊戲函數內，我們會改變數據，這些數據稍後需要存儲在常量數據存儲寄存器 `c4` 中，函數內的全局變量會改變，在 `recv_internal()` 中我們保存：

```func
#include "lib/stdlib.func";
#include "struct/const.func";
#include "struct/storage.func";
#include "struct/msg.func";
#include "struct/game.func";
#include "struct/admin.func";
#include "struct/get-methods.func";


() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int hash = slice_hash(cs); 
    throw_if(0, cs~load_uint(4) & 1);

    init_data();

    slice sender_addr = cs~load_msg_addr();

    if (in_msg_body.slice_empty?()) {
        game::start(sender_addr, msg_value, hash);
        pack_data();
        throw(0);
    }
}
```

這裡可能會產生一個問題，為什麼在一切都正常工作的情況下拋出異常。根據 [TVM 4.5.1 條款](https://ton.org/tvm.pdf) 文檔，0-31 是為異常保留的代碼，這些代碼與 [exit_code](https://docs.ton.org/learn/tvm-instructions/tvm-exit-codes) 相同，0 是標準的成功退出代碼。

然後一切都很簡單，`op`，我們已經分析了其意義，`main.func` 的最終代碼：

```func
#include "lib/stdlib.func";
#include "struct/const.func";
#include "struct/storage.func";
#include "struct/msg.func";
#include "struct/game.func";
#include "struct/admin.func";
#include "struct/get-methods.func";


() recv_internal(int my_balance, int msg_value, cell in_msg_full, slice in_msg_body) impure {
    slice cs = in_msg_full.begin_parse();
    int hash = slice_hash(cs); 
    throw_if(0, cs~load_uint(4) & 1);

    init_data();

    slice sender_addr = cs~load_msg_addr();

    if (in_msg_body.slice_empty?()) {
        game::start(sender_addr, msg_value, hash);
        pack_data();
        throw(0);
    }

    int op = in_msg_body~load_uint(32);
    int is_admin = equal_slices(sender_addr, db::admin_addr);
    if (op == op::add_balance()) {
        db::available_balance += msg_value;
        pack_data();
        throw(0);
    }

    if (op == op::maintain()) {
        throw_if(0xfffe, is_admin == 0);
        adm::maintain(in_msg_body);
        throw(0);
    }

    if (op == op::withdraw()) {
        throw_if(0xfffd, is_admin == 0);
        adm::withdraw();
        pack_data();
        throw(0);
    }

    throw(0xffff);
}
```

## 分析 game.func

終於到達了遊戲邏輯，`game.func` 文件以輔助函數支付獎金開始：

```func
() game::payout(slice sender_addr, int amount, slice msg) impure inline_ref {
    cell body = begin_cell()
        .store_uint(0, 32)
        .store_slice(msg)
        .end_cell();

    cell msg = begin_cell()
        .store_uint(0x18, 6)
        .store_slice(sender_addr)
        .store_coins(amount)
        .store_uint(1, 1 + 4 + 4 + 64 + 32 + 1 + 1)
        .store_ref(body)
        .end_cell();

    send_raw_message(msg, 0);
}  
```

遊戲本身從檢查發送到合約的金額是否等於 `1 TON` 開始：

```func
() game::start(slice sender_addr, int msg_value, int hash) impure inline_ref {
    throw_unless(exit::invalid_bet(), msg_value == 1ton());
}
```

接下來，組合隨機數的哈希值，該哈希值由當前時間、`c4` 寄存器中的哈希、在 `recv_internal` 中從消息中生成的哈希和 `cur_lt()`（當前交易的邏輯時間）組成：

```func
() game::start(slice sender_addr, int msg_value, int hash) impure inline_ref {
    throw_unless(exit::invalid_bet(), msg_value == 1ton());
    int new_hash = slice_hash(
        begin_cell()
            .store_uint(db::hash, 256)
            .store_uint(hash, 256)
            .store_uint(cur_lt(), 64)
            .store_uint(now(), 64)
        .end_cell()
        .begin_parse()
    );
}
```

使用哈希可以生成隨機數，但在使用 `rand()` 函數生成隨機數之前，我們使用 [randomize](https://docs.ton.org/develop/func/stdlib/#randomize) 隨機化哈希：

```func
() game::start(slice sender_addr, int msg_value, int hash) impure inline_ref {
    throw_unless(exit::invalid_bet(), msg_value == 1ton());
    int new_hash = slice_hash(
        begin_cell()
            .store_uint(db::hash, 256)
            .store_uint(hash, 256)
            .store_uint(cur_lt(), 64)
            .store_uint(now(), 64)
        .end_cell()
        .begin_parse()
    );

    randomize(new_hash);

}
```

> 我注意到這是實現隨機數的可能變體之一，您可以在文檔中閱讀隨機數 - https://docs.ton.org/develop/smart-contracts/guidelines/random-number-generation

為了實現中獎概率，生成一個0到10000之間的數字，這裡很簡單，我們看數字落在哪個百分位，根據此決定是否發送獎金：

```func
() game::start(slice sender_addr, int msg_value, int hash) impure inline_ref {
    throw_unless(exit::invalid_bet(), msg_value == 1ton());
    int new_hash = slice_hash(
        begin_cell()
            .store_uint(db::hash, 256)
            .store_uint(hash, 256)
            .store_uint(cur_lt(), 64)
            .store_uint(now(), 64)
        .end_cell()
        .begin_parse()
    );

    randomize(new_hash);
    db::hash = new_hash;

    int number = rand(10000); ;; [0; 10000)
    db::last_number = number;
    db::available_balance += 1ton();

    if (number < 10) { ;; win 1/2 available balance
        int win = db::available_balance / 2;
        int commission = muldiv(win, 10, 100);
        win -= commission;

        db::available_balance -= (win + commission);
        db::service_balance += commission;

        game::payout(sender_addr, win, msg::jackpot());

        return ();
    }

    if (number < 1000) { ;; win x5
        int win = 5 * 1ton();
        int commission = muldiv(win, 10, 100);
        win -= commission;

        db::available_balance -= (win + commission);
        db::service_balance += commission;
        game::payout(sender_addr, win, msg::x5());

        return ();
    }

    if (number < 2000) { ;; win x2
        int win = 2 * 1ton();
        int commission = muldiv(win, 10, 100);
        win -= commission;

        db::available_balance -= (win + commission);
        db::service_balance += commission;
        game::payout(sender_addr, win, msg::x2());

        return ();
    }

}
```

## 結論

我會在我的頻道 https://t.me/ton_learn 上寫類似的TON網絡教學和分析。期待您的訂閱。