# Microwire SPI 协议解析库（Kingst 逻辑分析仪插件）

## 1. 项目简介

本项目是 Kingst 逻辑分析仪（上位机软件）的协议解析库，用于解析 Microwire SPI 总线数据。  
编译生成的动态库（Windows 下为 `SpiAnalyzer.dll`）放入 Kingst 上位机软件根目录后，即可在软件中以“Microwire SPI”协议解析器的形式使用。

解析器基于 Saleae/Kingst Analyzer SDK，按时钟、片选和数据线对 Microwire SPI 总线进行采样和解码，并支持结果气泡显示与文本/CSV 导出。

---

## 2. Microwire SPI 协议解析逻辑概览

核心解码流程实现在 [src/SpiAnalyzer.cpp](src/SpiAnalyzer.cpp) 中的 `SpiAnalyzer::WorkerThread` 与 `SpiAnalyzer::GetWord`：

- 通过 `Clock` 与 `Enable`（片选）建立一次完整的传输区间；
- 在每个字传输中，按“步骤（step_count）”逐段采样：
  1. **Step 0（起始位）**  
     - 固定 1 bit，表示 Microwire 传输的起始位；
  2. **Step 1（Op Code）**  
     - 固定 2 bit，用于区分读/写等操作码；
  3. **Step 2（地址位）**  
     - 位数随配置而变：  
       - 当 `BitsPerTransfer = 8` 时：地址为 **11 bit**；  
       - 当 `BitsPerTransfer = 16` 时：地址为 **10 bit**；
  4. **Step ≥ 3（数据段）**  
     - 每一步都是一组完整的数据，位宽等于 `BitsPerTransfer`（8 或 16 bit）；
- MOSI/MISO 取样逻辑：
  - 根据配置的 `ClockInactiveState` 和 `DataValidEdge`（CPOL/CPHA/Level）决定在**上升沿、下降沿或电平中间**去采样(通常为电平采样)；
  - 前三步（起始、Op Code、地址）只对 MOSI 进行采样，MISO 在数据阶段才有效；
- 时序错误检测：
  - 若时钟空闲电平与设置不符，会在时间轴上标记错误，并生成错误帧（`SPI_ERROR_FLAG`），提示“CLK 空闲电平与设置不匹配”。

每一段（起始/OpCode/地址/数据）都会被封装成一个 `Frame`：
- `mType = 0`：起始位；
- `mType = 1`：Op Code；
- `mType = 2`：地址；
- `mType ≥ 3`：数据字；

在结果气泡和导出文件中，会根据 `mType` 选择对应的显示位宽与文字说明。

---

## 3. 主要代码结构

- [src/SpiAnalyzer.h](src/SpiAnalyzer.h) / [src/SpiAnalyzer.cpp](src/SpiAnalyzer.cpp)  
  - 实现 `SpiAnalyzer` 主类：  
    - 采样通道绑定（MOSI/MISO/CLK/Enable）；  
    - 解析主线程 `WorkerThread`；  
    - 分段取样与 Microwire 协议状态机 `GetWord`；  
    - 错误检测、进度上报、仿真数据入口等；
  - 对外导出接口（供上位机动态加载）：  
    - `GetAnalyzerName()` → `"Microwire SPI"`  
    - `CreateAnalyzer()` / `DestroyAnalyzer()`。

- [src/SpiAnalyzerSettings.h](src/SpiAnalyzerSettings.h) / [src/SpiAnalyzerSettings.cpp](src/SpiAnalyzerSettings.cpp)  
  - 管理协议解析器的所有用户配置项和 UI 交互：
    - 通道选择：MOSI / MISO / Clock / Enable；
    - 位序：`MSB First` / `LSB First`；
    - 字长：`Org 8` / `Org 16`；
    - 时钟极性 CPOL：`Clock Low/High when inactive`；
    - 时钟相位 CPHA：`Leading Edge` / `Trailing Edge` / `Level`；
    - 片选有效电平：`Active Low` / `Active High`；
    - 是否显示解码标记（marker）。
  - 负责设置的序列化/反序列化：
    - `SaveSettings()` / `LoadSettings()` 使用 `"KingstSpiAnalyzer"` 作为标识存储在上位机配置中。

- [src/SpiAnalyzerResults.h](src/SpiAnalyzerResults.h) / [src/SpiAnalyzerResults.cpp](src/SpiAnalyzerResults.cpp)  
  - 负责将解析结果转换成：
    - 波形气泡文本（`GenerateBubbleText`）；
    - 文本/CSV 导出（`GenerateExportFile`）；
    - 表格视图文本（`GenerateFrameTabularText`）。
  - 导出 CSV 列格式：  
    `Time [s], Packet ID, MOSI, MISO`  
    - 时间按触发点与采样率折算为秒；  
    - `Packet ID` 为 Kingst/Saleae SDK 中的分组编号（如有）；  
    - MOSI/MISO 字符串根据帧类型（起始/OpCode/地址/数据）自动选择合适位宽显示。

