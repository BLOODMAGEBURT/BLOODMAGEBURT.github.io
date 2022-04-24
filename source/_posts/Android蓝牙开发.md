---
title: Android蓝牙开发
date: 2022-04-24 09:55:00
Tags: bluetooth
---



#### 1.分类

- 传统蓝牙（适合设备间流式传输和通信，比如蓝牙耳机连接）
- 低功耗蓝牙（BLE bluetooth low energy）

#### 2.基础知识

- 蓝牙开发4大步骤

  - 设置蓝牙
  - 扫描配对设备或可用设备
  - 连接设备
  - 传输数据

  <!--more-->

- 蓝牙权限

  ```xml
  <manifest ... >
    <uses-permission android:name="android.permission.BLUETOOTH" />
    <uses-permission android:name="android.permission.BLUETOOTH_ADMIN" />
  
    <!-- If your app targets Android 9 or lower, you can declare
         ACCESS_COARSE_LOCATION instead. -->
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    ...
  </manifest>
  ```

- 使用配置文件

  配置文件定义了设备间蓝牙通信的无线接口规范，如，蓝牙耳机有 **[BluetoothHeadset](https://developer.android.com/reference/android/bluetooth/BluetoothHeadset?hl=zh-cn)** ，健康设备有 **[BluetoothHealthAppConfiguration](https://developer.android.com/reference/android/bluetooth/BluetoothHealthAppConfiguration?hl=zh-cn)** ，通过这些对象，可以方便的对设备进行交互操作

#### 3. 传统模式四大步骤详解

1. 设置蓝牙

   1. 确定手机是否支持蓝牙

      ```kotlin
      val bluetoothAdapter: BluetoothAdapter? = BluetoothAdapter.getDefaultAdapter()
      if (bluetoothAdapter == null) {
          // Device doesn't support Bluetooth
      }
      ```

   2. 确定蓝牙功能是否开启

      ```kotlin
      if (bluetoothAdapter?.isEnabled == false) {
          val enableBtIntent = Intent(BluetoothAdapter.ACTION_REQUEST_ENABLE)
          startActivityForResult(enableBtIntent, REQUEST_ENABLE_BT)
      }
      ```

   3. 可选项，监听蓝牙连接状态

      如果应用需要根据蓝牙的连接状态做出相应的操作，则可以监听 **[ACTION_STATE_CHANGED](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter?hl=zh-cn#ACTION_STATE_CHANGED)** 广播，每当蓝牙状态变更，系统会发出此广播

2. 查找设备

   > 请注意 **配对** 和 **连接** 之间的区别
   >
   > 配对：代表两个设备知道彼此的存在，且保留有验证身份的共享通信密钥
   >
   > 连接：代表两台设备共享一个RFCOMM通道，随时可以通信
   
   1. 查询已配对的设备
   
      ```kotlin
      val pairedDevices: Set<BluetoothDevice>? = bluetoothAdapter?.bondedDevices
      pairedDevices?.forEach { device ->
          val deviceName = device.name
          val deviceHardwareAddress = device.address // MAC address
      }
      ```
   
      
   
   2. 发现设备
   
      调用 开始扫描的方法 [startDiscovery()](https://developer.android.com/reference/android/bluetooth/BluetoothAdapter?hl=zh-cn#startDiscovery()) ，注册广播接收者，当发现设备时系统会发出广播，在广播接收者中获取设备信息（名称、 MAC地址）
   
      *特别注意： 当获取要要配对的设备后，一定要主动调用停止扫描的方法  `cancelDiscovery()`*
   
      ```kotlin
      override fun onCreate(savedInstanceState: Bundle?) {
          ...
      
          // Register for broadcasts when a device is discovered.
          val filter = IntentFilter(BluetoothDevice.ACTION_FOUND)
          registerReceiver(receiver, filter)
      }
      
      // Create a BroadcastReceiver for ACTION_FOUND.
      private val receiver = object : BroadcastReceiver() {
      
          override fun onReceive(context: Context, intent: Intent) {
              val action: String = intent.action
              when(action) {
                  BluetoothDevice.ACTION_FOUND -> {
                      // Discovery has found a device. Get the BluetoothDevice
                      // object and its info from the Intent.
                      val device: BluetoothDevice =
                              intent.getParcelableExtra(BluetoothDevice.EXTRA_DEVICE)
                      val deviceName = device.name
                      val deviceHardwareAddress = device.address // MAC address
                  }
              }
          }
      }
      
      override fun onDestroy() {
          super.onDestroy()
          ...
      
          // Don't forget to unregister the ACTION_FOUND receiver.
          unregisterReceiver(receiver)
      }
      ```
   
      如果希望自己的设备被别的蓝牙扫描到，可启用可检测性
   
      如果尚未在设备上启用蓝牙，则启用设备可检测性会自动启用蓝牙
   
      ```kotlin
      val discoverableIntent: Intent = Intent(BluetoothAdapter.ACTION_REQUEST_DISCOVERABLE).apply {
          putExtra(BluetoothAdapter.EXTRA_DISCOVERABLE_DURATION, 300)
      }
      startActivity(discoverableIntent)
      ```
   
   3. 连接设备
   
      1. 作为服务器连接
   
         > 当您需要连接两台设备时，其中一台设备必须保持开放的 `BluetoothServerSocket`，从而充当服务器。服务器套接字的用途是侦听传入的连接请求，并在接受请求后提供已连接的 `BluetoothSocket`。从 `BluetoothServerSocket` 获取 `BluetoothSocket` 后，您可以（并且应该）舍弃 `BluetoothServerSocket`，除非您的设备需要接受更多连接。
   
         1. 通过调用 `listenUsingRfcommWithServiceRecord()` 获取 `BluetoothServerSocket`
         2. 通过调用 `accept()` 开始侦听连接请求，获取 `BluetoothSocket`
         3. 如果您无需接受更多连接，请调用 `close()`
   
         ```kotlin
         private inner class AcceptThread : Thread() {
             
             private val mmServerSocket: BluetoothServerSocket? by lazy(LazyThreadSafetyMode.NONE) {
                 bluetoothAdapter?.listenUsingInsecureRfcommWithServiceRecord(NAME, MY_UUID)
             }
         
             override fun run() {
                 // Keep listening until exception occurs or a socket is returned.
                 var shouldLoop = true
                 while (shouldLoop) {
                     val socket: BluetoothSocket? = try {
                         mmServerSocket?.accept()
                     } catch (e: IOException) {
                         Log.e(TAG, "Socket's accept() method failed", e)
                         shouldLoop = false
                         null
                     }
                     socket?.also {
                         manageMyConnectedSocket(it)
                         mmServerSocket?.close()
                         shouldLoop = false
                     }
                 }
             }
         
             // Closes the connect socket and causes the thread to finish.
             fun cancel() {
                 try {
                     mmServerSocket?.close()
                 } catch (e: IOException) {
                     Log.e(TAG, "Could not close the connect socket", e)
                 }
             }
         }
         ```
   
      2. 作为客服端连接
   
         > 如果远程设备在开放服务器套接字上接受连接，则为了发起与此设备的连接，您必须首先获取表示该远程设备的 `BluetoothDevice` 对象。如要了解如何创建 `BluetoothDevice`，请参阅[查找设备](https://developer.android.com/guide/topics/connectivity/bluetooth?hl=zh-cn#FindingDevices)。然后，您必须使用 `BluetoothDevice` 来获取 `BluetoothSocket` 并发起连接。
   
         1. 使用 `BluetoothDevice`，通过调用 `createRfcommSocketToServiceRecord(UUID)` 获取 `BluetoothSocket`
         2. 通过调用 `connect()` 发起连接。请注意，此方法为阻塞调用。
   
         ```kotlin
         private inner class ConnectThread(device: BluetoothDevice) : Thread() {
         
             private val mmSocket: BluetoothSocket? by lazy(LazyThreadSafetyMode.NONE) {
                 device.createRfcommSocketToServiceRecord(MY_UUID)
             }
         
             public override fun run() {
                 // Cancel discovery because it otherwise slows down the connection.
                 bluetoothAdapter?.cancelDiscovery()
         
                 mmSocket?.use { socket ->
                     // Connect to the remote device through the socket. This call blocks
                     // until it succeeds or throws an exception.
                     socket.connect()
         
                     // The connection attempt succeeded. Perform work associated with
                     // the connection in a separate thread.
                     manageMyConnectedSocket(socket)
                 }
             }
         
             // Closes the client socket and causes the thread to finish.
             fun cancel() {
                 try {
                     mmSocket?.close()
                 } catch (e: IOException) {
                     Log.e(TAG, "Could not close the client socket", e)
                 }
             }
         }
         ```
   
   4. 管理连接
   
      > 成功连接多台设备后，每台设备都会有已连接的 `BluetoothSocket`。这一点非常有趣，因为这表示您可以在设备之间共享信息。
   
      1. 使用 `getInputStream()` 和 `getOutputStream()`，分别获取通过套接字处理数据传输的 `InputStream` 和 `OutputStream`。
      2. 使用 `read(byte[])` 和 `write(byte[])` 读取数据以及将其写入数据流
   
      ```kotlin
      private const val TAG = "MY_APP_DEBUG_TAG"
      
      // Defines several constants used when transmitting messages between the
      // service and the UI.
      const val MESSAGE_READ: Int = 0
      const val MESSAGE_WRITE: Int = 1
      const val MESSAGE_TOAST: Int = 2
      // ... (Add other message types here as needed.)
      
      class MyBluetoothService(
              // handler that gets info from Bluetooth service
              private val handler: Handler) {
      
          private inner class ConnectedThread(private val mmSocket: BluetoothSocket) : Thread() {
      
              private val mmInStream: InputStream = mmSocket.inputStream
              private val mmOutStream: OutputStream = mmSocket.outputStream
              private val mmBuffer: ByteArray = ByteArray(1024) // mmBuffer store for the stream
      
              override fun run() {
                  var numBytes: Int // bytes returned from read()
      
                  // Keep listening to the InputStream until an exception occurs.
                  while (true) {
                      // Read from the InputStream.
                      numBytes = try {
                          mmInStream.read(mmBuffer)
                      } catch (e: IOException) {
                          Log.d(TAG, "Input stream was disconnected", e)
                          break
                      }
      
                      // Send the obtained bytes to the UI activity.
                      val readMsg = handler.obtainMessage(
                              MESSAGE_READ, numBytes, -1,
                              mmBuffer)
                      readMsg.sendToTarget()
                  }
              }
      
              // Call this from the main activity to send data to the remote device.
              fun write(bytes: ByteArray) {
                  try {
                      mmOutStream.write(bytes)
                  } catch (e: IOException) {
                      Log.e(TAG, "Error occurred when sending data", e)
      
                      // Send a failure message back to the activity.
                      val writeErrorMsg = handler.obtainMessage(MESSAGE_TOAST)
                      val bundle = Bundle().apply {
                          putString("toast", "Couldn't send data to the other device")
                      }
                      writeErrorMsg.data = bundle
                      handler.sendMessage(writeErrorMsg)
                      return
                  }
      
                  // Share the sent message with the UI activity.
                  val writtenMsg = handler.obtainMessage(
                          MESSAGE_WRITE, -1, -1, mmBuffer)
                  writtenMsg.sendToTarget()
              }
      
              // Call this method from the main activity to shut down the connection.
              fun cancel() {
                  try {
                      mmSocket.close()
                  } catch (e: IOException) {
                      Log.e(TAG, "Could not close the connect socket", e)
                  }
              }
          }
      }
      ```



#### 4.BLE 类型开发



