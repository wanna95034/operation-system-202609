# FAT 模拟文件系统：程序命名与架构规范

> 项目中文名：FAT 模拟文件系统  
> 项目英文名：FatFsLab  
> 规范版本：v1.0  
> 适用范围：题目五“模拟磁盘文件系统实现”的源代码、测试、图形界面和文档  
> 设计原则：磁盘格式遵循指导书，软件分层借鉴 xv6-riscv，磁盘空间分配只采用 FAT。

## 1. 规范关键词

本文使用以下关键词：

- **必须（MUST）**：违反后会破坏题目要求、磁盘格式或模块边界；
- **应当（SHOULD）**：正常情况下必须遵循，确有理由时应在代码注释或设计文档中说明；
- **可以（MAY）**：可选增强，不得影响必做功能。

## 2. 项目标识与产物命名

| 对象 | 统一名称 |
|---|---|
| 项目目录 | `fatfs-lab` |
| CMake 项目名 | `FatFsLab` |
| C++ 根命名空间 | `fatfs` |
| 核心静态库 | `fatfs_core` |
| 图形程序 | `fatfs_lab` / Windows 下为 `fatfs_lab.exe` |
| 命令行测试程序 | `fatfs_cli` / Windows 下为 `fatfs_cli.exe` |
| 默认磁盘镜像 | `fatfs.img` |
| 默认日志文件 | `fatfs.log` |
| 单元测试程序 | `fatfs_tests` |
| 架构文档目录 | `docs/` |
| 运行数据目录 | `data/` |

不得使用 `test.exe`、`main2.cpp`、`new.cpp`、`final.cpp`、`temp.cpp` 等无法表达职责的名称。

## 3. 技术边界

第一版按以下技术方案组织：

- 核心语言：C++17；
- 构建系统：CMake；
- GUI：Qt Widgets（如果环境不具备 Qt，可替换 UI，但不得改动核心接口）；
- 测试：独立测试目标，可使用轻量测试框架或自定义断言；
- 持久化：一个固定 8192 字节的二进制磁盘镜像；
- 路径：第一版只支持绝对路径；
- 文件名：严格采用指导书规定的 3+2 格式和 ASCII 字符；
- 文件内容：文本内容，不允许出现作为文件结束标志的 `#` 字符。

核心模块中不得包含 Qt 类型。`QString`、`QByteArray`、`QTreeWidget` 等只允许出现在 `ui/` 或适配层中。核心层统一使用标准 C++ 类型。

## 4. 总体架构

### 4.1 分层结构

```text
┌─────────────────────────────────────────┐
│ UI Layer                                │
│ Qt 窗口、目录树、表格、对话框、提示信息 │
└───────────────────┬─────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Application Layer                       │
│ 命令解析、参数转换、用例编排、视图模型   │
└───────────────────┬─────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ File System Layer                       │
│ 文件、目录、路径、打开文件表             │
└───────────────────┬─────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ FAT Layer                               │
│ 块分配、块回收、FAT 链映射与校验         │
└───────────────────┬─────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│ Storage Layer                           │
│ 双缓冲、块读写、磁盘镜像                 │
└───────────────────┬─────────────────────┘
                    ↓
               fatfs.img
```

### 4.2 依赖规则

模块依赖必须单向向下：

```text
ui → app → fs → fat → storage → core
```

必须遵守：

- `storage` 不得引用 FAT、目录、文件和 GUI；
- `fat` 不得解析路径或操作窗口；
- `fs` 不得直接调用宿主机文件流，必须经过 `storage`；
- `app` 只编排用例，不直接修改 FAT 或目录项；
- `ui` 不得包含块分配、路径解析和文件读写算法；
- 测试可以直接引用被测模块，但不得为了测试破坏生产接口。

禁止形成循环依赖，例如 `FatTable` 回调 `FileSystem`，或 `VirtualDisk` 依赖 `DirectoryManager`。

## 5. 推荐目录结构

