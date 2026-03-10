# BLE + IoT Android 专项面试题

> 对应岗位：Android 安卓工程师（BLE/IoT 方向）
> 核心要求：5年+ Android，3年+ IoT，2年+ BLE，主导过完整 BLE 项目上线
> 更新日期：2026-03-05

---

# 模块一：BLE 核心基础

---

## Q1：BLE 完整通信流程是什么？GATT 协议结构如何理解？

**答：**

BLE 通信基于 GATT（Generic Attribute Profile）协议，数据组织结构如下：

```
设备（Peripheral，外设）
└── GATT Server
    ├── Service A（功能模块，UUID 标识）
    │   ├── Characteristic 1（数据点）
    │   │   ├── Value（实际数据）
    │   │   ├── Properties（READ / WRITE / NOTIFY / INDICATE）
    │   │   └── Descriptor（CCCD：Client Characteristic Configuration）
    │   └── Characteristic 2
    └── Service B
```

**完整通信流程：**

```kotlin
// ============================================================
// 完整 BLE 通信封装（生产可用级别）
// ============================================================

class BleManager(private val context: Context) {

    // BLE 连接状态
    enum class BleState {
        DISCONNECTED, SCANNING, CONNECTING, CONNECTED, DISCOVERING_SERVICES, READY
    }

    private val _state = MutableStateFlow(BleState.DISCONNECTED)
    val state: StateFlow<BleState> = _state.asStateFlow()

    // 接收数据的 SharedFlow（多个订阅者都能收到数据）
    private val _dataFlow = MutableSharedFlow<BleData>(replay = 0)
    val dataFlow: SharedFlow<BleData> = _dataFlow.asSharedFlow()

    private var gatt: BluetoothGatt? = null
    private val scope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    // ---- Step 1：扫描 ----
    fun startScan(targetName: String? = null, targetMac: String? = null) {
        _state.value = BleState.SCANNING

        val filters = buildList {
            if (targetName != null) add(
                ScanFilter.Builder().setDeviceName(targetName).build()
            )
            if (targetMac != null) add(
                ScanFilter.Builder().setDeviceAddress(targetMac).build()
            )
        }

        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_LATENCY)  // 低延迟快速扫描
            .setReportDelay(0)                                 // 立即上报，不缓冲
            .build()

        val scanner = BluetoothAdapter.getDefaultAdapter().bluetoothLeScanner
        scanner.startScan(filters.ifEmpty { null }, settings, scanCallback)

        // 超时自动停止扫描（10秒）
        scope.launch {
            delay(10_000)
            if (_state.value == BleState.SCANNING) {
                scanner.stopScan(scanCallback)
                _state.value = BleState.DISCONNECTED
            }
        }
    }

    private val scanCallback = object : ScanCallback() {
        override fun onScanResult(callbackType: Int, result: ScanResult) {
            BluetoothAdapter.getDefaultAdapter().bluetoothLeScanner.stopScan(this)
            connect(result.device)
        }
        override fun onScanFailed(errorCode: Int) {
            _state.value = BleState.DISCONNECTED
            // errorCode: 1=已在扫描, 2=应用注册失败, 3=内部错误, 4=功能不支持
        }
    }

    // ---- Step 2：连接 ----
    fun connect(device: BluetoothDevice) {
        _state.value = BleState.CONNECTING
        // autoConnect=false：直接连接，响应快；true：等设备进入范围再连（后台重连用）
        gatt = device.connectGatt(context, false, gattCallback, BluetoothDevice.TRANSPORT_LE)
    }

    // ---- Step 3：GATT 回调处理 ----
    private val gattCallback = object : BluetoothGattCallback() {

        override fun onConnectionStateChange(gatt: BluetoothGatt, status: Int, newState: Int) {
            when {
                newState == BluetoothProfile.STATE_CONNECTED && status == BluetoothGatt.GATT_SUCCESS -> {
                    _state.value = BleState.DISCOVERING_SERVICES
                    // 连接成功后必须调用 discoverServices 才能操作 Characteristic
                    gatt.discoverServices()
                }
                newState == BluetoothProfile.STATE_DISCONNECTED -> {
                    _state.value = BleState.DISCONNECTED
                    gatt.close()
                    // status != GATT_SUCCESS 时说明是异常断连，可触发重连
                    if (status != BluetoothGatt.GATT_SUCCESS) {
                        scheduleReconnect(gatt.device)
                    }
                }
            }
        }

        override fun onServicesDiscovered(gatt: BluetoothGatt, status: Int) {
            if (status != BluetoothGatt.GATT_SUCCESS) return
            _state.value = BleState.READY

            // 服务发现完成后，开启需要的 Characteristic 通知
            enableNotification(gatt, HEART_RATE_SERVICE_UUID, HEART_RATE_CHAR_UUID)
        }

        // 收到设备主动推送的数据（NOTIFY/INDICATE）
        override fun onCharacteristicChanged(gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic) {
            val rawBytes = characteristic.value
            scope.launch {
                _dataFlow.emit(BleData(characteristic.uuid, rawBytes))
            }
        }

        // 主动读数据的回调（gatt.readCharacteristic 后触发）
        override fun onCharacteristicRead(gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic, status: Int) {
            if (status == BluetoothGatt.GATT_SUCCESS) {
                scope.launch {
                    _dataFlow.emit(BleData(characteristic.uuid, characteristic.value))
                }
            }
        }

        // 写数据的回调（gatt.writeCharacteristic 后触发）
        override fun onCharacteristicWrite(gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic, status: Int) {
            val success = status == BluetoothGatt.GATT_SUCCESS
            // 通知调用方写入结果
        }

        // MTU 协商回调（requestMtu 后触发）
        override fun onMtuChanged(gatt: BluetoothGatt, mtu: Int, status: Int) {
            // MTU 协商成功，之后可以发送更大的数据包
            // 默认 MTU=23，有效载荷=20字节；协商后最大 MTU=517，有效载荷=514字节
        }
    }

    // ---- Step 4：开启通知 ----
    private fun enableNotification(gatt: BluetoothGatt, serviceUuid: UUID, charUuid: UUID) {
        val characteristic = gatt.getService(serviceUuid)?.getCharacteristic(charUuid) ?: return

        // 第一步：告诉本地 BLE 栈开启通知
        gatt.setCharacteristicNotification(characteristic, true)

        // 第二步：写 CCCD Descriptor，告诉设备"我要订阅通知"（缺这步设备不会推送）
        // CCCD UUID 是固定的：00002902-0000-1000-8000-00805f9b34fb
        val cccdUuid = UUID.fromString("00002902-0000-1000-8000-00805f9b34fb")
        val descriptor = characteristic.getDescriptor(cccdUuid)
        descriptor?.value = BluetoothGattDescriptor.ENABLE_NOTIFICATION_VALUE  // NOTIFY
        // descriptor?.value = BluetoothGattDescriptor.ENABLE_INDICATION_VALUE  // INDICATE（有确认回执）
        gatt.writeDescriptor(descriptor)
    }

    // ---- Step 5：断开 ----
    fun disconnect() {
        gatt?.disconnect()  // 先断开，等 onConnectionStateChange 回调后再 close
    }

    private fun cleanup() {
        gatt?.close()       // 释放 GATT 资源，之后 gatt 不可再用
        gatt = null
        _state.value = BleState.DISCONNECTED
    }
}

data class BleData(val uuid: UUID, val bytes: ByteArray)
```

