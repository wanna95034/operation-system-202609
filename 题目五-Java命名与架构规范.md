# 题目五：Java 程序命名与架构规范

> 更新：2026-09-23。适用于 `operation-system-202609` 的 Java 源码、测试、命令行和 GUI。课程原文见 [指导书要求提取](./题目五-指导书要求提取.md)，具体实施顺序见 [Java 开发方案与阶段任务](./题目五-Java开发方案与阶段任务.md)。
> 状态：编码前规范；文中类、方法和路径均为拟建立的项目结构，当前尚未实现。

## 1. 命名基线

| 对象 | 规定名称 / 规则 | 示例 |
| --- | --- | --- |
| Git 仓库 | `operation-system-202609` | 与当前 GitHub 仓库名称保持一致 |
| Maven 坐标 | `edu.oslab:fatfs-lab` | `groupId:artifactId` |
| Java 根包 | `edu.oslab.fatfs` | `edu.oslab.fatfs.fat` |
| Java 文件 | 与唯一公开顶层类型同名 | `FatTable.java` |
| 包 | 全小写、单词清晰 | `openfile`、`directory` |
| 类、接口、record、枚举 | `PascalCase`，名词表达职责 | `BlockDevice`、`OpenMode` |
| 方法、字段、局部变量 | `camelCase`，方法用动词起头 | `readBlock`、`nextBlock` |
| 常量 | `UPPER_SNAKE_CASE` | `BLOCK_SIZE`、`FAT_END` |
| 单元测试 | `<被测类型>Test` | `FatTableTest` |
| 集成测试 | `<场景>IT` | `RestartPersistenceIT` |
| 测试方法 | `行为_when条件_预期结果` 的英文驼峰组合 | `append_whenCrossingBlock_updatesFatChain` |
| 默认镜像 | `data/fatfs.img` | 用户可通过启动参数指定其他路径 |
| 可运行包 | `fatfs-lab-1.0.0.jar` | 与 Maven 版本一致 |

名称中优先使用完整领域词：`blockNumber`、`directoryEntry`、`openFileTable`；局部极短循环变量除外，避免 `blk`、`ent`、`tmpFs` 等含义不清的缩写。`FAT` 是公认术语，类型名使用 `FatTable`、`FatChain`，常量仍写 `FAT_END`。同一个概念只用一个名字：盘块编号统一 `blockNumber`，目录项索引统一 `slotIndex`，文件数据长度统一 `byteLength`。

## 2. 工程目录和依赖方向

```text
src/main/java/edu/oslab/fatfs/
├─ bootstrap/   FatFsApplication、启动参数和对象组装
├─ model/       DiskLayout、DirectoryEntry、FileCursor、OpenMode、FileAttributes、快照
├─ error/       FsErrorCode、FsException
├─ storage/     BlockDevice、FileChannelDisk
├─ buffer/      BlockBufferPool、BufferSlot
├─ fat/         FatTable、FatChain
├─ directory/   DirectoryEntryCodec、DirectoryManager
├─ path/        FileName、VirtualPath、PathResolver
├─ openfile/    OpenFileTable、OpenFileEntry
├─ fs/          FileSystemService、DefaultFileSystemService、ConsistencyChecker
├─ app/         Command、CommandDispatcher、FsController
├─ cli/         ShellSession、TerminalInput、IdlePolicy
└─ ui/          MainFrame、树/表模型与操作对话框

src/test/java/edu/oslab/fatfs/  与主代码包结构对应
docs/                          磁盘格式、环境、演示与测试记录
scripts/                       run-gui.cmd、run-cli.cmd
data/                          本地模拟磁盘镜像
logs/                          本地运行日志
```

依赖方向固定为：`ui/cli → app → fs → path/directory/openfile/fat → buffer → storage`。`model` 和 `error` 为共享基础类型。低层包不得导入 `javax.swing`，不得依赖 `cli` 或 `ui`。`FileSystemService` 接收领域类型和字节数组；CLI 解析字符串，GUI 收集表单输入，两者共用命令分发和错误映射。