```text
fatfs-lab/
├─ CMakeLists.txt
├─ README.md
├─ docs/
│  ├─ architecture.md
│  ├─ disk-format.md
│  └─ test-plan.md
├─ src/
│  ├─ core/
│  │  ├─ constants.h
│  │  ├─ fs_error.h
│  │  ├─ result.h
│  │  └─ types.h
│  ├─ storage/
│  │  ├─ virtual_disk.h
│  │  ├─ virtual_disk.cpp
│  │  ├─ block_buffer_pool.h
│  │  └─ block_buffer_pool.cpp
│  ├─ fat/
│  │  ├─ fat_table.h
│  │  ├─ fat_table.cpp
│  │  ├─ fat_chain.h
│  │  └─ fat_chain.cpp
│  ├─ fs/
│  │  ├─ file_name.h
│  │  ├─ file_name.cpp
│  │  ├─ directory_entry.h
│  │  ├─ directory_entry.cpp
│  │  ├─ directory_manager.h
│  │  ├─ directory_manager.cpp
│  │  ├─ path_resolver.h
│  │  ├─ path_resolver.cpp
│  │  ├─ open_file_table.h
│  │  ├─ open_file_table.cpp
│  │  ├─ file_system.h
│  │  └─ file_system.cpp
│  ├─ app/
│  │  ├─ command.h
│  │  ├─ command_parser.h
│  │  ├─ command_parser.cpp
│  │  ├─ shell_session.h
│  │  ├─ shell_session.cpp
│  │  ├─ fs_controller.h
│  │  └─ fs_controller.cpp
│  ├─ ui/
│  │  ├─ main_window.h
│  │  ├─ main_window.cpp
│  │  ├─ main_window.ui
│  │  ├─ create_file_dialog.h
│  │  └─ create_file_dialog.cpp
│  ├─ cli_main.cpp
│  └─ gui_main.cpp
├─ tests/
│  ├─ unit/
│  │  ├─ virtual_disk_test.cpp
│  │  ├─ fat_table_test.cpp
│  │  ├─ directory_entry_test.cpp
│  │  ├─ path_resolver_test.cpp
│  │  └─ open_file_table_test.cpp
│  ├─ integration/
│  │  ├─ file_lifecycle_test.cpp
│  │  ├─ directory_lifecycle_test.cpp
│  │  └─ persistence_test.cpp
│  └─ test_main.cpp
├─ data/
│  └─ .gitkeep
└─ assets/
   └─ icons/
```

## 6. C++ 命名规范

### 6.1 文件和目录

- 源文件和头文件必须使用 `snake_case`：`open_file_table.cpp`；
- 一个主要类对应一组同名 `.h/.cpp` 文件；
- 测试文件使用 `<被测对象>_test.cpp`；
- 目录名使用小写单词：`storage/`、`fat/`、`fs/`；
- 磁盘格式结构不得与 GUI 文件放在同一目录。

### 6.2 类型

- 类、结构体、枚举、类型别名使用 `PascalCase`；
- 接口类不使用 `I` 前缀，名称直接表达职责；
- 磁盘持久化结构以 `Disk` 结尾；
- 纯内存对象不得使用 `Disk` 后缀。

示例：

```cpp
class VirtualDisk;
class FatTable;
class DirectoryManager;
class PathResolver;

struct DirectoryEntryDisk;
struct OpenFile;
struct ResolvedPath;

enum class FsError;
enum class EntryType;

using BlockNo = std::uint8_t;
using ByteOffset = std::uint32_t;
```

### 6.3 函数和方法

- 函数和方法使用 `lowerCamelCase`；
- 名称以动词开头；
- 查询函数使用 `find`、`get`、`is`、`has`、`can`；
- 修改函数使用 `create`、`write`、`update`、`remove`、`allocate`、`release`；
- 不允许使用 `doIt()`、`process()`、`handle()` 等含义不完整的公共接口。

示例：

```cpp
Result<void> createImage(const std::filesystem::path& path);
Result<BlockNo> allocateBlock();
Result<ResolvedPath> resolvePath(std::string_view path);
bool isDirectory(const DirectoryEntry& entry);
```

### 6.4 变量

- 局部变量和参数使用 `lowerCamelCase`；
- 私有成员使用尾下划线：`imagePath_`、`openFiles_`；
- 布尔值必须使用 `is`、`has`、`can`、`should` 开头；
- 块号统一使用 `blockNo`，逻辑块号使用 `logicalBlock`；
- 字节偏移统一使用 `byteOffset`，块内偏移使用 `offsetInBlock`；
- 数量使用 `Count` 后缀，索引使用 `Index` 后缀；
- 禁止使用含义不明的 `a`、`b`、`n`、`tmp`、`data2`，短循环变量 `i` 除外。

### 6.5 常量和枚举

- 编译期常量使用 `kPascalCase`；
- 宏只用于条件编译，不用于普通常量；
- 枚举必须使用 `enum class`；
- 枚举值使用 `PascalCase`。

```cpp
constexpr std::size_t kBlockSize = 64;
constexpr std::size_t kBlockCount = 128;

enum class OpenMode {
    Read,
    Write,
};
```

### 6.6 命名空间

统一使用根命名空间 `fatfs`，子模块可以使用：

```cpp
namespace fatfs::storage { }
namespace fatfs::fat { }
namespace fatfs::fs { }
namespace fatfs::app { }
namespace fatfs::ui { }
```