---

## Q2：BLE 连接不稳定怎么排查和解决？

**答：**

这是 IoT 开发最核心的问题，分三层解决：**预防 → 检测 → 恢复**

```kotlin
// ============================================================
// 层1：预防断连
// ============================================================

// 预防1：心跳保活（防止系统或设备主动断连）
class BleHeartbeat(private val gatt: BluetoothGatt, private val charUuid: UUID) {
    private var heartbeatJob: Job? = null

    fun start(scope: CoroutineScope, intervalMs: Long = 30_000) {
        heartbeatJob = scope.launch {
            while (isActive) {
                delay(intervalMs)
                // 定期读取一个 Characteristic，保持连接活跃
                val service = gatt.services.firstOrNull()
                val char = service?.getCharacteristic(charUuid)
                if (char != null) {
                    gatt.readCharacteristic(char)
                }
            }
        }
    }

    fun stop() {
        heartbeatJob?.cancel()
    }
}

// 预防2：关闭省电优化（华为/小米/OPPO 等厂商有激进的省电策略）
// AndroidManifest.xml 中声明，引导用户手动关闭省电优化
// Settings → 应用 → 你的APP → 电池 → 不受限制


// ============================================================
// 层2：检测断连（监控连接质量）
// ============================================================

class BleConnectionMonitor {
    private var lastDataTime = 0L
    private var noDataJob: Job? = null

    // 每次收到数据更新时间戳
    fun onDataReceived() {
        lastDataTime = System.currentTimeMillis()
    }

    // 启动超时检测：超过 N 秒没有数据则判定连接异常
    fun startWatchdog(scope: CoroutineScope, timeoutMs: Long = 60_000, onTimeout: () -> Unit) {
        noDataJob = scope.launch {
            while (isActive) {
                delay(10_000)  // 每 10 秒检查一次
                val elapsed = System.currentTimeMillis() - lastDataTime
                if (elapsed > timeoutMs) {
                    onTimeout()  // 触发重连
                }
            }
        }
    }

    fun stop() { noDataJob?.cancel() }
}


// ============================================================
// 层3：断连后自动重连（指数退避策略）
// ============================================================

class BleReconnectManager(
    private val device: BluetoothDevice,
    private val onReconnected: () -> Unit
) {
    private var retryCount = 0
    private val maxRetry = 5
    // 退避间隔：1s, 2s, 4s, 8s, 16s（每次翻倍，避免频繁重连耗电）
    private val baseDelayMs = 1_000L

    fun scheduleReconnect(scope: CoroutineScope) {
        if (retryCount >= maxRetry) {
            // 超过最大重试次数，通知用户
            return
        }

        val delay = baseDelayMs * (1L shl retryCount)  // 2^retryCount * baseDelay
        retryCount++

        scope.launch {
            delay(delay)
            // 重新发起连接
            val gatt = device.connectGatt(context, false, gattCallback, BluetoothDevice.TRANSPORT_LE)
            // 连接成功后调用 onReconnected()，重置 retryCount = 0
        }
    }
}


// ============================================================
// 常见断连原因和解决方案（背熟）
// ============================================================

// 原因1：status=133（GATT_ERROR）
// 现象：连接后马上断开，status=133
// 原因：底层 GATT 协议错误，通常是连接太频繁或设备未完全准备好
// 解决：断开后延迟 1-2 秒再重连，并调用 gatt.close() 清理

// 原因2：华为/小米后台省电策略强制断开
// 现象：App 切到后台 30 分钟后断连
// 解决：
// 1. 启动前台 Service（显示通知），系统不会随意 kill
// 2. 申请忽略电池优化权限
// 3. 使用 autoConnect=true 重连（设备重新进入范围自动恢复）

// 原因3：MTU 协商问题导致传输失败后断连
// 解决：连接后先协商 MTU，再发送大数据
gatt.requestMtu(512)  // onMtuChanged 回调后再发数据
```

---

## Q3：如何优化 BLE 功耗？

**答：**