只由 `FileChannelDisk` 访问宿主机镜像。`VirtualPath` 表示模拟盘中的 `/usr/a.tx`，不传给 `java.nio.file.Files` 作实际文件创建。`Path` 只用于宿主机上的镜像、日志、配置和脚本。

## 3. 核心类型的职责边界

| 包 / 类型 | 对外职责 | 不应承担的内容 |
| --- | --- | --- |
| `model.DiskLayout` | 定义块数、块大小、FAT 和根目录位置 | 执行磁盘 I/O |
| `storage.BlockDevice` | 按编号读写一个完整块、刷新和关闭 | 解释目录项、分配 FAT |
| `storage.FileChannelDisk` | 实现块设备、镜像尺寸检查、独占锁 | 解析虚拟路径 |
| `buffer.BlockBufferPool` | 恰好两个 64 字节槽；命中、占用、替换、脏写回 | 处理文件权限 |
| `fat.FatTable` / `FatChain` | FAT 元数据、链校验、分配、连接和回收 | 生成 GUI 状态表 |
| `directory.DirectoryEntryCodec` | 明确的 8 字节编码/解码 | Java 对象序列化 |
| `directory.DirectoryManager` | 单块目录槽位读改写、判空 | 自动创建父目录 |
| `path.VirtualPath` / `PathResolver` | 绝对路径校验、规范化、逐层检索 | 修改磁盘 |
| `openfile.OpenFileTable` | 固定 5 项、模式与指针状态 | 管理宿主机文件句柄 |
| `fs.FileSystemService` / `DefaultFileSystemService` | 定义并实现 11 项操作、预检、提交顺序 | 访问 Swing 控件 |
| `fs.ConsistencyChecker` | 诊断 FAT、目录和引用一致性 | 自动格式化或擅自修盘 |
| `app.CommandDispatcher` | 将课程命令映射为核心操作 | 再写一套 FAT 算法 |
| `app.FsController` | 串行提交、生成只读状态快照 | 用界面状态代替磁盘状态 |
| `cli.ShellSession` | 输入循环、状态、空闲策略、重置/退出 | 从计时线程直接改写 FAT |
| `ui.MainFrame` | 用户输入和真实状态展示 | 决定块分配算法 |

`FatFsApplication` 通过构造器组装依赖。一个镜像对应一套服务和缓冲池；不设置全局静态可变 FAT、打开表或磁盘单例。资源类型实现 `AutoCloseable`，明确关闭顺序：停止新命令、完成或停止当前命令、关闭打开文件、写回缓冲/FAT、刷新镜像、释放锁和终端资源。

## 4. 接口与参数约定

以下签名是方向性约定；编码时如调整类型，应同步更新开发方案、测试和文档。

```java
interface BlockDevice extends AutoCloseable {
    void readBlock(int blockNumber, byte[] destination);
    void writeBlock(int blockNumber, byte[] source);
    void flush();
}

interface FileSystemService {
    void createFile(VirtualPath path, FileAttributes attributes);
    void openFile(VirtualPath path, OpenMode mode);
    ReadResult readFile(VirtualPath path, int length);
    WriteResult writeFile(VirtualPath path, byte[] data, int length);
    CloseResult closeFile(VirtualPath path);
    void deleteFile(VirtualPath path);
    byte[] typeFile(VirtualPath path);
    void changeAttributes(VirtualPath path, FileAttributes attributes);
    void makeDirectory(VirtualPath path);
    DirectorySnapshot listDirectory(VirtualPath path);
    void removeDirectory(VirtualPath path);
}
```

`FileSystemService` 是应用层使用的接口，`DefaultFileSystemService` 是第一版实现；控制器依赖接口，磁盘状态和命令提交仍只有一套真实实现。

