
# 介绍

Move是一种安全、沙盒式和形式化验证的下一代编程语言，它的第一个用例是[Diem](https://en.wikipedia.org/wiki/Diem_(digital_currency))区块链（当时名字叫Libra, 脸书团队开发的项目, 译者注）,Move为其实现提供了基础。 Move 允许开发人员编写灵活管理和转移数字资产的程序，同时提供安全保护，防止对那些链上资产的攻击。不仅如此，Move 也可用于区块链世界之外的开发场景。

Move 的诞生从[Rust](https://www.rust-lang.org/)中吸取了灵感，Move也是因为使用具有移动（move）语义的资源类型作为数字资产（例如货币）的显式表示而得名。

今天，Move 及其虚拟机正在为 [多个区块链](https://github.com/MystenLabs/awesome-move#move-powered-blockchains) 提供动力，其中大部分仍处于早期开发阶段。

## 什么是Move Patterns

本书旨在讨论面向资源的语言的软件设计模式和最佳实践，尤其是Move及其风格。这部分涵盖了 Move 中广泛使用的编程模式，为什么会存在Move设计模式，主要有以下三个方面。

- 面向资源编程
  Move是一种新的编程语言，其特点是面向资源编程，对于区块链最核心的 Token 资产进行了更为贴合的处理，实现了真正意义上的数字资产化。Move与传统面向对象的编程语言有着很大不同，所以一些面向对象的设计模式在Move中是不存在的（例如Uniswap中的工厂模式）。当然，这些模式中的一些很可能也可以在其他一些基于资源的语言和生态系统中实现。

- 状态存储机制
  在Solidity中，能够定义并保存自己的状态变量，变量的值放在全局储存上，在合约中可以直接通过全局变量直接读取或者修改它。

  ```solidity
  // A solidity examply
  // set msg.sender to owner
  contract A {
      // 定义一个状态变量
      address owner;
      function setOwner() public {
  	// 通过变量名直接修改
          owner = msg.sender;
      }
  }
  ```

  但是在Move中存储方式是完全不一样的，Move合约并不直接存储资源，代码中的每一个变量都是一个资源对象，是资源对象那么必须通过显示的接口去明确的调用。

  Move中对资源的访问都是通过[全局存储操作](http://movebook.chrisyy.top/global-storage-operators.html)接口来访问的。操作函数包括`move_to`或者`move from`或者`borrow_global`，`borrow_global_mut`等函数。从全局储存里面取出资源，或者存放到账户下去，引用资源对象或者修改，都是需要**要开发者显示的去表示。**

  ```move
  module example::m {
      // A Coin type
      // 一种Coin类型的资源
      struct Coin has key, store{
          value: u64
      }
      // send sender a coin value of 100
      // 在sender地址下存放100个coin
      public entry fun mint(sender: &signer) {
          move_to(sender, Coin {
              value: 100
          });
      }
  }
  ```

- Ability 
  Ability是 Move 语言中的一种类型特性，用于控制对给定类型的值允许哪些操作。
  
  - [copy](<http://movebook.chrisyy.top/abilities.html#copy>)复制：允许此类型的值被复制
  - [drop](<http://movebook.chrisyy.top/abilities.html#drop>) 丢弃：允许此类型的值被弹出/丢弃，没有话表示必须在函数结束之前将这个值销毁或者转移出去。
  - [store](<http://movebook.chrisyy.top/abilities.html#store>) 存储：允许此类型的值存在于全局存储中或者某个结构体中
  - [key](<http://movebook.chrisyy.top/abilities.html#key>) 键值：允许此类型作为全局存储中的键(具有 `key` 能力的类型才能保存到全局存储中)

作为面向资源的编程语言，以上三个特点也是与其他语言非常不同的地方，基于资源编程的Move编程模式，也主要是围绕这些特性产生的。

此外本书不是 Move 或任何其他面向资源的语言的指南。 有关 Move 本身的书籍，请参阅 [awesome-move#books](https://github.com/MystenLabs/awesome-move#books)。 另请参阅 [awesome-move](https://github.com/MystenLabs/awesome-move) 以获取来自 Move 编程语言社区的代码和内容的精选列表。

## Technical disclaimer

This book is designed to be viewed digitally with hyperlink support (such as PDF or web format). For now the [full software pattern format](https://en.wikipedia.org/wiki/Software_design_pattern#Documentation) is not followed, instead the patterns are simply defined by a short summary and examples.

## License

[![CC BY 4.0 license button][cc-by-png]][cc-by]

Move Patterns: Design Patterns for Resource Based Programming © 2022 by [Ville Sundell](https://github.com/villesundell) is licensed under CC BY 4.0. To view a copy of this license, visit [http://creativecommons.org/licenses/by/4.0/](http://creativecommons.org/licenses/by/4.0/).

[cc-by-png]: https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by.svg "CC BY 4.0 license button"
[cc-by]: https://creativecommons.org/licenses/by/4.0/ "Creative Commons Attribution 4.0 International License"