头文件不得写 `using namespace`。源文件也应避免全局 `using namespace std;`。

## 7. 固定常量与类型

所有磁盘格式常量必须集中在 `src/core/constants.h`，不得在代码中散落魔法数字。

```cpp
namespace fatfs {

using BlockNo = std::uint8_t;
using FatValue = std::uint8_t;
using EntryIndex = std::uint8_t;
using ByteOffset = std::uint32_t;

constexpr std::size_t kBlockSize = 64;
constexpr std::size_t kBlockCount = 128;
constexpr std::size_t kDiskSize = kBlockSize * kBlockCount;

constexpr BlockNo kFatFirstBlock = 0;
constexpr std::size_t kFatBlockCount = 2;
constexpr BlockNo kRootDirectoryBlock = 2;
constexpr BlockNo kFirstDataBlock = 3;

constexpr std::size_t kDirectoryEntrySize = 8;
constexpr std::size_t kEntriesPerDirectory = 8;
constexpr std::size_t kMaxOpenFiles = 5;

constexpr std::chrono::minutes kDefaultShellIdleTimeout{10};
constexpr std::chrono::milliseconds kShellTimerPollInterval{200};

constexpr FatValue kFatFree = 0;
constexpr FatValue kFatBad = 254;
constexpr FatValue kFatEnd = 255;

constexpr char kEmptyEntryMarker = '$';
constexpr char kFileEndMarker = '#';

}  // namespace fatfs
```

## 8. 磁盘数据结构命名与编码

### 8.1 目录项的逻辑结构

```cpp
struct DirectoryEntry {
    std::string baseName;
    std::string extension;
    std::uint8_t attributes;
    BlockNo startBlock;
    std::uint8_t blockCount;
};
```

`DirectoryEntry` 是内存对象，可以包含 `std::string`，但不得直接写入磁盘。

### 8.2 磁盘目录项

磁盘目录项固定为 8 字节：

```cpp
#pragma pack(push, 1)
struct DirectoryEntryDisk {
    char baseName[3];
    char extension[2];
    std::uint8_t attributes;
    BlockNo startBlock;
    std::uint8_t blockCount;
};
#pragma pack(pop)

static_assert(sizeof(DirectoryEntryDisk) == 8);
```

但是，正式写盘时应当优先使用显式编解码函数，避免依赖编译器结构体布局：

```cpp
std::array<std::byte, 8> encodeDirectoryEntry(const DirectoryEntry& entry);
Result<DirectoryEntry> decodeDirectoryEntry(
    const std::array<std::byte, 8>& bytes);
```

不得把以下内容写入磁盘：

- 指针；
- `std::string`、`std::vector` 等对象的内存表示；
- `bool` 的原始表示；
- Qt 对象；
- 依赖平台对齐的未打包结构体。

### 8.3 属性命名

不用 C++ 位域保存属性，统一使用掩码：

```cpp
constexpr std::uint8_t kAttrReadOnly = 1U << 0;
constexpr std::uint8_t kAttrSystem = 1U << 1;
constexpr std::uint8_t kAttrRegular = 1U << 2;
constexpr std::uint8_t kAttrDirectory = 1U << 3;
```

辅助函数：

```cpp
bool hasAttribute(std::uint8_t attributes, std::uint8_t mask);
bool isValidAttribute(std::uint8_t attributes);
```

## 9. 核心模块职责与公共接口

### 9.1 `VirtualDisk`

唯一允许直接操作 `fatfs.img` 的类。

```cpp
class VirtualDisk {
public:
    Result<void> create(const std::filesystem::path& imagePath);
    Result<void> open(const std::filesystem::path& imagePath);
    Result<void> close();

    Result<void> readBlock(BlockNo blockNo, BlockBuffer& output);
    Result<void> writeBlock(BlockNo blockNo, const BlockBuffer& input);

    bool isOpen() const noexcept;
};
```

规则：

- 创建后的镜像必须恰好为 8192 字节；
- `readBlock()`、`writeBlock()` 每次只能访问一个 64 字节块；
- 块号越界必须返回错误；
- 不得公开底层 `fstream`；
- 析构时应安全关闭文件，使用 RAII 管理资源。

### 9.2 `BlockBufferPool`

封装指导书要求的两个缓冲区。

```cpp
using BlockBuffer = std::array<std::byte, kBlockSize>;

class BlockBufferPool {
public:
    Result<BufferHandle> acquire(BlockNo blockNo);
    Result<void> flush(BufferHandle& handle);
    void release(BufferHandle& handle);
};
```

规则：

