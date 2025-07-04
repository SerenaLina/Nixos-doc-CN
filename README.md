### **引言（Introduction）**

Nix 是一个纯函数式的软件包管理器。  
这意味着它像 Haskell 这样的纯函数式编程语言一样处理软件包 —— 软件包是通过没有副作用的函数构建的，并且在构建完成之后永远不会改变。

Nix 将软件包存储在 **Nix 存储区** 中，通常是目录 `/nix/store`。在该目录下，每个软件包都有自己的唯一子目录，例如：

`/nix/store/b6gvzjyb2pg0kjfwrjmg1vfhh54ad73z-firefox-33.1/`

其中 `b6gvzjyb2pg0...` 是一个唯一标识符，它包含了该软件包的所有依赖信息（这是软件包构建依赖图的一个加密哈希）。这种方式实现了许多强大的功能：

---

#### **多个版本（Multiple versions）**

你可以同时安装同一个软件包的多个版本或变体。  
这在不同应用程序依赖于该软件包的不同版本时尤其重要 —— 它避免了所谓的 “DLL 地狱”（即动态链接库冲突问题）。

由于 Nix 的哈希方案，不同版本的软件包会被放入 Nix 存储中的不同路径中，因此它们彼此不会干扰。

这样一来，执行升级或卸载操作不会破坏其他应用程序，因为这些操作不会“破坏性地”更新或删除被其他软件包使用的文件。

---

#### **完整依赖性（Complete dependencies）**

Nix 可以帮助你确保软件包依赖关系的声明是完整的。  
在其他软件包管理系统（例如 RPM）中，为每个软件包声明依赖关系时，没有办法保证依赖声明是完整的。如果你忘记声明某个依赖项，那么只要你本地安装了它，软件包就能构建和运行；但在最终用户的系统上，如果缺少该依赖，就会失败。

而在 Nix 中，软件包不会被安装到像 `/usr/bin` 这样的“全局”位置，而是被安装到特定路径下，如：

`/nix/store/5lbfaxb722zp…-openssl-0.9.8d/include`

这就大大降低了依赖声明不完整的风险。因为构建工具（如编译器）不会在这些路径中自动搜索依赖项，如果你能成功构建某个软件包，那就说明你已经显式声明了它的依赖。

---

#### **运行时依赖（Runtime dependencies）**

一旦软件包构建完成，Nix 会通过扫描二进制文件中 Nix 存储路径的哈希部分（例如 `r8vvq9kq...`）来找到其运行时依赖。  
这听起来似乎不安全，但实际上运作得非常好。
#### **多用户支持（Multi-user support）**

Nix 支持**多用户模式**。  
这意味着**非特权用户**（即普通用户）也可以**安全地安装软件**。每个用户可以拥有不同的“配置文件”（profile），也就是一组在 Nix 存储区中安装的软件包，这些软件包会出现在该用户的 `PATH` 环境变量中。

如果一个用户安装了某个软件包，而另一个用户之前已经安装过该软件包，那么这个软件包**不会被重复构建或下载**。

同时，Nix 也确保**安全性** —— 一个用户**无法在其他用户可能会使用的包中注入木马程序**。

---

#### **原子升级与回滚（Atomic upgrades and rollbacks）**

由于在 Nix 中，软件包管理操作**不会覆盖 Nix 存储区中的现有包**，而是将新版本放入不同的路径中，这就使得操作具备**原子性**。

也就是说，在升级某个软件包的过程中，**不会存在一个时间窗口**，这个窗口内的软件包有一部分是旧版本，有一部分是新版本 —— 这是非常重要的，因为如果某个程序在这个阶段被启动，很可能会崩溃。

而且，因为旧版本的软件包没有被覆盖，升级后它们仍然存在。  
这意味着你可以**随时回滚**到旧版本：

`$ nix-env --upgrade --attr nixpkgs.some-package $ nix-env --rollback`

---

#### **垃圾回收（Garbage collection）**

当你像这样卸载一个软件包时：


`$ nix-env --uninstall firefox`

该软件包**不会立即从系统中删除**（因为你可能还会想要回滚，或者它可能仍然存在于其他用户的配置文件中）。

你可以通过运行垃圾回收器来**安全地删除未使用的软件包**：