```kotlin
// ============================================================
// 功耗优化策略（面试必讲，结合你的戒指项目）
// ============================================================

// 策略1：根据场景切换扫描模式
// 前台使用：低延迟模式（功耗高但响应快）
// 后台监听：低功耗模式（功耗低但延迟高）
fun startScanWithPowerMode(isForeground: Boolean) {
    val settings = ScanSettings.Builder()
        .setScanMode(
            if (isForeground) ScanSettings.SCAN_MODE_LOW_LATENCY  // 前台
            else ScanSettings.SCAN_MODE_LOW_POWER                 // 后台
        )
        .setReportDelay(if (isForeground) 0 else 5000)  // 后台批量上报，减少唤醒次数
        .build()
}

// 策略2：连接参数优化
// Connection Interval（连接间隔）：越小响应越快，耗电越多
// Slave Latency（从设备延迟）：允许设备跳过几个连接间隔，省电
// Supervision Timeout（监督超时）：多久没通信判定断连
// 建议：实时数据（心率）用小间隔；静息状态（睡眠监测）用大间隔

// Android 8.0+ 支持请求更新连接参数
gatt.requestConnectionPriority(BluetoothGatt.CONNECTION_PRIORITY_HIGH)    // 实时数据，7.5ms间隔
gatt.requestConnectionPriority(BluetoothGatt.CONNECTION_PRIORITY_BALANCED) // 均衡，默认
gatt.requestConnectionPriority(BluetoothGatt.CONNECTION_PRIORITY_LOW_POWER) // 省电，100ms+间隔

// 策略3：批量传输代替频繁小包
// ❌ 错误：每秒发 1 个字节（频繁唤醒蓝牙芯片）
// ✅ 正确：缓冲 5 秒数据后一次性发送（大包）
class DataBuffer {
    private val buffer = mutableListOf<ByteArray>()
    private var bufferJob: Job? = null

    fun addData(data: ByteArray, scope: CoroutineScope, onFlush: (ByteArray) -> Unit) {
        buffer.add(data)

        // 5秒内没有新数据，则批量发送
        bufferJob?.cancel()
        bufferJob = scope.launch {
            delay(5000)
            val combined = buffer.flatten().toByteArray()
            onFlush(combined)
            buffer.clear()
        }
    }
}

// 策略4：不需要时及时断开连接，按需重连
// ❌ 常驻连接（耗电，占用设备连接槽）
// ✅ 同步完数据后断开，下次需要时再连（适合步数、睡眠等非实时数据）

// 策略5：使用 Bluetooth LE Audio（蓝牙 6.0 新特性，了解即可）
// LE Audio 基于 Isochronous Channels，延迟更低、功耗更低
// Android 13+ 开始支持
```

---

## Q4：BLE 数据丢包怎么处理？大数据包如何分片传输？

**答：**

```kotlin
// ============================================================
// BLE 数据分片传输（超出 MTU 时必须手动分片）
// ============================================================

// MTU 默认 23 字节，有效载荷（payload）= MTU - 3 = 20 字节
// 协商后最大 MTU=517，有效载荷=514字节
// 固件升级包通常几百 KB，必须分片传输

class BlePacketSender(
    private val gatt: BluetoothGatt,
    private val characteristic: BluetoothGattCharacteristic
) {
    // 协商到的 MTU，连接后更新
    private var mtu = 23
    private val payloadSize get() = mtu - 3  // 有效载荷大小

    // 分片发送大数据（如固件升级包）
    suspend fun sendLargeData(data: ByteArray): Boolean {
        val totalPackets = (data.size + payloadSize - 1) / payloadSize  // 向上取整
        var offset = 0
        var packetIndex = 0

        while (offset < data.size) {
            val end = minOf(offset + payloadSize, data.size)
            val chunk = data.sliceArray(offset until end)

            // 构造数据包：[包序号(2字节)] + [总包数(2字节)] + [数据]
            val packet = buildPacket(packetIndex, totalPackets, chunk)

            val success = writeCharacteristicWithRetry(packet)
            if (!success) return false  // 发送失败

            offset = end
            packetIndex++

            // 流控：避免发太快，设备来不及处理
            // WriteType=NO_RESPONSE 不等回调，速度快但可能丢包
            // WriteType=DEFAULT 等 onCharacteristicWrite 回调，可靠但慢
            delay(20)  // 简单流控：每包间隔 20ms
        }
        return true
    }

    // 带重试的写入（最多重试 3 次）
    private suspend fun writeCharacteristicWithRetry(data: ByteArray, maxRetry: Int = 3): Boolean {
        repeat(maxRetry) { attempt ->
            characteristic.value = data
            characteristic.writeType = BluetoothGattCharacteristic.WRITE_TYPE_DEFAULT  // 有回执
            val result = gatt.writeCharacteristic(characteristic)
            if (result) {
                // 等待 onCharacteristicWrite 回调（这里用 CompletableDeferred 同步等待）
                val success = waitForWriteAck(timeout = 3000)
                if (success) return true
            }
            delay(100L * (attempt + 1))  // 重试间隔递增
        }
        return false
    }

    private fun buildPacket(index: Int, total: Int, data: ByteArray): ByteArray {
        return ByteArray(4 + data.size).apply {
            // 小端序写入包序号
            this[0] = (index and 0xFF).toByte()
            this[1] = ((index shr 8) and 0xFF).toByte()
            // 小端序写入总包数
            this[2] = (total and 0xFF).toByte()
            this[3] = ((total shr 8) and 0xFF).toByte()
            // 数据
            data.copyInto(this, destinationOffset = 4)
        }
    }

    // 使用 CompletableDeferred 把回调转换为协程挂起（面试加分项）
    private var writeAckDeferred: CompletableDeferred<Boolean>? = null

    private suspend fun waitForWriteAck(timeout: Long): Boolean {
        writeAckDeferred = CompletableDeferred()
        return withTimeoutOrNull(timeout) {
            writeAckDeferred!!.await()
        } ?: false
    }

    // 在 onCharacteristicWrite 中调用
    fun onWriteAck(success: Boolean) {
        writeAckDeferred?.complete(success)
    }
}


// ============================================================
// 接收端：按序号重组分片数据
// ============================================================

class BlePacketReceiver {
    private val packetBuffer = HashMap<Int, ByteArray>()  // 序号 → 数据
    private var totalPackets = -1

    fun onDataReceived(rawBytes: ByteArray): ByteArray? {
        if (rawBytes.size < 4) return null

        val packetIndex = (rawBytes[0].toInt() and 0xFF) or ((rawBytes[1].toInt() and 0xFF) shl 8)
        val total       = (rawBytes[2].toInt() and 0xFF) or ((rawBytes[3].toInt() and 0xFF) shl 8)
        val payload     = rawBytes.sliceArray(4 until rawBytes.size)

        totalPackets = total
        packetBuffer[packetIndex] = payload

        // 收到所有分片，按序号重组
        if (packetBuffer.size == totalPackets) {
            val fullData = (0 until totalPackets)
                .mapNotNull { packetBuffer[it] }
                .reduce { acc, bytes -> acc + bytes }  // 拼接
            packetBuffer.clear()
            return fullData
        }
        return null  // 还没收完
    }
}
```