- 内部只能有两个缓冲槽；
- 一个缓冲槽同一时间只能由一个调用者占用；
- 修改后必须标记为 dirty；
- `release()` 之后不得继续使用该句柄；
- 第一版不实现 LRU，只采用“空闲槽优先，必要时写回脏块”。

### 9.3 `FatTable`

负责 FAT 的加载、分配、链接、映射、回收和校验。

```cpp
class FatTable {
public:
    Result<void> load();
    Result<void> flush();
    Result<void> format();

    Result<BlockNo> allocateBlock();
    Result<void> releaseBlock(BlockNo blockNo);
    Result<void> releaseChain(BlockNo startBlock);

    Result<BlockNo> getNext(BlockNo blockNo) const;
    Result<void> setNext(BlockNo blockNo, FatValue next);
    Result<BlockNo> mapLogicalBlock(
        BlockNo startBlock,
        std::size_t logicalBlock,
        bool shouldAllocate);

    Result<std::vector<BlockNo>> collectChain(BlockNo startBlock) const;
    Result<void> validate() const;
};
```

规则：

- 分配范围只能是 `3..127`；
- `0..2` 永远不得作为普通数据块返回；
- 新分配块必须清零并标记为 `kFatEnd`；
- 遍历链最多访问 128 项，超过即返回 `FatLoopDetected`；
- 遇到 `kFatFree`、`kFatBad`、系统块或越界值必须报告损坏；
- 不得在多个模块中重复实现 FAT 遍历算法。

### 9.4 `DirectoryManager`

负责一个目录块内的 8 个目录项。

```cpp
class DirectoryManager {
public:
    Result<std::vector<DirectoryEntry>> list(BlockNo directoryBlock);
    Result<EntryLocation> find(
        BlockNo directoryBlock,
        const FileName& name);
    Result<EntryLocation> insert(
        BlockNo directoryBlock,
        const DirectoryEntry& entry);
    Result<void> update(
        const EntryLocation& location,
        const DirectoryEntry& entry);
    Result<void> remove(const EntryLocation& location);
    Result<bool> isEmpty(BlockNo directoryBlock);
    Result<void> initializeEmpty(BlockNo directoryBlock);
};
```

规则：

- 空目录项以第一个字节 `$` 判断；
- 一次目录修改必须以完整 64 字节块为单位写回；
- 名称比较在进入该模块前必须已经规范化；
- 普通文件项和目录项使用同一 8 字节格式；
- 根目录和子目录使用同一套读写算法。

### 9.5 `PathResolver`

只负责绝对路径拆分和逐级查找，不修改磁盘。

```cpp
struct ResolvedPath {
    BlockNo parentDirectoryBlock;
    EntryLocation location;
    DirectoryEntry entry;
};

struct ResolvedParent {
    BlockNo parentDirectoryBlock;
    FileName childName;
};

class PathResolver {
public:
    Result<ResolvedPath> resolve(std::string_view absolutePath);
    Result<ResolvedParent> resolveParent(std::string_view absolutePath);
};
```

规则：

- 第一版只接受以 `/` 开头的绝对路径；
- 连续 `/` 可以规范化为一个；
- 根路径 `/` 必须作为特殊情况处理；
- 中间路径项必须是目录；
- 路径解析不得隐式创建目录；
- `resolveParent()` 用于创建文件和目录，必须返回最后一级名称。

### 9.6 `OpenFileTable`

```cpp
struct FileCursor {
    std::uint32_t byteOffset;
};

struct OpenFile {
    bool isUsed;
    std::string absolutePath;
    DirectoryEntry entry;
    EntryLocation entryLocation;
    OpenMode mode;
    FileCursor readCursor;
    FileCursor writeCursor;
    std::uint32_t byteLength;
};

class OpenFileTable {
public:
    Result<std::size_t> insert(OpenFile file);
    Result<OpenFile&> get(std::size_t fileHandle);
    Result<OpenFile&> findByPath(std::string_view absolutePath);
    Result<void> remove(std::size_t fileHandle);
    bool contains(std::string_view absolutePath) const;
};
```

规则：

- 最大表项数固定为 5；
- 同一绝对路径在表中只能出现一次；
- 文件句柄是表下标，不得等同于磁盘块号；
- 读写位置在内部保存为逻辑字节偏移；
- GUI 展示时可以将逻辑偏移转换为物理块号和块内偏移；
- 删除文件、显示整个文件、修改属性前必须查询该表。

### 9.7 `FileSystem`

这是核心层唯一推荐暴露给应用层的门面对象。

