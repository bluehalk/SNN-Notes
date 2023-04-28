# TrueNorth: Design and Tool Flow of a 65 mW 1 Million Neuron Programmable Neurosynaptic Chip

我们开发了TrueNorth，一个65毫瓦的实时神经突触处理器，实现了一个非冯-诺伊曼的、低功耗、高度并行、可扩展和容错的架构。TrueNorth芯片有4096个突触核心，包含100万个数字神经元和2.56亿个突触，通过一个事件驱动的路由基础设施紧密互联.

a synapse is a structure that permits a neuron to pass an electrical or chemical signal to another neuron or to the target effector cell.

**TrueNorth原生Corelet语言、Corelet编程环境（CPE）[4]和一个不断扩大的可组合算法库[5]被用来开发TrueNorth架构的应用。**

### TrueNorth 的七条准则:

1. Minimizing Active Power：基于相关感知信息在时间和空间上的稀疏性，TrueNorth是一个事件驱动的架构，**使用异步电路**以及**使同步电路成为事件驱动的技术**。我们**消除了耗电的全局时钟网络，将内存和计算放在一起（尽量减少数据传输的距离）**，并**实现了稀疏的内存访问模式**。
2. Minimizing Static Power：TrueNorth芯片采用低功耗制造工艺
3. Maximizing Parallelism：为了通过4096个并行内核实现高性能，我们通过使用同步电路进行计算，并通过时分复用神经元计算，共享一个物理电路来计算256个神经元的状态，从而使内核面积最小化。
4. Real-Time Operation：TrueNorth架构采用分层通信，用高扇形横梁进行本地通信，用片上网络进行远距离通信，并采用全球系统同步以确保实时运行。
5. Scalability
6. Defect Tolerance
7. 硬件-软件一对一的等价性：全数字化的实现和确定性的全球系统同步使硬件和软件之间的这种契约成为可能。这些创新加速了芯片的测试和验证（包括前和后），并增强了可编程性，如相同的程序可以在芯片和模拟器上运行。

event-driven communication circuits

我们的主要贡献是一个新颖的异步-同步混合流程，将异步和同步设计方法中的元素互连起来，加上支持设计和验证的工具。该流程还包括以事件驱动的方式操作同步电路的技术。

## TRUENORTH ARCHITECTURE

TrueNorth架构与传统的冯-诺依曼架构有着本质的区别。与冯-诺依曼机器不同，我们不使用将指令映射到线性存储器中的顺序程序。TrueNorth架构实现了由连接它们的网络耦合起来的尖峰神经元。我们通过指定神经元的行为和它们之间的连接来对芯片进行编程。神经元通过发送尖峰来相互交流。交流的数据可以用尖峰的频率、时间和空间分布进行编码。

## Neurosynaptic Core（TrueNorth Core）——the basic building block of the architecture.

Just High Level

![Untitled](TrueNorth%20Design%20and%20Tool%20Flow%20of%20a%2065%20mW%201%20Millio%20b957dc187b4b431e99336120e670b0d1/Untitled.png)

- members：
    
    axons：轴突 垂直方向的，用来传递动作电位，连接上一轮神经元存在buffer中的数据
    
    synapse：突触 =black dot。在core中就是用synaptic crossbar来代表synapse的位置（有的地方有有的地方没有），即轴突和树突的交叉点。
    
    synaptic：突触的
    
    neurons：神经元
    
    dendrite：树突
    
- computation：
    
    首先是充电过程，可以看作是关于前一时刻电压和本时刻输入的一个函数，用以得到当前时刻的电压。然后是放电过程，我们判断当前的电压是否超出阈值，如果超出我们输出1，否则输出0。最后，如果进行了放电的话，我们会进行一个电位的重置，重置我们也有硬重置和软重置两种方法
    
    1. A neurosynaptic core receives **spikes** from the network and stores them in the input buffers.
    2. When a **1 kHz synchronization trigger signal** called **a tick arrives**, the spikes for the current tick are read from the input buffers, and distributed across the corresponding horizontal axons.
    3. 沿着axon方向传递的时候，如果某个位置有synapse（black dot），那么通过dendrite传递给对应的neuron
    4.  每个神经元整合其传入的尖峰并更新其膜电位。
    5. When all spikes are integrated in a neuron, the leak value is subtracted from the membrane potential.
    6. If the updated membrane potential exceeds the threshold, a spike is generated and sent into the network.

