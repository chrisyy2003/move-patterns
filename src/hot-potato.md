# Hot Potato

|||
|-|-|
| **Name** | Hot Potato |
| **Origin** | [Sui Project](https://github.com/MystenLabs/sui/blob/20e68787b3ace2b408ba0c1d8d9117fc5206cb05/sui_programmability/examples/defi/sources/FlashLender.move#L19) / [Todd Nowacki](https://github.com/tnowacki) |
| **Example** | [FlashLender.move](https://github.com/MystenLabs/sui/blob/a6156aeaf332b9f257cf04063a9a751a7a431360/sui_programmability/examples/defi/sources/flash_lender.move) |
| **Depends on** | None |
| **Known to work on** | Move |

## 概述

Hot Potato模式受益于Move中的Ability，Hot Potato是一个没有`key`、`store`和`drop`能力的结构，强制该结构在创建它的模块中使用掉。这种模式在闪电贷款这样的需要原子性的程序中是理想的，因为在同一交易中必须启动和偿还贷款。

```move
struct Hot_Potato {}
```

相较于Solidity中的闪电贷的实现，Move中的实现是优雅的。在Solidity中会涉及到较多的动态调用，并且存在重入，拒绝服务攻击等问题。但在Move中，当函数返回了一个不具有任何的ability的potato时，由于没有drop的ability也，所以没办法储存到全局里面去，也没有办法去储存到其他结构体中。在函数结束的时也不能丢弃，所以必须解构这个资源，或者传给另外一个可以使用这个potato的一个函数。

所以通过这个方式，可以来实现函数的调用流程。模块可以在没有调用者任何背景和条件下，**保证调用者一定会按照预先设定的顺序去调用函数**。

> 闪电贷本质也是一个调用顺序的问题
> 

## 如何使用

### Aptos

Aptos上[Liqudswasp](https://github.com/pontem-network/liquidswap/blob/main/sources/swap/liquidity_pool.move#L99)项目实现了FlashLoan，这里提取了核心的代码。

```move
public fun flashloan<X, Y, Curve>(x_loan: u64, y_loan: u64): (Coin<X>, Coin<Y>, Flashloan<X, Y, Curve>)
    acquires LiquidityPool, EventsStore {
        let pool = borrow_global_mut<LiquidityPool<X, Y, Curve>>(@liquidswap_pool_account);
        ...
        let reserve_x = coin::value(&pool.coin_x_reserve);
        let reserve_y = coin::value(&pool.coin_y_reserve);
        // Withdraw expected amount from reserves.
        let x_loaned = coin::extract(&mut pool.coin_x_reserve, x_loan);
        let y_loaned = coin::extract(&mut pool.coin_y_reserve, y_loan);
        ...
        // Return loaned amount.
        (x_loaned, y_loaned, Flashloan<X, Y, Curve> { x_loan, y_loan })
    }

public fun pay_flashloan<X, Y, Curve>(
        x_in: Coin<X>,
        y_in: Coin<Y>,
        loan: Flashloan<X, Y, Curve>
    ) acquires LiquidityPool, EventsStore {
        ...
        let Flashloan { x_loan, y_loan } = loan;

        let x_in_val = coin::value(&x_in);
        let y_in_val = coin::value(&y_in);

        let pool = borrow_global_mut<LiquidityPool<X, Y, Curve>>(@liquidswap_pool_account);

        let x_reserve_size = coin::value(&pool.coin_x_reserve);
        let y_reserve_size = coin::value(&pool.coin_y_reserve);

        // Reserve sizes before loan out
        x_reserve_size = x_reserve_size + x_loan;
        y_reserve_size = y_reserve_size + y_loan;

        // Deposit new coins to liquidity pool.
        coin::merge(&mut pool.coin_x_reserve, x_in);
        coin::merge(&mut pool.coin_y_reserve, y_in);
        ...
    }
```

### Sui

sui[官方示例](https://github.com/MystenLabs/sui/blob/main/sui_programmability/examples/defi/sources/flash_lender.move)中同样实现了闪电贷。

当用户借款时调用`loan`函数返回一笔资金`coin`和一个记录着借贷金额`value`但没有任何`ability`的`receipt`收据，如果用户试图不归还资金，那么这个收据将被丢弃从而报错，所以必须调用`repay`函数从而销毁收据。收据的销毁完全由模块控制，销毁时验证传入的金额是否等于收据中的金额，从而保证闪电贷的逻辑正确。

```move
module example::flash_lender {
    use sui::balance::{Self, Balance};
    use sui::coin::{Self, Coin};
    use sui::object::{Self, ID, UID};
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

    /// A shared object offering flash loans to any buyer willing to pay `fee`.
    struct FlashLender<phantom T> has key {
        id: UID,
        /// Coins available to be lent to prospective borrowers
        to_lend: Balance<T>,
        /// Number of `Coin<T>`'s that will be charged for the loan.
        /// In practice, this would probably be a percentage, but
        /// we use a flat fee here for simplicity.
        fee: u64,
    }

    /// A "hot potato" struct recording the number of `Coin<T>`'s that
    /// were borrowed. Because this struct does not have the `key` or
    /// `store` ability, it cannot be transferred or otherwise placed in
    /// persistent storage. Because it does not have the `drop` ability,
    /// it cannot be discarded. Thus, the only way to get rid of this
    /// struct is to call `repay` sometime during the transaction that created it,
    /// which is exactly what we want from a flash loan.
    struct Receipt<phantom T> {
        /// ID of the flash lender object the debt holder borrowed from
        flash_lender_id: ID,
        /// Total amount of funds the borrower must repay: amount borrowed + the fee
        repay_amount: u64
    }

    /// An object conveying the privilege to withdraw funds from and deposit funds to the
    /// `FlashLender` instance with ID `flash_lender_id`. Initially granted to the creator
    /// of the `FlashLender`, and only one `AdminCap` per lender exists.
    struct AdminCap has key, store {
        id: UID,
        flash_lender_id: ID,
    }
    
    // === Creating a flash lender ===

    /// Create a shared `FlashLender` object that makes `to_lend` available for borrowing.
    /// Any borrower will need to repay the borrowed amount and `fee` by the end of the
    /// current transaction.
    public fun new<T>(to_lend: Balance<T>, fee: u64, ctx: &mut TxContext): AdminCap {
        let id = object::new(ctx);
        let flash_lender_id = object::uid_to_inner(&id);
        let flash_lender = FlashLender { id, to_lend, fee };
        // make the `FlashLender` a shared object so anyone can request loans
        transfer::share_object(flash_lender);

        // give the creator admin permissions
        AdminCap { id: object::new(ctx), flash_lender_id }
    }

    // === Core functionality: requesting a loan and repaying it ===

    /// Request a loan of `amount` from `lender`. The returned `Receipt<T>` "hot potato" ensures
    /// that the borrower will call `repay(lender, ...)` later on in this tx.
    /// Aborts if `amount` is greater that the amount that `lender` has available for lending.
    public fun loan<T>(
        self: &mut FlashLender<T>, amount: u64, ctx: &mut TxContext
    ): (Coin<T>, Receipt<T>) {
        let to_lend = &mut self.to_lend;
        assert!(balance::value(to_lend) >= amount, ELoanTooLarge);
        let loan = coin::take(to_lend, amount, ctx);
        let repay_amount = amount + self.fee;
        let receipt = Receipt { flash_lender_id: object::id(self), repay_amount };

        (loan, receipt)
    }

    /// Repay the loan recorded by `receipt` to `lender` with `payment`.
    /// Aborts if the repayment amount is incorrect or `lender` is not the `FlashLender`
    /// that issued the original loan.
    public fun repay<T>(self: &mut FlashLender<T>, payment: Coin<T>, receipt: Receipt<T>) {
        let Receipt { flash_lender_id, repay_amount } = receipt;
        assert!(object::id(self) == flash_lender_id, ERepayToWrongLender);
        assert!(coin::value(&payment) == repay_amount, EInvalidRepaymentAmount);

        coin::put(&mut self.to_lend, payment)
    }
}
```

## 总结

Hot Potato设计模式不仅仅只适用于闪电贷的场景，还可以用来控制更复杂的函数调用顺序。

例如我们想要一个制作土豆的合约，当用户调用`get_potato`时，会得到一个没有任何能力的`potato`，我们想要用户得倒之后，按照切土豆、煮土豆最后才能吃土豆的一个既定流程来操作。所以用户为了完成交易那么必须最后调用`consume_potato`，但是该函数限制了土豆必须被`cut`和`cook`，所以需要分别调用`cut_potato`和`cook_potato`，`cook_potato`中又限制了必须先被`cut`，从而合约保证了调用顺序必须为get→cut→cook→consume，从而控制了调用顺序。

```move
module example::hot_potato {
    /// Without any capability,
    struct Potato {
        has_cut: bool,
        has_cook: bool,
    }
    /// When calling this function, the `sender` will receive a `Potato` object.
    /// The `sender` can do nothing with the `Potato` such as store, drop,
    /// or move_to the global storage, except passing it to `consume_potato` function.
    public fun get_potato(_sender: &signer): Potato {
        Potato {
            has_cut: false,
            has_cook: false,
        } 
    }

    public fun cut_potatoes(potato: &mut Potato) {
        assert!(!potato.has_cut, 0);
        potato.has_cut = true;
    }

    public fun cook_potato(potato: &mut Potato) {
        assert!(!potato.has_cook && potato.has_cut, 0);
        potato.has_cook = true;
    }

    public fun consume_potato(_sender: &signer, potato: Potato) {
        assert!(potato.has_cook && potato.has_cut, 0);
        let Potato {has_cut: _, has_cook: _ } = potato; // destroy the Potato.
    }
}
```