`$ nix-collect-garbage`

这会删除所有**没有被任何用户配置文件或当前运行程序使用的软件包**。


---


---

#### **函数式软件包语言（Functional package language）**

Nix 中的软件包是通过 **Nix 表达式** 构建的，Nix 表达式是一种简单的**函数式语言**。  
一个 Nix 表达式描述了构建软件包所需的所有信息（称为“**导出对象** derivation”）：包括其它依赖软件包、源码、构建脚本、构建时所需的环境变量等。

Nix 极力保证构建的**确定性**：

> 也就是说，同一个 Nix 表达式，无论构建多少次，最终的构建结果都应该是一样的。

因为它是函数式语言，所以非常容易支持构建软件包的不同“变体”：只需要将表达式写成一个函数，然后使用不同的参数多次调用它即可。  
借助哈希机制，这些变体在 Nix 存储中也不会发生冲突。

---

#### **透明的源码/二进制部署（Transparent source/binary deployment）**

通常，Nix 表达式描述的是如何从源码构建一个软件包。  
因此，当你执行以下命令时：

```bash
$ nix-env --install --attr nixpkgs.firefox
```

它可能会触发大量的构建任务 —— 不仅仅是 Firefox 本身，还包括它的所有依赖（甚至可能包括 C 标准库和编译器），除非这些依赖已经存在于 Nix 存储中。

这是一种**源码部署模型**。

但对于大多数用户而言，从源码构建软件包并不理想 —— 实在是**太耗时**了。

为此，Nix 可以自动跳过从源码构建的过程，改为使用**二进制缓存**（binary cache），它是一个提供预编译二进制文件的网络服务。

例如，当需要从源码构建路径：

```
/nix/store/b6gvzjyb2pg0…-firefox-33.1
```

时，Nix 会**首先检查**是否存在如下网址的缓存文件：

```
https://cache.nixos.org/b6gvzjyb2pg0….narinfo
```

如果存在，就会从那里**下载预构建的二进制文件**；  
如果不存在，才会退回到从源码构建的方式。

---

#### **Nix 软件包集合（Nix Packages collection）**

我们提供了大量的 Nix 表达式，包含了数百个现有的 Unix 软件包，  
这个集合被称为 **Nix Packages Collection（Nixpkgs）**。

---

#### **构建环境管理（Managing build environments）**

Nix 对开发者来说非常有用，它能非常方便地**自动设置软件包的构建环境**。

只要你有一个 Nix 表达式来描述你的软件包依赖，你就可以使用命令：

```bash
$ nix-shell
```

该命令会构建或下载依赖（如果还没在本地 Nix 存储中），然后启动一个 Bash shell，其中所有必需的环境变量（例如编译器的搜索路径）都已自动设置好。

---


---

#### **构建环境管理 示例（Managing build environments - Example）**

例如，下面这个命令会获取 Pan 新闻阅读器（newsreader）的所有依赖项，这些依赖项在它的 Nix 表达式中有定义：

```bash
$ nix-shell '<nixpkgs>' --attr pan
```

执行完后，你将进入一个 shell，在其中你可以编辑、构建和测试这个软件包，例如：

```bash
[nix-shell]$ unpackPhase
[nix-shell]$ cd pan-*
[nix-shell]$ configurePhase
[nix-shell]$ buildPhase
[nix-shell]$ ./pan/gui/pan
```

> 这些命令代表典型的构建步骤：解压 → 进入目录 → 配置 → 构建 → 启动 GUI。

---

#### **可移植性（Portability）**

Nix 可运行于 **Linux** 和 **macOS**。

---

#### **NixOS**

**NixOS** 是一个基于 Nix 的 Linux 发行版。  
它不仅使用 Nix 来进行软件包管理，还用来管理系统配置（例如构建 `/etc` 目录中的配置文件）。

这带来了以下优点：

- **系统配置可回滚**：整个系统配置可以轻松回到之前的状态。
    
- **用户无需 root 权限** 也可以安装软件。
    

更多信息与下载，请访问 [NixOS 官网](https://nixos.org/)。

---

#### **许可证（License）**

Nix 在 **GNU LGPLv2.1** 或（你可以选择）**任何更高版本**的条款下发布。

---