---

# 模块二：BLE 进阶场景

---

## Q5：固件 OTA 升级如何实现？

**答：**

OTA（Over The Air）是通过 BLE 把新固件传输到设备并完成升级，是 IoT 开发的核心功能之一。

```kotlin
// ============================================================
// OTA 升级完整流程
// ============================================================

// OTA 一般流程（以标准 DFU 协议为例，各厂商有差异）：
// 1. App 通知设备进入 OTA 模式（写特定指令）
// 2. 设备重启到 Bootloader 模式，断开当前连接
// 3. App 重新连接到 Bootloader（通常 MAC 地址 +1）
// 4. 协商 MTU，获取最大包大小
// 5. 发送固件文件大小，设备确认
// 6. 分片发送固件数据
// 7. 发送完成，发送校验指令（CRC32）
// 8. 设备校验通过后自动重启，升级完成

class OtaManager(
    private val bleManager: BleManager,
    private val scope: CoroutineScope
) {
    private val _progress = MutableStateFlow(0f)
    val progress: StateFlow<Float> = _progress.asStateFlow()

    private val _status = MutableStateFlow<OtaStatus>(OtaStatus.Idle)
    val status: StateFlow<OtaStatus> = _status.asStateFlow()

    suspend fun startOta(firmwareBytes: ByteArray) {
        try {
            // 1. 进入 OTA 模式
            _status.value = OtaStatus.EnteringOtaMode
            bleManager.writeCommand(OTA_ENTER_CMD)
            delay(2000)  // 等待设备重启

            // 2. 重新连接 Bootloader
            _status.value = OtaStatus.ConnectingBootloader
            bleManager.reconnect()

            // 3. MTU 协商，获取最大包大小
            val mtu = bleManager.requestMtu(512)
            val chunkSize = mtu - 3

            // 4. 发送文件大小，等待设备确认
            _status.value = OtaStatus.Transferring
            val sizeBytes = ByteBuffer.allocate(4)
                .order(ByteOrder.LITTLE_ENDIAN)
                .putInt(firmwareBytes.size)
                .array()
            bleManager.writeCommand(sizeBytes)
            bleManager.waitForAck()  // 等待设备确认

            // 5. 分片发送固件
            var offset = 0
            while (offset < firmwareBytes.size) {
                val end = minOf(offset + chunkSize, firmwareBytes.size)
                val chunk = firmwareBytes.sliceArray(offset until end)

                bleManager.writeData(chunk)
                bleManager.waitForAck()

                offset = end
                _progress.value = offset.toFloat() / firmwareBytes.size
            }

            // 6. 发送 CRC32 校验值
            _status.value = OtaStatus.Verifying
            val crc = CRC32().apply { update(firmwareBytes) }.value
            bleManager.writeCommand(crc.toByteArray())
            val verified = bleManager.waitForAck()

            _status.value = if (verified) OtaStatus.Success else OtaStatus.Failed("CRC 校验失败")

        } catch (e: TimeoutCancellationException) {
            _status.value = OtaStatus.Failed("升级超时")
        } catch (e: Exception) {
            _status.value = OtaStatus.Failed(e.message ?: "未知错误")
        }
    }
}

sealed class OtaStatus {
    object Idle : OtaStatus()
    object EnteringOtaMode : OtaStatus()
    object ConnectingBootloader : OtaStatus()
    object Transferring : OtaStatus()
    object Verifying : OtaStatus()
    object Success : OtaStatus()
    data class Failed(val reason: String) : OtaStatus()
}
```

---

## Q6：多设备并发连接如何管理？

**答：**

Android 官方限制同时连接的 BLE 设备数量（通常 7 个，实际各厂商不同），多设备管理需要设计连接池。

```kotlin
// ============================================================
// 多设备连接管理器
// ============================================================

class MultiDeviceBleManager {

    // 连接池：MAC → BleConnection
    private val connections = ConcurrentHashMap<String, BleConnection>()
    private val MAX_CONNECTIONS = 5  // 保守值，避免超出系统限制

    // 每个设备独立的数据流
    private val _deviceDataFlow = MutableSharedFlow<DeviceData>(
        replay = 0,
        extraBufferCapacity = 100  // 缓冲区防止数据丢失
    )
    val deviceDataFlow: SharedFlow<DeviceData> = _deviceDataFlow.asSharedFlow()

    // 连接单个设备
    fun connect(device: BluetoothDevice, scope: CoroutineScope): Boolean {
        val mac = device.address

        // 已经连接，不重复连接
        if (connections.containsKey(mac)) return true

        // 超出上限，拒绝连接
        if (connections.size >= MAX_CONNECTIONS) return false

        val connection = BleConnection(device, scope) { data ->
            scope.launch { _deviceDataFlow.emit(DeviceData(mac, data)) }
        }
        connections[mac] = connection
        connection.connect()
        return true
    }

    // 断开单个设备
    fun disconnect(mac: String) {
        connections.remove(mac)?.disconnect()
    }

    // 断开所有设备（App 关闭时调用）
    fun disconnectAll() {
        connections.values.forEach { it.disconnect() }
        connections.clear()
    }

    // 向指定设备写数据
    fun write(mac: String, data: ByteArray): Boolean {
        return connections[mac]?.write(data) ?: false
    }

    // 广播：向所有连接的设备发送相同指令
    fun broadcast(data: ByteArray) {
        connections.values.forEach { it.write(data) }
    }

    // 获取所有设备的连接状态
    fun getConnectionStates(): Map<String, BleState> {
        return connections.mapValues { it.value.state.value }
    }
}

data class DeviceData(val mac: String, val data: ByteArray)


// ============================================================
// 多设备场景下的并发写保护
// ============================================================

// BLE GATT 操作是串行的！同时发起多个 gatt.writeCharacteristic() 会导致失败
// 必须用队列保证同一个 gatt 的操作串行执行

class BleOperationQueue(private val gatt: BluetoothGatt) {
    // Channel 作为操作队列（无界缓冲）
    private val operationChannel = Channel<BleOperation>(capacity = Channel.UNLIMITED)
    private var isRunning = false

    fun enqueue(operation: BleOperation) {
        operationChannel.trySend(operation)  // 加入队列
    }

    // 启动队列消费者（一次只执行一个操作）
    fun start(scope: CoroutineScope) {
        if (isRunning) return
        isRunning = true
        scope.launch {
            for (operation in operationChannel) {
                operation.execute(gatt)          // 执行操作
                operation.waitForCompletion()    // 等待回调确认完成
            }
        }
    }
}

// 操作类型
sealed class BleOperation {
    abstract suspend fun execute(gatt: BluetoothGatt)
    abstract suspend fun waitForCompletion()

    class WriteOperation(
        private val characteristic: BluetoothGattCharacteristic,
        private val data: ByteArray
    ) : BleOperation() {
        private val deferred = CompletableDeferred<Boolean>()

        override suspend fun execute(gatt: BluetoothGatt) {
            characteristic.value = data
            gatt.writeCharacteristic(characteristic)
        }
        override suspend fun waitForCompletion() {
            withTimeout(5000) { deferred.await() }
        }
        fun complete(success: Boolean) { deferred.complete(success) }
    }
}
```

