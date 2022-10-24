# Introduction

Move是一种安全、沙盒式和形式化验证的下一代编程语言，它的第一个用例是 [Diem](https://en.wikipedia.org/wiki/ Diem_(digital_currency)) 区块链(当时名字叫Libra, 脸书团队开发的项目, 译者注), Move 为其实现提供了基础。 Move 允许开发人员编写灵活管理和转移数字资产的程序，同时提供安全保护，防止对那些链上资产的攻击。不仅如此，Move 也可用于区块链世界之外的开发场景。

Move 的诞生从[Rust](https://www.rust-lang.org/)中吸取了灵感，Move也是因为使用具有移动(move)语义的资源类型作为数字资产(例如货币)的显式表示而得名。

今天，Move 及其虚拟机正在为 [多个区块链](https://github.com/MystenLabs/awesome-move#move-powered-blockchains) 提供动力，其中大部分仍处于早期开发阶段。

## What is Move Patterns for? 

本书旨在讨论面向资源的语言的软件设计模式和最佳实践，尤其是Move(特别是sui move)及其风格。这部分涵盖了 Move 中广泛使用的编程模式；其中一些只能存在于 Move 中。然而，这些模式中的一些很可能也可以在其他一些基于资源的语言和生态系统中实现。

本书不是 Move 或任何其他面向资源的语言的指南。 有关 Move 本身的书籍，请参阅 [awesome-move#books](https://github.com/MystenLabs/awesome-move#books)。 另请参阅 [awesome-move](https://github.com/MystenLabs/awesome-move) 以获取来自 Move 编程语言社区的代码和内容的精选列表。

## Technical disclaimer

This book is designed to be viewed digitally with hyperlink support (such as PDF or web format). For now the [full software pattern format](https://en.wikipedia.org/wiki/Software_design_pattern#Documentation) is not followed, instead the patterns are simply defined by a short summary and examples.

## License

[![CC BY 4.0 license button][cc-by-png]][cc-by]

Move Patterns: Design Patterns for Resource Based Programming © 2022 by [Ville Sundell](https://github.com/villesundell) is licensed under CC BY 4.0. To view a copy of this license, visit [http://creativecommons.org/licenses/by/4.0/](http://creativecommons.org/licenses/by/4.0/).

[cc-by-png]: https://mirrors.creativecommons.org/presskit/buttons/88x31/svg/by.svg "CC BY 4.0 license button"
[cc-by]: https://creativecommons.org/licenses/by/4.0/ "Creative Commons Attribution 4.0 International License"