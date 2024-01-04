---
title: '浅析 MoveScription 合约'
description: '简单分析 MoveScription 合约的安全性，从源码种分析铸造时所付的 Sui 不会丢失的原理。'
pubDate: 'Jan 03 2024'
heroImage: '/blog-placeholder-1.jpg'
---

在 `movescription` 合约中，锁定和返还 Sui 是该合约中的关键环节，尤其是在铸造、管理和烧毁（`Movescription`）的过程中，现在让我们来分析这些过程：

## 锁定 SUI

1. **铸造过程**：当新的 `Movescription` 被铸造时，将收取一定的 SUI 费用。这个费用被封装在 `Movescription` 对象的余额中。这一过程的相关代码在 `do_mint` 函数中，其中铸币费（`fee_coin`）被转换为余额（`fee_balance`）并存储在 `Movescription` 对象中。

```rust
    public fun do_mint(
        tick_record: &mut TickRecord,
        fee_coin: Coin<SUI>,
        clk: &Clock,
        ctx: &mut TxContext
    ) {
        assert!(tick_record.version == VERSION, EVersionMismatched);
        assert!(tick_record.remain > 0, ENotEnoughToMint);
        let now_ms = clock::timestamp_ms(clk);
        assert!(now_ms >= tick_record.start_time_ms, ENotStarted);
        tick_record.total_transactions = tick_record.total_transactions + 1;

        let sender: address = tx_context::sender(ctx);
        let tick: String = tick_record.tick;

        let mint_fee_coin = if(coin::value<SUI>(&fee_coin) == tick_record.mint_fee){
            fee_coin
        }else{
            let mint_fee_coin = coin::split<SUI>(&mut fee_coin, tick_record.mint_fee, ctx);
            transfer::public_transfer(fee_coin, sender);
            mint_fee_coin
        };
        let fee_balance: Balance<SUI> = coin::into_balance<SUI>(mint_fee_coin);

        ...

        emit(MintTick {
            sender: sender,
            tick: tick,
        });
    }
```

2. **铭文结构**：每个 `Movescription` 结构体都包括一个类型为 `Balance<SUI>` 的 `acc` 字段。这个字段持有与铭文相关联的锁定的 SUI 余额。

   ```rust
   struct Movescription has key, store {
       ...
       acc: Balance<SUI>,
       ...
   }
   ```

3. **费用处理**：当铭文被铸造时，SUI 费用被锁定在 `Movescription` 的 `acc` 字段中。这一点在 `do_mint` 函数中很明显，其中 `fee_balance` 是处理 `fee_coin` 的结果。

## 烧毁后返还 SUI

1. **烧毁过程**：`burn` 函数允许用户烧毁铭文。当这种情况发生时，与铭文相关联的任何锁定的 SUI 应该返还给用户。

```rust
    public entry fun burn(
        tick_record: &mut TickRecord,
        inscription: Movescription,
        ctx: &mut TxContext
    ) {
        let acc = do_burn(tick_record, inscription, ctx);
        transfer::public_transfer(acc, tx_context::sender(ctx));
    }

```

2. **将余额转换为代币**：在 `do_burn` 函数中，铭文的 `acc` 余额被转换回 `Coin<SUI>` 对象。这是通过使用 `coin::from_balance<SUI>` 函数完成的，有效地解锁了 SUI 并使其可以用于转移。

   ```rust
   public fun do_burn(
       tick_record: &mut TickRecord,
       inscription: Movescription,
       ctx: &mut TxContext
   ) : Coin<SUI> {
       ...
       let acc: Coin<SUI> = coin::from_balance<SUI>(acc, ctx);
       ...
   }
   ```

3. **将 SUI 返还给用户**：转换后，`burn` 函数使用 `transfer::public_transfer` 将解锁的 SUI 发送回发起烧毁交易的用户（sender）。

   ```rust
   public entry fun burn(
       tick_record: &mut TickRecord,
       inscription: Movescription,
       ctx: &mut TxContext
   ) {
       let acc = do_burn(tick_record, inscription, ctx);
       transfer::public_transfer(acc, tx_context::sender(ctx));
   }
   ```

## 结论

在 `Movescription` 合约中，SUI 在铸造过程中作为费用被锁定在 `Movescription` 结构体的余额(balance)中。当铭文被烧毁时，锁定的 SUI 被转换回 `Coin<SUI>` 并返还给用户。这一机制确保了 SUI 在铭文铸造时被安全锁定，用户(sender)可以随时 `Burn` 烧毁铭文，将锁定的 `sui` 拿回自己的钱包。

`movescripton` 模块在 Sui 上实现了一种 SFT，创造出了智能铭文