---

## Q7：BLE 跨 Android 版本兼容性问题有哪些？如何处理？

**答：**

```kotlin
// ============================================================
// Android 版本兼容性问题汇总（面试必备）
// ============================================================

object BleCompatHelper {

    // ---- Android 12（API 31）：权限重构 ----
    // 12以前：需要 ACCESS_FINE_LOCATION（位置权限）扫描蓝牙
    // 12以后：新增 BLUETOOTH_SCAN / BLUETOOTH_CONNECT / BLUETOOTH_ADVERTISE
    //         扫描不再强制需要位置权限（neverForLocation = true 时）

    fun requestBlePermissions(activity: Activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.S) {
            // Android 12+
            ActivityCompat.requestPermissions(
                activity,
                arrayOf(
                    Manifest.permission.BLUETOOTH_SCAN,
                    Manifest.permission.BLUETOOTH_CONNECT
                ),
                REQUEST_CODE
            )
        } else {
            // Android 11 及以下
            ActivityCompat.requestPermissions(
                activity,
                arrayOf(Manifest.permission.ACCESS_FINE_LOCATION),
                REQUEST_CODE
            )
        }
    }

    // ---- Android 13（API 33）：onCharacteristicChanged 新增 value 参数 ----
    // API 33 废弃了旧版 onCharacteristicChanged(gatt, characteristic)
    // 新版添加了 value 参数，避免多线程竞争导致数据错乱

    val gattCallback = object : BluetoothGattCallback() {
        // 旧版（API < 33）
        @Deprecated("Deprecated in API 33")
        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic
        ) {
            if (Build.VERSION.SDK_INT < Build.VERSION_CODES.TIRAMISU) {
                handleData(characteristic.uuid, characteristic.value.copyOf())
                // 注意：必须 copyOf()！characteristic.value 是共享内存，多线程可能被覆盖
            }
        }

        // 新版（API >= 33）：value 参数是独立的字节数组，不用 copyOf
        override fun onCharacteristicChanged(
            gatt: BluetoothGatt,
            characteristic: BluetoothGattCharacteristic,
            value: ByteArray
        ) {
            handleData(characteristic.uuid, value)
        }
    }

    private fun handleData(uuid: UUID, data: ByteArray) { /* 处理数据 */ }

    // ---- Android 8（API 26）：后台扫描限制 ----
    // App 在后台时，每 30 分钟只能扫描 5 次（每次最多 30 秒）
    // 解决：使用 PendingIntent 版本的 startScan，系统代为扫描
    fun startBackgroundScan(context: Context, filters: List<ScanFilter>) {
        val intent = Intent(context, BleReceiver::class.java)
        val pendingIntent = PendingIntent.getBroadcast(
            context, 0, intent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val settings = ScanSettings.Builder()
            .setScanMode(ScanSettings.SCAN_MODE_LOW_POWER)
            .setCallbackType(ScanSettings.CALLBACK_TYPE_FIRST_MATCH)  // 只在首次发现时回调
            .build()

        val scanner = BluetoothAdapter.getDefaultAdapter().bluetoothLeScanner
        // 这个版本的 startScan 使用 PendingIntent，不受后台扫描限制
        scanner.startScan(filters, settings, pendingIntent)
    }

    // ---- Android 10（API 29）：后台定位权限 ----
    // 后台扫描 BLE 需要 ACCESS_BACKGROUND_LOCATION
    // 但申请这个权限审核严格，建议用前台 Service 替代

    // ---- 厂商问题 ----
    // 华为：后台 BLE 连接容易被断开 → 前台 Service + 前台通知
    // 小米：MIUI 系统对 BLE 有额外省电限制 → 引导用户手动加入白名单
    // 三星：部分型号 MTU 协商有 Bug → 固定使用较小 MTU（247）
}
```

---

# 模块三：IoT 架构设计

---

## Q8：如何设计 IoT 设备通信的整体架构？

**答：**

```
// ============================================================
// IoT 设备通信分层架构（面试核心，手画架构图）
// ============================================================

UI 层（Fragment/Compose）
    ↕ StateFlow / SharedFlow
ViewModel 层（业务逻辑）
    ↕ 调用
Repository 层（数据决策）
    ↕ 调用
IoT Service 层（设备通信协议封装）
    ├── BleManager（蓝牙连接管理）
    ├── ProtocolParser（协议解析/封包）
    └── DataSyncManager（数据同步策略）
        ↕ 同步到
Cloud API（远程服务）
    ↕
Local Database（Room 本地缓存）
```

