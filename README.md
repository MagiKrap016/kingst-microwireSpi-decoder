# Microwire SPI 协议解析库说明

## 1. 功能与定位

本项目为 **Kingst 逻辑分析仪（上位机软件）** 提供一个 **Microwire SPI 协议解析插件**：

- 以动态库形式集成（Windows 下为 `SpiAnalyzer.dll`）；
- 解码 Microwire SPI 总线上的 **起始位、Op Code、地址位和数据段**；
- 支持在波形界面显示 **气泡文本 / 标记（marker）**，并可导出为 **文本 / CSV 文件**；
- 基于 Kingst / Saleae Analyzer SDK 开发，可作为二次开发和协议扩展示例。

将编译生成的动态库拷贝到 Kingst 上位机软件安装目录后，即可在“协议解码/Analyzer”列表中看到 **“Microwire SPI”** 解析器，选择后即可对采集到的 SPI 波形进行在线协议解码与导出。

---

## 2. 工程目录与文件结构

项目主要目录及功能如下：

- **src（核心源码）** 目录 [src](src)：
  - SpiAnalyzer.h / SpiAnalyzer.cpp：解析器主类与解码线程；
  - SpiAnalyzerSettings.h / SpiAnalyzerSettings.cpp：解析器配置界面与参数管理；
  - SpiAnalyzerResults.h / SpiAnalyzerResults.cpp：解析结果显示与导出；
  - SpiSimulationDataGenerator.h / SpiSimulationDataGenerator.cpp：仿真 Microwire SPI 波形生成。

- **vs2013（Windows 工程）** 目录 [vs2013](vs2013)：
  - SpiAnalyzer.sln、SpiAnalyzer.vcxproj：Visual Studio 2013 工程文件；
  - Debug/Release/x64：不同配置输出目录（Release/x64 下生成 SpiAnalyzer.dll）。

- **Linux（Linux 构建）** 目录 [Linux](Linux)：
  - makefile：用于在 Linux 平台编译生成共享库。

- **Mac（macOS 构建）** 目录 [Mac](Mac)：
  - makefile：用于在 macOS 平台编译生成共享库。

---

## 3. Microwire SPI 解码原理概览

Microwire SPI 协议解析的核心流程实现在 [src/SpiAnalyzer.cpp](src/SpiAnalyzer.cpp) 中的 `SpiAnalyzer::WorkerThread` 与 `SpiAnalyzer::GetWord`：

### 3.1 基本时序与分段

- 以 `Clock` 与 `Enable`（片选）为边界，确定一次完整的传输区间；
- 在每个传输区间内，使用内部计数 `step_count` 分段采样：
  1. **Step 0：起始位（Start Bit）**
     - 固定 1 bit；
     - 用于标识一次 Microwire 传输的开始；
  2. **Step 1：Op Code 段**
     - 固定 2 bit；
     - 区分读 / 写等操作类型；
  3. **Step 2：地址段（Address）**
     - 位数与数据位宽相关：
       - 当 `BitsPerTransfer = 8`：地址为 **11 bit**；
       - 当 `BitsPerTransfer = 16`：地址为 **10 bit**；
  4. **Step ≥ 3：数据段（Data）**
     - 每一步对应一个完整数据字；
     - 位宽等于 `BitsPerTransfer`（8 bit 或 16 bit）。

### 3.2 MOSI / MISO 采样与时钟配置

- 按解析器设置中的：
  - `ClockInactiveState`（时钟空闲电平，CPOL）；
  - `DataValidEdge`（数据有效边 / 电平，CPHA / Level）；
  决定在 **上升沿 / 下降沿 / 电平中间** 进行采样（典型为电平采样模式）。
- 起始位、Op Code 与地址段只对 **MOSI** 进行采样；
- 数据段会同时解码 **MOSI / MISO**，用于区分主发从收与主收从发的数据流。

### 3.3 错误检测与 Frame 类型

- 若时钟空闲电平与配置不一致，会：
  - 在波形时间轴上打出错误标记；
  - 生成错误帧（`SPI_ERROR_FLAG`），并给出 **“CLK 空闲电平与设置不匹配”** 等提示。