```cpp
class FileSystem {
public:
    Result<void> format(const std::filesystem::path& imagePath);
    Result<void> mount(const std::filesystem::path& imagePath);
    Result<void> unmount();

    Result<void> createFile(
        std::string_view path,
        std::uint8_t attributes);
    Result<std::size_t> openFile(
        std::string_view path,
        OpenMode mode);
    Result<std::string> readFile(
        std::size_t fileHandle,
        std::size_t length);
    Result<std::size_t> writeFile(
        std::size_t fileHandle,
        std::string_view content);
    Result<void> closeFile(std::size_t fileHandle);
    Result<void> deleteFile(std::string_view path);
    Result<std::string> typeFile(std::string_view path);
    Result<void> changeAttributes(
        std::string_view path,
        std::uint8_t attributes);

    Result<void> makeDirectory(std::string_view path);
    Result<std::vector<DirectoryEntry>> listDirectory(
        std::string_view path);
    Result<void> removeDirectory(std::string_view path);
};
```

应用层和 GUI 不得越过 `FileSystem` 直接修改 FAT 或目录块。

## 10. 外部命令与内部方法的对应命名

为了兼容指导书，同时保持 C++ 命名统一，采用以下映射：

| 用户命令 | 内部方法 | 命令枚举 |
|---|---|---|
| `create_file` | `createFile()` | `CommandType::CreateFile` |
| `open_file` | `openFile()` | `CommandType::OpenFile` |
| `read_file` | `readFile()` | `CommandType::ReadFile` |
| `write_file` | `writeFile()` | `CommandType::WriteFile` |
| `close_file` | `closeFile()` | `CommandType::CloseFile` |
| `delete_file` | `deleteFile()` | `CommandType::DeleteFile` |
| `typefile` | `typeFile()` | `CommandType::TypeFile` |
| `change` | `changeAttributes()` | `CommandType::ChangeAttributes` |
| `md` | `makeDirectory()` | `CommandType::MakeDirectory` |
| `dir` | `listDirectory()` | `CommandType::ListDirectory` |
| `rd` | `removeDirectory()` | `CommandType::RemoveDirectory` |

命令字符串只允许在 `CommandParser` 中解析，核心层不得依赖命令文本。

### 10.1 Shell 会话与空闲超时重置

命令行程序 `fatfs_cli` 必须提供空闲超时机制，防止会话长期占用打开文件和未刷新的缓冲区。

默认配置：

| 配置项 | 默认值 | 说明 |
|---|---:|---|
| 空闲超时 | 10 分钟 | 从最后一次用户输入完成时开始计时 |
| 检查周期 | 200 毫秒 | 使用单调时钟检查，不依赖系统时间变化 |
| 超时动作 | `reset` | 安全重置当前 Shell 会话 |
| 允许范围 | 60～3600 秒 | 超出范围的配置必须拒绝 |

启动参数：

```text
fatfs_cli --idle-timeout 600 --idle-action reset
fatfs_cli --idle-timeout 300 --idle-action exit
```

Shell 内部命令：

| 命令 | 作用 |
|---|---|
| `timeout status` | 显示当前超时秒数、动作和剩余时间 |
| `timeout set <seconds>` | 修改当前会话的空闲超时时间 |
| `timeout action reset` | 超时后重置会话并继续显示提示符 |
| `timeout action exit` | 超时后安全退出命令行程序 |
| `timeout reset` | 手动重置空闲计时器 |
| `exit` | 立即执行安全清理并退出 |

会话对象：

```cpp
enum class ShellIdleAction {
    Reset,
    Exit,
};

enum class ShellState {
    Ready,
    RunningCommand,
    Resetting,
    Exiting,
    Faulted,
};

class ShellSession {
public:
    Result<void> run();
    void recordUserActivity();
    Result<void> setIdleTimeout(std::chrono::seconds timeout);
    void setIdleAction(ShellIdleAction action);
    Result<void> resetSession();
    Result<void> shutdown();
};
```

空闲时间必须使用 `std::chrono::steady_clock` 计算，禁止使用系统日期时间，避免用户修改时间或时钟校准造成误触发。

以下事件必须重置计时器：

- 用户提交任意非空输入，包括非法命令；
- 用户执行 `timeout reset`；
- 一条命令执行完成并重新显示提示符。

以下状态不得触发空闲超时：

- 文件系统命令仍在执行；
- Shell 正在执行安全重置；
- Shell 正在写回缓冲区或卸载磁盘镜像。

超时触发后的安全重置顺序：

```text
ShellState = Resetting
    ↓
停止接受新命令
    ↓
按打开顺序关闭全部已打开文件
    ↓
写方式文件补写结束符 # 并更新目录项
    ↓
刷新脏块、FAT 和磁盘文件
    ↓
清空打开文件表
    ↓
当前目录恢复为 /
    ↓
清空未提交的命令输入和会话级临时状态
    ↓
reset 模式：ShellState = Ready，重新显示提示符
exit 模式：卸载镜像并结束进程
```