```kotlin
// ============================================================
// 协议解析层：把原始字节转换成业务对象
// ============================================================

// 设备通信协议（与硬件团队共同定义）
// 数据帧格式：[帧头0xAA][帧头0x55][命令码1字节][数据长度2字节][数据N字节][CRC2字节]
//               0      1       2            3   4           5..N+4    N+5  N+6

object ProtocolParser {

    const val HEADER_1 = 0xAA.toByte()
    const val HEADER_2 = 0x55.toByte()

    // 命令码定义（与硬件团队约定）
    const val CMD_HEART_RATE = 0x01.toByte()
    const val CMD_STEPS      = 0x02.toByte()
    const val CMD_SLEEP      = 0x03.toByte()
    const val CMD_BATTERY    = 0x04.toByte()
    const val CMD_OTA_ACK    = 0xF0.toByte()

    // 解析数据帧
    fun parse(bytes: ByteArray): DeviceFrame? {
        if (bytes.size < 7) return null  // 最小帧长度

        // 验证帧头
        if (bytes[0] != HEADER_1 || bytes[1] != HEADER_2) return null

        val cmd = bytes[2]
        val dataLen = ((bytes[3].toInt() and 0xFF) shl 8) or (bytes[4].toInt() and 0xFF)

        if (bytes.size < 5 + dataLen + 2) return null  // 数据不完整

        val data = bytes.sliceArray(5 until 5 + dataLen)
        val receivedCrc = ((bytes[5 + dataLen].toInt() and 0xFF) shl 8) or
                          (bytes[6 + dataLen].toInt() and 0xFF)

        // CRC 校验
        val calculatedCrc = calculateCrc(bytes.sliceArray(0 until 5 + dataLen))
        if (receivedCrc != calculatedCrc) return null  // 数据损坏

        return DeviceFrame(cmd, data)
    }

    // 封包：业务数据 → 原始字节帧
    fun buildFrame(cmd: Byte, data: ByteArray): ByteArray {
        val len = data.size
        val frameBody = ByteArray(5 + len).apply {
            this[0] = HEADER_1
            this[1] = HEADER_2
            this[2] = cmd
            this[3] = ((len shr 8) and 0xFF).toByte()
            this[4] = (len and 0xFF).toByte()
            data.copyInto(this, destinationOffset = 5)
        }
        val crc = calculateCrc(frameBody)
        return frameBody + byteArrayOf(((crc shr 8) and 0xFF).toByte(), (crc and 0xFF).toByte())
    }

    private fun calculateCrc(data: ByteArray): Int {
        var crc = 0xFFFF
        for (byte in data) {
            crc = crc xor (byte.toInt() and 0xFF)
            repeat(8) {
                crc = if (crc and 0x01 != 0) (crc shr 1) xor 0xA001 else crc shr 1
            }
        }
        return crc and 0xFFFF
    }
}

data class DeviceFrame(val cmd: Byte, val data: ByteArray)


// ============================================================
// 数据同步策略（本地存储 + 云端同步）
// ============================================================

class HealthDataSyncManager(
    private val bleManager: BleManager,
    private val localDb: HealthRecordDao,
    private val remoteApi: HealthApi,
    private val scope: CoroutineScope
) {
    init {
        // 监听 BLE 数据，自动解析并存储
        scope.launch {
            bleManager.dataFlow.collect { bleData ->
                when (bleData.uuid) {
                    HEART_RATE_UUID -> {
                        val hr = parseHeartRate(bleData.bytes)
                        val record = HealthRecord(heartRate = hr, timestamp = System.currentTimeMillis())
                        localDb.insert(record)   // 先存本地
                        syncToCloud(record)       // 再同步云端（异步，不阻塞）
                    }
                }
            }
        }
    }

    private fun syncToCloud(record: HealthRecord) {
        scope.launch {
            try {
                remoteApi.uploadHealthRecord(record)
                localDb.markSynced(record.id)   // 标记已同步
            } catch (e: Exception) {
                // 同步失败，保留本地记录，下次网络恢复时重试
                // 可以用 WorkManager 做离线重试
            }
        }
    }
}
```

---

## Q9：IoT 设备数据安全传输如何保障？

**答：**

```kotlin
// ============================================================
// BLE 数据安全方案
// ============================================================

// 层1：BLE 配对加密（Pairing）
// 通过 BLE 配对，连接层的数据自动加密（AES-128 CCM）
// 如何触发配对：
device.createBond()  // 发起配对，用户需要确认（有些设备自动配对）

// 检查是否已配对
if (device.bondState == BluetoothDevice.BOND_BONDED) {
    // 已配对，连接层数据已加密
}

// 层2：应用层加密（当 BLE 配对不可控时）
object AesEncryption {
    // 与设备共享一个对称密钥（通过安全通道分发，如扫码绑定时服务端下发）
    private const val ALGORITHM = "AES/GCM/NoPadding"

    fun encrypt(data: ByteArray, key: ByteArray): ByteArray {
        val secretKey = SecretKeySpec(key, "AES")
        val iv = ByteArray(12).also { SecureRandom().nextBytes(it) }  // 随机 IV（每次不同）
        val cipher = Cipher.getInstance(ALGORITHM)
        cipher.init(Cipher.ENCRYPT_MODE, secretKey, GCMParameterSpec(128, iv))
        val encrypted = cipher.doFinal(data)
        // 把 IV 拼在前面一起发送（接收方解密时需要）
        return iv + encrypted
    }

    fun decrypt(data: ByteArray, key: ByteArray): ByteArray {
        val iv = data.sliceArray(0 until 12)
        val ciphertext = data.sliceArray(12 until data.size)
        val secretKey = SecretKeySpec(key, "AES")
        val cipher = Cipher.getInstance(ALGORITHM)
        cipher.init(Cipher.DECRYPT_MODE, secretKey, GCMParameterSpec(128, iv))
        return cipher.doFinal(ciphertext)
    }
}

// 层3：安全存储密钥（Android Keystore）
// 密钥存在系统安全硬件中，App 被反编译也拿不到原始密钥
object KeystoreHelper {
    private const val KEY_ALIAS = "ble_device_key"

    fun getOrCreateKey(): SecretKey {
        val keyStore = KeyStore.getInstance("AndroidKeyStore").apply { load(null) }

        // 如果密钥已存在，直接返回
        keyStore.getKey(KEY_ALIAS, null)?.let { return it as SecretKey }

        // 生成新密钥
        val keyGenerator = KeyGenerator.getInstance(KeyProperties.KEY_ALGORITHM_AES, "AndroidKeyStore")
        keyGenerator.init(
            KeyGenParameterSpec.Builder(KEY_ALIAS, KeyProperties.PURPOSE_ENCRYPT or KeyProperties.PURPOSE_DECRYPT)
                .setBlockModes(KeyProperties.BLOCK_MODE_GCM)
                .setEncryptionPaddings(KeyProperties.ENCRYPTION_PADDING_NONE)
                .setKeySize(256)
                .setUserAuthenticationRequired(false)  // 不需要指纹验证
                .build()
        )
        return keyGenerator.generateKey()
    }
}
```

