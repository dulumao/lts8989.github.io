# 2 代码和项目组织

> **本章概要**
* 习惯性地组织我们的代码
* 高效处理抽象：接口和泛型
* 关于如何构建项目的最佳实践

以干净、惯用和可维护的方式组织 Go 代码库并非易事。理解与代码和项目组织相关的所有最佳实践需要经验和错误。例如，要避免哪些陷阱，例如变量遮蔽和嵌套代码滥用，如何构建包，何时何地使用接口或泛型，init 函数，实用程序包？在本章中，我们将深入研究常见的组织错误。