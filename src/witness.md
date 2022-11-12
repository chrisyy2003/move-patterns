# Witness

|||
|-|-|
| **Name** | Witness |
| **Origin** | [FastX](https://github.com/MystenLabs/sui/blob/c1ed7cc451119432d32d8eacf471f314e0ce5275/fastx_programmability/sources/Coin.move#L85) / [Sam Blackshear](https://github.com/sblackshear) |
| **Example** | [Sui Move by Example](https://examples.sui.io/patterns/witness.html) / [Damir Shamanaev](https://github.com/damirka) |
| **Depends on** | None |
| **Known to work on** | Move |

## 概述

witness是一种临时资源，相关资源只能被使用一次，资源在使用后被丢弃，确保不能重复使用相同的资源来初始化任何其他结构，通常用来确认一个类型的的所有权。

witness得益于Move中的类型系统。一个类型实例化的时候，它只能在定义这个类型的模块中创建。

一个简单的例子，在framework里面定义了coin合约用来定义token标准，如果想要注册token那么合约会调用`publish_coin`。

```solidity
module framework::coin {
    /// The witness patameter ensures that the function can only be called by the module defined T.
    public fun publish_coin<T: drop>(_witness: T) {
        // register this coin to the registry table
    }
}
module examples::xcoin {
    use framework::coin;
    /// The Witness type.
    struct X has drop {}
    /// Only this module defined X can call framework::publish_coin<X>
    public fun publish() {
        coin::publish_coin<X>(X {});
    }
}
module hacker::hack {
    use framework::coin;
    use examples::xcoin::X;

    public fun publish() {
        // Illegal, X can not be constructed here.
        coin::publish_coin<X>(X {}); 
    }
}
```

那么如果此时有一个hacker想要抢先注册这个token，那么需要构造模块中的x提前调用`publish_coin`函数，但是由于Move中的类型系统限制了这种情况的发生，因为模块外部是不能构造其他模块的结构体资源。

```solidity
┌─ /sources/m.move:25:31
   │
25 │         coin::publish_coin<X>(X {}); 
   │                               ^^^^ Invalid instantiation of '(examples=0x1)::xcoin::X'.
All structs can only be constructed in the module in which they are declared
```

## 如何使用

witness在Sui中与其他Move公链有一些区别。

如果结构类型与定义它的模块名称相同且是大写，并且没有字段或者只有一个布尔字段，则意味着它是一个one-time witness类型。该类型只会在模块初始化时使用，在合约中验证是否是one-time witness类型，可以通过sui framwork中[types::is_one_time_witness](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/types.move#L10)来验证。

例如在sui的coin库中，如果需要注册一个coin类型，那么需要调用[create_currency](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/coin.move#L176)函数。函数参数则就需要一个one-time witness类型。为了传递该类型参数，需要在模块初始化`init`函数参数中第一个位置传递，即：

```solidity
// 注册一个M_COIN类型的通用Token
module examples::m_coin {
    use sui::coin;
    use sui::transfer;
    use sui::tx_context::{Self, TxContext};

		// 必须是模块名大写字母
    struct M_COIN has drop{}

		// 第一个位置传递
    fun init (witness: M, ctx: &mut TxContext) {
        let cap = coin::create_currency(witness, 8, ctx);
        transfer::transfer(cap, tx_context::sender(ctx));
    }
}
```

sui中的初始化函数只能有一个或者两个参数，且最后的参数一定是`&mut TxContext`类型，one-time witness类型同样是模块初始化时自动传递的。

> `init`函数如果传递除了上述提到的以外的参数，Move编译器能够编译通过，但是部署时Sui的验证器会报错。此外如果第一个传递的参数不是one-time witness类型，同样也只会在部署时Sui验证才会报错。
> 

## 总结

witness模式通常其他模式一同使用，例如Wrapper和capability模式。

除了在sui的coin标准库中使用到了wintess，以下例子同样也有使用到：

- [Liquidity pool](https://github.com/MystenLabs/sui/blob/main/sui_programmability/examples/defi/sources/pool.move)
- [Regulated coin](https://github.com/MystenLabs/sui/blob/main/sui_programmability/examples/fungible_tokens/sources/regulated_coin.move)