必须遵守：

- “重置会话”绝不等于格式化磁盘，不得删除或重新初始化 `fatfs.img`；
- 超时不得中断正在执行的文件系统操作；
- 重置前必须尝试安全关闭所有打开文件；
- 任一写回步骤失败时，不得静默清空状态，应进入 `Faulted` 状态并输出错误；
- `Faulted` 状态下拒绝新的修改命令，只允许重试刷新、卸载或退出；
- GUI 会话不受 CLI 空闲超时影响，除非后续单独提出 GUI 自动锁定需求；
- 命令输入与计时检测必须解耦，不能因为阻塞式 `std::getline()` 导致计时器无法运行；
- Windows 实现可以封装可轮询的控制台输入，平台相关代码必须放在 `app` 的输入适配器中，不得进入文件系统核心层。

## 11. 错误处理规范

预期内的用户错误不得通过异常表达，统一返回 `Result<T>` 和 `FsError`。

```cpp
enum class FsError {
    None,
    DiskNotOpen,
    DiskImageInvalid,
    BlockOutOfRange,
    DiskReadFailed,
    DiskWriteFailed,
    DiskFull,
    FatCorrupted,
    FatLoopDetected,
    InvalidPath,
    InvalidFileName,
    InvalidAttributes,
    ParentNotFound,
    EntryNotFound,
    EntryAlreadyExists,
    NotAFile,
    NotADirectory,
    DirectoryFull,
    DirectoryNotEmpty,
    RootDirectoryProtected,
    OpenFileTableFull,
    FileAlreadyOpen,
    FileNotOpen,
    FileBusy,
    PermissionDenied,
    InvalidOpenMode,
    EndOfFile,
};
```

规范：

- 错误枚举必须表达原因，不能只返回 `false`；
- 错误信息的中文翻译放在应用层，不写入底层算法；
- 底层错误必须向上传递，不得悄悄忽略；
- 编程错误可使用 `assert`，用户输入错误不得触发断言；
- 任何失败操作都不得留下半写入的目录项或孤立 FAT 链。

## 12. 文件名与路径规范

### 12.1 文件名

- 基本名最多 3 字符；
- 扩展名最多 2 字符；
- 文件格式为 `abc.tx`；
- 目录没有扩展名；
- 名称只允许 ASCII 字母和数字；
- `$`、`.`、`/` 不得作为名称字符；
- `.` 只允许作为基本名和扩展名的分隔符；
- 名称比较默认区分大小写；若后续决定不区分，必须在 `FileName` 中统一规范化。

### 12.2 路径

- 根目录表示为 `/`；
- 文件路径示例：`/usr/bin/a.tx`；
- 目录路径示例：`/usr/bin`；
- 第一版不支持相对路径、`.` 和 `..`；
- 路径标准化由 `PathResolver` 统一完成；
- 其他模块不得自行使用字符串切割重复实现路径解析。

## 13. FAT 与文件内容规则

### 13.1 FAT 不变量

在任意成功操作后，必须满足：

1. FAT 恰好有 128 项；
2. 第 0～2 块不会被普通文件或子目录分配；
3. `kFatFree` 只表示空闲块；
4. `kFatEnd` 只表示链尾或系统保留块；
5. `kFatBad` 表示坏块，不得被分配；
6. FAT 链中的普通值必须属于 `3..127`；
7. FAT 链不得成环；
8. 同一数据块不得同时属于两个文件或目录；
9. 目录项的 `blockCount` 必须与实际 FAT 链长度一致。

### 13.2 文件结束符

- 文件数据以 `#` 作为结束标志；
- 用户写入内容不得包含 `#`；
- 打开文件时扫描 FAT 链找到 `#`，计算实际字节长度；
- 写指针定位在 `#` 之前，即新数据覆盖旧 `#` 后重新追加 `#`；
- 如果文件数据恰好填满一个块，必须分配新块存储 `#`；
- 目录项的 `blockCount` 包含用于保存结束符的块。

### 13.3 块映射

除 `FatTable::mapLogicalBlock()` 外，任何模块不得手工跳转 FAT 链。

```text
logicalBlock = byteOffset / kBlockSize
offsetInBlock = byteOffset % kBlockSize
physicalBlock = mapLogicalBlock(startBlock, logicalBlock, shouldAllocate)
```

## 14. 修改操作的一致性顺序

### 14.1 创建文件或目录

```text
验证路径、名称、重名、目录容量和磁盘空间
    ↓
分配并初始化数据块/目录块
    ↓
更新并写回 FAT
    ↓
最后写入父目录项
    ↓
必要时加入打开文件表
```

