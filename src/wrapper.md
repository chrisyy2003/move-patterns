# Wrapper

|||
|-|-|
| **Name** | Wrapper |
| **Origin** | [Ville Sundell](https://github.com/villesundell) |
| **Example** | [TaoHe project](https://github.com/taoheorg/taohe) |
| **Depends on** | None |
| **Known to work on** | Move |

## 概述

Wrapper是将一个对象嵌套在另一个对象的模式。例如在上个Capability的例子中，通常合约初始化时会铸造一个Capability，那么在这之后如果我想将这个Capability转让给其他人怎么办？

在Move中我们往一个地址下存放数据时，会使用到`move_to`函数，但是此函数接受的参数是`signer`，也就是没有账号的签名发起的交易是没有办法**主动向地址下存放资源**。所以在Move编程实践的当中就产生了Wrapper这么一个模式。

Wrapper模式指的是，如果需要主动给一个地址发送一个资源时，首先把要发送的对象放在一个Wrapper对象里面包装一下，随后用户需要**主动调用**接受Wrapper的函数，此时函数会通过Wrapper这个资源取出来其中被包装的对象，由于是用户主动调用的该函数，所以可以直接存放到用户下面。

> 这种场景总主要是因为不能直接发送，所以需要先预把对象，预存到一个地方，然后接受者再主动去确认拿取，就如同现实生活中Offer需要确认一般，所以这种模式一般也叫做Offer模式

## 如何使用

### Aptos

为了实现资源的转移，可以在合约中可以定义一个Offer资源，类型参数接受一个泛型`T`，从而可以给任意类型包装，然后定义一个`receipt`字段进行访问控制。

用户接受Offer时，首先从Offer地址下取出来，随后在验证地址是否和Offer中的地址相同。

```move
module example::offer {
    use std::signer;

    struct OwnerCapability has key, store {}

    /// 定义一个Offer来包装一个对象
    struct Offer<T: key + store> has key, store {
        receipt: address,
        offer: T,
    }
    /// 发送一个Offer到地址to下
    public entry fun send_offer(sender: &signer, to: address) {
        move_to<Offer<OwnerCapability>>(sender, Offer<OwnerCapability> {
            receipt: to,
            offer: OwnerCapability {},
        });
    }
    /// 地址to调用函数从而接受Offer中的对象
    public entry fun accept_role(sender: &signer, grantor: address) acquires Offer {
        assert!(exists<Offer<OwnerCapability>>(grantor), 0);
        let Offer<OwnerCapability> { receipt, offer: admin_cap } = move_from<Offer<OwnerCapability>>(grantor);
        assert!(receipt == signer::address_of(sender), 1);
        move_to<OwnerCapability>(sender, admin_cap);
    } 
}
```

### Sui

Sui中由于资源都是一个Object，每一个对象都是属于一个Owner的，所以需要主动给用户发送资源时，只需要使用`transfer::transfer(obj, to)`，不需要使用Offer模式

## 总结

Wrapper中由于定义了一个`receipt`字段限制了只能有指定的地址来接受Offer，在其他场景，例如需要赋予多个人一个Capability又如何解决？此外在Move中，一个账户下只能存放一种类型的资源，那么针对上面的情况如果想再存放另外一个人的Offer也是不可行的。

所以在我们实践中，可以将`receipt`字段替换为一个`vector`或者一个`table`来储存多个目标地址。

```move
/// 定义一个Offer来包装一个对象
struct Wrapper<T: key + store> has key, store {
    receipt: address,
    map: VecMap<T>,
}
```

------

此外在其他场景中如果没有提供存放资源的接口，那么可以通过Wrapper来将其保存。

例如下面的`Coin`模块，由于Move中的特性，定义的资源只能在定义这个资源的模块内操作，所以当mint函数返回一个`Coin`对象时，用户不能直接使用`move_to`来存放。

```move
module example::coin {
    struct Coin has key, store {
        value: u64
    }
    struct MintCapability has key, store {}

    public fun mint(amount: u64, _cap: &MintCapability): Coin {
        Coin { value: amount }
    }
}
```

那么此时用户可以定义一个自己的Wrapper结构将其包装，随后就可以将这个嵌套结构一同放到自己的地址下。

```move
struct Wrapper has key, store {
        coin: Coin
}
```