- 每一段起始 / Op Code / 地址 / 数据，都会封装为一个 `Frame`，其 `mType` 含义为：
  - `mType = 0`：起始位；
  - `mType = 1`：Op Code；
  - `mType = 2`：地址；
  - `mType ≥ 3`：数据字；

在结果气泡与导出文本中，会根据 `mType` 自动选择合适的位宽与格式进行显示，便于区分不同字段。

---

## 4. 主要源码与接口说明

### 4.1 SpiAnalyzer（解析器本体）

- 文件： [src/SpiAnalyzer.h](src/SpiAnalyzer.h)、[src/SpiAnalyzer.cpp](src/SpiAnalyzer.cpp)
- 主要职责：
  - 绑定并管理采样通道：`MOSI / MISO / Clock / Enable`；
  - 创建、运行 Microwire 解码主线程 `WorkerThread`；
  - 按 Step 状态机执行 `GetWord` 完成分段取样；
  - 进行时序与配置一致性检查，生成错误帧与进度上报；
  - 对接仿真数据入口，为 Simulation 模式提供波形。
- 导出接口（由 Kingst / Saleae SDK 动态加载）：
  - `GetAnalyzerName()`：返回协议名称 **"Microwire SPI"**；
  - `CreateAnalyzer()` / `DestroyAnalyzer()`：创建与销毁解析器实例。

### 4.2 SpiAnalyzerSettings（配置与 UI）

- 文件： [src/SpiAnalyzerSettings.h](src/SpiAnalyzerSettings.h)、[src/SpiAnalyzerSettings.cpp](src/SpiAnalyzerSettings.cpp)
- 主要职责：
  - 管理解析器所有可配置参数（通道、极性、位宽等）；
  - 构建上位机“设置/Settings”对话框；
  - 校验配置合法性（如通道不重叠、必要通道已选择等）；
  - 将配置序列化 / 反序列化到上位机配置中：
    - `SaveSettings()` / `LoadSettings()` 使用标识 **"KingstSpiAnalyzer"**。

### 4.3 SpiAnalyzerResults（结果显示与导出）

- 文件： [src/SpiAnalyzerResults.h](src/SpiAnalyzerResults.h)、[src/SpiAnalyzerResults.cpp](src/SpiAnalyzerResults.cpp)
- 主要职责：
  - 生成波形上的气泡文本：`GenerateBubbleText`；
  - 生成表格视图文本：`GenerateFrameTabularText`；
  - 生成文本 / CSV 导出文件：`GenerateExportFile`。
- CSV 导出列格式：
  - `Time [s], Packet ID, MOSI, MISO`
    - 时间以秒为单位，基于触发点和采样率计算；
    - `Packet ID` 为 SDK 中的分组编号（如有）；
    - MOSI/MISO 会根据帧类型和数据位宽自动格式化显示。

### 4.4 SpiSimulationDataGenerator（仿真数据）

- 文件： [src/SpiSimulationDataGenerator.h](src/SpiSimulationDataGenerator.h)、[src/SpiSimulationDataGenerator.cpp](src/SpiSimulationDataGenerator.cpp)
- 主要职责：
  - 在无真实 DUT 的情况下生成标准的 Microwire SPI 波形；
  - 用于快速验证解析器逻辑与 UI 显示效果。

---

## 5. 解析器可配置参数说明

在 Kingst 上位机中添加 **“Microwire SPI”** 协议解析器后，可在“设置 / Settings”中配置以下参数（由 `SpiAnalyzerSettings` 提供）：

### 5.1 通道选择

- **MOSI**（Master Out, Slave In）：主机发送、从机接收数据线；
- **MISO**（Master In, Slave Out）：主机接收、从机发送数据线；
- **Clock（CLK）**：时钟线；
- **Enable（片选 / SS）**：可选，指定一次事务的起止；
- 要求：通道之间 **不能重叠**，至少需设置 MOSI 或 MISO 中的一个通道。

### 5.2 数据位序（Shift Order）

- `Most Significant Bit First (MSB First)`：高位先行（标准 Microwire 模式）；
- `Least Significant Bit First (LSB First)`：低位先行。

### 5.3 传输位宽（Bits per Transfer）

- `Org 8`：每个数据字 8 bit；
- `Org 16`：每个数据字 16 bit；
- 同时影响地址字段长度：
  - Org 8：地址为 11 bit；
  - Org 16：地址为 10 bit。