- `BlockDevice` 的数组参数恰好可容纳 64 字节；实现逐次处理短读、短写，只有到 EOF 仍未凑齐块才判定镜像损坏。无进展情况有明确失败出口。
- 核心读取和写入单位是字节。`ReadResult` 包含 `byte[] data` 与 EOF 状态；`WriteResult` 记录实际写入字节数；`CloseResult` 能区分已关闭和原本未打开。不可用 `null` 或模糊的字符串表示这些状态。
- `DirectorySnapshot`、FAT 快照、打开表快照对外只读；若内部有数组，构造和返回时都做防御性复制。
- 原始 `byte[]` 的内容与 GUI 显示文本分开。文本输入编码为 UTF-8，`length` 是编码后的字节数，不能把 `String.length()` 当作字节长度。
- `FileSystemService` 只接受校验过的 `VirtualPath`；外部字符串在 `app` 边界转换，避免每个模块各自解析路径。
- 块号、属性字节、FAT 值的计算使用 `int`；读盘时对 Java 有符号 `byte` 做无符号转换，写盘前做范围检查。

## 5. 磁盘格式的专有命名

| 常量 / 字段 | 值或语义 | 使用位置 |
| --- | --- | --- |
| `BLOCK_COUNT` | 128 | 镜像和 FAT 项数 |
| `BLOCK_SIZE` | 64 字节 | 块设备与每个缓冲槽 |
| `IMAGE_SIZE` | 8192 字节 | 新建与挂载校验 |
| `FAT_FIRST_BLOCK` | 0 | FAT 起点 |
| `FAT_BLOCK_COUNT` | 2 | FAT 占两块 |
| `ROOT_DIRECTORY_BLOCK` | 2 | 根目录固定块 |
| `DIRECTORY_ENTRY_SIZE` | 8 字节 | 登记项编解码 |
| `DIRECTORY_SLOT_COUNT` | 8 | 每目录容量 |
| `MAX_OPEN_FILES` | 5 | 打开表大小 |
| `FAT_FREE` | 0 | 空闲块 |
| `FAT_BAD` | 254 | 坏块，默认示例为 23、49 |
| `FAT_END` | 255 | 链尾和系统保留块占用标记 |
| `EMPTY_ENTRY_MARKER` | `$`，目录项首字节 | 空槽 |
| `CONTENT_END_MARKER` | `#`，文件数据字节 | 文件内容终止 |

对文件目录项，第 7 字节命名 `allocatedBlockCount`；打开表中的实际长度命名 `byteLength`。两个值不能互换。目录项字节偏移为：名称 0～2，扩展名 3～4，属性 5，起始块 6，块数/保留字节 7。子目录第 7 字节写 `0`，即使目录本身占一个物理块。

一个目录固定占一块，含 8 个登记项；没有 `.`、`..` 项。文件内容占块公式为 `max(1, ceil((byteLength + 1) / 64))`，其中额外的 1 字节留给 `#`。公式使用足够宽的整数实现，避免长度溢出。已有镜像挂载时只校验，不自动格式化。

## 6. 名称、路径、属性与命令

虚拟基本名 1～3 字节，文件扩展名 0～2 字节。名称为 US-ASCII 非空格可打印字符，排除 `$`、`.`、`/`；文件名中的 `.` 只作基本名和扩展名分隔。示例 `a-b.tx`、`x_y.t` 合法。比较区分大小写；同目录文件和目录不能重名。路径以 `/` 开头，根为 `/`，拒绝相对路径以及 `.`、`..` 段。具体边界案例以开发方案第 5.2 节为准。

属性值使用 `FileAttributes` 封装：`4` 普通读写，`1` 普通只读，`2` 系统读写，`3` 系统只读，`8` 目录。`create_file` 只接受 `2/4`；属性修改按项目规则接受 `1/2/3/4`；其他位组合拒绝。只读限制写打开与追加，不自行扩大为禁止删除。

保留指导书对外命令的拼写，Java 内部方法统一为动词驼峰形式：

