# **自定义固件开发指南：实现完整设备仿真**

---

## **目录**

### **第一部分：基础概念**

1. [介绍](#1-介绍)
   - [1.1 本指南的目的](#11-本指南的目的)
   - [1.2 目标读者](#12-目标读者)
   - [1.3 如何使用本指南](#13-如何使用本指南)
2. [关键定义](#2-关键定义)
3. [设备兼容性](#3-设备兼容性)
   - [3.1 支持的FPGA硬件](#31-支持的FPGA硬件)
   - [3.2 PCIe硬件注意事项](#32-pcie硬件注意事项)
   - [3.3 系统要求](#33-系统要求)
4. [需求](#4-需求)
   - [4.1 硬件](#41-硬件)
   - [4.2 软件](#42-软件)
   - [4.3 环境设置](#43-环境设置)
5. [收集捐赠设备信息](#5-收集捐赠设备信息)
   - [5.1 使用Arbor进行PCIe设备扫描](#51-使用arbor进行pcie设备扫描)
   - [5.2 提取和记录设备属性](#52-提取和记录设备属性)
6. [初始固件定制](#6-初始固件定制)
   - [6.1 修改配置空间](#61-修改配置空间)
   - [6.2 插入设备序列号(DSN)](#62-插入设备序列号dsn)
7. [Vivado项目设置和定制](#7-vivado项目设置和定制)
   - [7.1 生成Vivado项目文件](#71-生成vivado项目文件)
   - [7.2 修改IP块](#72-修改ip块)

### **第二部分：中级概念和实现**

8. [高级固件定制](#8-高级固件定制)
   - [8.1 为仿真配置PCIe参数](#81-为仿真配置pcie参数)
   - [8.2 调整BAR和内存映射](#82-调整bar和内存映射)
   - [8.3 仿真设备电源管理和中断](#83-仿真设备电源管理和中断)
9. [仿真设备特定功能](#9-仿真设备特定功能)
   - [9.1 实现高级PCIe功能](#91-实现高级pcie功能)
   - [9.2 仿真供应商特定功能](#92-仿真供应商特定功能)
10. [事务层包(TLP)仿真](#10-事务层包tlp仿真)
    - [10.1 理解和捕获TLP](#101-理解和捕获tlp)
    - [10.2 为特定操作制作自定义TLP](#102-为特定操作制作自定义tlp)

### **第三部分：高级技术和优化**

11. [构建、烧录和测试](#11-构建烧录和测试)
    - [11.1 综合与实现](#111-综合与实现)
    - [11.2 烧录比特流](#112-烧录比特流)
    - [11.3 测试与验证](#113-测试与验证)
12. [高级调试技术](#12-高级调试技术)
    - [12.1 使用Vivado的集成逻辑分析器](#121-使用vivado的集成逻辑分析器)
    - [12.2 PCIe流量分析工具](#122-pcie流量分析工具)
13. [故障排除](#13-故障排除)
    - [13.1 设备检测问题](#131-设备检测问题)
    - [13.2 内存映射和BAR配置错误](#132-内存映射和bar配置错误)
    - [13.3 DMA性能和TLP错误](#133-dma性能和tlp错误)
14. [仿真精度和优化](#14-仿真精度和优化)
    - [14.1 精确时间仿真技术](#141-精确时间仿真技术)
    - [14.2 动态响应系统调用](#142-动态响应系统调用)
15. [固件开发最佳实践](#15-固件开发最佳实践)
    - [15.1 持续测试和文档](#151-持续测试和文档)
    - [15.2 管理固件版本](#152-管理固件版本)
    - [15.3 安全考虑](#153-安全考虑)
16. [附加资源](#16-附加资源)

---

## **第一部分：基础概念**

---

## **1. 介绍**

### **1.1 本指南的目的**

本指南的主要目的是提供一个逐步的方法来开发用于FPGA设备的自定义直接内存访问(DMA)固件，以准确仿真PCIe硬件。这使得硬件测试、系统调试、安全研究和硬件仿真等应用成为可能。

通过遵循本指南，您将学习如何：

- 从捐赠设备收集必要信息。
- 定制固件以仿真特定硬件设备。
- 使用Vivado和Visual Studio Code等工具设置开发环境。
- 理解与PCIe和DMA操作相关的关键概念。

### **1.2 目标读者**

本指南适用于：

- **固件开发人员**：对创建用于硬件仿真、测试或绕过硬件限制的自定义固件感兴趣的工程师。
- **硬件工程师**：需要仿真特定设备的硬件测试和开发专业人员。
- **安全研究人员**：进行漏洞评估、恶意软件分析或需要硬件仿真的安全测试的个人。
- **FPGA爱好者**：对FPGA定制和低级硬件仿真感兴趣的爱好者和学习者。

### **1.3 如何使用本指南**

本指南分为三个部分：

- **第一部分：基础概念**：涵盖固件开发的基本概念、设置和初始步骤。
- **第二部分：中级概念和实现**：深入探讨更复杂的主题，如高级固件定制、TLP仿真和初步调试技术。
- **第三部分：高级技术和优化**：探索高级调试、故障排除、优化策略和最佳实践。

建议按顺序阅读本指南，以便在处理高级主题之前建立坚实的理解。

---

## **2. 关键定义**

理解术语对于有效地遵循本指南至关重要。以下是与PCIe、DMA和设备仿真相关的关键定义：

- **DMA (Direct Memory Access)**：一种允许硬件设备直接从或写入系统内存的能力，无需CPU干预，从而实现高速数据传输。
- **TLP (Transaction Layer Packet)**：PCIe架构中的基本通信单元，封装控制和数据信息。
- **BAR (Base Address Register)**：PCIe设备中的寄存器，定义设备内存映射到系统内存空间的内存和I/O地址区域。
- **FPGA (Field-Programmable Gate Array)**：一种可重新配置的集成电路，可以编程以执行特定的硬件功能。
- **MSI/MSI-X (Message Signaled Interrupts)**：PCIe设备用于向CPU发送中断的机制，无需使用传统的中断线。
- **设备序列号(DSN)**：与特定设备相关的唯一标识符，通常用于高级设备识别。
- **PCIe配置空间**：一个标准化的内存区域，PCIe设备在其中提供关于自身的信息并配置操作参数。
- **捐赠设备**：用于提取配置和识别详细信息以在FPGA上仿真其行为的PCIe硬件设备。

---

## **3. 设备兼容性**

### **3.1 支持的FPGA硬件**

虽然本指南主要关注**Squirrel DMA (35T)**卡，但这些方法也适用于其他基于FPGA的DMA硬件：

- **Squirrel (35T)**
  - **描述**：适用于标准内存获取和设备仿真的经济型FPGA设备。
- **Enigma-X1 (75T)**
  - **描述**：中档FPGA，提供增强的资源，适合更高要求的内存操作。
- **ZDMA (100T)**
  - **描述**：高性能FPGA，优化用于快速内存交互，适合大规模内存读/写。
- **Kintex-7**
  - **描述**：功能强大的FPGA，适用于复杂项目和大规模DMA解决方案。

### **3.2 PCIe硬件注意事项**

为了确保顺利仿真，必须解决几个PCIe特定的功能：

- **IOMMU/VT-d设置**
  - **建议**：禁用IOMMU（Intel的VT-d）或AMD的等效功能，以允许不受限制的DMA访问。
  - **理由**：IOMMU可能会限制DMA操作，可能会干扰内存获取和仿真。
- **内核DMA保护**
  - **建议**：在现代系统中禁用内核DMA保护功能。
  - **步骤**：
    - **Windows**：在BIOS/UEFI设置中禁用安全启动或基于虚拟化的安全性（VBS）等功能。
    - **注意**：禁用这些功能可能会使系统面临安全风险；确保您在安全环境中操作。
- **PCIe插槽要求**
  - **建议**：使用与FPGA设备要求匹配的兼容PCIe插槽（例如，x1，x4）。
  - **理由**：确保与主机系统的最佳性能和兼容性。

### **3.3 系统要求**

- **主机系统**
  - **处理器**：多核CPU（Intel i5/i7或AMD等效）
  - **内存**：至少16 GB RAM
  - **存储**：至少100 GB可用空间的SSD
  - **操作系统**：Windows 10/11（64位）或兼容的Linux发行版
- **外围设备**
  - **JTAG编程器**：用于将固件烧录到FPGA
  - **PCIe插槽**：确保主机系统有一个与DMA卡兼容的可用PCIe插槽

---

## **4. 需求**

### **4.1 硬件**

- **捐赠PCIe设备**
  - **目的**：用于仿真的设备ID和配置数据的来源。
  - **示例**：网络适配器、存储控制器或任何未使用的通用PCIe卡。
- **DMA FPGA卡**
  - **描述**：能够执行DMA操作的基于FPGA的设备。
  - **示例**：Squirrel (35T)、Enigma-X1 (75T)、ZDMA (100T)、Kintex-7
- **JTAG编程器**
  - **目的**：用于将固件烧录到FPGA。
  - **示例**：Xilinx Platform Cable USB II、Digilent JTAG-HS3

### **4.2 软件**

- **Xilinx Vivado Design Suite**
  - **描述**：用于合成和构建固件项目的FPGA开发软件。
  - **下载**：[Xilinx Vivado](https://www.xilinx.com/support/download.html)
- **Visual Studio Code**
  - **描述**：用于编辑Verilog或VHDL代码的代码编辑器。
  - **下载**：[Visual Studio Code](https://code.visualstudio.com/)
- **PCILeech-FPGA**
  - **描述**：用于DMA固件开发的代码库和基础代码。
  - **代码库**：[PCILeech-FPGA on GitHub](https://github.com/ufrisk/pcileech-fpga)
- **Arbor**
  - **描述**：用于收集设备信息的PCIe设备扫描工具。
  - **下载**：[Arbor by MindShare](https://www.mindshare.com/software/Arbor)
  - **注意**：需要创建账户；提供14天试用。
- **替代工具**
  - **Telescan PE**
    - **描述**：作为Arbor的替代品的PCIe流量分析工具。
    - **下载**：[Teledyne LeCroy Telescan PE](https://www.teledynelecroy.com/protocolanalyzer/pci-express/telescan-pe-software/resources/analysis-software)
    - **注意**：免费但需要手动注册批准。

### **4.3 环境设置**

#### **4.3.1 安装Xilinx Vivado Design Suite**

- **步骤**：
  1. 访问[Xilinx Vivado下载页面](https://www.xilinx.com/support/download.html)。
  2. 下载与您的FPGA设备兼容的版本。
  3. 运行安装程序并按照屏幕上的说明进行操作。
  4. 在安装过程中选择必要的组件。
  5. 启动Vivado以确保正确安装。

#### **4.3.2 安装Visual Studio Code**

- **步骤**：
  1. 访问[Visual Studio Code下载页面](https://code.visualstudio.com/)。
  2. 下载并安装适用于您的操作系统的版本。
  3. 安装Verilog或VHDL支持的扩展（例如，**Verilog-HDL/SystemVerilog**）。

#### **4.3.3 克隆PCILeech-FPGA代码库**

- **步骤**：
  1. 打开终端或命令提示符。
  2. 导航到您想要的目录：     ```bash
     cd ~/Projects/```
  3. 克隆代码库：     ```bash
     git clone https://github.com/ufrisk/pcileech-fpga.git```
  4. 导航到克隆的目录：     ```bash
     cd pcileech-fpga```

#### **4.3.4 设置干净的开发环境**

- **建议**：在隔离的环境中工作，以防止意外的交互。
- **步骤**：
  1. 使用专用的开发机器或虚拟机。
  2. 确保没有其他应用程序干扰PCIe操作或FPGA编程。

---

## **5. 收集捐赠设备信息**

准确的设备仿真依赖于从捐赠设备中提取和复制关键信息。这种全面的数据收集使您的FPGA能够忠实地模拟目标硬件的PCIe配置和行为，确保与主机系统的兼容性和功能。

### **5.1 使用Arbor进行PCIe设备扫描**

**Arbor**是一款设计用于深入扫描PCIe设备的强大且用户友好的工具。它提供了有关连接硬件的配置空间的详细见解，使其成为提取设备仿真所需信息的宝贵资源。

#### **5.1.1 安装Arbor**

要开始使用Arbor进行设备扫描，您必须首先在系统上安装该软件。

**步骤**：

1. **访问Arbor下载页面**：

   - 使用您喜欢的网络浏览器导航到官方[Arbor下载页面](https://www.mindshare.com/software/Arbor)。
   - 确保您直接访问该网站以避免任何恶意重定向。

2. **创建账户（如果需要）**：

   - Arbor可能需要您创建用户账户以访问下载链接。
   - 提供必要的信息，例如您的姓名、电子邮件地址和组织。
   - 如果提示，请验证您的电子邮件以激活您的账户。

3. **下载Arbor**：

   - 登录后，找到Arbor的下载部分。
   - 选择与您的操作系统兼容的版本（例如，Windows 10/11 64位）。
   - 点击**下载**按钮并将安装程序保存到计算机上的已知位置。

4. **安装Arbor**：

   - 找到下载的安装程序文件（例如，`ArborSetup.exe`）。
   - 右键单击安装程序并选择**以管理员身份运行**以确保其具有必要的权限。
   - 按照屏幕上的说明完成安装过程。
     - 接受许可协议。
     - 选择安装目录。
     - 如果需要，选择创建桌面快捷方式。

5. **验证安装**：

   - 完成后，确保Arbor列在您的开始菜单或桌面上。
   - 启动Arbor以确认其无错误打开。

#### **5.1.2 扫描PCIe设备**

安装Arbor后，您可以继续扫描系统中连接的PCIe设备。

**步骤**：

1. **启动Arbor**：

   - 双击桌面上的Arbor图标或通过开始菜单找到它。
   - 如果用户账户控制（UAC）提示，请允许应用程序对您的设备进行更改。

2. **导航到本地系统选项卡**：

   - 在Arbor界面中，找到导航窗格或选项卡。
   - 点击**本地系统**以访问扫描本地机器的工具。

3. **扫描PCIe设备**：

   - 查找通常位于界面顶部或底部的**扫描**或**重新扫描**按钮。
   - 点击**扫描/重新扫描**以启动检测过程。
   - 等待扫描过程完成；这可能需要几分钟，具体取决于连接的设备数量。

4. **查看检测到的设备**：

   - 扫描完成后，Arbor将显示所有检测到的PCIe设备的列表。
   - 设备通常按其名称、设备ID和其他识别信息列出。

#### **5.1.3 识别捐赠设备**

识别正确的捐赠设备对于准确仿真至关重要。

**步骤**：

1. **在列表中找到您的捐赠设备**：

   - 滚动浏览Arbor检测到的设备列表。
   - 查找与您的捐赠硬件的品牌和型号匹配的设备。
   - 设备可能按其供应商名称、设备类型或功能列出。

2. **验证设备详细信息**：

   - 点击设备以选择它。
   - 确认**设备ID**和**供应商ID**与您的捐赠设备匹配。
     - **提示**：这些ID通常可以在设备的文档中找到或在制造商的网站上找到。

3. **查看详细配置**：

   - 选择设备后，找到并点击类似**查看详细信息**或**属性**的选项。
   - 这将打开一个详细视图，显示设备的配置空间和功能。

4. **与物理硬件交叉引用**：

   - 如果列出了多个相似设备，请将**插槽号**或**总线地址**与安装捐赠设备的物理插槽进行交叉引用。

#### **5.1.4 捕获设备数据**

从捐赠设备中提取详细信息对于准确仿真至关重要。

**要提取的信息**：

- **设备ID (0xXXXX)**：
  - 一个16位的标识符，唯一标识设备型号。
- **供应商ID (0xYYYY)**：
  - 分配给制造商的16位标识符。
- **子系统ID (0xZZZZ)**：
  - 标识特定子系统或变体。
- **子系统供应商ID (0xWWWW)**：
  - 标识子系统的供应商。
- **修订ID (0xRR)**：
  - 指示设备的修订级别。
- **类代码 (0xCCCCCC)**：
  - 一个24位代码，定义设备的类型（例如，网络控制器、存储设备）。
- **基址寄存器(BARs)**：
  - 定义设备使用的内存或I/O空间的寄存器。
  - 包括BAR0到BAR5，每个可能是32位或64位。
- **功能**：
  - 列出支持的功能，如MSI/MSI-X、电源管理、PCIe链路速度和宽度。
- **设备序列号(DSN)**：
  - 一个64位的唯一标识符，如果设备支持的话。

**步骤**：

1. **导航到PCI配置选项卡**：

   - 在设备的详细视图中，找到并选择**PCI配置**或**配置空间**选项卡。

2. **记录相关详细信息**：

   - 仔细记录每个所需字段。
   - 使用截图或将值复制到文本文件或电子表格中以确保准确性。
   - 确保十六进制值记录正确，包括`0x`前缀（如果使用）。

3. **展开功���列表**：

   - 查找标记为**功能**或**高级功能**的部分。
   - 记录每个功能及其参数（例如，MSI计数、支持的电源状态）。

4. **详细检查BARs**：

   - 对于每个BAR，注意：
     - **BAR编号（例如，BAR0）**：
     - **类型（内存或I/O）**：
     - **位宽（32位或64位）**：
     - **大小（例如，256 MB）**：
     - **可预取状态（是/否）**：

5. **保存数据以供参考**：

   - 将所有信息编译成一个组织良好的文档。
   - 清晰标记每个部分，以便在固件定制时轻松参考。

6. **仔细检查条目**：

   - 查看所有记录的数据以确保准确性。
   - 通过重新访问Arbor界面纠正任何差异。

### **5.2 提取和记录设备属性**

在捕获数据后，了解每个属性的意义并确保它们已被准确记录是至关重要的。

**确保您已准确记录以下内容**：

1. **设备ID**：

   - **目的**：唯一标识设备型号。
   - **用途**：对于主机操作系统加载正确的驱动程序至关重要。

2. **供应商ID**：

   - **目的**：标识制造商。
   - **用途**：与设备ID结合使用以匹配设备驱动程序。

3. **子系统ID和子系统供应商ID**：

   - **目的**：指定子系统的设备和供应商ID，允许区分变体。
   - **用途**：对于具有多种配置或OEM特定版本的设备很重要。

4. **修订ID**：

   - **目的**：指示硬件修订。
   - **用途**：有助于识别可能需要不同驱动程序或固件的特定硬件版本。

5. **类代码**：

   - **目的**：对设备类型进行分类（例如，存储设备、网络控制器）。
   - **用途**：允许操作系统了解设备的一般功能。

6. **基址寄存器(BARs)**：

   - **目的**：定义设备将使用的内存或I/O地址区域。
   - **用途**：对于将设备内存映射到系统地址空间至关重要。

7. **功能**：

   - **目的**：列出设备支持的高级功能。
   - **示例**：
     - **MSI/MSI-X**：用于高效中断处理的消息信号中断。
     - **电源管理**：如D0、D1、D2、D3hot、D3cold等状态。
     - **PCIe链路速度/宽度**：决定数据传输能力。

8. **设备序列号(DSN)**：

   - **目的**：设备的唯一64位标识符。
   - **用途**：用于高级识别，可能是某些驱动程序所需的。

**最佳实践**：

- **组织数据**：

  - 创建一个结构化的文档或电子表格。
  - 使用清晰的标题和子标题标记每个属性。

- **包括单位和格式**：

  - 指明大小的单位（例如，MB，KB）。
  - 对十六进制值使用一致的格式（例如，`0x1234`）。

- **与规格交叉引用**：

  - 如果可用，查阅设备的数据表以验证值。
  - 这有助于识别任何差异或不寻常的配置。

- **保护数据**：

  - 安全存储收集的信息。
  - 注意任何专有或机密信息。

---

## **6. 初始固件定制**

在详细记录捐赠设备的信息后，下一步是定制您的FPGA固件以准确仿真捐赠设备。这涉及修改PCIe配置空间并确保内存映射正确对齐。

### **6.1 修改配置空间**

PCIe配置空间是定义设备如何被识别和与主机系统交互的关键组件。将此空间定制为与捐赠设备匹配对于成功仿真至关重要。

#### **6.1.1 导航到配置文件**

配置空间在项目中的特定SystemVerilog (.sv)文件中定义。

**路径**：

- **标准路径**：  ```
  pcileech-fpga/pcileech-wifi-main/src/pcie_7x_0_core_top.v ```

- **替代路径（取决于目录结构）**：  ```
  src\pcie_7x\pcie_7x_0_core_top.v ```

#### **6.1.2 在Visual Studio Code中打开文件**

编辑配置文件需要一个支持SystemVerilog语法高亮的合适代码编辑器。

**步骤**：

1. **启动Visual Studio Code**：

   - 点击VS Code图标或通过开始菜单找到它。

2. **打开文件**：

   - 使用**文件 > 打开文件**或按`Ctrl + O`。
   - 导航到上述配置文件路径。
   - 选择`pcileech_pcie_cfg_a7.sv`并点击**打开**。

3. **验证语法高亮**：

   - 确保编辑器识别`.sv`文件扩展名。
   - 如果需要，安装SystemVerilog支持的扩展。

4. **熟悉文件结构**：

   - 滚动浏览文件以了解现有的分配和注释。
   - 查找定义配置寄存器的部分。

#### **6.1.3 修改设备ID和供应商ID**

更新这些标识符对于主机系统识别仿真设备为捐赠设备至关重要。

**步骤**：

1. **搜索`cfg_deviceid`**：

   - 使用搜索功能（`Ctrl + F`）。
   - 找到定义`cfg_deviceid`的行。

2. **更新设备ID**：

   ```verilog
   cfg_deviceid <= 16'hXXXX;  // 用捐赠设备的设备ID替换XXXX   ```

   - **示例**：
     - 如果捐赠设备的设备ID是`0x1234`，则更新为：       ```verilog
       cfg_deviceid <= 16'h1234;       ```

3. **搜索`cfg_vendorid`**：

   - 找到定义`cfg_vendorid`的行。

4. **更新供应商ID**：

   ```verilog
   cfg_vendorid <= 16'hYYYY;  // 用捐赠设备的供应商ID替换YYYY   ```

   - **示例**：
     - 如果捐赠设备的供应商ID是`0xABCD`，则更新为：       ```verilog
       cfg_vendorid <= 16'hABCD;       ```

5. **确保格式正确**：

   - 验证十六进制值以`16'h`为前缀。
   - 保持一致的缩进和注释风格。

#### **6.1.4 修改子系统ID和修订ID**

这些标识符提供有关设备变体和硬件修订的附加详细信息。

**步骤**：

1. **搜索`cfg_subsysid`**：

   - 找到定义`cfg_subsysid`的行。

2. **更新子系统ID**：

   ```verilog
   cfg_subsysid <= 16'hZZZZ;  // 用捐赠设备的子系统ID替换ZZZZ   ```

   - **示例**：
     - 如果捐赠设备的子系统ID是`0x5678`，则更新为：       ```verilog
       cfg_subsysid <= 16'h5678;       ```

3. **搜索`cfg_subsysvendorid`**：

   - 找到定义`cfg_subsysvendorid`的行。

4. **更新子系统供应商ID（如果适用）**：

   ```verilog
   cfg_subsysvendorid <= 16'hWWWW;  // 用捐赠设备的子系统供应商ID替换WWWW   ```

   - **示例**：
     - 如果捐赠设备的子系统供应商ID是`0x9ABC`，则更新为：       ```verilog
       cfg_subsysvendorid <= 16'h9ABC;       ```

5. **搜索`cfg_revisionid`**：

   - 找到定义`cfg_revisionid`的行。

6. **更新修订ID**：

   ```verilog
   cfg_revisionid <= 8'hRR;   // 用捐赠设备的修订ID替换RR   ```

   - **示例**：
     - 如果捐赠设备的修订ID是`0x01`，则更新为：       ```verilog
       cfg_revisionid <= 8'h01;       ```

#### **6.1.5 更新类代码**

类代码通知主机设备的类型和功能。

**步骤**：

1. **搜索`cfg_classcode`**：

   - 找到定义`cfg_classcode`的行。

2. **更新类代码**：

   ```verilog
   cfg_classcode <= 24'hCCCCCC;  // 用捐赠设备的类代码替换CCCCCC   ```

   - **示例**：
     - 如果捐赠设备的类代码是`0x020000`（以太网控制器），则更新为：       ```verilog
       cfg_classcode <= 24'h020000;       ```

3. **验证正确的位宽**：

   - 确保类代码是24位值。
   - 十六进制值应以`24'h`为前缀。

#### **6.1.6 保存更改**

在进行所有修改后，保存并查看更改非常重要。

**步骤**：

1. **保存文件**：

   - 点击**文件 > 保存**或按`Ctrl + S`。

2. **查看更改**：

   - 重新阅读修改的行以确认准确性。
   - 检查是否有任何语法错误或拼写错误。

3. **可选 - 使用版本控制**：

   - 如果使用Git或其他版本控制系统，请使用有意义的消息提交更改。
     - **示例**：       ```
       git add pcileech_pcie_cfg_a7.sv
       git commit -m "更新PCIe配置为捐赠设备标识符" ```

### **6.2 插入设备序列号(DSN)**

设备序列号(DSN)是一些设备用于高级功能的唯一标识符。包括它可以增强仿真的真实性。

#### **6.2.1 定位DSN字段**

DSN通常在同一配置文件中定义。

**步骤**：

1. **搜索`cfg_dsn`**：

   - 在`pcileech_pcie_cfg_a7.sv`中，使用搜索功能（`Ctrl + F`）找到`cfg_dsn`。

2. **了解现有分配**：

   - DSN可能设置为默认值或清零。     ```verilog
     cfg_dsn <= 64'h0000000000000000;  // 默认DSN```

#### **6.2.2 插入DSN**

更新DSN涉及将其设置为捐赠设备的确切值。

**步骤**：

1. **更新`cfg_dsn`**：

   ```verilog
   cfg_dsn <= 64'hXXXXXXXX_YYYYYYYY;  // 用捐赠DSN替换   ```

   - **示例**：
     - 如果捐赠设备的DSN是`0x0011223344556677`，则更新为：       ```verilog
       cfg_dsn <= 64'h0011223344556677;       ```

2. **处理DSN不可用**：

   - 如果捐赠设备没有DSN或不需要，将其设置为零：     ```verilog
     cfg_dsn <= 64'h0000000000000000;  // 无DSN```

3. **确保格式正确**：

   - DSN是64位值；确保其格式正确。
   - 使用`64'h`前缀表示十六进制值。

4. **添加注释以提高清晰度**：

   - 包括注释以指示DSN来源。     ```verilog
     cfg_dsn <= 64'h0011223344556677;  // 捐赠DSN```

#### **6.2.3 保存更改**

通过保存和查看来完成修改。

**步骤**：

1. **保存文件**：

   - 点击**文件 > 保存**或按`Ctrl + S`。

2. **验证语法**：

   - 查看编辑器中是否有任何红色下划线或错误指示。
   - 在继续之前纠正任何问题。

3. **记录更改**：

   - 如果使用版本控制，请使用适当的消息提交更新。
     - **示例**：       ```
       git commit -am "插入捐赠设备序列号(DSN)到配置中" ```

---

## **7. Vivado项目设置和定制**

在将固件文件更新为反映捐赠设备的配置后，下一步是将这些更改集成到Vivado项目中。这涉及生成项目文件、定制IP核以及为综合和实现准备设计。

### **7.1 生成Vivado项目文件**

Vivado使用Tcl脚本自动化项目创建和配置。通过运行这些脚本，您可以确保所有设置根据您的FPGA设备正确应用。

#### **7.1.1 打开Vivado**

从Vivado的新会话开始，确保以前的设置或项目不会干扰您当前的工作。

**步骤**：

1. **启动Vivado**：

   - 在开始菜单或桌面上找到Vivado应用程序。
   - 点击打开它。

2. **选择正确的版本**：

   - 如果安装了多个版本，请确保使用与您的FPGA兼容的版本（例如，Vivado 2020.1）。

3. **等待启动屏幕**：

   - 允许Vivado完全初始化后再继续。

#### **7.1.2 访问Tcl控制台**

Tcl控制台允许您直接执行脚本和命令。

**步骤**：

1. **打开Tcl控制台**：

   - 在Vivado界面中，转到菜单栏。
   - 点击**窗口** > **Tcl控制台**。
   - Tcl控制台将出现在窗口底部。

2. **调整控制台大小（可选）**：

   - 拖动控制台的顶部边框以调整其大小以获得更好的可见性。

3. **清除以前的命令**：

   - 如果有任何命令存在，您可以清除它们以获得一个干净的开始。

#### **7.1.3 导航到项目目录**

确保Tcl控制台指向项目脚本所在的正确目录。

**对于Squirrel DMA (35T)**：

**路径**：

- 您的项目目录，通常为：  ```
  C:/Users/YourUsername/Documents/pcileech-fpga/pcileech-wifi-main/ ```

**步骤**：

1. **设置工作目录**：

   - 在Tcl控制台中输入：     ```tcl
     cd C:/Users/YourUsername/Documents/pcileech-fpga/pcileech-wifi-main/```
     - 用系统上的实际位置替换路径。

2. **验证目录更改**：

   - 在Tcl控制台中输入`pwd`。
   - 控制台应显示当前目录，确认更改。

#### **7.1.4 生成Vivado项目**

运行适当的Tcl脚本将设置项目的所有必要配置。

**步骤**：

1. **运行Tcl脚本**：

   - 对于**Squirrel (35T)**：     ```tcl
     source vivado_generate_project_squirrel.tcl -notrace```
   - 对于**Enigma-X1 (75T)**：     ```tcl
     source vivado_generate_project_enigma_x1.tcl -notrace```
   - 对于**ZDMA (100T)**：     ```tcl
     source vivado_generate_project_100t.tcl -notrace```

2. **等待脚本完成**：

   - 脚本将执行多个命令：
     - 创建项目。
     - 添加源文件。
     - 配置项目设置。
   - 监控Tcl控制台的进度消息。
   - 解决可能出现的任何错误，例如缺少文件或路径不正确。

3. **确认项目生成**：

   - 完成后，控制台将指示项目已创建。
   - 项目文件（`.xpr`和相关目录）将存在于项目目录中。

#### **7.1.5 打开生成的项目**

现在项目已生成，您可以在Vivado中打开它以进行进一步定制。

**步骤**：

1. **打开项目**：

   - 在Vivado中，点击**文件** > **打开项目**。
   - 导航到您的项目目录。

2. **选择项目文件**：

   - 对于**Squirrel**：     ```
     pcileech_squirrel_top.xpr ```
   - 点击`.xpr`文件以选择它。

3. **点击打开**：

   - Vivado将加载项目，显示设计层次结构和源文件。

4. **验证项目内容**：

   - 在**项目管理器**窗口中，确保列出了所有源文件。
   - 检查打开时是否有任何警告或错误。

### **7.2 修改IP块**

PCIe IP核是必须配置以匹配捐赠设备规格的关键组件。定制IP核可确保FPGA在PCIe协议级别上与捐赠硬件相同。

#### **7.2.1 访问PCIe IP核**

PCIe IP核是您Vivado项目中实例化的IP块。

**步骤**：

1. **找到PCIe IP核**：

   - 在**源**窗格中，确保选择了**层次结构**选项卡。
   - 展开设计层次结构以找到PCIe IP核。
     - 通常命名为`pcie_7x_0.xci`或类似名称。

2. **打开IP定制窗口**：

   - 右键单击`pcie_7x_0.xci`。
   - 从上下文菜单中选择**定制IP**。
   - **IP配置**窗口将打开。

3. **等待IP设置加载**：

   - IP定制界面可能需要几分钟初始化。
   - 确保所有选项和选项卡完全加载后再继续。

#### **7.2.2 定制设备ID和BARs**

在IP核中配置设备标识符对于主机系统的正确枚举至关重要。

**步骤**：

1. **导航到设备和供应商标识符**：

   - 在IP定制窗口中，选择**设备和供应商标识符**选项卡或部分。

2. **输入设备ID**：

   - 找到标记为**设备ID**的字段。
   - 输入捐赠设备的设备ID（例如，`0x1234`）。

3. **输入供应商ID**：

   - 找到**供应商ID**字段。
   - 输入捐赠设备的供应商ID（例如，`0xABCD`）。

4. **输入子系统ID和子系统供应商ID**：

   - 输入**子系统ID**（例如，`0x5678`）。
   - 输入**子系统供应商ID**（例如，`0x9ABC`）。

5. **设置修订ID**：

   - 输入**修订ID**（例如，`0x01`）。

6. **设置类代码**：

   - 输入**类代码**（例如，`0x020000`用于以太网控制器）。

7. **配置其他标识符（如果可用）**：

   - 一些IP核允许设置**编程接口**、**设备功能**等。
   - 根据需要将这些设置与捐赠设备匹配。

#### **7.2.3 配置BAR大小**

BARs定义设备如何将其内部内存和寄存器映射到主机系统。

**步骤**：

1. **导航到基址寄存器(BARs)**：

   - 在IP定制窗口中，选择**BARs**选项卡或部分。

2. **配置每个BAR**：

   - 对于**BAR0**到**BAR5**，根据捐赠设备设置以下参数：
     - **启用BAR**：选中或取消选中以匹配捐赠设备。
     - **BAR大小**：从下拉菜单中选择大小（例如，**256 MB**，**64 KB**）。
     - **BAR类型**：
       - **内存（32位寻址）**
       - **内存（64位寻址）**
       - **I/O**
     - **可预取**：如果捐赠设备的BAR可预取，则选中。

3. **示例配置**：

   - **BAR0**：
     - 启用
     - 大小：**256 MB**
     - 类型：**���存（64位）**
     - 可预取：**是**
   - **BAR1**：
     - 禁用（如果捐赠设备不使用BAR1）

4. **确保对齐和不重叠空间**：

   - 验证总内存映射不超过FPGA的能力。
   - 确保BAR大小符合PCIe规范要求。

5. **高级设置（如果适用）**：

   - 一些设备可能有特殊要求，例如扩展ROM BAR。
   - 如果需要，配置这些设置。

#### **7.2.4 完成IP定制**

在配置所有必要的设置后，您需要应用更改。

**步骤**：

1. **查看所有设置**：

   - 浏览IP定制窗口中的每个选项卡。
   - 确认所有条目与捐赠设备的规格匹配。

2. **应用更改**：

   - 点击**确定**或**生成**以应用设置。
   - 如果提示，确认您希望继续更改。

3. **重新生成IP核**：

   - Vivado将重新生成IP核以反映新配置。
   - 监控**消息**窗格以查看是否有任何错误或警告。

4. **更新项目中的IP**：

   - 确保更新的IP核正确集成到您的项目中。
   - Vivado可能会提示更新IP依赖项；允许其进行更新。

#### **7.2.5 锁定IP核**

锁定IP核可防止在综合和实现期间进行意外更改。

**目的**：

- **防止覆盖**：确保您的手动配置得到保留。
- **保持一致性**：在构建过程中保持IP核处于已知状态。

**步骤**：

1. **打开Tcl控制台**：

   - 在Vivado中，如果尚未打开，请转到**窗口** > **Tcl控制台**。

2. **执行锁定命令**：

   - 输入以下命令：     ```tcl
     set_property -name {IP_LOCKED} -value true -objects [get_ips pcie_7x_0]```
   - 按**Enter**执行。

3. **验证锁定**：

   - 检查**消息**窗格以确认。
   - IP核现在应标记为已锁定。

4. **解锁（如果需要）**：

   - 要在将来进行进一步更改，可以解锁IP核：     ```tcl
     set_property -name {IP_LOCKED} -value false -objects [get_ips pcie_7x_0]```
   - 记得在更改后重新锁定。

5. **记录操作**：

   - 在您的项目文档中注明IP核已锁定。
   - 这有助于团队成员了解项目的配置状态。

---

## **第二部分：中级概念和实现**

---

## **8. 高级固件定制**

为了实现捐赠设备的精确仿真，需要进一步深入定制固件。这涉及对齐PCIe参数、调整基址寄存器(BARs)以及仿真电源管理和中断机制，以匹配捐赠设备的规格。这些步骤确保仿真设备与主机系统无缝交互，并在行为上与原始硬件相同。

### **8.1 为仿真配置PCIe参数**

准确的仿真要求您的FPGA设备的PCIe参数与捐赠设备的参数精确匹配。这包括链路速度、链路宽度、功能指针和最大有效载荷大小等设置。正确的配置确保与主机系统的兼容性以及与设备交互的驱动程序和应用程序的正确操作。

#### **8.1.1 匹配PCIe链路速度和宽度**

PCIe链路速度和宽度是决定设备数据吞吐量和性能的关键参数。与捐赠设备匹配这些设置对于准确仿真至关重要。

**步骤**：

1. **访问PCIe IP核设置**：

   - **打开您的Vivado项目**：
     - 启动Vivado并打开您之前创建或修改的项目。
     - 确保所有源文件已正确添加到项目中。

   - **找到PCIe IP核**：
     - 在**源**窗格中，展开层次结构以找到PCIe IP核实例，通常命名为`pcie_7x_0`。
     - 与IP核关联的文件通常是`pcie_7x_0.xci`。

   - **定制IP核**：
     - 右键单击`pcie_7x_0.xci`并选择**定制IP**。
     - **IP配置**窗口将打开，显示各种配置选项。

2. **设置最大链路速度**：

   - **导航到链路参数**：
     - 在IP定制窗口中，点击**链路参数**选项卡或部分。
     - 此部分包含与PCIe链路特性相关的设置。

   - **配置最大链路速度**：
     - 找到**最大链路速度**选项。
     - 将其设置为与捐赠设备的链路速度匹配。
       - **示例**：
         - 如果捐赠设备以**Gen2 (5.0 GT/s)**运行，选择**5.0 GT/s**。
         - 如果它以**Gen1 (2.5 GT/s)**或**Gen3 (8.0 GT/s)**运行，选择相应的选项。
     - **注意**：确保您的FPGA和物理硬件支持所选链路速度。

3. **设置链路宽度**：

   - **配置链路宽度**：
     - 在同一**链路参数**部分，找到**链路宽度**设置。
     - 将其设置为与捐赠设备的链路宽度匹配。
       - **示例**：
         - 如果捐赠设备使用**x4**链路，将**链路宽度**设置为**4**。
         - 选项通常包括**1**、**2**、**4**、**8**、**16**通道。
     - **注意**：物理连接器和FPGA必须支持所选链路宽度。

4. **保存并重新生成**：

   - **应用更改**：
     - 配置链路速度和宽度后，点击**确定**以应用更改。
     - Vivado可能会提示您由于更改而重新生成IP核。
     - 确认并允许重新生成过程完成。

   - **验证设置**：
     - 重新生成完成后，重新访问IP核设置以确保配置正确应用。
     - 检查**消息**窗口是否有任何警告或错误。

#### **8.1.2 设置功能指针**

PCIe配置空间中的功能指针指向各种功能结构，如MSI、电源管理等。正确设置这些指针可确保主机系统能够定位和利用设备的功能。

**步骤**：

1. **在固件中找到功能指针**：

   - **打开配置文件**：
     - 在Visual Studio Code中，打开`pcileech_pcie_cfg_a7.sv`文件，路径为：       ```
       pcileech-fpga/pcileech-wifi-main/src/pcileech_pcie_cfg_a7.sv ```

   - **了解功能指针**：
     - 功能指针是一个8位寄存器，指向PCIe配置空间中的第一个功能结构，通常从标准配置头之后开始。

2. **设置功能指针值**：

   - **找到`cfg_cap_pointer`的分配**：
     - 搜索代码中`cfg_cap_pointer`的定义行。       ```verilog
       cfg_cap_pointer <= 8'hXX; // 当前值```

   - **更新功能指针**：
     - 用捐赠设备的功能指针值替换`XX`。
       - **示例**：
         - 如果捐赠设备的功能指针是`0x60`，则更新为：           ```verilog
           cfg_cap_pointer <= 8'h60; // 更新为匹配捐赠设备```

   - **确保正确对齐**：
     - 功能结构必须在4字节边界上对齐。
     - 功能指针应指向配置空间内的有效偏移。

3. **保存更改**：

   - **保存配置文件**：
     - 进行更改后，点击**文件 > 保存**或按`Ctrl + S`。

   - **验证语法**：
     - 确保更改未引入语法错误。

   - **注释以提高清晰度**：
     - 添加注释以解释更改以供将来参考。       ```verilog
       cfg_cap_pointer <= 8'h60; // 设置为捐赠设备的功能指针，偏移0x60```

#### **8.1.3 调整最大有效载荷和读取请求大小**

这些参数定义单个PCIe事务中可以传输的最大数据量。与捐赠设备匹配这些设置可确保兼容性和最佳性能。

**步骤**：

1. **设置最大有效载荷大小**：

   - **访问设备功能**：
     - 在PCIe IP核定制窗口中，导航到**设备功能**或**功能**选项卡。

   - **配置支持的最大有效载荷大小**：
     - 找到**支持的最大有效载荷大小**设置。
     - 将其设置为捐赠设备支持的值。
       - **选项**：
         - **128字节**、**256字节**、**512字节**、**1024字节**、**2048字节**、**4096字节**。
       - **示例**：
         - 如果捐赠设备支持最大有效载荷大小为**256字节**，选择**256字节**。

2. **设置最大读取请求大小**：

   - **配置支持的最大读取请求大小**：
     - 在同一选项卡中，找到**支持的最大读取请求大小**设置。
     - 将其设置为匹配捐赠设备的能力。
       - **示例**：
         - 如果捐赠设备支持最大读取请求大小为**512字节**，选择**512字节**。

3. **调整固件参数**：

   - **打��`pcileech_pcie_cfg_a7.sv`**：
     - 确保配置文件在Visual Studio Code中打开。

   - **更新固件常量**：
     - 找到定义`max_payload_size_supported`和`max_read_request_size_supported`的行。       ```verilog
       max_payload_size_supported <= 3'bZZZ; // 当前值
       max_read_request_size_supported <= 3'bWWW; // 当前值```

   - **设置适当的值**：
     - 用大小的二进制表示替换`ZZZ`和`WWW`。
       - **映射**：
         - **128字节**：`3'b000`
         - **256字节**：`3'b001`
         - **512字节**：`3'b010`
         - **1024字节**：`3'b011`
         - **2048字节**：`3'b100`
         - **4096字节**：`3'b101`
       - **示例**：
         - 对于**256字节**有效载荷大小：           ```verilog
           max_payload_size_supported <= 3'b001; // 支持高达256字节```
         - 对于**512字节**读取请求大小：           ```verilog
           max_read_request_size_supported <= 3'b010; // 支持高达512字节```

4. **保存更改**：

   - **保存文件**：
     - 更新值后，保存文件。

   - **验证一致性**：
     - 确保固件中的值与PCIe IP核中配置的值匹配。

   - **添加注释**：
     - 记录更改以供将来参考。       ```verilog
       max_payload_size_supported <= 3'b001; // 256字节根据捐赠设备
       max_read_request_size_supported <= 3'b010; // 512字节根据捐赠设备```

### **8.2 调整BAR和内存映射**

基址寄存器(BARs)定义设备向主机公开的内存区域。正确配置BAR和内存映射对于准确仿真和设备驱动程序的正常操作至关重要。

#### **8.2.1 设置BAR大小**

配置BAR大小可确保设备在枚举期间请求正确的地址空间，并且主机正确映射这些区域。

**步骤**：

1. **访问BAR配置**：

   - **定制PCIe IP核**：
     - 在Vivado中，右键单击`pcie_7x_0.xci`并选择**定制IP**。

   - **导航到BARs选项卡**：
     - 在IP定制窗口中，点击**基址寄存器(BARs)**选项卡。

2. **配置BAR大小和类型**：

   - **匹配捐赠设备的BARs**：
     - 对于每个BAR（BAR0到BAR5），设置大小和类型以匹配捐赠设备。

   - **设置BAR大小**：
     - 从下拉菜单中为每个BAR选择适当的大小。
       - **示例**：
         - 如果**BAR0**是**64 KB**，将**BAR0大小**设置为**64 KB**。
         - 如果**BAR1**是**128 MB**，将**BAR1大小**设置为**128 MB**。

   - **设置BAR类型**：
     - 为每个BAR选择**32位**或**64位**寻址。
     - 指定BAR是**内存**还是**I/O**类型。
     - 根据捐赠设备设置**可预取**状态。

   - **启用或禁用BARs**：
     - 确保仅启用捐赠设备使用的BARs。

3. **更新BRAM配置**：

   - **调整BRAM IP核**：
     - 在`ip`目录中，找到与BARs对应的BRAM配置。
       - **文件**：         ```
         pcileech-fpga/pcileech-wifi-main/ip/bram_bar_zero4k.xci
         pcileech-fpga/pcileech-wifi-main/ip/bram_pcie_cfgspace.xci ```

   - **修改BRAM大小**：
     - 打开每个BRAM IP核并调整内存大小以匹配相应的BAR大小。
     - 确保总内存不超过FPGA的容量。

4. **保存并重新生成**：

   - **应用更改**：
     - 配置BARs和更新BRAM大小后，点击IP定制窗口中的**确定**。

   - **重新生成IP核**：
     - Vivado可能会提示您由于更改而重新生成IP核。
     - 允许重新生成完成。

   - **检查错误**：
     - 查看**消息**窗口是否有任何与BAR配置相关的警告或错误。

#### **8.2.2 在固件中定义BAR地址空间**

设置BAR大小和类型后，您需要定义固件如何处理对这些BAR的访问。

**步骤**：

1. **打开BAR控制器文件**：

   - **找到源文件**：
     - 在Visual Studio Code中，打开：       ```
       pcileech-fpga/pcileech-wifi-main/src/pcileech_tlps128_bar_controller.sv ```

2. **映射地址范围**：

   - **定义地址解码逻辑**：
     - 实现逻辑以检测何时访问BAR。       ```verilog
       always_comb begin
         if (bar_hit[0]) begin
           // 处理对BAR0的访问
         end else if (bar_hit[1]) begin
           // 处理对BAR1的访问
         end
         // 继续处理其他BARs
       end```

   - **实现BAR访问处理**：
     - 为每个BAR定义如何管理读写操作。
       - **示例**：         ```verilog
         if (bar_hit[0]) begin
           case (addr_offset)
             16'h0000: data_out <= reg0;
             16'h0004: data_out <= reg1;
             // 其他寄存器
             default: data_out <= 32'h0;
           endcase
         end```

3. **实现地址解码逻辑**：

   - **计算地址偏移**：
     - 使用传入地址计算BAR内的偏移。       ```verilog
       addr_offset = incoming_address - bar_base_address[0];```

   - **处理数据传输**：
     - 实现读写操作的逻辑。       ```verilog
       if (cfg_write) begin
         // 将数据写入适当的寄存器
       end else if (cfg_read) begin
         // 从适当的寄存器读取数据
       end```

4. **保存更改**：

   - **保存文件**：
     - 实现逻辑后，保存`pcileech_tlps128_bar_controller.sv`文件。

   - **验证功能**：
     - 确保逻辑正确处理所有可能的访问。

#### **8.2.3 处理多个BARs**

正确管理多个BARs对于暴露多个内存或I/O区域的设备至关重要。

**步骤**：

1. **为每个BAR实现逻辑**：

   - **分离逻辑块**：
     - 为了清晰起见，在控制器中为每个BAR创建单独的代码块。       ```verilog
       // BAR0处理
       if (bar_hit[0]) begin
         // BAR0特定逻辑
       end
       // BAR1处理
       if (bar_hit[1]) begin
         // BAR1特定逻辑
       end```

   - **定义寄存器和内存**：
     - 根据需要为每个BAR分配寄存器或内存块。

2. **确保不重叠的地址空间**：

   - **验证地址范围**：
     - 确认每个BAR的地址空间不重叠。
     - 将BAR大小对齐到2的幂边界，以符合PCIe规范。

   - **更新地址解码**：
     - 调整地址解码逻辑以考虑每个BAR的大小和基址。

3. **测试BAR访问**：

   - **仿真测试**：
     - 使用仿真工具测试对每个BAR的读写操作。
     - 验证读取或写入的正确数据。

   - **硬件测试**：
     - 编程FPGA后，使用主机上的软件工具访问每个BAR。
     - **示例**：
       - 在Linux上使用`lspci`检查BAR映射。
       - 编写执行内存映射I/O到BARs的测试程序。

### **8.3 仿真设备电源管理和中断**

仿真电源管理功能和实现中断对于需要与主机操作系统的电源和中断处理机制密切交互的设备至关重要。

#### **8.3.1 电源管理配置**

实现电源管理允许设备支持各种电源状态，有助于系统范围的电源效率和符合操作系统的期望。

**步骤**：

1. **在PCIe IP核中启用电源管理**：

   - **访问功能**：
     - 在PCIe IP核定制窗口中，选择**功能**选项卡。

   - **启用电源管理**：
     - 选中**电源管理**选项，以在设备的配置空间中包括该功能。

2. **设置支持的电源状态**：

   - **配置支持的状态**：
     - 指定设备支持的电源状态，例如：
       - **D0（完全开启）**
       - **D1、D2（中间状态）**
       - **D3hot、D3cold（低功耗状态）**
     - 将这些设置与捐赠设备的功能匹配。

3. **在固件中实现电源状态逻辑**：

   - **打开`pcileech_pcie_cfg_a7.sv`**：
     - 修改固件以处理电源状态转换。

   - **处理电源管理控制和状态寄存器(PMCSR)**：
     - 实现对PMCSR的读写访问。       ```verilog
       // PMCSR地址
       localparam PMCSR_ADDRESS = 12'h44; // 示例地址

       // PMCSR寄存器
       reg [15:0] pmcsr_reg;

       // 处理PMCSR写入
       always @(posedge clk) begin
         if (cfg_write && cfg_address == PMCSR_ADDRESS) begin
           pmcsr_reg <= cfg_writedata[15:0];
           // 根据pmcsr_reg[1:0]更新电源状态
         end
       end       ```

   - **管理电源状态影响**：
     - 实现逻辑以根据当前电源状态改变设备行为。

4. **保存更改**：

   - **保存固件文件**：
     - 确保所有修改已保存。

   - **验证功能**：
     - 通过仿真或硬件测试测试电源管理功能。

#### **8.3.2 MSI/MSI-X配置**

实现MSI/MSI-X允许设备使用基于消息的中断，这比传统的基于引脚的中断更高效且可扩展。

**步骤**：

1. **在PCIe IP核中启用MSI/MSI-X**：

   - **访问中断配置**：
     - 在PCIe IP核定制窗口中，导航到**中断**或**MSI/MSI-X**选项卡。

   - **选择中断类型**：
     - 根据捐赠设备选择**MSI**或**MSI-X**。

   - **配置支持的向量数量**：
     - 设置与捐赠设备匹配的中断向量数量。
       - **MSI**支持最多32个向量。
       - **MSI-X**支持最多2048个向量。

   - **启用功能**：
     - 确保MSI或MSI-X功能包括在设备的配置空间中。

2. **在固件中实现中断逻辑**：

   - **打开`pcileech_pcie_tlp_a7.sv`**：
     - 修改固件以处理中断生成。

   - **定义中断信号**：
     - 声明MSI/MSI-X请求的信号。       ```verilog
       reg msi_req;```

   - **实现中断生成逻辑**：
     - 定义触发中断的条件。       ```verilog
       // 示例中断条件
       wire interrupt_condition = /*条件逻辑*/;

       // 生成MSI中断
       always @(posedge clk) begin
         if (interrupt_condition) begin
           msi_req <= 1'b1;
         end else begin
           msi_req <= 1'b0;
         end
       end       ```

   - **连接到PCIe核**：
     - 确保`msi_req`信号正确连接到PCIe IP核的中断接口。

3. **保存更改**：

   - **保存固件文件**：
     - 实现中断逻辑后，保存文件。

   - **检查时序约束**：
     - 验证新逻辑未引入时序违规。

#### **8.3.3 实现中断处理逻辑**

定义何时以及如何生成中断对于设备与主机中断处理机制的交互至关重要。

**步骤**：

1. **定义中断条件**：

   - **识别触发事件**：
     - 确定应导致中断的特定事件。
       - **示例**：
         - 数据准备好进行处理。
         - 错误条件。
         - 任务完成。

   - **实现条件逻辑**：
     - 使用组合或顺序逻辑检测这些事件。

2. **创建中断生成模块**：

   - **模块化设计**：
     - 将中断逻辑实现为单独的模块，以提高清晰度和重用性。       ```verilog
       module interrupt_controller(
         input wire clk,
         input wire reset,
         input wire event_trigger,
         output reg msi_req
       );
         always @(posedge clk or posedge reset) begin
           if (reset) begin
             msi_req <= 1'b0;
           end else if (event_trigger) begin
             msi_req <= 1'b1;
           end else begin
             msi_req <= 1'b0;
           end
         end
       endmodule```

   - **与主固件集成**：
     - 实例化模块并将其连接到主固件逻辑。

3. **确保正确的时序和顺序**：

   - **遵循PCIe规范**：
     - 确保中断根据协议生成和清除。

   - **管理中断延迟**：
     - 优化逻辑以最小化事件发生和中断生成之间的延迟。

4. **测试中断传递**：

   - **仿真**：
     - 使用仿真工具验证中断生成的正确性。

   - **硬件测试**：
     - 编程FPGA并使用主机端软件确认中断的接收和处理。

   - **调试工具**：
     - 利用集成逻辑分析器(ILA)核实时监控信号。

5. **保存更改**：

   - **最终确定代码**：
     - 确保所有更改已保存并记录。

   - **审查和改进**：
     - 根据测试结果迭代设计。

---

## **10. 事务层包(TLP)仿真**

事务层包(TLP)是PCIe中通信的基本单位。准确的TLP仿真对于设备与主机系统的正确交互至关重要。

### **10.1 理解和捕获TLP**

#### **10.1.1 学习TLP结构**

- **组件**：

  - **头部**：包含字段如**事务层包类型(Type)**、**长度**、**请求者ID**、**标签**、**地址**等。
  - **数据有效载荷**：存在于内存写入和其他一些TLP中。
  - **CRC**：确保数据完整性。

- **理解TLP类型**：

  - **内存读取请求**
  - **内存读取完成**
  - **内存写入**
  - **配置读取/写入**
  - **供应商定义的消息**

#### **10.1.2 从捐赠设备捕获TLP**

- **步骤**：

  1. **设置PCIe协议分析器**：

     - 使用硬件工具如**Teledyne LeCroy PCIe分析器**。

  2. **捕获事务**：

     - 在捐赠设备正常操作期间监控并记录TLP。

  3. **分析捕获的TLP**：

     - 使用分析器的软件解剖TLP并理解其结构和顺序。

#### **10.1.3 记录关键TLP事务**

- **步骤**：

  1. **识别关键事务**：

     - 关注设备初始化、配置、数据传输和错误处理所必需的TLP。

  2. **创建详细文档**：

     - 对于每个关键TLP，记录字段值、顺序和发送条件。

  3. **理解时序和顺序**：

     - 注意TLP之间的时序和所需的响应时间。

### **10.2 为特定操作制作自定义TLP**

#### **10.2.1 在固件中实现TLP处理**

- **要修改的文件**：

  - `pcileech_pcie_tlp_a7.sv`    ```
    pcileech-wifi-main/src/pcileech_pcie_tlp_a7.sv ```

- **步骤**：

  1. **创建TLP生成函数**：

     - 在`pcileech_pcie_tlp_a7.sv`中，编写函数以组装具有所需头部和有效载荷的TLP。
     - **示例**：       ```verilog
       function automatic [127:0] generate_tlp;
         input [15:0] requester_id;
         input [7:0] tag;
         input [7:0] length;
         input [31:0] address;
         input [31:0] data;
         begin
           generate_tlp = { /* TLP头部和有效载荷 */ };
         end
       endfunction```

  2. **处理TLP接收**：

     - 实现逻辑以解析传入的TLP并提取必要信息。
     - 使用状态机管理不同的TLP类型。

  3. **确保合规性**：

     - 验证TLP符合PCIe规范的格式和时序。

  4. **实现完成处理**：

     - 对于内存读取请求，生成适当的完成TLP。

  5. **保存更改**：

     - 实现更改后，保存文件。

#### **10.2.2 处理不同的TLP类型**

- **内存读取请求**：

  - **实现**：

    - 解析请求头部。
    - 从适当的内存位置获取数据。
    - 组装并发送带有数据的完成TLP。

- **内存写入请求**：

  - **实现**：

    - 接收TLP并提取数据有效载荷。
    - 将数据写入指定的内存位置。

- **配置读取/写入请求**：

  - **实现**：

    - 访问配置空间寄存器。
    - 对于读取，返回请求的数据。
    - 对于写入，更新寄存器值。

- **供应商定义的消息**：

  - **实现**：

    - 根据捐赠设备的协议实现解析和响应逻辑。

#### **10.2.3 验证TLP时序和顺序**

- **步骤**：

  1. **使用仿真工具**：

     - 使用测试平台仿真固件以验证TLP处理。

  2. **使用ILA监控**：

     - 插入ILA核以在硬件测试期间捕获TLP相关信号。

  3. **检查时序约束**：

     - 确保TLP在PCIe标准允许的时序窗口内处理和响应。

  4. **合规性测试**：

     - 使用PCIe合规性工具验证对标准的遵从性。

  5. **保存更改**：

     - 在测试和验证后保存所有修改的文件。

---

## **第三部分：高级技术和优化**

---

## **11. 构建、烧录和测试**

在完成所有定制后，是时候构建固件，将其编程到FPGA上，并彻底测试以确保其正常功能。

### **11.1 综合与实现**

#### **11.1.1 运行综合**

综合将您的高级代码转换为门级表示。

- **步骤**：

  1. **开始综合**：

     - 在Vivado中，点击**流程导航器**中的**运行综合**。

  2. **监控进度**：

     - 注意任何警告或错误。
     - **常见警告**：
       - **未连接端口**：确保所有必要信号已连接。
       - **时序约束未满足**：可能需要调整约束。

  3. **查看综合报告**：

     - 检查**利用率摘要**以确保设计适合FPGA。

#### **11.1.2 运行实现**

实现将综合设计映射到FPGA的资源。

- **步骤**：

  1. **开始实现**：

     - 综合成功后，点击**运行实现**。

  2. **分析时序报告**：

     - 确保所有时序约束都已满足。
     - **解决违规**：
       - 调整逻辑或约束以修复设置或保持时间违规。

  3. **验证布局**：

     - 检查关键组件是否已优化放置。

#### **11.1.3 生成比特流**

比特流是用于编程FPGA的二进制文件。

- **步骤**：

  1. **生成比特流**：

     - 点击**生成比特流**。

  2. **等待完成**：

     - 这可能需要一些时间，具体取决于设计的复杂性。

  3. **查看比特流生成日志**：

     - 确保生成过程中没有错误。

### **11.2 烧录比特流**

#### **11.2.1 连接FPGA设备**

- **步骤**：

  1. **准备硬件**：

     - 确保FPGA板已通电并通过JTAG连接。
     - 参考您的FPGA板手册以获取特定连接说明。

  2. **打开硬件管理器**：

     - 在Vivado中，导航到**流程导航器 > 编程和调试 > 打开硬件管理器**。

#### **11.2.2 编程FPGA**

- **步骤**：

  1. **连接到目标**：

     - 在硬件管理器中，点击**打开目标**并选择**自动连接**。
     - Vivado应检测到您的FPGA设备。

  2. **编程设备**：

     - 在硬件窗口中，右键单击您的FPGA设备并选择**编程设备**。
     - 选择生成的比特流文件（带有`.bit`扩展名）。
     - 点击**编程**将固件烧录到FPGA上。
     - 等待编程过程完成。

#### **11.2.3 验证编程**

- **步骤**：

  1. **检查状态**：

     - 确保编程完成无错误。
     - Vivado将在完成后显示成功消息。

  2. **观察LED或指示灯**：

     - 一些FPGA板有指示成功编程或活动状态的LED。

### **11.3 测试与验证**

#### **11.3.1 验证设备枚举**

- **Windows**：

  - **步骤**：
    1. **打开设备管理器**：
       - 按`Win + X`并选择**设备管理器**。
    2. **检查设备属性**：
       - 在适当的设备类别下查看（例如，**网络适配器**、**存储控制器**）。
       - 确认**设备ID**、**供应商ID**和其他标识符与捐赠设备匹配。

- **Linux**：

  - **步骤**：
    1. **使用lspci**：

       ```bash
       lspci -nn
       ```

    2. **验证设备列表**：
       - 检查仿真设备是否以正确的ID出现。
       - **示例输出**：

         ```
         03:00.0 Network controller [0280]: VendorID DeviceID
         ```

#### **11.3.2 测试设备功能**

- **步骤**：

  1. **安装必要的驱动程序**：

     - 如果需要，使用捐赠设备的驱动程序。
     - 按照制造商的说明安装。

  2. **执行功能测试**：

     - 运行与设备交互的应用程序。
     - 测试数据传输、配置和任何特殊功能。
     - **示例**：
       - 对于网络卡，执行ping测试或数据流。
       - 对于存储控制器，执行读/写操作。

  3. **监控系统行为**：

     - 检查系统稳定性和无错误。
     - 确保设备在各种工作负载下按预期运行。

#### **11.3.3 监控错误**

- **Windows**：

  - **步骤**：
    1. **检查事件查看器**：
       - 按`Win + X`并选择**事件查看器**。
       - 导航到**Windows日志 > 系统**。
    2. **查找与PCIe相关的错误**：
       - 搜索与PCIe或特定设备相关的警告或错误。

- **Linux**：

  - **步骤**：
    1. **检查dmesg日志**：

       ```bash
       dmesg | grep pci
       ```

    2. **识别问题**：
       - 查找指示PCIe通信或设备初始化问题的消息。

---

## **12. 高级调试技术**

当问题出现时，高级调试工具和技术可以帮助有效识别和解决问题。

### **12.1 使用Vivado的集成逻辑分析器**

集成逻辑分析器(ILA)允许实时监控FPGA内部信号。

#### **12.1.1 插入ILA核**

- **步骤**：

  1. **添加ILA IP核**：

     - 在Vivado中，打开**IP目录**。
     - 搜索**ILA**。
     - 在设计中实例化ILA核。

  2. **连接信号**：

     - 将要监控的信号连接到ILA探针。
     - **示例**：

       ```verilog
       ila_0 your_ila_instance (
         .clk(clk),
         .probe0(signal_to_monitor)
       );
       ```

     - **文件路径**：

       ```
       pcileech-wifi-main/src/pcileech_squirrel_top.sv
       ```

#### **12.1.2 配置触发条件**

- **步骤**：

  1. **设置探针属性**：

     - 定义每个探针的宽度以匹配信号宽度。

  2. **定义触发器**：

     - 在ILA仪表板中设置将触发数据捕获的条件。
     - **示例**：
       - 当检测到特定TLP类型或发生错误条件时触发。

#### **12.1.3 捕获和分析数据**

- **步骤**：

  1. **运行设计**：

     - 使用启用ILA的比特流编程FPGA。

  2. **打开硬件管理器**：

     - 在Vivado中访问ILA界面。

  3. **捕获数据**：

     - 启动ILA并等待触发条件。
     - 一旦触发，ILA将捕获波形数据。

  4. **分析波形**：

     - 使用波形查看器检查信号行为。
     - 识别异常或验证正确操作。

### **12.2 PCIe流量分析工具**

使用外部工具可以提供更深入的PCIe通信见解。

#### **12.2.1 PCIe协议分析器**

- **示例**：

  - **Teledyne LeCroy PCIe分析器**
  - **Keysight PCIe分析器**

- **步骤**：

  1. **设置分析器**：

     - 将分析器连接在主机系统和FPGA设备之间。

  2. **配置捕获设置**：

     - 定义要捕获的数据范围（例如，特定TLP类型，错误条件）。

  3. **捕获流量**：

     - 在设备操作期间记录PCIe事务。

  4. **分析结果**：

     - 检查TLP的合规性和正确性。
     - 识别任何协议违规或意外行为。

#### **12.2.2 基于软件的工具**

- **示例**：

  - **Wireshark与PCIe插件**
  - **ChipScope Pro**（用于Xilinx设备）

- **步骤**：

  1. **安装必要的插件**：

     - 确保工具中启用了PCIe支持。

  2. **监控PCIe总线**：

     - 捕获并显示PCIe数据包。

  3. **分析通信**：

     - 查找数据中的异常或错误。
     - 验证TLP的正确形成和顺序。

---

## **13. 故障排除**

本节提供了您在固件开发和测试过程中可能遇到的常见问题的解决方案。

### **13.1 设备检测问题**

**问题**：主机系统无法识别FPGA设备。

#### **可能的原因和解决方案**

1. **设备ID不正确**：
   - **原因**：固件中的ID与主机期望的不匹配。
   - **解决方案**：验证并更正固件中的**设备ID**、**供应商ID**和**子系统ID**。

2. **PCIe链路训练失败**：
   - **原因**：PCIe链路未建立。
   - **解决方案**：
     - 检查物理连接。
     - 确保**链路宽度**和**链路速度**配置正确。

3. **电源问题**：
   - **原因**：FPGA设备电源不足。
   - **解决方案**：验证电源连接和电压水平。

4. **固件错误**：
   - **原因**：固件中的错误阻止正常操作。
   - **解决方案**：检查代码中的语法错误或配置错误。

### **13.2 内存映射和BAR配置错误**

**问题**：设备的内存区域不可访问，或访问它们会导致系统错误。

#### **可能的原因和解决方案**

1. **BAR大小或类型不正确**：
   - **原因**：BAR配置与捐赠设备不匹配。
   - **解决方案**：在PCIe IP核和固件中调整BAR大小和类型。

2. **地址解码错误**：
   - **原因**：固件未能正确解释地址。
   - **解决方案**：调试固件中的地址解码逻辑。

3. **重叠的地址空间**：
   - **原因**：BARs重叠或与其他设备冲突。
   - **解决方案**：确保BAR地址正确对齐且不重叠。

### **13.3 DMA性能和TLP错误**

**问题**：DMA操作期间发生数据传输速率慢或错误，或TLP格式错误。

#### **可能的原因和解决方案**

1. **DMA逻辑效率低下**：
   - **原因**：DMA引擎未优化。
   - **解决方案**：实现缓冲和流水线以提高吞吐量。

2. **TLP格式错误**：
   - **原因**：TLP格式不正确。
   - **解决方案**：检查TLP组装代码以确保符合PCIe规范。

3. **流量控制问题**：
   - **原因**：流量控制信用处理不当。
   - **解决方案**：在固件中实现适当的流量控制机制。

---

## **14. 仿真精度和优化**

提高仿真精度可确保兼容性和性能，使仿真设备与捐赠设备无异。

### **14.1 精确时间仿真技术**

- **实现时序约束**：
  - 使用Vivado的时序约束匹配捐赠设备的时序特性。
  - 将约束应用于关键路径以确保满足所需的设置和保持时间。

- **使用时钟域交叉(CDC)技术**：
  - 正确处理跨不同时钟域的信号以防止亚稳态。
  - 根据需要使用同步器或FIFO。

- **仿真设备行为**：
  - 使用仿真工具建模和验证设备在不同条件下的行为。
  - 验证操作的时序和顺序与捐赠设备匹配。

### **14.2 动态响应系统调用**

- **实现状态机**：
  - 设计状态机，允许设备动态响应各种命令和状态。
  - 确保设备能够优雅地处理意外或无序请求。

- **监控和响应主机命令**：
  - 实现逻辑以解码和响应配置写入、供应商特定命令和其他交互。
  - 相应地更新内部寄存器和状态。

- **优化固件逻辑**：
  - 精简固件代码以减少延迟并提高响应能力。
  - 删除不必要的延迟或数据路径中的瓶颈。

---

## **15. 固件开发最佳实践**

遵循最佳实践有助于保持代码质量，促进协作，并确保项目的长期性。

### **15.1 持续测试和文档**

- **定期测试**：
  - 在每次重大更改后测试固件以尽早发现问题。
  - 使用测试平台和仿真验证逻辑在实现之前。

- **自动化测试**：
  - 实现自动化测试脚本以验证功能和性能。
  - 如果在团队环境中工作，使用持续集成工具。

- **维护文档**：
  - 记录设计，包括框图、状态机和接口。
  - 使用清晰的提交消息跟踪更改，并相应更新设计文档。

### **15.2 管理固件版本**

- **使用版本控制系统**：
  - 使用**Git**等系统管理代码版本并与他人协作。
  - 使用清晰的目录结构和命名约定组织代码库。

- **标记发布和里程碑**：
  - 标记固件的稳定版本以供将来参考。
  - 使用分支进行实验功能或重大更改。

- **备份和恢复**：
  - 定期备份您的工作以防止数据丢失。
  - 根据需要使用基于云的代码库或本地备份。

### **15.3 安全考虑**

- **安全编码实践**：
  - 遵循指南以防止常见漏洞，如缓冲区溢出或竞争条件。
  - 验证所有输入并优雅地处理错误。

- **数据保护**：
  - 确保设备处理的任何敏感数据都受到保护。
  - 如果需要，实施加密或访问控制。

- **合规性和道德**：
  - 了解与设备仿真相关的法律和道德考虑。
  - 确保遵守相关法律、法规和许可协议。

---

## **16. 附加资源**

增强您的理解并通过以下资源保持更新：

- **Xilinx文档**
  - [Xilinx用户指南](https://www.xilinx.com/support/documentation/user_guides.htm)
  - 包括有关Vivado、IP核和FPGA开发的详细信息。

- **PCI-SIG规范**
  - [PCI Express基础规范](https://pcisig.com/specifications)
  - PCIe标准的官方规范。

- **FPGA教程和论坛**
  - [FPGA4Fun](http://www.fpga4fun.com/)
  - [Stack Overflow FPGA问题](https://stackoverflow.com/questions/tagged/fpga)
  - 社区驱动的讨论和教程。

- **Verilog和VHDL资源**
  - [ASIC World Verilog教程](https://www.asic-world.com/verilog/index.html)
  - [VHDL参考指南](https://www.vhdlwhiz.com/vhdl-reference-guide/)

- **Vivado设计套件用户指南**
  - [Vivado用户指南](https://www.xilinx.com/support/documentation/sw_manuals/xilinx2020_1/ug893-vivado-ip-subsystems.pdf)

- **PCIe协议分析工具**
  - [Teledyne LeCroy](https://teledynelecroy.com/protocolanalyzer/)
  - 提供一系列PCIe分析工具。

---

## **18. 支持和贡献**

您的支持有助于维护和改进本指南及相关项目。

