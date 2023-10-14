# Summer23-SmartNIC  

A repository for summer internship designing a SmartNIC combining RISCV and NetFPGA
  
<ins>Internship Week 1 to Week 3</ins> 

1. NetFPGA-Plus  
  a.	Compile and build a well-known Open-NIC code  
  b.	Compile the build NetFPGA-plus, fix and broken settings  
  c.	Run the NetFPGA code, update the driver with Open-NIC updates

##     NetFPGA-Plus Part a:   
###    Compile and build a good-known Open-NIC code ([AMD OpenNIC Project](https://github.com/Xilinx/open-nic)):

#### OpenNIC Project Overview: 

On-chip FPGA resources are ideal regarding the demands of a SmartNIC(Smart Network Interface Card) nowadays. The OpenNIC project’s design is organized along the lines of the “shell” and “role,” concepts first introduced in Project Catapult by Microsoft Research.<sup>[[1]](#1)</sup>  The NIC shell contains the RTL sources and design files for targetting several of the AMD Xilinx Alveo boards featuring UltraScale+ FPGAs. The shell delivers NIC implementation supporting up to four PCI-e physical functions (PFs) and two 100Gbps Ethernet ports.<sup>[[2]](#2)</sup> For host PCIe PF interfaces, a standard set of hardware IP blocks are provided - the CMAC and QDMA subsystems respectively - and a Linux kernel driver for the NIC shell.

A block diagram of the OpenNIC system appears below:
![image](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/Open_NIC_Shell.png)
#### <p align="center">OpenNIC Block Diagram </p>

The CMAC Ethernet Subsystem implements a 100Gbps Ethernet MAC/PHY interface and moves data between the external Ethernet connection and FPGA’s programmable logic fabric via the H2C (Host-to-Card) and C2H (Card-to-Host) engines.     
The QDMA IP Subsystem executes a high-performance and configurable scatter-gather DMA machine to access FPGA’s integrated PCIe block. It supports multiple Physical or Virtual Functions with scalable queues and delivers excellent small-packet performance with low latency.<sup>[[2]](#2)</sup>

The **goal** of OpenNIC is to enable fast prototyping of hardware-accelerated network-attached applications (i.e. accelerates SmartNIC development) despite not being a fully-fledged SmartNIC solution.<sup>[[2]](#2)</sup>   
  
The OpenNIC shell’s CMAC and QDMA IP blocks have many configuration options. The OpenNIC design leaves various IP options and hence simplifies merging interfaces to these IP blocks with network-specific adapters and wrappers around them.

To select a specific OpenNIC design, it supports as many as four PCIe physical functions (PFs) and two 100Gbps Ethernet ports. The corresponding resource consumption depends on the configuration parameters including the number of CMAC blocks and the number of physical functions per QDMA block needed. Since the shell consumes relatively very few of the FPGA’s on-chip resources, it leaves plenty of programmable logic for other user functions. 

#### User Plugin Integration
OpenNIC shell provides 2 user logic boxes for instantiating custom RTL logic, one running at 250MHz and the other at 322MHz.  
The 250MHz box contains the unique SmartNIC datapath that bridges between the PCIe (QDMA) and Ethernet (CMAC) interfaces, while the 322MHz box is used to customize the CMAC subsystem for any specialized networking need.   

The clock-speed references in these user logic boxes do not define their roles, they are the natural clock rates at which these roles operate in the overall SmartNIC system: 250MHz for QDMA and 322MHz for CMAC. <sup>[[1]](#1)</sup> The interfaces between the QDMA subsystem and the 250MHz box use a variant of the AXI4-stream protocol, the 250MHz AXI4-stream. 

When selecting 2 CMAC ports, there are two CMAC subsystems with dedicated data and control interfaces. The CMAC subsystem connects to the 322MHz user logic box deploying a variant of the AXI4-stream protocol, the 322MHz AXI4-stream similarly. 

The NIC shell communicates with the user logic boxes through multiple AXI-Stream interfaces - AXI-lite interface running at 125MHz for register access, AXI4-stream interface running at either 250MHz or 322MHz for data path and Synchronous reset interface running at 125MHz.<sup>[[2]](#2)</sup>

### OpenNIC Implementation on Alveo U280 Board  
#### How to Build

To build the OpenNIC shell by running the Tcl script `build.tcl` under `script` in Vivado, firstly, it should be capable with the appropriate Vivado version from Vivado 2020.x, 2021.x or 2022.1. Take the example of 2020.2:`source /tools/Xilinx/Vivado/2020.2/settings64.sh`.
  
To implement the NIC shell, run the following command **under `script` dictionary** with a proper `Mode` choice(i.e. `tcl`, `batch` or `gui`):
```Bash
vivado -mode tcl -source build.tcl -tclargs  
-board au280 -overwrite 1 -impl 1 -post_impl 1 -num_phys_func 2 -num_cmac_port 2
```
  
The number of QDMA physical functions per QDMA subsystem and the number of CMAC ports are given in the block diagrams of the Shell configuration of OpenNIC code above.
  
#### Building for Simulation
For Behavioral simulation, Vivado GUI is recommended. Remember to choose the correct Vivado version and enter `start_gui` in the Ubuntu command line.  
*N.B: Servers in Oxford Computer Infrastructure Group may lack the configuration environment of Modelsim and Cocotb, hence third-party simulator had not been verified or tested during the internship*

Once opening the GUI, firstly add the corresponding `Directories` (i.e. `open-nic-shell` under the home directory).  
  
Then, in `Settings` of `Project Manager` under Flow Navigator, select `Project Device` to be the corresponding board (here we select `Xilinx Alveo U280 Accelerator Card` for our FPGA board). 
  
Afterwards, set the `Simulation Top Module Name` to `p2p_250mhz` in `Simulation` of Project Settings. Choose `Blackbox Model(stub file)` in the `Elaboration` part. 

If a third-party simulator is preferred in future releases, choose the corresponding Compiled Library Location wherever in the open-nic-shell directory but Simulator Executive Path should be the path of simulator installation (e.g. the directory containing the vsim executable for Modelsim). 

#### Programming onto FPGA board
After bitstream generation, FPGA can be programmed in two ways: 
1. Program the device directly. The FPGA configuration will be lost after reboot. 
2. Program and boot from the configuration memory.

Known Issues are discussed of boot server failure and CMAC license updates are [discussed here](https://github.com/MrJimbo2002/open-nic-shell/blob/80777515c83cc04d8497522669aa82dd914d1e08/README.md?plain=1#L528)

## NetFPGA-Plus Part b & c:   
### Compile the built NetFPGA-Plus, fix the broken settings and run the NetFPGA code with the most recent compatible driver for any Open-NIC update ([NetFPGA-Plus Project](https://github.com/NetFPGA/NetFPGA-PLUS)):

#### NetFPGA-Plus Project Overview:

The NetFPGA platform is used to build working prototypes of high-speed, hardware-accelerated networking systems. Reference designs included with the system include an IPv4 router, an Ethernet switch, and a four-port NIC. The open-source project intends to explore how to build Ethernet switches and  Internet Protocol(IP) routers using hardware rather than software, and to prototype advanced services for next-generation networks.<sup>[[3]](#3)</sup>

NetFPGA-Plus is available on multiple platforms, including Xilinx Alveo U200, U250, U280 and Xilinx VCU1525. These are FPGA-based PCIe boards with I/O capabilities for 100 Gbps operation, incorporating Xilinx’s Ultrascale+ FPGA. External memories include DDR4 and HBM, depending on the specific platform adopted.<sup>[[4]](#4)</sup>

Apart from the reprogrammable development board, there are reference sample courseware implementations presented - Reference NIC, Reference Switch and Reference Router subset projects. Considering potential utility in the data control plane, FPGA was regarded to incorporate a SmartNIC. Consequently, SmartNICs tend to accelerate data centre servers by offloading data encryption from the servers' CPU to the DPU (Data Processing Unit). Hence, Reference NIC was selected in the following exploration of the entire project. 

The modular structure for all reference projects in the NetFPGA platform is a pipeline at which each stage is a separate module. The block diagram of the pipeline is shown below. 

![image](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/NF_ReadME_Reference/NetFPGA_Block%20Diagram.png)

The CMAC subsystem on Xilinx open-nic-shell sends the received packet to the input arbiter module via nf_mac_attachment. The input arbiter has three **input** interfaces: two from the CMAC subsystem and one from the QDMA subsystem, both on open-nic-shell.  
Each input to the arbiter connects to an input queue, which is a small fall-through FIFO. The simple arbiter rotates between all the input queues in a round-robin manner, each time selecting a non-empty queue and writing one full packet from it to the next stage in the data path, which is the **output port lookup module**.<sup>[[5]](#5)</sup>

Then, the **output port lookup module** 'decides' which port a packet goes out of. After that decision is made, the packet is then handed to the output queues module. The lookup module implements a very basic lookup scheme, sending all packets from 100G ports to the CPU and vice versa, based on the source port indicated in the packet's header. <sup>[[5]](#5)</sup> Notice that although only one physical DMA module was stated in relevant Verilog scripts, there are 4 virtual DMA ports represented - Global Ports, Master AXI4-Stream Ports (interface to data path), Slave AXI4-Stream Ports (interface to RX queues) and Slave AXI-Lite Ports. The virtual DMA ports are distinguished by the SRC_PORT/DST_PORT field.

Once a packet arrives at the **output_queues** module, it already has a marked destination (provided on a side channel - The TUSER field). According to the destination, it enters a dedicated output queue. There are five such output queues: one for the targeted 4-lane 100G port and the remaining one for the DMA block. Note the hardware specifications of the Alveo Series FPGA board: one PCIe Gen4x8 with CCIX and two QSFP28 (100GbE). A packet may be dropped if its output queue is full or almost full.

When a packet reaches the head of its output queue, it is sent to the corresponding output port of either **CMAC subsystem** or **QDMA subsystem**, illustrated by nf_datapath on the block diagram. The output queues are arranged in an interleaved order: one physical Ethernet port, one DMA port etc. Hence, Even queues are therefore assigned to physical Ethernet ports, and odd queues are assigned to the virtual DMA ports.<sup>[[5]](#5)</sup>

### Testing

#### Prerequisite

Following **`Wiki`** of the GitHub Repository of NetFPGA-Plus, firstly set up the required installation environment in the `Getting Started Guide`. (already done in our group remote Server04)

Then, go to the `Building Your First Project` page, following the instructions step by step to generate corresponding bitstream files for any NetFPGA-Plus design ( i.e. select any one from Reference Router, Reference Switch or Reference NIC).

Looking into Linux commanding, type `vim tools/settings.sh`, then change the relevant setting of User-defined blocks between line 26 and line 29. Remember, `NFPLUS_FOLDER` refers to `~/NetFPGA-PLUS` , `NF_PROJECT_NAME` matches to  `reference_xxx`, `NF_DESIGN_DIR` corresponds to `${NFPLUS_FOLDER}/hw/projects/${NF_PROJECT_NAME}`. Whenever changing configuration setups, we need to trace back through `source tools/settings.sh` before proceeding with the compilation steps.
  
There is **no** need for us to fix or update any Open-NIC settings.

#### [Running Tests](https://github.com/NetFPGA/NetFPGA-PLUS/wiki/Hardware-Test)

For the sake of performance, the PCI Express and 100G physical layer interfaces are replaced with simulation-only core modules that stimulate AXI4-Stream slave ports, record data received from AXI4-Stream master ports, and provide a simple way of performing register reads and writes via an AXI4-Lite master.<sup>[[6]](#6)</sup>

The top-level file nf_test.py can be found inside NetFPGA-PLUS/tools/scripts. Tests are run by the `./nf_test.py` command followed by the arguments indicating if it is a hardware or simulation test and what is the specific test that we would like to run. So when running the test, test mode should be specified (sim or hw).

An end-user-supplied script, run.py, writes out two sets of AXI Stream files: the stimuli, and the expected results. The simulation is run, and the resulting log files are automatically reconciled with the expected output by nf_sim_reconcile_axi_logs.py.

For simulation test verification, under `~/NetFPGA-PLUS/tools/scripts`, type the corresponding major and minor arguments, taking `./nf_test.py sim --major loopback --minor minsize` as the example below:

![image](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/NF_ReadME_Reference/NF_Plus_SimulationTest_2.png)

N.B of Barrier Synchronization: Instead of specifying times for packet and register operations, the infrastructure uses a barrier statement for synchronization. The barrier blocks until all expected packets arrive, or it times out, causing the test to fail. This ensures that register operations occur at the correct time relative to the packet operations.<sup>[[6]](#6)</sup>

*P.S. Hardware Tests for any network design were left to be determined since OpenNIC Kernel was not connected, NetFPGA nfX interfaces and 100G NIC ethY ports were not found or linked for some unknown reason.*
  
  
<ins>**Internship Week 4 to Week 6**</ins>  
  
2. RISC-V  
  a.	Identify a suitable RISC-V code to use (preference: Alveo compatible)  
  b.	Compile and build code for Alveo U280  
  c.	Run the image on the FPGA

## RISC-V Part a: 

###  Identify a suitable RISC-V code to use (preference: Alveo compatible)  
#### RISC-V Overview

RISC-V is an **open-source** Instruction Set Architecture (ISA) based on the Reduced Instruction Set Computing (RISC) principles, maintained by RISC-V International. An ISA outlines the set of instructions a processor can understand and execute, bridging between hardware and software. It shapes the capabilities and performance of a processor. The choice of ISA influences how software is developed, and it has a lasting impact on a processor's efficiency, compatibility, and flexibility. 

RISC-V refers to a **r**educed **i**nstruction **s**et **c**omputer **a**rchitecture. The key architectural features of RISC-V include a load-store architecture, a fixed-length 32-bit instruction format, and a small number of general-purpose registers. RISC-V supports various integer instruction set extensions, such as RV32I (32-bit), RV64I (64-bit), and RV128I (128-bit), which define the base integer instruction set for different address space sizes.<sup>[[7]](#7)</sup>  

Thanks to its open nature, developers are encouraged to experiment and specialize in customizing the architecture to suit specific needs, resulting in a tailored solution. In terms of hardware support, several semiconductor companies have developed RISC-V processors and systems-on-chip (SoCs), such as the PULPino and the RISC-V BOOM Out-of-Order Superscalar processor. 

On the software side, the RISC-V ecosystem includes support for various operating systems, including Linux, FreeBSD, and real-time operating systems (RTOS) like FreeRTOS and Zephyr. <sup>[[7]](#7)</sup>  

RISC-V processors are suitable for a wide range of use cases, including low-power embedded systems, IoT devices, data centres, desktop systems, and high-performance computing applications. Specifically, RISC-V processors are also gaining traction in the data centre and high-performance computing (HPC) markets. The modularity of the RISC-V ISA allows efficient processing of complex workloads such as AI, machine learning, and big data analytics. 

Hence, the summer intern project intends to discover the prospective value of RISC-V processing core architecture in available resources of Alveo U280 data accelearing card. The 'CPU' core is likely to assign the card substantial computational capability and even serves as a data control plane as well in future verification. 

#### Discussion of various open-source projects

A very comprehensive comparison of various open-source RISC-V core projects was summarized in [Excel Spreadsheet.](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/RISCV_ReadME_Reference/Tabular_Summary.xlsx)

Ibex, OpenTitan and Berkeley Out-of-Order Machine (BOOM) are not directly concentrated onto FPGA targets and so were put aside at that moment.

Regarding Alveo compatible RISC-V code, corresponding four established projects were considered: Aquila SoC, Pulp, Pulpissimo and Vivado-Risc-V.

##### Aquila SoC Briefing 

[Aquila](https://github.com/eisl-nctu/aquila) is an open-source 32-bit RISC-V RV32IMA-compliant processor core for Xilinx FPGAs. The processor core is encapsulated as a reusable IP for Xilinx Vivado EDA tools. Currently, the microarchitecture of Aquila implements the classical five-stage pipeline RISC architecture with in-order execution. The default behaviour of the Aquila SoC, when configured into a Xilinx FPGA, is to execute a boot code stored in pre-initialized on-chip memory, which loads a binary executable (ELF file) from the Host PC through the UART connection. <sup>[[8]](#8)</sup>  

However, the Aquila configuration architecture currently does not hold any available 100G Ethernet or PCIe DMA interfaces. Moreover,  since its current specification layouts only cover Layer 1 data and instruction caches, plenty of resources were not deployed in newer releases of Xilinx FPGA accelerating cards. Therefore, Auila SoC was not considered to be explored further. 

##### PULP Briefing 

[PULP](https://github.com/pulp-platform/pulp) (Parallel Ultra-Low-Power) is an open-source multi-core computing platform for advanced microcontroller architecture. The PULP architecture includes either the RI5CY core or the zero-riscy one as the main core, autonomous Input/Output subsystem (uDMA), new memory subsystem, new peripherals and new SDK(Software Development Kit).

PULP equips the RISCY processor by default in bitstream generation as an in-order, single-issue core with 4 pipeline stages. It implements several ISA extensions: hardware loops, post-incrementing load and store instructions, bit-manipulation instructions, MAC operations, supports fixed-point operations, packed-SIMD instructions and the dot product. Additionally, PULP includes a new efficient I/O subsystem via a uDMA (micro-DMA) which communicates with the peripherals autonomously. The core just needs to program the uDMA and wait for it to handle the transfer. <sup>[[9]](#9)</sup>  

Although the RTL simulation platform had not been verified due to a lack of *vsim* setup, there is a direct deliverable generation of bitstream by the 'make build' command. However, the current version of the PULP has very limited testing on ModelSimor QuestaSim simulation. The Pulp platform only covers vcu118 and zcu102 boards for supported FPGA ports and no example scripts for ASIC synthesis. Since considerable work was estimated to make it compatible with the Alveo U280 board, the Pulp is left to be determined in the future. 

#### Pulpissimo discussion as prospective alternative RISC-V core

[Pulpissimo](https://github.com/pulp-platform/pulpissimo) is the microcontroller architecture of more recent PULP chips, which is a single-core platform. The PULPissimo architecture includes: either the RI5CY core or the Ibex one as the main core, autonomous Input/Output subsystem (uDMA), new memory subsystem, peripherals and SDK, and support for Hardware Processing Engines (HWPEs).<sup>[[10]](#10)</sup> 

Apart from sharing characteristics of RISCY and uDMA with Pulp, PULPissimo also supports the integration of hardware accelerators (Hardware Processing Engines) that share memory with the RI5CY core and are programmed on the memory map.  Specific IPs plug streaming accelerators into a PULPissimo or PULP system on the data and control plane.<sup>[[10]](#10)</sup>

PULPissimo is a Microcontroller provided in the SystemVerilog RTL description, provided Makefile targets for RTL simulation with Mentor Questa SIM and Cadence Xcelium(both were not verified yet). In terms of Operating systems, FreeRTOS allows us to build bare metal applications using the FreeRTOS kernel. Other than the FreeRTOS kernel, it is possible to build a bare-metal application, despite in that case driver support is not yet fully fleshed out.

For FPGA implementation for a supported target FPGA board, firstly generate the necessary synthesis including scripts by running the corresponding make target `make scripts`. This will parse the using the PULP bender dependency management tool to generate tcl scripts for all the IPs used in the PULPissimo project. 

Now switch to the fpga subdirectory and make a target to generate the bitstream: `cd fpga` and `make <board_target>`. 
N.B. If there is a fault message referring to a synthesis error on soc_interconnect.sv:277, check the Issues Tab [here.](https://github.com/pulp-platform/pulpissimo/issues/354) 

To execute the given application on the FPGA board, Pulpissimo's L2 memory should be loaded with the binary. To do so, there is OpenOCD in conjunction with GDB to communicate with the internal RISC-V debug module. Hence, Pulpissimo was considered to be a relatively mature and complete open-source project for a RISC-V core implemented onto the FPGA board. The main reason for not adopting Pulpssimo is that the already supported vcu108 is not the closest one compared to the au250 board from the Vivado-RISC-V project. It shall be attainable if we want to replace the RISC-V processor architecture for the prospective SmartNIC project version 2.0.

#### Vivado-RISC-V Research - Original Project Overview

[Vivado-RISC-V](https://github.com/eugene-tarassov/vivado-risc-v) repository contains an FPGA prototype of a fully functional RISC-V Linux server with networking, an online Linux package repository and daily package updates. It includes scripts and sources to generate RISC-V SoC HDL, Xilinx Vivado project, FPGA bitstream, and bootable SD card. The SD card contains RISC-V Open Source Supervisor Binary Interface (OpenSBI), U-Boot, Linux kernel and Debian root FS. Linux package repositories and regular updates are provided by Debian. <sup>[[11]](#11)</sup> However, since our SmartNIC hardware configuration currently had no bootable SD card interface, there is **no** need to change any setting of OpenSBI, U-Boot, Linux Kernel or Debian root FS. In other words, there is no device component found for `/dev/disk/by-path/*-usb*/` in the `mk-sd-card` file. 

The operating system of the Vivado-RISC-V prototype project runs on bare-metal or RTOS software. The modified bootrom is compatible with FPGA and extended Linux kernel and U-Boot device tree. For Linux boot, it will load and execute `boot.elf` invoking from OpenSBI and U-Boot. BSCAN block is used to support both RISC-V debugging and FPGA access over the same JTAG cable.

Rocket Chip is used as RISC-V implementation: UC Berkeley Architecture Research - Rocket Chip Generator. Rocket Chip is an open-source Sysem-on-Chip design *generator* that emits synthesizable RTL. It leverages the Chisel hardware construction language to compose a library of sophisticated generators for cores, caches and interconnects into an integrated SoC. Rocket Chip generates general-purpose processor cores that use the open RISC-V ISA and provide an in-order core generator known as Rocket. For heterogeneous SoC specialization, Rocket Chip supports the integration of custom accelerators in the form of instruction set extensions, coprocessors, or fully independent novel cores. In short, Rocket Chip is configured to include instruction and data caches, coherent interconnect, floating point, and all the relevant infrastructure. The rocket-chip generator invokes the Chisel compiler as a Scala program to emit RTL describing a complete SoC. Check Rocket Chip configuration classes from the `rocket.scala` directory. <sup>[[12]](#12)</sup> 

## RISC-V Part b & c: 
### Compile and build code for Alveo U280, and consequently run the image on the FPGA

The initial RISC-V SoC in this repo contains DDR, UART, SD and Ethernet controllers. DDR is provided by Vivado. UART, SD and Ethernet are open-source Verilog. The original Ethernet controller is based on the Verilog Ethernet Components project, which is a collection of Ethernet-related components for gigabit, 10G, and 25G packet processing.<sup>[[11]](#11)</sup>  The ethernet MAC addresses were reconfigured in Makefile to be 100G compatible with the corresponding NetFPGA-Plus ethernet module. 

The Makefile creates a Vivado project directory, e.g. `workspace/rocket64b2/vivado-u250-riscv`. It could be helpful to open the project in Vivado GUI, to see RISC-V SoC structure, make changes, add peripherals and rebuild the bitstream. The SoC occupies a portion of FPGA, leaving plenty of space for experiments and developing additional hardware for the next integration part.

As for details of the GitHub repository modification, firstly copy the u250 directory named `u280` under the `vivado-risc-v/board` folder. Then, change the corresponding Board_part and Xilinx Part in Makefile.inc and riscv-2021.2.tcl invoked from the Xilinx Board Store website as   
```
BOARD_PART  ?= xilinx.com:au280:part0:1.2  
XILINX_PART ?= xcu280-fsvh2892-2L-e
```

Then, in the same folder, there was an [error](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/RISCV_ReadME_Reference/VivadoRISCV_Version3_X1Y44.png) message pointing to module 'gtwizard_ultrascale_0', change the `CONFIG.CHANNEL_ENABLE` to `X0Y44`. Change the `C0_CLOCK_BOARD_INTERFACE` in instance `ddr4_0` with `sysclk0` from `riscv-2022.2.tcl`
  
Moreover, according to memory core [faulse](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/RISCV_ReadME_Reference/VivadoRISCV_Version3_SLR.png) and [invalid pins](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/RISCV_ReadME_Reference/VivadoRISCV_Version3_Invalid%20Pin.png), carefully compare the official xdc constraint files and seek for any difference between au280 and au250 board. Pay extra attention to qsfp28 and ethernet submodules across all the files in the `vivado-risc-v/board` folder except `bootrom.dts`. 

Taking the top module riscv_wrapper as the example, comment on the following scripts in the qsfp28 submodule definition from Line 42 to Line 54:
```
/*output wire         qsfp0_modsell,     
output wire         qsfp0_resetl,     
input  wire         qsfp0_modprsl,*/
input  wire         qsfp0_intl 
/*output wire         qsfp0_lpmode,  
output wire         qsfp0_refclk_reset,  
output wire [1:0]   qsfp0_fs*/
```

Similar procedures were conducted in the instantiation part from Line 171 to Line 184. Script Details of each file can be found under `/home/yuzhejin/vivado-riscv_version_final/board/u280/` either in the GitHub repository or from Oxford Computer Infrastructure Group Server04. 

<ins>Programming Outcome</ins>

The explicit Memory Interface Generator(MIG) from Alveo U280 DDR4 reflected the successful implementation of the **modified** Vivado-RISC-V project with Rocket Chip. The System Monitor remained at the standard mode. 

![image](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/RISCV_ReadME_Reference/vivado-riscv_MIG_gui.png)
  
Overall, the RISC-V core architecture could be smoothly integrated into our most updated Xilinx accelerator cards. Hence, it leads to an upcoming valuable exploration of how to make our NetFPGA-Plus networked system possess computational processing power like a CPU core. 

<ins>**Internship Week 7 to Week 8**</ins>

3.	Integration  
  a.	Compile NetFPGA and RISC-V on the same device, without integration of interfaces  
  b.<sup>*</sup>	Run the image on the FPGA

## Integration Part a: 

###  Compile NetFPGA and RISC-V on the same device, without integration of interfaces    

Before the actual integration part, it is advisable to use Vivado Block Design to add an IP. The IO block is the best place to mod the design for an additional peripheral device, like GPIO. Validate and synthesize the design, but don't build bitstream yet - device tree and RISC-V HDL need to be updated first.

Besides, Rocket Chip, the SoC architecture generator, contains Scala packages. These packages are all found within the `src/main/scala` directory including rocket, tile, tilelink, system and util. These Scala utilities for generator configuration were referenced in the next block diagram generation. 

##### Top Hierarchy Block Diagram 

To begin with, reviewing the initial RISC-V SoC repository, open-source Verilog scripts described  UART, SD and Ethernet controllers. Ethernet controller is regarded as a collection of gigabit, 10G and 25G packet processing. Its MAC address was expected to be reconfigured in Makefile to be 100G CMAC compatible with the related NetFPGA-Plus submodule. Specifically, the original `eth_mac_10g_fifo` and `eth_phy_10g` modules were targeted to be changed for successful merging. 

Similarly, examine the `Design Sources` file one by one in Hierarchy layouts from Vivado GUI Project Manager, seeking any difference between NetFPGA-Plus and Vivado-RISC-V project interfaces.  `qsfp0_phy_1_inst` and `riscv_qdma_0`were to be combined from RISC-V to NetFPGA-Plus interface.   
  
On the other hand, according to the modern CPU cache memory model, cache memory is built directly into the CPU to give the processor the fastest possible access to memory locations and provides nanosecond order speed access time to frequently referenced instructions and data, dividing into three levels (L1, L2 and L3 cache).

Then, taking the reference of [Untethered lowRISC RocketChip](https://www.cl.cam.ac.uk/~jrrk2/docs/untether-v0.2/overview/) and [Chipyard RocketChip](https://chipyard.readthedocs.io/en/stable/Generators/Rocket-Chip.html) high-level views, the design should contain multiple Rocket tiles each of which consists of a Rocket core and L1 instruction and data caches. The number of Rocket tiles is selected in Vivado-RISC-V `make CONFIG=rocket64bx`. All tiles share a unified and banked L2 cache and an I/O bus.   

A revised high hierarchy block diagram of our integrated SmartNIC project is shown below: 

![image](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/main/SmartNIC_Block_Diagram.png)  

## Integration Part b:
### Run the image on the FPGA   

To merge original NetFPGA-Plus and Vivado-RISC-V projects into our combined `smartnic_integration`, it follows the general principle for a Vivado engineering mission breakdown: 
  1. Module source files(v,vhdl)
  2. IP cores
  3. Top modules
  
After copying all the initial directories recursively to the `smartnic_integreation` folder, first and foremost, keep **invariant** for all high-level block configurations such as `qdma` and `nf_datapath` manually defined modules and all established invoked IP cores. 

Considering estimated variant module source files and top modules, replace all the repetitive functional modules into a Global variable and its instantiation signals in the form of 

```
   .NetFPGA(Global Signal)
   .RISC-V(Global Signal))
```

In detail, take the examples of top modules for both initial projects. To write up a unified new top module of smartnic_integration, comment definition of `/* QSFP28 #0 */` following wire definitions in RISC-V top module `riscv_wrapper`. The reverse modification is likely to work the same, hence varying the RISC-V top module to be compatible with NetFPGA-Plus will be discussed only for the remaining merging part. Delete repetitive qsfp0_tx and qsfp_rx ports definitions and relevant instantiations in `riscv_wrapper` as 

```
*/ output wire         qsfp0_tx1_p,
   output wire         qsfp0_tx1_n,
   input  wire         qsfp0_rx1_p,  
   input  wire         qsfp0_rx1_n,*/

ethernet_u280 ethernet_u280_i (
   ...
*/.qsfp0_tx1_p(qsfp0_tx1_p),
  .qsfp0_tx1_n(qsfp0_tx1_n),
  .qsfp0_rx1_p(qsfp0_rx1_p),
  .qsfp0_rx1_n(qsfp0_rx1_n),*/
   ...
```

Likewise, in the `ethernet-u280.v` module source, delete configurations of `Internal 156.25 MHz clock`  and again repetitive qsfp0_tx and qsfp_rx ports definitions and relevant instantiations:

```
qsfp0_gt1_inst(
   ...
*/.gtytxn_out(qsfp0_tx1_n),
   gtytxp_out(qsfp0_tx1_p),*/
   ...
```

Then, ` vim /home/yuzhejin/vivado-riscv_version_final/ethernet/ethernet.v`, change the AXI LITE Slave Interface, AXI Master Interface TX and AXI Master Interface RX to be compatible with datastream interface environment in `top.v`, `top_wrapper.sv` and `nf_attachment.sv` under `/home/yuzhejin/NetFPGA PLUS_final_version/hw/lib/common/hdl` folder. Keep unchanged in terms of unique deployed interface types(e.g. regs in `ethernet.v`), port information and functional logics if NetFPGA-Plus is not invoked and delete repetitive ones. Notice that the PHY MDIO Interface for MDIO(Management Data Input/Output) shall be invariant. 

Likewise, revise and edit corresponding source files in Vivado GUI hierarchy: `eth_mac_phy_10g_fifo.v` and `eth_phy_10g.v`, being consistent with `open_nic_shell.sv` under `/hw/lib/xilinx/xilinx_shell_v1_0_0/hdl/`. 

Then, integrate Makefiles in both projects to a joined Makefile for the smartnic_integration project. Completely change previous makefile location pointers. Remember some of the makefile settings were invoked from `settings.sh` from NetFPGA-Plus. Similar procedures apply to tcl files: `vivado.tcl`, `ethernet-u280.tcl`, `eth_mac_fifo.tcl` and `riscv-2020.2.tcl` for original Vivado-RISC-V, while `reference_nic.tcl`,  `reference_nic_defines.tcl` and `au280_timing.tcl` for initial NetFPGA-Plus project. 

Finally, only select one set of clock wizard configurations and constraint sets (sdc or xdc suffix files). 

#### Discussion of Expected Outcome for Integrated SmartNIC 

Due to the relatively limited internship duration length, system verification of complete Vivado engineering projects and further verification tests were left to be taken in the future. According to the designed high-level [block diagram](https://github.com/MrJimbo2002/Summer23-SmartNIC/blob/d291ca1c1d2442f8c6ed40dc79665e770341fc5b/README.md?plain=1#L260), the DRAM Memory on Alveo U280 connects between RISC-V processor core and NetFPGA-Plus data management plane. The 'CPU' core is likely to assign the card substantial computational capability and even serves as a data control plane in future verification. For example, the initial lookup module in NetFPGA-Plus could be upgraded to implement a more complicated lookup scheme not only based on the source port but also other parameters like destination port and packet size. The lookup table function demands a high degree of technical sophistication aided by the processor to comprehend a multitude of network protocols, ranging from the foundational TCP/IP suite to more specialized protocols. 
  
Provided Ping tests and Transaction results are expected to generate **same order** latency, while the Iperf test is estimated to witness **orders of growth** in bandwidth measurement. Besides, the SmartNIC could insert additional high-level algorithms in need. From the hardware level, it tends to increase the executive efficiency of every network node and consequently, the entire networking system, breaking through prospective bottlenecks of network latency and data transmission rates.  

Moreover, the SmartNIC enables running the network software processes and freeing up the valuable processing power of CPU for data centre servers via PCIe slot. It is likely to provide accelerated functionality to computers, such as network traffic engineering, data stream acceleration and deep packet inspection(DPI). The SmartNIC accelerates data centre servers by offloading data encryption from the servers' CPU to the DPU(Data Processing Unit). This leads to an increase in the level of security for data storage encryption and partially covers the work of DPI that evaluates the data transmission path and header of a packet. 

In conclusion, the processor core stands as the linchpin of a network system, orchestrating packet processing, ensuring security, optimizing traffic, and offering scalability and customization. Its multifaceted functions are pivotal in delivering the high performance, reliability, and versatility demanded by modern network infrastructures, making it an indispensable component in the ever-evolving realm of network technology.

Considering the future of RISC-V core on FPGA-based smart NICs, current market products like NVIDIA BlueField DPU, Intel Mount Evans IPU and AMD Pensando DPU have already shown their extraordinary values in network systems. Its diverse functions are pivotal in delivering exceptional and reliable modern network infrastructures in the ever-evolving realm of the semiconductor industry. A wide range of market potentials is still to be determined from the mentioned data transmission security and analytics to the Internet of Things and 5G in terms of containerization of network mobile components.

## References
<a id="1">[1]</a> 
Ivo Bolsens: 
OpenNIC empowers next-generation SmartNIC research  
https://www.linkedin.com/pulse/opennic-empowers-next-generation-smartnic-research-ivo-bolsens/  

<a id="2">[2]</a> 
Xilinx open-nic   
https://github.com/Xilinx/open-nic

<a id="3">[3]</a> 
NetFPGA Development System  
https://digilent.com/reference/netfpga/netfpga
  
<a id="4">[4]</a> 
NetFPGA-Plus Official Github Wiki  
https://github.com/NetFPGA/NetFPGA-PLUS/wiki

<a id="5">[5]</a> 
NetFPGA-Plus Official Github Wiki - Reference NIC  
https://github.com/NetFPGA/NetFPGA-PLUS/wiki/Reference-NIC

<a id="6">[6]</a> 
NetFPGA SUME Simulations Github   
https://github.com/NetFPGA/NetFPGA-SUME-public/wiki/NetFPGA-SUME-Simulations

<a id="7">[7]</a> 
RISC-V vs ARM: A Comprehensive Comparison of Processor Architectures   
https://www.wevolver.com/article/risc-v-vs-arm-a-comprehensive-comparison-of-processor-architectures

<a id="8">[8]</a> 
The Aquila SoC  
https://github.com/eisl-nctu/aquila

<a id="9">[9]</a> 
PULP  
https://github.com/pulp-platform/pulp

<a id="10">[10]</a> 
Pulpissimo  
https://github.com/pulp-platform/pulpissimo

<a id="11">[11]</a> 
Vivado-Risc-V  
https://github.com/eugene-tarassov/vivado-risc-v

<a id="12">[12]</a> 
The Rocket Chip Generator  
https://www2.eecs.berkeley.edu/Pubs/TechRpts/2016/EECS-2016-17.html
