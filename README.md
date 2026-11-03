# brpc 1.4.0 移植到 loongarch64

brpc 是 Doris BE 的 RPC 骨架。它只支持 x86_64 / aarch64，loongarch64 需要 **8 处补丁**。
补丁文件见本仓库 `brpc-1.4.0-loongarch64.patch`，另需新增 `atomicops_internals_loongarch64_gcc.h`（取自 brpc 上游 master）。

## 补丁清单

| 文件 | 问题 | 处理 |
|---|---|---|
| `src/butil/build_config.h` | 无 loongarch 分支，直接 `#error` | 加 `ARCH_CPU_LOONGARCH64_FAMILY` / `ARCH_CPU_64_BITS` / `ARCH_CPU_LITTLE_ENDIAN` |
| `src/butil/atomicops.h` | 无原子操作实现 | 引用 `atomicops_internals_loongarch64_gcc.h`（上游实现，ll.w/sc.w、ll.d/sc.d） |
| `src/butil/time.h` | `clock_cycles()` 无实现 | 用 `rdtime.d`（GCC 16 支持） |
| `src/bthread/processor.h` | `cpu_relax()` 用 x86 `pause` | loongarch 用 `nop`（与上游一致） |
| `src/bthread/context.h` / `context.cpp` | 无协程上下文切换汇编 | 从 brpc master 搬 `BTHREAD_CONTEXT_PLATFORM_linux_loongarch64` 汇编块（make/jump_fcontext） |
| `CMakeLists.txt` | 自带 `-D__STRICT_ANSI__` 与 `-std=gnu++11` 叠加，导致 libstdc++ 两个 `__int128` 特化同时生效 | 删除 `-D__STRICT_ANSI__` |
| `CMakeLists.txt` | GCC 16 不再隐式包含 `<cstdint>` | CXXFLAGS 加 `-include cstdint` |

## 关键坑：`-D__STRICT_ANSI__`

libstdc++ 的 `type_traits` 有两个块：`__GLIBCXX_TYPE_INT_N_0`（无条件）和 `__int128`（`__STRICT_ANSI__` 时才生效）。
brpc 用 `-std=gnu++11`（不定义 `__STRICT_ANSI__`）却**手动加了 `-D__STRICT_ANSI__`**，两个块同时生效 → 重复定义：

    error: ‘struct std::__is_integral_helper<__int128>’ 重定义

## 链接注意

- glog 会引 libunwind 符号 → 需要 `-lunwind`（或者按 Doris 那样用 `WITH_UNWIND=OFF` 的 glog）。
- brpc 还引用全局命名空间的 `snappy::GetUncompressedLength` → 需要 `-lsnappy`。
- openssl 在 `installed/lib`，brpc 等在 `installed/lib64`，两个 `-L` 都要给。

## 验证（运行时，不只是编译）

自写测试程序（`echo` 服务）：

    [1] bthread: counter=16000 期望=16000 -> OK      # 8 个协程各 2000 次 bthread_yield，验证上下文切换汇编
    [2] brpc RPC: OK (5 次同步调用全部正确)

产物：`libbrpc.a`(389MB) 与 `libbrpc.so`。
