# Capability

|||
|-|-|
| **Name** | Capability |
| **Origin** | [Libra Project](https://github.com/move-language/move/blob/5e034dde19a5320d7e2bdc9da25114e816b4454d/language/stdlib/modules/libra_coin.mvir#L8) / Unknown Author |
| **Example** | [Sui Move by Example](https://examples.sui.io/patterns/capability.html) / [Damir Shamanaev](https://github.com/damirka) |
| **Depends on** | None |
| **Known to work on** | Move |

## 概述

Capability是一个能够证明**资源所有者**特定权限的资源（注意：它是一个资源也就是一个Move中的结构体），其作用主要是用来进行访问控制。

例如当我们想限制某个资源的铸造权，管理权，函数调用权时，便可以采用Capability这种设计模式。这也是Move智能合约里面使用最广泛的一个设计模式，例如sui-framework中的[TreasuryCap](https://github.com/MystenLabs/sui/blob/main/crates/sui-framework/sources/coin.move#L32-L35)。这是也是已知最古老的 Move 设计模式，可追溯到 Libra 项目及其代币智能合约，其中功能用于授权铸币。

## 如何使用

Capability本质是一个资源对象，只是被可信任的用户持有。通常在合约中我们可以定义一个`AdminCap`来代表本模块的控制权限，如果某个用户持有就可以用户可信，其中资源对象内不需要任何的字段。

```move
struct AdminCap has key, store {}
```

一般Capability生成在模块初始化的时候，例如Sui中的init函数，就可以赋予部署者一个Capability的资源，然后通过`move_to`然后储存到它的账户下。

然后当需要使用到有访问权限的函数时，此时函数就会检查调用者地址下是否存在这个Capability资源，如果存在那么说明调用者拥有正确的访问权限。

### Aptos

```move
module example::capability {
    use std::signer;

    // 定义一个OwnerCapability类型
    struct OwnerCapability has key, store {}

    // 向管理者地址下存放一个OwnerCapability资源
    public entry fun init(sender: signer) {
        assert!(signer::address_of(&sender) == @example, 0);
        move_to(&sender, OwnerCapability {})
    }

    // Only user with OwnerCapability can call this function
    // 只有具有OwnerCapability的用户才能调用此函数
    public entry fun admin_fun(sender: &signer) acquires OwnerCapability {
        assert!(exists<OwnerCapability>(signer::address_of(sender)), 1);
        let _cap = borrow_global<OwnerCapability>(signer::address_of(sender));
        // do something with the cap.
    }
}
```

### Sui

sui中的Move与Aptos或者starcoin中的Core Move有所不同，sui封装了全局操作函数，具体可以查看sui的[官方文档](https://docs.sui.io/learn/sui-move-diffs)。

```move
module capability::m {
    use sui::transfer;
    use sui::object::{Self, UID};
    use sui::tx_context::{Self, TxContext};

    struct OwnerCapability has key { id: UID }

    /// A Coin Type
    struct Coin has key, store {
        id: UID,
        value: u64
    }

    /// Module initializer is called once on module publish.
    /// Here we create only one instance of `OwnerCapability` and send it to the publisher.
    fun init(ctx: &mut TxContext) {
        transfer::transfer(OwnerCapability {
            id: object::new(ctx)
        }, tx_context::sender(ctx))
    }

    /// The entry function can not be called if `OwnerCapability` is not passed as
    /// the first argument. Hence only owner of the `OwnerCapability` can perform
    /// this action.
    public entry fun mint_and_transfer(
        _: &OwnerCapability, to: address, ctx: &mut TxContext
    ) {
        transfer::transfer(Coin {
            id: object::new(ctx),
            value: 100,
        }, to)
    }
}
```

## 总结

相较于其他语言的访问控制（例如Solidity中定一个`address owner`即可，或者定义一个`mapping`），Move中的访问控制实现上是复杂的，主要由于Move中独特的存储架构，模组不存储状态变量，需要将资源存储到一个账户下面。

但是单单对于对于访问控制来说，实现方式有很多种，例如在Move中可以使用`VecMap`等数据结构达到同样的效果，但是Capability这种模式跟面向资源编程的概念更加契合且容易使用。

> 需要注意的是，既然Capability作为一个凭证，显然是不能`copy`的能力的，如果别人拿到`copy`之后，它可以通过复制从而获得更多的Capability。同样在正常的业务逻辑下，Capability也是不能随意的丢弃也就是不能有`drop`能力，因为丢弃的话显然会造成一些不可逆的影响。
