### DS402（CIA402） 驱动器
### SDO
### PDO   
- 0x1800(CIA301协议)参数(TPDO)：周期发送/一次性发/同步发送 
- 0x1A00(CIA301协议)映射 发送什么数据  
### NMT 
### NMTerror
### SYNC
### CANID高4bits区分不同对象的功能码
### 0x2000厂家参数
### 0x6000设备标准配置 子索引参数

### canfestival初始化
```
   initTimer();   //定时器初始化
   CANObject->canHandle = CAN1;canInit(CAN1,CAN_BAUD_1M);	//can硬件初始化
   CANOpenMasterInit(CANObject);//加载配置文件
   setNodeId (CANObject, 1);//设置节点id1
   setState(CANObject,Initialisation);//初始化
   setState(CANObject,Pre_operational);//预操作
   setState(CANObject,Operational);//操作模式
   setNodeId (CANObject, 2);//设置节点id2
   setState(CANObject,Initialisation);//初始化
   setState(CANObject,Pre_operational);//预操作
   setState(CANObject,Operational);//操作模式
   setNodeId (CANObject, 3);//设置节点id3
   setState(CANObject,Initialisation);//初始化
   setState(CANObject,Pre_operational);//预操作
   setState(CANObject,Operational);//操作模式
```

### 周期性发送SYNC
#### 修改字典：
- index 0x1005 UNS32_Master_obj1005=0x40000080;//改成4就是周期性发送
- index 0x1006 UNS32_Master_obj1006=1000000;//0x1006是周期发送sync的周期单位us

### 发送PDO
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

### PDO与SDO差异
- PDO的CAN内容中的8个字节全部可以拿来传信息，PDO通过映射索引内容关联读写字典中其他索引对应的内容
- SDO的CAN内容中的4个字节已经拿来放读写功能码（1byte）索引（2bytes）和子索引（1byte）了，所以can信息最多4bytes，但可以直接读写字典索引中的内容


### 发送SDO
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

### 发SDO中CAN帧内容和代码解析
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
//0x80异常   sdo中止代码（CIA301协议）  emcy紧急报文 0x81 0x82 ...
//6060=0x6060索引
//00 子索引
unsigned long abortCode=0;
char sendData[4]={3,0,0,0};//数据内容
writeNetworkDict(SmasterObjdict_Data,0x01,0x6060,0x00,4,uint8,&sendData,0);
while(getwriteResultNetworkDict(&masterObjdict_Data,0x01,&abortCode)==SDO_UPLOAD_IN_PROGRESS){;}//正在发送SDO 等待发送完成 01：1号节点
```