失败时必须释放本次新分配块并恢复 FAT。

### 14.2 追加写文件

```text
验证打开模式与属性
    ↓
保存原 FAT 链和目录项快照
    ↓
覆盖旧结束符并写入新数据
    ↓
必要时分配并连接新块
    ↓
写入新的结束符
    ↓
写回 FAT
    ↓
最后更新目录项 blockCount
```

磁盘满或写入失败时必须回滚本次新增 FAT 链。

### 14.3 删除文件

```text
验证文件存在且未打开
    ↓
完整收集并校验 FAT 链
    ↓
保存目录块和 FAT 快照
    ↓
清除父目录项
    ↓
释放 FAT 链
```

不得边遍历边释放尚未完成校验的链。

### 14.4 关闭写文件

- 确保末尾存在且只存在一个有效 `#`；
- 刷新脏数据块；
- 刷新 FAT；
- 更新目录项；
- 最后删除打开文件表项。

## 15. UI 与核心层交互规范

GUI 只能通过 `FsController` 调用文件系统：

```cpp
class FsController {
public:
    void createFile(const CreateFileRequest& request);
    void openFile(const OpenFileRequest& request);
    void writeFile(const WriteFileRequest& request);
    void refreshViews();
};
```

规则：

- 槽函数中不得编写 FAT 遍历代码；
- GUI 不持有 `VirtualDisk`、`FatTable` 的可修改引用；
- 所有操作完成后统一调用 `refreshViews()`；
- FAT 表、目录表、打开文件表使用只读快照展示；
- 中文错误提示由 `FsError` 映射生成；
- 弹窗负责收集参数，不负责验证磁盘状态。

建议界面对象命名：

| 控件 | 对象名 |
|---|---|
| 主窗口 | `mainWindow` |
| 目录树 | `directoryTree` |
| 当前目录表 | `directoryTable` |
| FAT 表 | `fatTableView` |
| 打开文件表 | `openFileTableView` |
| 磁盘块视图 | `diskBlockView` |
| 命令输入框 | `commandInput` |
| 日志输出框 | `operationLog` |
| 创建文件按钮 | `createFileButton` |
| 写文件按钮 | `writeFileButton` |

## 16. 日志规范

运行日志采用统一格式：

```text
[时间] [级别] [模块] 操作和结果
```

示例：

```text
[17:30:42] [INFO] [FileSystem] created /usr/a.tx at block 7
[17:31:08] [ERROR] [FatTable] allocation failed: disk full
```

级别：

- `DEBUG`：块号、FAT 链、偏移等调试信息；
- `INFO`：用户操作成功；
- `WARN`：可恢复的异常状态；
- `ERROR`：操作失败或镜像损坏。

核心层不得直接调用 `QMessageBox`。日志通过回调或日志接口交给应用层。

## 17. 测试命名与组织规范

测试名称采用：

```text
<对象>_<条件>_<预期结果>
```

示例：

```cpp
TEST(FatTable, allocateBlock_whenDiskHasSpace_returnsFirstFreeBlock);
TEST(FatTable, collectChain_whenChainLoops_returnsFatLoopDetected);
TEST(PathResolver, resolve_whenParentMissing_returnsParentNotFound);
TEST(FileSystem, writeFile_whenCrossingBlock_allocatesNextBlock);
TEST(ShellSession, idleTimeout_whenReady_resetsSession);
TEST(ShellSession, idleTimeout_whenCommandRunning_doesNotInterruptCommand);
TEST(ShellSession, resetSession_whenWriteFileOpen_flushesAndClosesFile);
```

测试必须覆盖：

- 0、1、63、64、65 字节写入；
- FAT 链尾、坏块、越界和环；
- 根目录、空目录、满目录和多级目录；
- 打开文件表第 5 项和第 6 项；
- 只读文件写入；
- 打开文件删除、显示和修改属性；
- 磁盘满时的回滚；
- 程序重启后的持久化；
- 文件数据正好占满块时的结束符处理。
- 空闲超时后打开文件表被清空、当前目录恢复为 `/`；
- `exit` 超时动作安全卸载镜像并返回成功状态；
- 命令执行时间超过空闲阈值时不会被超时机制中断；
- 超时重置写回失败时进入 `Faulted`，且不会继续接受修改命令。

每个测试必须创建独立临时镜像，不得复用 `data/fatfs.img`，测试结束后删除临时文件。

## 18. C++ 编码风格

