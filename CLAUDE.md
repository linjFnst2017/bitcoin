# CLAUDE.md

本文件为 Claude Code (claude.ai/code) 在该仓库中工作时提供指导。

## 构建

```sh
# 配置构建（要求 out-of-source 构建）
cmake -B build

# 构建全部
cmake --build build -j "$(($(nproc)+1))"

# 构建特定目标（迭代开发时更快）
cmake --build build --target bitcoind bitcoin-cli
cmake --build build --target bitcoin-qt
cmake --build build --target test_bitcoin
cmake --build build --target bench_bitcoin

# 查看所有可配置选项
cmake -B build -LH
```

## 测试

```sh
# 运行所有单元测试
ctest --test-dir build

# 运行单元测试，失败时显示详细输出
ctest --test-dir build --output-on-failure

# 运行指定的单元测试套件
build/bin/test_bitcoin --run_test=<套件名>

# 运行套件中的某个测试用例
build/bin/test_bitcoin --run_test=<套件名>/<测试用例>

# 运行功能（集成）测试
build/test/functional/test_runner.py

# 运行单个功能测试
build/test/functional/test_runner.py <测试名>
```

## 代码检查和格式化

```sh
# 运行所有 linter（CI 检查）
ci/lint/run-lint.py

# 格式化 C++ 代码（对变更行使用 clang-format-diff）
git diff | contrib/devtools/clang-format-diff.py -p1

# 运行 clang-tidy（需要 clang 编译器和 compile_commands.json）
cd src && run-clang-tidy -p ../build -j $(nproc) ./path/to/file.cpp

# 格式化 Python 代码
git diff | yapf-diff
```

## 代码架构

Bitcoin Core 是一个 C++20 CMake 项目（约 v31.99，预发布版本），通过 `src/interfaces/` 实现了清晰的**分层隔离**。

### `src/` 下的核心目录

| 目录 | 用途 |
|-----------|---------|
| [src/node/](src/node/) | 节点状态 — 验证、交易池、区块存储、挖矿、P2P 连接。禁止直接调用 wallet/ 或 qt/ 的代码。 |
| [src/wallet/](src/wallet/) | 钱包逻辑 — 币选择、密钥管理 (scriptpubkeyman)、SQLite/BDB 存储、PSBT、手续费估算。 |
| [src/qt/](src/qt/) | Qt6 GUI — 主窗口、交易视图、币控制、RPC 控制台。 |
| [src/rpc/](src/rpc/) | RPC 处理器 — 区块链、交易池、挖矿、钱包、节点、原始交易等。 |
| [src/kernel/](src/kernel/) | `libbitcoinkernel` 共享库 — 共识关键代码。最小依赖，兼容 C 的 API。 |
| [src/consensus/](src/consensus/) | 纯共识规则 — `amount.h`、`tx_check`、`tx_verify`、`merkle`、`params`、`validation.h`。 |
| [src/crypto/](src/crypto/) | 加密原语 — SHA256（含 ARM/x86 优化）、ChaCha20、AES、RIPEMD-160、secp256k1（子仓库）。 |
| [src/script/](src/script/) | 脚本解释器、Miniscript 描述符引擎、签名提供者。 |
| [src/bench/](src/bench/) | 基准测试（基于 nanobench 框架）。 |
| [src/test/](src/test/) | 单元测试 (Boost.Test)、模糊测试和测试工具。 |
| [src/util/](src/util/) | 共享工具 — 字符串编码、时间、文件系统 (`fs` 命名空间)、BIP32、ASMap、日志、线程池。 |
| [src/interfaces/](src/interfaces/) | node/wallet/qt 之间的抽象层 — 进程分离的 "API 边界"。 |
| [src/common/](src/common/) | 节点和钱包共享的代码（参数、初始化、URL 处理）。 |
| [src/ipc/](src/ipc/) | Cap'n Proto IPC 多进程支持（bitcoin-node、bitcoin-gui 作为独立进程运行）。 |
| [src/indexes/](src/indexes/) | 可选索引 — txindex、blockfilter、coinstatsindex。 |
| [src/compat/](src/compat/) | 平台兼容性适配层（assumeutxo、字节交换、大小端等）。 |

### 主要可执行文件

| 二进制文件 | 源码 | 用途 |
|--------|--------|---------|
| `bitcoind` | [src/bitcoind.cpp](src/bitcoind.cpp) | 节点+钱包守护进程 |
| `bitcoin-cli` | [src/bitcoin-cli.cpp](src/bitcoin-cli.cpp) | RPC 客户端 |
| `bitcoin-qt` | [src/qt/](src/qt/) | Qt GUI 节点 |
| `bitcoin-tx` | [src/bitcoin-tx.cpp](src/bitcoin-tx.cpp) | 交易工具 |
| `bitcoin-wallet` | [src/bitcoin-wallet.cpp](src/bitcoin-wallet.cpp) | 钱包管理工具 |
| `bitcoin-util` | [src/bitcoin-util.cpp](src/bitcoin-util.cpp) | 实用工具 |
| `bench_bitcoin` | [src/bench/](src/bench/) | 基准测试运行器 |
| `test_bitcoin` | [src/test/](src/test/) | 单元测试运行器 |
| `libbitcoinkernel` | [src/kernel/](src/kernel/) | 共识库 |

### `test/` 下的数据

| 路径 | 用途 |
|------|---------|
| [test/functional/](test/functional/) | Python 集成测试（约 300+ 脚本，按功能命名） |
| [test/fuzz/](test/fuzz/) | 模糊测试框架 |
| [test/util/](test/util/) | 测试辅助脚本 |

### 代码风格要点

- **命名**：类/函数使用 `UpperCamelCase`，变量使用 `snake_case`，成员变量加 `m_` 前缀，全局变量加 `g_` 前缀，常量使用 `ALL_CAPS`
- **大括号**：类/函数换行，`if`/`for`/`while` 同行
- **缩进**：4 空格，不缩进 `public:`/`private:` 和 `namespace`
- **头文件包含**：第一个包含 `<bitcoin-build-config.h>`（IWYU pragma keep），然后是本项目的头文件
- **类型转换**：优先使用命名转换（`static_cast`、`int{x}`）而非 C 风格转换
- **文档风格**：公共 API 使用 Doxygen (`/** ... */`)