- [src/SpiSimulationDataGenerator.h](src/SpiSimulationDataGenerator.h) / [src/SpiSimulationDataGenerator.cpp](src/SpiSimulationDataGenerator.cpp)  
  - 为上位机“仿真模式”生成测试用的 Microwire SPI 波形（便于在无真实信号源时调试插件）。

- 工程与构建文件：
  - Windows：  
    - [vs2013/SpiAnalyzer.sln](vs2013/SpiAnalyzer.sln)  
    - [vs2013/SpiAnalyzer.vcxproj](vs2013/SpiAnalyzer.vcxproj)
  - Linux：  
    - [Linux/makefile](Linux/makefile)
  - macOS：  
    - [Mac/makefile](Mac/makefile)

---

## 4. 可配置参数说明（在 Kingst 上位机中）

解析器在上位机软件中以“Microwire SPI”出现时，你可以在“设置/Settings”对话框中配置如下选项（由 `SpiAnalyzerSettings` 提供界面与检查）：

- 通道选择：
  - `MOSI`（Master Out, Slave In）  
  - `MISO`（Master In, Slave Out）  
  - `Clock`（CLK）  
  - `Enable`（SS/片选），可选；  
  - 要求通道之间不能重叠，至少需设置 MOSI 或 MISO 其中之一。
- 位序（Shift Order）：
  - `Most Significant Bit First (Standard)`  
  - `Least Significant Bit First`
- 传输位宽（Bits per Transfer）：
  - `Org 8`：8 bit 数据；  
  - `Org 16`：16 bit 数据；  
  - 同时影响地址位长度（11 bit / 10 bit）。
- 时钟极性（Clock Inactive State / CPOL）：
  - `Clock is Low when inactive (CPOL = 0)`  
  - `Clock is High when inactive (CPOL = 1)`
- 时钟相位（Data Valid Edge / CPHA）：
  - `Data is Valid on Clock Leading Edge (CPHA = 0)`  
  - `Data is Valid on Clock Trailing Edge (CPHA = 1)`  
  - `Data is Valid on Clock Level (CPHA = 2)`
- 片选有效电平（Enable Active State）：
  - `Enable line is Active Low (Standard)`  
  - `Enable line is Active High`
- 是否显示解码标记（Show Decode Marker）：
  - 可以勾选/取消显示每一位采样位置上的箭头或标记。

---

## 5. 编译与生成动态库

### 5.1 Windows（推荐）

1. 使用 Visual Studio 2013 打开工程文件：  
   - [vs2013/SpiAnalyzer.sln](vs2013/SpiAnalyzer.sln)
2. 选择生成配置：
   - 配置：`Release`  
   - 平台：`x64`
3. 编译解决方案：
   - 在 VS 中执行“生成解决方案”；
4. 成功后，将在类似目录生成目标动态库：  
   - [vs2013/Release/x64/SpiAnalyzer.dll](vs2013/Release/x64/SpiAnalyzer.dll)

### 5.2 Linux / macOS（可选）

1. 安装 C++ 编译环境（gcc/clang、make 等）；  
2. 在对应目录执行：
   - Linux：在 `Linux` 目录中运行 `make`；  
   - macOS：在 `Mac` 目录中运行 `make`；  
3. 生成对应平台的共享库（`.so` / `.dylib`），可供支持该接口的上位机或工具使用。

---

## 6. 在 Kingst 上位机中的使用方法

1. 使用上述步骤生成 Windows 版本动态库：  
   - `SpiAnalyzer.dll`
2. 将该 DLL 文件复制到 Kingst 逻辑分析仪上位机软件的**安装根目录**。
3. 重启 Kingst 上位机软件：
   - 在“协议解码/Analyzer”列表中，应能看到名为 **“Microwire SPI”** 的解析器；
4. 在软件中配置解析选项：
   - 为 MOSI/MISO/CLK/Enable 选择对应的逻辑通道；  
   - 根据实际被测设备设置 CPOL/CPHA、位序、位宽等；  
   - 可选择是否显示 decode marker。
5. 开始采集：
   - 采集完成后，在波形中可看到 Microwire SPI 的起始位、OpCode、地址和数据等解析结果；  
   - 可通过导出功能生成文本或 CSV 文件，便于后期分析。

---

## 7. 适用场景

- 对使用 Microwire SPI 协议的存储器、MCU 或外设进行总线调试与协议分析；
- 在 Kingst 逻辑分析仪上位机软件中统一管理、查看、导出 Microwire SPI 通信数据；
- 需要在无真实硬件时进行协议解码插件调试，可利用内置仿真数据生成功能。

如需进一步扩展（例如增加更多 Microwire 变种、特殊命令解码、专用字段解析等），可以在现有 `GetWord` 状态机基础上增加新的帧类型和结果显示逻辑。