| 对外命令 | `FileSystemService` 方法 | 关键约束 |
| --- | --- | --- |
| `create_file` | `createFile` | 创建后以写方式加入打开表 |
| `open_file` | `openFile` | 同模式重复打开不重复占位；不同模式冲突 |
| `read_file` | `readFile` | 未打开则自动读打开，遇 `#` 停止 |
| `write_file` | `writeFile` | 未打开则自动写打开，只从尾部追加 |
| `close_file` | `closeFile` | 未打开返回“无需关闭”，写关闭确保结束符与刷新 |
| `delete_file` | `deleteFile` | 已打开拒绝，回收整条链 |
| `typefile` | `typeFile` | 已打开拒绝，直接显示到 `#`，不占打开表 |
| `change` | `changeAttributes` | 已打开拒绝 |
| `md` | `makeDirectory` | 父目录存在，申请并初始化一块 |
| `dir` | `listDirectory` | 返回真实目录项快照 |
| `rd` | `removeDirectory` | 保护根，仅允许空子目录 |

命令解析器仅负责语法、引号、长度和参数校验；运行结果由 `FsController` 从核心服务取得。CLI 与 GUI 内命令框复用解析器。菜单调用同一控制器，错误码与中文提示保持一致。

## 7. 错误、状态与持久化

`FsErrorCode` 使用 `UPPER_SNAKE_CASE`，例如 `INVALID_PATH`、`NAME_CONFLICT`、`DIRECTORY_FULL`、`DISK_FULL`、`OPEN_TABLE_FULL`、`OPEN_MODE_CONFLICT`、`FILE_BUSY`、`CORRUPT_FAT_CHAIN`、`DISK_IO_FAILURE`。`FsException` 携带错误码、操作对象和必要原因；底层 `IOException` 保留为异常原因。展示层负责中文说明，核心不返回已格式化的 UI 文案。

前置条件可判定失败时，磁盘、目录和打开表均不改变；例如目录满、空间不足、表满、只读写入。写入到一半的真实 I/O 故障无法保证回滚，进入 `FAULTED` 并拒绝后续修改，保留诊断状态。成功命令在报告成功前，必须完成相关数据、FAT、目录项写回并刷新。镜像重启一致性需要集成测试验证。

CLI 状态名固定为 `READY`、`RUNNING_COMMAND`、`RESETTING`、`EXITING`、`FAULTED`。默认空闲阈值 `600` 秒、检查间隔约 `200ms`、可配置范围 `60..3600` 秒，动作 `reset` 或 `exit`。使用单调时间计算，只在 `READY` 检查；命令完成后重新计时。重置清打开表和旧会话输入，保留镜像，不重新格式化。GUI 不受 CLI 超时策略控制。

## 8. 编码、测试与提交约定

- Java 代码采用 4 空格缩进；公开顶层类型一文件一个。值对象优先不可变；数组类型做防御性复制。
- 资源通过 `AutoCloseable` 和明确生命周期释放；关闭错误不能静默吞掉。构造器传入依赖与单调时钟，方便用假设备和假时钟验证边界。
- 对块设备测试完整块、尺寸异常、越界和短 I/O；对 FAT 测试环、坏块、空闲后继和回收；对文件测试 63/64/65 字节、满盘与重新挂载。
- 测试临时镜像放在独立临时目录，不操作正式 `data/fatfs.img`。测试方法描述可观察行为，断言结果和磁盘字节，不只断言内部方法被调用。
- GUI 组件与模型在 Swing 事件分发线程更新；磁盘命令由串行后台执行器运行。观察用快照由核心统一生成，禁止独立伪造 FAT/目录状态。
- 运行镜像、日志、`target/`、本地 `.skill/`、`.agents/skills/` 不加入 Git；源码、测试、小型测试资源和文档可提交。工程创建时补齐 `.gitignore` 的 Java/Maven 条目。

开始编码时先落实 P0～P2 的环境、工程和块 I/O。任何接口、磁盘字节布局或命令语义的变更，都同步更新此规范、[开发阶段任务](./题目五-Java开发方案与阶段任务.md)和相应测试。
