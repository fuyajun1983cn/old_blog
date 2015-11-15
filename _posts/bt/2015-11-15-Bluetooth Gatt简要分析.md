---
layout: post
title:  "Bluetooth Gatt简要剖析"
date:   2015-11-15 16:46:51 +0800
categories: bt update
---
#<center><font color='black'>Bluetooth Gatt Profile简要剖析</font></center>#
--------------------

##<font color='blue'>GATT概述</font>  
GATT设计为可供应用程序或其他Profile使用，使得客户端与服务器端能进行通信。
服务器包含了许多属性，GATT Profile定义了如何使用ATT协议来发现，读，写和获取这些属性的方法，以及配置属性的广播。

GATT可用于任何物理链路之上，使用ATT协议，L2CAP频道。
GATT 构建在ATT之上，它建立了ATT传输和保存数据的通用操作和框架。
GATT定义了两种角色：服务器和客户端。这两个角色不一定与GAP中定义的四个角色中的某个角色绑定。是由上层更高的Profile指定。
GATT和ATT可用于BR/EDR/LE。在LE中，是必须有的。  

GATT定义了两种角色：服务器和客户端
* 客户端发起请求，并接受响应。
* 服务器端接受请求，并发送响应，指示或通知。 
两种角色可动态变换，即一个设备可以同时充当客户端和服务器端角色。

![GATT服务器端与客户端交互示意图]({{site.url}}/images/bt/gatt_client_server.png)


如果一个设备声称支持GATT Profile，则必须实现其定义的一些能力。
GATT Profile主要处理如下一些场景： 
* 交换配置信息（Exchange）
* 发现设备的服务和特征。（Discovery）
* 读取一个特征值。（Read）
* 写入一个特征值。（Write）
* 通知一个特征值。（Notification）
* 指示一个特征值。（Indication）

##<font color='blue'>GATT数据格式</font>  
GATT传输的数据包含在ATT协议PDU（协议数据单元）中。
主要有如下一些形式的数据：***命令，请求，响应，指示，通知***。下面是PDU的结构图：
![PDU结构图]({{site.url}}/images/bt/gatt1.png)  

数据操作码（Opcode）包含上述五种类型，与之相应的数据存储在属性参数里。其中命令和请求是作用于服务器端设备属性集中的值。服务端的一个属性由如下四部分组成：***属性句柄***、***属性类型***、***属性值***以及***属性权限***。它的逻辑表示如下图所示：
![数据操作码格式]({{site.url}}/images/bt/gatt2.png)

其中，属性句柄是索引，取值范围：0x0001 ～ 0xFFFF，服务器上属性句柄值不会随时间改变。属性类型是一个UUID，代表某种类型。属性值代表该属性的实际数据，而属性权限则由上层Profile指定为读写权限。

GATT服务器存储通过ATT协议传输的数据并接受来自GATT客户端的ATT请求，命令和确认信息。 GATT同时指定了GATT服务器的数据格式。属性被格式化为服务集(Services)和特征集(Characteristics)。 服务包含一系列特征集。 特征集包含一个单一值以及一个或多个描述特征值的描述符。

##<font color='blue'>基于GATT的Profile层次结构</font>  
GATT Profile指定了Profile数据交换的结构。这个结构定义用于Profile的基本元素：服务和特征。包含在ATT的属性当中。 最顶层是Profile，一个Profile是由一个或多个服务组成的，这些服务是实现某个用例必需的。 一个服务则是由许多特征或其他服务的引用组成的。

GATT Profile指定了Profile数据交换的结构。这个结构定义用于Profile的基本元素：服务和特征。包含在ATT的属性当中。 最顶层是Profile，一个Profile是由一个或多个服务组成的，这些服务是实现某个用例必需的。 一个服务则是由许多特征或其他服务的引用组成的。


每个特征包含一个值和关于这个值的其他信息。 所有的服务和特征以及特征的组件（如值或描述符）包含了Profile数据，都存储在服务器的属性（Attribute）中。 服务有两种类型：主服务和次服务。主服务提供设备的主要功能，次服务提供设备的辅助功能。至少被设备上的一个主服务引用。

为保持兼容早期的客户，一个服务定义的后续修改只能增加新的引用服务或可选的特征，服务定义的行为也不能修改。 服务可用在一个或多个profile中。


**特征（Characteristic）**
	一个特征包含一个使用在服务中的值以及关于该值怎样访问的属性和配置信息，同时还包含该值如何显示或表示的信息。一个特征的定义包含一个特征声明，特征属性和一个值。它也可能包含描述符，描述符描述了对应的特征值在服务器中的值或允许的配置。

GATT Profile定义的特征如下：  
1.  Server Configuration  
2.  Primary Service Discovery  
3.  Relationship Discovery  
4.  Characteristic Discovery  
5.  Characteristic Descriptor Discovery  
6.  Reading a Characteristic Value  
7.  Writing a Characteristic Value  
8.  Notification of a Characteristic Value  
9.  Indication of a Characteristic Value  
10. Reading a Characteristic Descriptor  
11. Writing a Characteristic Descriptor  

示例图如下：
![GATT Profile层次结构图]({{site.url}}/images/bt/gatt3.png)

##<font color='blue'>Android Gatt相关类介绍</font>  
GATT相关的操作定义在`IBluetoothGatt`，应用程序通过BluetoothGatt类来使用GATT Profile。 所有具体的操作都会Route到GattService.BluetoothGattBinder类，并最终通过JNI接口调用到HAL接口。Java层类之间的关系如下：  
![Android Gatt相关类关系图]({{site.url}}/images/bt/GATT.png)

目前Android提供的接口只支持连接到作为BLE设备的Gatt Server端，即Android设备只能作为Gatt Client端， 要连接到一个作为GATT Server端的BLE设备，需要执行如下一些操作：  
1. 扫描BLE设备。  
```java
mHandler.postDelayed(new Runnable() {
    @Override
    public void run() {
        mScanning = false;
        mBluetoothAdapter.stopLeScan(mLeScanCallback);
        invalidateOptionsMenu();
    }
}, SCAN_PERIOD);

mScanning = true;
mBluetoothAdapter.startLeScan(mLeScanCallback);
```
需要提供相应的Callback函数获知扫描结果。

2. 根据扫描结果获知BLE设备的地址。  
```java
 final BluetoothDevice device = mBluetoothAdapter.getRemoteDevice(address);
```

3. 根据地址连接到BLE设备。  
```java
 mBluetoothGatt = device.connectGatt(this, false, mGattCallback);
```
需要提供相应的Callback函数获取设备连接的状态信息。

4. 发现服务。  
```java
//连接成功后，即可以调用：
mBluetoothGatt.discoverServices()
...
//必须先执行扫描服务，才能调用下面的接口
mBluetoothGatt.getServices()
```

5. 执行相关操作，如读BLE设备的特征值。  
```java
 mBluetoothGatt.readCharacteristic(characteristic);
```