- 缩进使用 4 个空格，不使用 Tab；
- 建议每行不超过 100～120 字符；
- 头文件使用 `#pragma once`；
- 一个函数只承担一个清晰职责；
- 公共方法必须说明前置条件、返回值和错误；
- 优先使用 `std::array`、`std::vector`、`std::string` 和 RAII；
- 禁止裸 `new`/`delete`；
- 能声明为 `const` 的方法和参数必须声明为 `const`；
- 能声明为 `noexcept` 的简单查询方法应声明为 `noexcept`；
- 避免全局可变对象，`FileSystem` 及其依赖由 `main` 创建并注入；
- 不得使用宿主机目录和文件 API 代替模拟文件系统操作；
- 不得把完整 8192 字节磁盘镜像长期加载到一个数组中绕过块 I/O；
- 所有编译目标应开启常用警告，并把新增警告视为缺陷。

## 19. 注释规范

注释解释“为什么”和不变量，不逐字翻译代码。

推荐：

```cpp
// Commit the directory entry last so an interrupted create operation
// cannot expose a file whose FAT chain is incomplete.
```

不推荐：

```cpp
// i 加 1
i++;
```

磁盘格式、回滚逻辑和 FAT 特殊值必须写注释。临时待办使用：

```cpp
// TODO(username): explain the remaining work and expected behavior.
```

提交前不得保留无说明的 `TODO`、注释掉的大段旧代码和调试输出。

## 20. Git 分支和提交命名建议

分支：

```text
feature/virtual-disk
feature/fat-allocation
feature/path-resolver
feature/file-operations
feature/gui
test/persistence
fix/fat-chain-loop
```

提交信息：

```text
feat(storage): implement fixed-size virtual disk
feat(fat): add FAT chain allocation and release
feat(fs): resolve absolute paths from root directory
test(fs): cover cross-block append writes
fix(fat): roll back newly allocated blocks on disk full
docs: add disk layout and architecture specification
```

一次提交应只解决一个主题。禁止使用“修改”“更新代码”“final”等无法表达内容的提交信息。

## 21. 首批实现顺序

后续编码必须按以下顺序推进：

1. `constants.h`、`types.h`、`FsError`、`Result<T>`；
2. `VirtualDisk` 及块边界测试；
3. `BlockBufferPool` 及双缓冲测试；
4. `FatTable` 的格式化、分配、映射和回收；
5. `DirectoryEntry` 编解码；
6. `DirectoryManager`；
7. `FileName` 与 `PathResolver`；
8. `OpenFileTable`；
9. `FileSystem` 的目录操作；
10. `FileSystem` 的文件生命周期和读写；
11. CLI 自动测试入口；
12. `ShellSession` 空闲计时、安全重置与退出；
13. Qt GUI 和状态可视化；
14. 一致性检查与完整集成测试。

在 `VirtualDisk`、FAT 和目录项编解码测试通过前，不开始 GUI 开发。

## 22. 架构验收清单

开始正式编码或合并模块前，逐项确认：

- [ ] 项目、可执行程序、镜像和命名空间名称统一；
- [ ] 模块依赖严格单向向下；
- [ ] 只有 `VirtualDisk` 操作宿主机磁盘文件；
- [ ] 所有磁盘访问以 64 字节块为单位；
- [ ] 双缓冲区封装在 `BlockBufferPool`；
- [ ] FAT 是唯一的数据块分配机制；
- [ ] 第 0～2 块不会分配给普通数据；
- [ ] 目录项编码固定为 8 字节；
- [ ] 路径解析只有一个实现；
- [ ] FAT 链遍历只有一个实现；
- [ ] GUI 不包含文件系统算法；
- [ ] 错误通过 `Result<T>` 向上传递；
- [ ] 创建、写入、删除具有失败回滚；
- [ ] 打开文件表最多 5 项且路径唯一；
- [ ] Shell 使用单调时钟检测空闲时间；
- [ ] 超时不会中断正在执行的命令；
- [ ] 超时重置会关闭文件、刷新数据并恢复根目录；
- [ ] 超时重置不会格式化或删除磁盘镜像；
- [ ] 单元测试不污染正式磁盘镜像；
- [ ] 代码中没有魔法数字、裸指针持久化和整盘常驻数组。

## 参考架构来源

- [xv6-riscv `fs.h`：磁盘布局、inode 与目录项](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.h)
- [xv6-riscv `fs.c`：块、文件、目录与路径分层](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/fs.c)
- [xv6-riscv `bio.c`：块缓冲接口](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/bio.c)
- [xv6-riscv `file.h`：内存 inode 与打开文件对象](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/file.h)
- [xv6-riscv `sysfile.c`：文件系统调用与创建流程](https://github.com/mit-pdos/xv6-riscv/blob/riscv/kernel/sysfile.c)