---

# 模块四：测试与工程规范

---

## Q10：BLE 相关代码如何做单元测试？

**答：**

BLE 开发的难点之一是设备依赖，单元测试要通过 Mock 解耦。

```kotlin
// ============================================================
// 设计可测试的 BLE 架构（接口隔离）
// ============================================================

// 定义接口（关键！）
interface IBleManager {
    val state: StateFlow<BleState>
    val dataFlow: SharedFlow<BleData>
    fun connect(device: BluetoothDevice)
    fun disconnect()
    suspend fun write(uuid: UUID, data: ByteArray): Boolean
}

// 真实实现
class BleManagerImpl : IBleManager { /* ... 真实 BLE 逻辑 ... */ }

// ViewModel 依赖接口而非具体实现
class DeviceViewModel(private val bleManager: IBleManager) : ViewModel() {
    val connectionState = bleManager.state

    fun connect(device: BluetoothDevice) {
        bleManager.connect(device)
    }
}


// ============================================================
// 单元测试：用 FakeBleManager 代替真实蓝牙
// ============================================================

class FakeBleManager : IBleManager {
    private val _state = MutableStateFlow(BleState.DISCONNECTED)
    override val state: StateFlow<BleState> = _state.asStateFlow()

    private val _dataFlow = MutableSharedFlow<BleData>()
    override val dataFlow: SharedFlow<BleData> = _dataFlow.asSharedFlow()

    // 记录调用历史，方便断言
    val connectCalls = mutableListOf<BluetoothDevice>()
    var writeResult = true  // 控制 write 返回成功还是失败

    override fun connect(device: BluetoothDevice) {
        connectCalls.add(device)
        _state.value = BleState.CONNECTED  // 模拟立刻连接成功
    }

    override fun disconnect() {
        _state.value = BleState.DISCONNECTED
    }

    override suspend fun write(uuid: UUID, data: ByteArray): Boolean = writeResult

    // 测试辅助：模拟设备发送数据
    suspend fun simulateIncomingData(uuid: UUID, data: ByteArray) {
        _dataFlow.emit(BleData(uuid, data))
    }
}

// 测试类
class DeviceViewModelTest {

    @get:Rule val instantExecutorRule = InstantTaskExecutorRule()

    private val fakeBle = FakeBleManager()
    private lateinit var viewModel: DeviceViewModel
    private val testScope = TestScope()

    @Before
    fun setup() {
        viewModel = DeviceViewModel(fakeBle)
    }

    @Test
    fun `connect updates state to CONNECTED`() = testScope.runTest {
        val mockDevice = mockk<BluetoothDevice>()

        viewModel.connect(mockDevice)

        assertEquals(BleState.CONNECTED, fakeBle.state.value)
        assertEquals(1, fakeBle.connectCalls.size)
    }

    @Test
    fun `incoming heart rate data updates UI state`() = testScope.runTest {
        val heartRateData = byteArrayOf(0x00, 72)  // 标准心率格式：flags + bpm

        // 模拟设备推送心率数据
        fakeBle.simulateIncomingData(HEART_RATE_CHAR_UUID, heartRateData)

        // 验证 ViewModel 正确解析并更新状态
        assertEquals(72, viewModel.heartRate.value)
    }

    @Test
    fun `write failure shows error message`() = testScope.runTest {
        fakeBle.writeResult = false  // 模拟写入失败

        viewModel.sendCommand(byteArrayOf(0x01))

        assertEquals("指令发送失败", viewModel.errorMessage.value)
    }
}
```

---

## Q11：如何与硬件团队协作定义 BLE 通信协议？

**答：**（考验跨团队协作经验）

```
BLE 协议定义最佳实践（实际工作经验）：

1. 制定协议文档（双方共同维护）
   内容包括：
   - Service UUID 列表（每个功能对应一个 Service）
   - Characteristic UUID 列表（含 Properties：R/W/N/I）
   - 数据帧格式定义（帧头、命令码、长度、数据、校验）
   - 每个命令码的请求/响应格式（含字节序：大端/小端）
   - 错误码定义
   - 版本号（协议升级时向后兼容）

2. 版本控制
   协议文档放到 Git 管理，任何修改都需要双方 review
   修改协议时，新旧协议并行支持一段时间（兼容期）

3. 联调流程
   - 先用 nRF Connect / LightBlue 工具直接读写 Characteristic 验证硬件
   - App 开发用 FakeBleManager 先跑通业务逻辑
   - 再和真实设备联调

4. 调试工具
   - nRF Connect（Android/iOS）：直接扫描、连接、读写 Characteristic
   - Wireshark + Ubertooth：抓包分析（需要硬件）
   - 设备端串口日志：硬件工程师在设备上打印发送/接收的原始字节

5. 常见协作问题和解决方案
   - 字节序不一致（App 用大端，设备用小端）→ 协议文档明确标注
   - CRC 算法不一致 → 协议文档给出伪代码，双方各自实现后互相验证
   - 数据格式有歧义 → 举具体例子（例：心率 75bpm 对应字节为 0x00 0x4B）
```

---

# 模块五：技术前沿（加分项）

---

## Q12：蓝牙 5.x / 6.0 有哪些新特性？

**答：**

