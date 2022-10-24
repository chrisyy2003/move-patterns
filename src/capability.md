# Capability

|||
|-|-|
| **Name** | Capability |
| **Origin** | [Libra Project](https://github.com/move-language/move/blob/5e034dde19a5320d7e2bdc9da25114e816b4454d/language/stdlib/modules/libra_coin.mvir#L8) / Unknown Author |
| **Example** | [Sui Move by Example](https://examples.sui.io/patterns/capability.html) / [Damir Shamanaev](https://github.com/damirka) |
| **Depends on** | None |
| **Known to work on** | Move |

## Summary

**能力**是证明**资源的所有者**被允许执行特定操作的资源，最常见的功能之一是`TreasuryCap`（在[sui::coin](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/coin.move#L19)中定义）。这是已知最古老的 Move 设计模式，可追溯到 Libra 项目及其代币智能合约，其中功能用于授权铸币。

## Examples

例如当我们想通过限制某个资源的铸造权，**管理权**时Capability是一个很好的选择。

```move
module examples::item {
    use sui::transfer;
    use sui::object::{Self, UID};
    use std::string::{Self, String};
    use sui::tx_context::{Self, TxContext};

    /// Type that marks Capability to create new `Item`s.
    struct AdminCap has key { id: UID }

    /// Custom NFT-like type.
    struct Item has key, store { id: UID, name: String }

    /// Module initializer is called once on module publish.
    /// Here we create only one instance of `AdminCap` and send it to the publisher.
    fun init(ctx: &mut TxContext) {
        transfer::transfer(AdminCap {
            id: object::new(ctx)
        }, tx_context::sender(ctx))
    }

    /// The entry function can not be called if `AdminCap` is not passed as
    /// the first argument. Hence only owner of the `AdminCap` can perform
    /// this action.
    public entry fun create_and_send(
        _: &AdminCap, name: vector<u8>, to: address, ctx: &mut TxContext
    ) {
        transfer::transfer(Item {
            id: object::new(ctx),
            name: string::utf8(name)
        }, to)
    }
}

```

*Example for Sui Move is taken from the book [Sui Move by Example](https://examples.sui.io/patterns/capability.html) by [Damir Shamanaev](https://github.com/damirka).*