### 5.4 时钟极性（Clock Inactive State / CPOL）

- `Clock is Low when inactive (CPOL = 0)`：时钟空闲为低电平；
- `Clock is High when inactive (CPOL = 1)`：时钟空闲为高电平。

### 5.5 时钟相位（Data Valid Edge / CPHA）

- `Data is Valid on Clock Leading Edge (CPHA = 0)`：前沿采样数据；
- `Data is Valid on Clock Trailing Edge (CPHA = 1)`：后沿采样数据；
- `Data is Valid on Clock Level (CPHA = 2)`：在稳定电平中点采样。

### 5.6 片选有效极性（Enable Active State）

- `Enable line is Active Low`：低电平有效；
- `Enable line is Active High`：高电平有效。

### 5.7 解码标记显示（Show Decode Marker）

- 可选择是否在波形上显示每一位采样位置的 **箭头 / marker**；
- 关闭后仅保留气泡与帧区域，更适合观察长帧数据。

---

## 6. 编译与生成动态库

### 6.1 Windows 平台（推荐方式）

1. 安装 Visual Studio 2013 及 C++ 开发组件；
2. 打开工程文件：
   - [vs2013/SpiAnalyzer.sln](vs2013/SpiAnalyzer.sln)
3. 在 VS 中选择构建配置：
   - `Configuration`：Release；
   - `Platform`：x64；
4. 执行“生成解决方案 / Build Solution”；
5. 编译成功后，在以下目录获得目标 DLL：
   - [vs2013/Release/x64/SpiAnalyzer.dll](vs2013/Release/x64/SpiAnalyzer.dll)

> **注意：** 请按 Kingst Analyzer SDK 文档配置 **Include 目录** 和 **库文件路径**，否则可能导致编译失败。

### 6.2 Linux / macOS 平台（可选）

1. 安装 gcc / clang、make 等编译工具；
2. 在对应目录执行命令：
   - Linux：进入 [Linux](Linux) 目录，执行 `make`；
   - macOS：进入 [Mac](Mac) 目录，执行 `make`；
3. 生成对应平台的共享库（`.so` / `.dylib`），可用于集成到兼容 Analyzer SDK 的其他工具中。

---

## 7. 在 Kingst 上位机中的使用步骤

1. 按第 6 节在 Windows 平台编译并获得 `SpiAnalyzer.dll`；
2. 将 `SpiAnalyzer.dll` 复制到 **Kingst 逻辑分析仪上位机软件的安装根目录**；
3. 重新启动 Kingst 上位机软件；
4. 在“协议解码 / Analyzer”列表中，选择 **“Microwire SPI”** 解析器；
5. 在“Settings / 设置”中：
   - 选择 MOSI / MISO / Clock / Enable 对应的物理通道；
   - 根据被测器件手册，设置 CPOL / CPHA、位宽和位序等参数；
   - 根据需要开启 / 关闭 marker 显示；
6. 开始采集并观察波形：
   - 可在波形上查看起始位、Op Code、地址和数据等解码结果；
   - 可将结果导出为文本或 CSV 文件，用于后续分析与归档。

---

## 8. 典型应用场景

- 调试采用 **Microwire SPI 接口** 的 EEPROM / Flash / MCU 或外设；
- 在 Kingst 逻辑分析仪软件中统一查看、对比和导出 Microwire SPI 读写数据；
- 在无真实硬件的情况下，通过仿真波形验证 Microwire 协议解析器的功能与界面显示；
- 作为基于 Kingst / Saleae Analyzer SDK 进行 **自定义协议解析器开发** 的参考工程。

如需扩展更多 Microwire 变种、专有命令或字段解析，可在 `GetWord` 状态机的基础上增加新的帧类型与显示逻辑，并在 `SpiAnalyzerResults` 中补充对应的文本格式化规则。

---

## 9. 参考与依赖

- Kingst Analyzer SDK 及示例工程：  
  https://github.com/flyghost/KingsVIS-Analyzer/tree/master
- 请严格按照 SDK 文档说明：
  - 配置工程的 **头文件包含路径（Include Directories）**；
  - 配置 **库文件 / 依赖模块**；
  否则工程可能无法通过编译或无法被 Kingst 上位机正常加载。
