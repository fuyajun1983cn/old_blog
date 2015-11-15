---
layout: post
title:  "Bluetooth Socket类分析"
date:   2015-11-15 16:46:51 +0800
categories: bt update
---
##Bluetooth Socket类分析  
Bluetooth Socket类的接口与TCP Socket类：`java.net.Socket`和java.net.ServerSocket类似，它也包含服务器端Socket和客户端Socket。  
* BluetoothServerSocket: 创建一个服务器端监听的Socket。并调用accept方法接受连接请求。当接受一个连接请求后，会返回一个`BluetoothSocket`对象来管理这个连接。  
* BluetoothSocket: 代表客户端Socket，该类初始化一个对外连接并管理此连接。通过connect方法向Server端发出连接请求。  

###Socket类型  
Android中，定义了三种Socket类型： 
```java
    /** Keep TYPE_ fields in sync with BluetoothSocket.cpp */
    /*package*/ static final int TYPE_RFCOMM = 1;
    /*package*/ static final int TYPE_SCO = 2;
    /*package*/ static final int TYPE_L2CAP = 3;
```

###创建服务器端Socket  
`BluetoothAdapter`提供了如下几个接口来创建BluetoothServerSocket： 
```java
// Create a listening, secure RFCOMM Bluetooth socket.
public BluetoothServerSocket listenUsingRfcommOn(int channel)
//Create a listening, secure RFCOMM Bluetooth socket with Service Record.
public BluetoothServerSocket listenUsingRfcommWithServiceRecord(String name, UUID uuid)
//Create a listening, insecure RFCOMM Bluetooth socket with Service Record.
public BluetoothServerSocket listenUsingInsecureRfcommWithServiceRecord(String name, UUID uuid)
//Create a listening, encrypted, RFCOMM Bluetooth socket with Service Record.
public BluetoothServerSocket listenUsingEncryptedRfcommWithServiceRecord
//Construct an unencrypted, unauthenticated, RFCOMM server socket.
public BluetoothServerSocket listenUsingInsecureRfcommOn(int port)
//Construct an encrypted, RFCOMM server socket.
public BluetoothServerSocket listenUsingEncryptedRfcommOn(int port)
//Construct a SCO server socket.
public static BluetoothServerSocket listenUsingScoOn()
```

###创建客户端Socket  
`BluetoothDevice`提供了如下几个接口来创建BluetoothSocket：  
/*
 * Create an RFCOMM {@link BluetoothSocket} ready to start a secure
 * outgoing connection to this remote device on given channel.
 */
public BluetoothSocket createRfcommSocket(int channel)

/**
 * Create an RFCOMM {@link BluetoothSocket} ready to start a secure
 * outgoing connection to this remote device using SDP lookup of uuid.
 */
 public BluetoothSocket createRfcommSocketToServiceRecord(UUID uuid)

/**
 * Create an RFCOMM {@link BluetoothSocket} socket ready to start an insecure
 * outgoing connection to this remote device using SDP lookup of uuid.
 */
public BluetoothSocket createInsecureRfcommSocketToServiceRecord(UUID uuid) 

/**
 * Construct an insecure RFCOMM socket ready to start an outgoing
 * connection.
 */
public BluetoothSocket createInsecureRfcommSocket(int port)

/**
 * Construct a SCO socket ready to start an outgoing connection.
 */
public BluetoothSocket createScoSocket() 


###Socket通信建立过程——服务器端  
服务器端开始监听，并准备接受连接：  
```java
//创建一个服务器端监听的Socket
  ...
  try {
        if (secure) {
            tmp = mAdapter.listenUsingRfcommWithServiceRecord(NAME_SECURE,
                MY_UUID_SECURE);
        } else {
            tmp = mAdapter.listenUsingInsecureRfcommWithServiceRecord(
                    NAME_INSECURE, MY_UUID_INSECURE);
        }
    } catch (IOException e) {
        Log.e(TAG, "Socket Type: " + mSocketType + "listen() failed", e);
    }
    mmServerSocket = tmp;

    ...

//开如接受连接  
  BluetoothSocket socket = null;

    // Listen to the server socket if we're not connected
    while (mState != STATE_CONNECTED) {
        try {
            // This is a blocking call and will only return on a
            // successful connection or an exception
            socket = mmServerSocket.accept();
        } catch (IOException e) {
            Log.e(TAG, "Socket Type: " + mSocketType + "accept() failed", e);
            break;
        }
...

//维护连接，并开始收发数据  
...
  // Get the BluetoothSocket input and output streams
    try {
        tmpIn = socket.getInputStream();
        tmpOut = socket.getOutputStream();
    } catch (IOException e) {
        Log.e(TAG, "temp sockets not created", e);
    }

    mmInStream = tmpIn;
    mmOutStream = tmpOut;
...
//之后，通过mmInStream或mmOutStream对象来收发数据。 
```

###Socket通信建立过程——客户端  
```java
//在指定的设备上创建一个Socket对象  
...
   // Get a BluetoothSocket for a connection with the
    // given BluetoothDevice
    try {
        if (secure) {
            tmp = device.createRfcommSocketToServiceRecord(
                    MY_UUID_SECURE);
        } else {
            tmp = device.createInsecureRfcommSocketToServiceRecord(
                    MY_UUID_INSECURE);
        }
    } catch (IOException e) {
        Log.e(TAG, "Socket Type: " + mSocketType + "create() failed", e);
    }
    mmSocket = tmp;
    ...

    //开始连接
    //连接前先关掉扫描
mAdapter.cancelDiscovery();

 try {
        // This is a blocking call and will only return on a
        // successful connection or an exception
        mmSocket.connect();
    } catch (IOException e) {
        // Close the socket
        try {
            mmSocket.close();
        } catch (IOException e2) {
            Log.e(TAG, "unable to close() " + mSocketType +
                    " socket during connection failure", e2);
        }
        connectionFailed();
        return;
    }
    ...


//维护连接，并开始收发数据  
...
  // Get the BluetoothSocket input and output streams
    try {
        tmpIn = socket.getInputStream();
        tmpOut = socket.getOutputStream();
    } catch (IOException e) {
        Log.e(TAG, "temp sockets not created", e);
    }

    mmInStream = tmpIn;
    mmOutStream = tmpOut;
...
//之后，通过mmInStream或mmOutStream对象来收发数据。 
```
##总结  
简要序列图如下：  
```sequence
BluetoothServerSocket->BluetoothServerSocket:BluetoothAdapter.listenUsingRfcommWithServiceRecord
BluetoothServerSocket->BluetoothServerSocket:accept
BluetoothSocket->BluetoothSocket: BluetoothDevice.createRfcommSocketToServiceRecor
BluetoothSocket->BluetoothServerSocket: connect
BluetoothServerSocket->BluetoothServerSocket: create a BluetoothSocket
BluetoothServerSocket->BluetoothSocket:InputSteam & OutputStream
BluetoothSocket->BluetoothServerSocket:InputSteam & OutputStream
BluetoothSocket->BluetoothSocket: close
BluetoothServerSocket->BluetoothServerSocket:close
```