Specificly，具体硬件对应：

![Untitled](TrueNorth%20Design%20and%20Tool%20Flow%20of%20a%2065%20mW%201%20Millio%20b957dc187b4b431e99336120e670b0d1/Untitled%201.png)

- 组成介绍：
    1. The neuron block is the TrueNorth core's main computational element.
    2. router：The TrueNorth architecture supports complex interconnected neural networks by building an on-chip network of neurosynaptic cores.
    3. Once the **spike packet** arrives at the router of the destination core, it is forwarded to the core's scheduler.
    4. Each TrueNorth core has two SRAM blocks. One **resides in the scheduler** and functions as the **spike queue** storing incoming events for delivery at future ticks. The other is the primary core memory, storing all synaptic connectivity, neuron parameters, membrane potential, and axon/core targets **for all neurons in the core**.
    5. Now that we have introduced the majority of the core blocks, the missing piece is their **coordination**. To that end, **the token controller orchestrates which actions occur in a core during each tick**. At every tick, the token controller requests 256 axon states from the scheduler corresponding to the current tick. Next, it sequentially analyzes 256 neurons. For each neuron, the token controller reads a row of the core SRAM（是当前neuron对应的那一列axon？） (corresponding to current neuron's connectivity) and combines this data with the axon states from the scheduler. The token controller then transitions any updates into the neuron block, and finally sends any pending spike events to the router.
- “数据流”——在SNN中数据即——**spike packet**
    1. **router**： The router **communicates** with **its own core and the four neighboring routers** in the east, west, north, and south directions, creating a 2-D mesh network.     
    Each spike packet carries a relative dx, dy address of the destination core, a destination axon index, a destination tick at which the spike is to be integrated, and several flags for debugging purposes. （很形象，spike就是信息，”小蝌蚪找妈妈”）
    When a spike arrives at the router of the destination core, **the router passes it to the scheduler, shown as (A) in Fig. 5.** 
    对应着computation的第一步。
    2. The main purpose of the **scheduler** is to **store input spikes in a queue until the specified destination tick**, given in the spike packet
    The scheduler **stores** each spike as a binary value in an SRAM of 16 × 256-bit entries, corresponding to 16 ticks and 256 axons.（16和256怎么来的？
    当一个神经突触核心收到一个tick时，调度器会读取当前tick的所有尖峰Ai(t)，并将其发送到令牌控制器（B）
    对应第二步，scheduler就是个SRAM？是不是对应到的是buffer？存储中转站，等待specified tick然后再发给Controller
    3. The **token controller** **controls the sequence of computations** carried out by a neurosynaptic core. After receiving spikes from the scheduler, it processes（处理） 256 neurons one by one（串行？）using the **membrane potential equation**（即下面这个公式）**.**
        
        ![Untitled](TrueNorth%20Design%20and%20Tool%20Flow%20of%20a%2065%20mW%201%20Millio%20b957dc187b4b431e99336120e670b0d1/Untitled%202.png)
        
        对于第j个神经元，突触连接wi,j从核心SRAM发送到令牌控制器（C），而其余的神经元参数（D）和当前膜电位（E）被发送到神经元块。如果突触连接wi,j和输入尖峰Ai(t)的位数和是1，the **token controller sends a clock signal and an instruction (F)** （指令是由controller分发的，像神经元块发送指令）indicating **axon type Gi to the neuron block**, which in turn adjusts the membrane potential based on 
        
        ![Untitled](TrueNorth%20Design%20and%20Tool%20Flow%20of%20a%2065%20mW%201%20Millio%20b957dc187b4b431e99336120e670b0d1/Untitled%203.png)
        
    4. 最终数据汇集到了**Neuron block**上面，**换句话说，神经元块只有在两个条件满足时才会接收指令和时钟脉冲:有一个活跃的输入尖峰和相关的突触是活跃的。因此，神经元块以事件驱动的方式进行计算。**