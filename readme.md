### 一、canopen通讯过程（不用协议栈直接控制）

##### 1.1开启节点

- 电机驱动器节点在上电后还是关闭状态，需要从主机发送特定的NMT start node指令，节点才会开启，节点开启后会自动按TxPDO的配置向总线发送can帧 NMT start node指令帧很简单，标准帧id为0x000，数据长度2字节数据内容为0x01 0N，其中N为对应节点的节点号
- id:0x000 data:01 0N

##### 1.2使能节点

- 节点开启后，驱动器还是处于闭锁状态，需要给从机对象字典6040的地址发送数据0x0006，使其切换到待机状态，待机状态下返回状态字变为0x0021 

- 1.2.1 pdo方式:

- canid:0x201, data:0x06 00（节点1的第一组RxPDO，对应内容为状态字设定,数据内容为 0x06 00的两个字节标准帧）

- 1.2.2 sdo方式

- canid：0x601，data:0x2b 40 60 00 06 00 00 00 

- 并且将收到从机发回的canid为581的回复。

##### 1.3启动

- 接下来可以将从机从待机状态切换到操作状态，操作状态下状态字将变成0x0027，方法是往状态字对应的0x6040地址写0x000f，我们往rxpdo4发送控制字和转矩。
- 发送0转矩可以让电机停下来，或者再次发送控制字0x0006，可以让驱动器直接进入待机状态。
- canid:501 data:0x0f 00 64 00 发送4字节标准帧(驱动器1电机转矩100旋转)

##### 1.4其他NMT管理canid:0x000 NMT节点状态切换命令

- 1）、canid:0x000  data:02 0N 可以关闭节点号为N的节点
- 2）、canid:0x000  data:81 0N 可以让节点号为N的节点重启 

##### 1.5 NMT 状态切换

- data中第一个字节：
- 1. 01h为启动命令（让节点进入操作状态）；
- 2. 02h为停止命令（让节点进入停止状态）；
- 3. 80h为进入预操作状态（让节点进入预操作状态）；
- 4. 81h为复位节点应用层（让节点的应用恢复初始状态）；
- 5. 82h为复位节点通讯
- data中第二个字节：
- 1. 第二个字节代表被控制的节点 Node-ID

##### 1.6 NMTN节点上线报文

- canid：700h+NodeID data：0x00 

##### 1.6 NMT节点状态与心跳报文

- canid：700h+NodeID 
- 1. data:04h 停止状态，
- 2. data:05h 操作状态，
- 3. data:7Fh 预操作状态

### 二、初始化（canfestival）

```
initTimer(); //定时器初始化
CANObject--canHandle = CAN1;
canInit(CAN1,CAN_BAUD_1M); //can硬件初始化
CANOpenMasterInit(CANObject);//加载配置文件
setNodeId (CANObject, 1);//设置节点id1
setState(CANObject,Initialisation);//初始化
setState(CANObject,Pre_operational);//预操作
setState(CANObject,Operational);//操作模式
setNodeId (CANObject, 2);//设置节点id2
setState(CANObject,Initialisation);//初始化
setState(CANObject,Pre_operational);//预操
setState(CANObject,Operational);//操作模式
setNodeId (CANObject, 3);//设置节点id3
setState(CANObject,Initialisation);//初始化
setState(CANObject,Pre_operational);//预操作
setState(CANObject,Operational);//操作模式
```

### 三、周期性发送SYNC（canfestival）

#### 修改字典：

- index 0x1005 UNS32_Master_obj1005=0x40000080;//改成4就是周期性发送
- index 0x1006 UNS32_Master_obj1006=1000000;//0x1006是周期发送sync的周期单位us

### 四、发送PDO（canfestival）

- PDO0索引（控制） 子索引       

- index 0x1800 

- index 0x1800 subindex 00h 入口数量        0x05 子索引的条数

- index 0x1800 subindex 01h 发送PDO标识符   0x181=0x180+NodeID=功能码+节点号

- index 0x1800 subindex 02h 传输类型        0xFF  周期发送

- index 0x1800 subindex 03h 禁止事件        0x00  两次发送之间留一定间隙

- index 0x1800 subindex 04h 事件定时器      0x03E0 单位毫秒

- PDO0索引（映射） 子索引

- index 0x1A00 

- index 0x1A00 subindex 00h 入口数量        0x02 子索引的条数

- index 0x1A00 subindex 01h  映射          0x7100 01 10  从0x7100索引的01子索引取16bits的数据

- index 0x1A00 subindex 02h  映射          0x7100 02 08  从0x7100索引的02子索引取08bits的数据

- 映射内容

- index 0x7100

- index 0x7100 subindex 00h  入口数量       0x02

- index 0x7100 subindex 01h                 0x2DFF

- index 0x7100 subindex 02h                 0xC3

- 组成CAN报文 0x181,2DFFC3

### 五、发送SDO（canfestival）

- SDO1索引       子索引
- index 0x1280   0x00    入口数目
- index 0x1280   0x01    COBID-C2S         0x00000601  //CAN-ID
- index 0x1280   0x02    COBID-S2C         0x00000581  //CAN-ID
- index 0x1280   0x03    NodeIDofServer    0x01        //1号节点
- SDO2索引       子索引
- index 0x1280   0x00    入口数目
- index 0x1280   0x01    COBID-C2S         0x00000602  //CAN-ID
- index 0x1280   0x02    COBID-S2C         0x00000582  //CAN-ID
- index 0x1280   0x03    NodeIDofServer    0x02        //2号节点

### 六、发SDO中CAN帧内容和代码解析（canfestival）

```
//发送sdo CAN帧内容：601，22 60 60 00 03 00 00 00 
//601=600+1发送给1好节点
//0x22为写入  
//0x2F写入1个  
//0x2B写入2个  
//0x27写入三个  
//0x23写入4个字节 
//0x40不计长度读取  
//0x60正常  
//0x80异常 sdo中止代码（CIA301协议） emcy紧急报文 0x81 0x82 ...
//6060=0x6060索引
//00 子索引
//03 速度模式
unsigned long abortCode=0;
char sendData[4]={3,0,0,0};//数据内容
writeNetworkDict(SmasterObjdict_Data,0x01,0x6060,0x00,4,uint8,&sendData,0);
while(getwriteResultNetworkDict(&masterObjdict_Data,0x01,&abortCode)==SDO_UPLOAD_IN_PROGRESS)
{
    ;
}//正在发送SDO 等待发送完成 01：1号节点
```

### 七、PDO与SDO差异（canfestival）

- PDO的CAN内容中的8个字节全部可以拿来传信息，PDO通过映射索引内容关联读写字典中其他索引对应的内容
- SDO的CAN内容中的4个字节已经拿来放读写功能码（1byte）索引（2bytes）和子索引（1byte）了，所以can信息最多4bytes，但可以直接读写字典索引中的内容
from https://www.bilibili.com/read/cv12636652