```
蓝牙版本特性演进（面试能说出来加分）：

蓝牙 5.0（2016）：
- 传输距离提升 4 倍（~40m → ~240m 室外）
- 传输速度提升 2 倍（1Mbps → 2Mbps）
- 广播容量提升 8 倍（支持更多广播数据）
- Android 8.0+ 开始支持

蓝牙 5.1（2019）：
- 方向查找（Direction Finding）：厘米级定位精度
- 可用于室内精确定位（IoT 资产追踪）

蓝牙 5.2（2020）：
- LE Audio：基于 Isochronous Channels（等时通道）
  - 更低延迟（适合助听器、实时音频）
  - Auracast 广播音频（一个设备广播到多个接收端，如机场广播）
  - LC3 编解码（比 SBC 更省电、更高质量）
- Enhanced ATT（EATT）：多路并发 ATT 操作，提升吞吐量
- Android 12+ 开始部分支持

蓝牙 5.3（2021）：
- 增强周期广播同步传输
- 连接子速率切换（Connection Subrating）：动态调整连接间隔，平衡响应速度和功耗

蓝牙 6.0（2024）：
- 频道探测（Channel Sounding）：厘米级双向测距，精度更高
- 监控广播（Monitored Advertisers）：更高效的设备发现
- LLCP 增强：链路层控制协议改进

IoT 通信新标准（了解即可）：
- Matter（原 CHIP）：智能家居统一标准，Thread + BLE + Wi-Fi
- Thread：基于 IEEE 802.15.4 的 IPv6 网状网络，低功耗
- Zigbee：低功耗网状网络，适合大量小型 IoT 传感器
```

---

## Q13：Android BLE 开发常见 Crash 如何快速定位？

**答：**

```kotlin
// ============================================================
// 常见 BLE Crash 类型和定位方法
// ============================================================

// Crash 1：NullPointerException on BluetoothGatt
// 原因：gatt 已 close 或 null，还在调用 readCharacteristic 等方法
// 定位：Firebase Crashlytics 查看 gatt 为 null 的调用栈
// 解决：
class SafeGattWrapper(private var gatt: BluetoothGatt?) {
    fun readCharacteristic(char: BluetoothGattCharacteristic): Boolean {
        return gatt?.readCharacteristic(char) ?: false  // null 安全调用
    }
    fun close() {
        gatt?.close()
        gatt = null  // 置 null，防止后续误用
    }
}


// Crash 2：IllegalStateException: Bluetooth is not enabled
// 原因：用户关闭蓝牙后 App 还在调用 BLE API
// 解决：监听蓝牙状态变化，及时响应

class BluetoothStateReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action == BluetoothAdapter.ACTION_STATE_CHANGED) {
            when (intent.getIntExtra(BluetoothAdapter.EXTRA_STATE, BluetoothAdapter.ERROR)) {
                BluetoothAdapter.STATE_OFF      -> handleBluetoothOff()
                BluetoothAdapter.STATE_ON       -> handleBluetoothOn()
                BluetoothAdapter.STATE_TURNING_OFF -> disconnectAllDevices()
            }
        }
    }
}

// AndroidManifest.xml 注册（静态注册，App 未启动也能收到）
// <receiver android:name=".BluetoothStateReceiver">
//     <intent-filter>
//         <action android:name="android.bluetooth.adapter.action.STATE_CHANGED"/>
//     </intent-filter>
// </receiver>


// Crash 3：DeadObjectException（调用已死亡的 Binder 服务）
// 原因：BluetoothGatt 内部 Binder 服务崩溃
// 解决：catch DeadObjectException，清理重连
fun safeGattOperation(operation: () -> Boolean): Boolean {
    return try {
        operation()
    } catch (e: DeadObjectException) {
        cleanup()        // 清理资源
        reconnect()      // 重连
        false
    }
}


// Crash 4：主线程 ANR（BLE 回调默认在非主线程，切回主线程更新 UI 时死锁）
// 解决：BLE 数据处理全在协程 IO 线程，只在需要时用 withContext(Main) 切主线程
override fun onCharacteristicChanged(gatt: BluetoothGatt, characteristic: BluetoothGattCharacteristic) {
    // ✅ 不在这里直接操作 UI，发射到 Flow，让 UI 层订阅处理
    val data = characteristic.value.copyOf()  // 注意 copyOf！
    scope.launch {
        _dataFlow.emit(BleData(characteristic.uuid, data))
    }
}


// ============================================================
// 日志策略：线上问题定位
// ============================================================
object BleLogger {
    private const val TAG = "BleSDK"

    // 关键事件打印日志（连接状态变化、数据收发、错误）
    fun logConnection(mac: String, state: String, status: Int) {
        if (BuildConfig.DEBUG) {
            Log.d(TAG, "[$mac] Connection: $state (status=$status)")
        }
        // 线上：上报 Firebase/自建日志系统，设置采样率避免日志过多
    }

    fun logData(mac: String, uuid: UUID, dataHex: String) {
        if (BuildConfig.DEBUG) {
            Log.d(TAG, "[$mac] Data [$uuid]: $dataHex")
        }
    }

    fun logError(mac: String, error: String, throwable: Throwable? = null) {
        Log.e(TAG, "[$mac] Error: $error", throwable)
        // 上报到 Crashlytics
        throwable?.let { FirebaseCrashlytics.getInstance().recordException(it) }
    }
}
```

---

## 面试 STAR 故事：BLE 稳定性优化（针对此岗位定制）

```
【背景】
我在 YLS 主导健康戒指 APP 开发时，接入了 BLE 戒指设备（支持心率、步数、睡眠监测）。
上线后收到用户反馈：部分机型（华为、OPPO）后台使用 30 分钟后数据停止更新。

【分析过程】
1. 给 BLE 回调加详细日志，收集用户日志
2. 发现规律：华为机型在 App 进后台后约 30 分钟，
   onConnectionStateChange 回调 status=8（GATT_CONN_TIMEOUT），触发断连
3. 根本原因：华为系统的激进省电策略，强制断开后台 BLE 连接

【解决方案】
1. 启动前台 Service（显示"健康数据同步中"通知），系统不再随意断连
2. 添加心跳机制：每 30 秒 readCharacteristic 一次，保持连接活跃
3. 断连后自动重连（指数退避：1s/2s/4s/8s）
4. 引导用户在手机设置中关闭 App 的电池优化

【结果】
- 连接断开频率降低 85%
- 数据丢失问题基本消除
- 用户满意度提升，应用商店评分从 3.8 → 4.3
- 整理成《BLE 多品牌兼容适配指南》文档，供团队后续参考
```

---

*文件生成时间：2026-03-05 | 针对岗位：Android BLE + IoT 专项*
