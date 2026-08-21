# Laporan Analisis Code MikuRay - V2Ray Client Android

**Tanggal Analisis:** 2026-08-21  
**Codebase:** MikuRay (Fork dari v2rayNG)  
**Bahasa:** Kotlin  
**Platform:** Android

---

## 1. ARSITEKTUR SERVICE

MikuRay menggunakan **dua mode operasi** yang berbeda, masing-masing dengan service tersendiri:

### 1.1 CoreVpnService (Mode VPN - Default)
- **File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`
- **Extends:** `VpnService` (Android VPN API)
- **Fungsi:** Menjalankan V2Ray/Xray core dengan VPN interface
- **Karakteristik:**
  - Membuat VPN interface menggunakan `Builder.establish()`
  - Tidak memerlukan root access
  - Mendukung per-app proxy
  - Mendukung tun2socks atau hev-socks5-tunnel

### 1.2 CoreRootService (Mode Root)
- **File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreRootService.kt`
- **Extends:** `Service` (Android Service biasa)
- **Fungsi:** Menjalankan V2Ray/Xray core dengan iptables (memerlukan root)
- **Karakteristik:**
  - Menggunakan iptables untuk routing
  - Memerlukan root access
  - Lebih fleksibel untuk advanced networking

### 1.3 CoreServiceManager (Central Orchestrator)
- **File:** `V2rayNG/app/src/main/java/com/v2ray/ang/core/CoreServiceManager.kt`
- **Fungsi:** Mengatur lifecycle core V2Ray/Xray
- **Tanggung Jawab:**
  - `startCoreLoop()`: Memulai core loop
  - `stopCoreLoop()`: Menghentikan core loop
  - `reloadCore()`: Me-reload core tanpa full restart (untuk network handover)
  - Mengelola `CoreController` native
  - Mengelola network monitoring
  - Mengelola browser dialer

---

## 2. MEKANISME CONNECT/DISCONNECT

### 2.1 Flow Koneksi (CoreVpnService)

```
User memulai VPN
    ↓
CoreVpnService.onStartCommand()
    ↓
setupVpnService()
    ├─ prepare() - cek VPN permission
    ├─ configureVpnService()
    │   ├─ Builder.addAddress()
    │   ├─ Builder.addRoute()
    │   ├─ Builder.addDnsServer()
    │   ├─ Builder.establish() → mInterface (ParcelFileDescriptor)
    │   └─ playConnect() 🔊 [LINE 235]
    └─ runTun2socks()
    ↓
startService()
    ↓
CoreServiceManager.startCoreLoop(mInterface)
    ├─ doStartCoreLoop()
    ├─ launchCore()
    │   ├─ Load config dari MMKV
    │   ├─ CoreConfigManager.getV2rayConfig()
    │   └─ coreController.startLoop(config, tunFd)
    └─ startNetworkMonitor()
```

### 2.2 Flow Disconnection

```
User menghentikan VPN
    ↓
stopAllService()
    ├─ isRunning = false
    ├─ RootLanSharing.stopClientSharing()
    ├─ wakeLock.release()
    ├─ playDisconnect() 🔊 [LINE 346]
    ├─ tun2SocksService?.stopTun2Socks()
    └─ CoreServiceManager.stopCoreLoop()
        ├─ coreController.stopLoop()
        ├─ browserDialer?.stop()
        ├─ networkMonitor?.unregister()
        └─ MessageUtil.sendMsg2UI(MSG_STATE_STOP_SUCCESS)
    ↓
mInterface.close()
    ↓
stopSelf()
```

---

## 3. BUG YANG DITEMUKAN

### 🔴 BUG MAJOR #1: Memory Leak di SoundPlayer
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/util/SoundPlayer.kt`  
**Severity:** HIGH  
**Lokasi:** Line 19-23

**Masalah:**
```kotlin
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)
    player?.start()
}
```

**Analisis:**
1. Jika `MediaPlayer.create()` gagal, akan return `null`
2. `player?.start()` tidak akan dipanggil jika null
3. **TIDAK ADA listener untuk `onCompletion`** - MediaPlayer tidak di-release setelah selesai
4. Setiap kali audio dimainkan, objek MediaPlayer lama di-release, tapi yang baru **tetap hidup selamanya**
5. Jika user connect-disconnect berkali-kali, akan terjadi **accumulation memory**

**Impact:**
- Memory leak jika user sering connect/disconnect
- Potensial crash pada low-memory devices
- MediaPlayer bisa terus memegang audio focus

**Rekomendasi Fix:**
```kotlin
private fun playSound(context: Context, resId: Int) {
    player?.release()
    player = MediaPlayer.create(context, resId)?.apply {
        setOnCompletionListener { mp ->
            mp.release()
            if (player === mp) {
                player = null
            }
        }
        start()
    }
}
```

---

### 🔴 BUG MAJOR #2: Race Condition di CoreVpnService
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`  
**Severity:** MEDIUM-HIGH  
**Lokasi:** Line 122-153 (onStartCommand)

**Masalah:**
```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ...
    if (!tryLockStart()) {
        LogUtil.w(AppConfig.TAG, "StartCore-VPN: Start already in progress")
        return START_NOT_STICKY  // ⚠️ MASALAH DI SINI
    }
    // ...
    serviceScope.launch {
        val ok = setupVpnService()
        if (!ok) {
            withContext(Dispatchers.Main) {
                unlockStart()
                stopSelf()  // ⚠️ DAN DI SINI
            }
        } else {
            startService()
            unlockStart()
        }
    }
    return START_STICKY  // ⚠️ Tapi return STICKY
}
```

**Analisis:**
1. Jika start sudah dalam progress, return `START_NOT_STICKY`
2. Tapi di flow normal, return `START_STICKY`
3. **Inkonsistensi:** Jika system membunuh service dan restart, bisa terjadi kondisi:
   - Service restart otomatis (karena STICKY)
   - Tapi `isStartingLock` masih true dari instance sebelumnya
   - Service baru langsung return NOT_STICKY
   - **Result: Service mati permanent**

4. `setupVpnService()` berjalan di coroutine, tapi return value `START_STICKY` sudah dikembalikan **sebelum setup selesai**

**Impact:**
- Service bisa stuck dalam state "starting" dan tidak bisa di-restart
- Jika system kill service di tengah setup, bisa menyebabkan zombie service
- User harus force close app untuk recovery

**Rekomendasi Fix:**
- Reset `isStartingLock` di `onCreate()` atau check timestamp
- Gunakan timeout untuk auto-unlock jika stuck
- Atau gunakan state machine yang lebih robust

---

### 🟡 BUG MINOR #1: Potensi Resource Leak di VPN Interface
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`  
**Severity:** LOW-MEDIUM  
**Lokasi:** Line 213-219 (configureVpnService)

**Masalah:**
```kotlin
try {
    if (::mInterface.isInitialized) {
        mInterface.close()
    }
} catch (e: Exception) {
    LogUtil.w(AppConfig.TAG, "Failed to close old interface", e)
}
```

**Analisis:**
- Exception di-catch tapi di-ignore (hanya warning log)
- Jika `close()` gagal, file descriptor lama **tetap terbuka**
- VPN interface baru dibuat di line 224: `mInterface = builder.establish()!!`
- **Result:** File descriptor leak

**Impact:**
- Accumulation file descriptors jika sering reconnect
- Bisa hit system limit (biasanya 1024 FD per process)
- App crash dengan "Too many open files"

**Rekomendasi:**
- Log error dengan severity lebih tinggi
- Consider retry atau fail-fast jika tidak bisa close

---

### 🟡 BUG MINOR #2: Thread Sleep di stopAllService
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`  
**Severity:** LOW  
**Lokasi:** Line 357-361

**Masalah:**
```kotlin
try {
    Thread.sleep(100)  // ⚠️ Blocking sleep
} catch (e: InterruptedException) {
    LogUtil.w(AppConfig.TAG, "StartCore-VPN: Sleep interrupted", e)
}
```

**Analisis:**
- Blocking main thread dengan `Thread.sleep()`
- Tidak jelas mengapa perlu delay 100ms
- Bisa menyebabkan ANR jika dipanggil dari main thread
- Di function `stopAllService` yang bisa dipanggil dari berbagai context

**Impact:**
- UI freeze selama 100ms saat disconnect
- Potensial ANR warning dari system
- Bad UX

**Rekomendasi:**
- Hapus sleep jika tidak diperlukan
- Atau gunakan coroutine delay dengan proper dispatcher
- Atau pindahkan ke background thread dengan callback

---

### 🟡 BUG MINOR #3: Potential NPE di stopCoreLoop
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/core/CoreServiceManager.kt`  
**Severity:** LOW  
**Lokasi:** Line 189-196

**Masalah:**
```kotlin
if (isRunning()) {
    CoroutineScope(Dispatchers.IO).launch {
        try {
            coreController.stopLoop()  // ⚠️ Unstructured coroutine
        } catch (e: Exception) {
            LogUtil.e(AppConfig.TAG, "StartCore-Manager: Failed to stop V2Ray loop", e)
        }
    }
}
```

**Analisis:**
- Coroutine diluncurkan dengan **unstructured concurrency**
- Tidak ada mekanisme untuk menunggu completion
- Function langsung lanjut ke line 199 (reconcileBrowserDialer) **tanpa menunggu core berhenti**
- Bisa menyebabkan race condition dimana:
  - `browserDialer.stop()` dipanggil sementara core masih berjalan
  - Notification dibatalkan sementara core masih active
  - `MSG_STATE_STOP_SUCCESS` dikirim tapi core belum berhenti

**Impact:**
- UI menunjukkan "Disconnected" tapi core masih berjalan
- Potensial zombie process
- Inconsistent state

**Rekomendasi:**
```kotlin
if (isRunning()) {
    runBlocking {
        withContext(Dispatchers.IO) {
            try {
                coreController.stopLoop()
            } catch (e: Exception) {
                LogUtil.e(AppConfig.TAG, "StartCore-Manager: Failed to stop V2Ray loop", e)
            }
        }
    }
}
```

---

## 4. LOKASI CODE NOTIFIKASI AUDIO

### 4.1 File Audio
**Lokasi:** `V2rayNG/app/src/main/res/raw/`
- `connect_sound.wav` - Audio saat connect
- `disconnect_sound.wav` - Audio saat disconnect

### 4.2 SoundPlayer Class
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/util/SoundPlayer.kt`

```kotlin
object SoundPlayer {
    private var player: MediaPlayer? = null

    fun playConnect(context: Context) {
        playSound(context, R.raw.connect_sound)
    }

    fun playDisconnect(context: Context) {
        playSound(context, R.raw.disconnect_sound)
    }

    private fun playSound(context: Context, resId: Int) {
        player?.release()
        player = MediaPlayer.create(context, resId)
        player?.start()
    }
}
```

### 4.3 Tempat Audio Dipanggil

#### Di CoreVpnService:
- **Connect:** Line 234-236
- **Disconnect:** Line 345-347

```kotlin
if (MmkvManager.decodeSettingsBool(AppConfig.PREF_SOUND_ON_CONNECT, true)) {
    SoundPlayer.playConnect(this)
}
```

#### Di CoreRootService:
- **Connect:** Line 57-59
- **Disconnect:** Line 106-108

### 4.4 Preference Control
**Key:** `AppConfig.PREF_SOUND_ON_CONNECT`  
**Defined:** `V2rayNG/app/src/main/java/com/v2ray/ang/AppConfig.kt` Line 168  
**Default Value:** `true` (enabled by default)

### 4.5 Cara Enable/Disable Audio
Audio dikontrol melalui preference `pref_sound_on_connect`:
- `true` = Audio diputar saat connect/disconnect
- `false` = Audio tidak diputar

User bisa mengubahnya melalui Settings UI (jika tersedia) atau dengan:
```kotlin
MmkvManager.encodeSettingsBool(AppConfig.PREF_SOUND_ON_CONNECT, false)
```

### 4.6 Cara Mengganti Audio Custom
1. Replace file di `V2rayNG/app/src/main/res/raw/`:
   - `connect_sound.wav`
   - `disconnect_sound.wav`
2. Format yang didukung: WAV, MP3, OGG (Android MediaPlayer)
3. Rebuild aplikasi

---

## 5. MEKANISME RECONNECT

### 5.1 Auto-Reconnect via Network Monitor
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/core/CoreServiceManager.kt`  
**Function:** `startNetworkMonitor()` (Line 217-227)

```kotlin
private fun startNetworkMonitor(service: Service) {
    if (Build.VERSION.SDK_INT < Build.VERSION_CODES.P) return
    if (networkMonitor != null) return

    val connectivity = service.getSystemService(Context.CONNECTIVITY_SERVICE) as? ConnectivityManager ?: return
    networkMonitor = NetworkMonitor(
        connectivity = connectivity,
        onUnderlyingNetworksChanged = { networks -> serviceControl?.setUnderlyingNetworks(networks) },
        onHandover = { reloadCore() },  // ⚠️ AUTO-RELOAD SAAT NETWORK HANDOVER
    ).also { it.register() }
}
```

**Kapan Terjadi:**
- Network switch (WiFi ↔ Mobile Data)
- Network handover (Cell tower switch)
- VPN underlying network berubah

**Mekanisme:**
1. `NetworkMonitor` mendeteksi network change
2. Callback `onHandover` dipanggil
3. `reloadCore()` dijalankan:
   - Stop core loop
   - Launch core lagi dengan config yang sama
   - **Tidak ada audio notification** (parameter `isReload = true`)

### 5.2 Service Restart oleh Android System
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/service/CoreVpnService.kt`  
**Function:** `onStartCommand()` (Line 122)

```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ...
    return START_STICKY  // ⚠️ Service akan di-restart otomatis
}
```

**Kapan Terjadi:**
- System membunuh service karena low memory
- User clear dari recent apps (tergantung Android version)
- System restart service setelah crash

**Behavior:**
- `START_STICKY` = Service di-restart dengan `intent = null`
- Service akan coba setup ulang dari awal

### 5.3 Intent-Based Restart
**File:** `V2rayNG/app/src/main/java/com/v2ray/ang/core/CoreServiceManager.kt`  
**Class:** `ReceiveMessageHandler` (Line 385-453)  
**Message:** `AppConfig.MSG_STATE_RESTART` (Line 414-430)

```kotlin
AppConfig.MSG_STATE_RESTART -> {
    LogUtil.i(AppConfig.TAG, "StartCore-Manager: Restart service")
    if (isOrderedBroadcast) resultCode = Activity.RESULT_OK

    val pendingResult = goAsync()
    CoroutineScope(Dispatchers.Default).launch {
        try {
            serviceControl.stopService()
            delay(500L)  // ⚠️ Delay 500ms
            LauncherManager.startService(serviceControl.getService())
        } finally {
            pendingResult.finish()
        }
    }
}
```

**Kapan Terjadi:**
- User klik "Restart" di UI
- Broadcast message diterima dari UI/Widget/Tile

---

## 6. CARA DISABLE AUTO-RECONNECT

### 6.1 Disable Network Monitor (Reconnect saat Network Change)

**Tidak ada setting built-in**, tapi bisa dimodifikasi code:

**Option 1: Comment out network monitor**
File: `CoreServiceManager.kt` Line 112
```kotlin
private fun doStartCoreLoop(service: Service, vpnInterface: ParcelFileDescriptor?) {
    // ...
    launchCore(service, vpnInterface)
    // startNetworkMonitor(service)  // ← COMMENT INI
}
```

**Option 2: Disable handover callback**
File: `CoreServiceManager.kt` Line 217-227
```kotlin
networkMonitor = NetworkMonitor(
    connectivity = connectivity,
    onUnderlyingNetworksChanged = { networks -> serviceControl?.setUnderlyingNetworks(networks) },
    onHandover = { /* reloadCore() */ },  // ← DISABLE CALLBACK
).also { it.register() }
```

### 6.2 Disable Service Auto-Restart

**Option 1: Change return value**
File: `CoreVpnService.kt` Line 152
```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // ...
    return START_NOT_STICKY  // ← CHANGE DARI START_STICKY
}
```

**Impact:**
- Service tidak akan di-restart otomatis oleh system
- Jika service dibunuh, akan tetap mati sampai user manual start
- **VPN akan disconnect jika app dibunuh**

### 6.3 Disable Intent Restart Handler

File: `CoreServiceManager.kt` Line 414-430
```kotlin
AppConfig.MSG_STATE_RESTART -> {
    // COMMENT SEMUA CODE DI SINI
    // Atau return early:
    return
}
```

### 6.4 Recommendation untuk User

**Jika ingin disable auto-reconnect:**
1. **Paling aman:** Disable network monitor saja (Option 6.1)
   - Core tetap jalan jika app dibunuh
   - Tidak reconnect saat network switch
   - User harus manual reconnect jika network berubah

2. **Paling agresif:** Kombinasi 6.1 + 6.2
   - Tidak ada auto-reconnect sama sekali
   - VPN mati permanent jika dibunuh
   - Paling hemat battery tapi worst UX

---

## 7. SUMMARY & REKOMENDASI

### 7.1 Bug Priority
1. **FIX IMMEDIATELY:**
   - 🔴 Bug #1: Memory leak di SoundPlayer
   - 🔴 Bug #2: Race condition di CoreVpnService startup

2. **FIX SOON:**
   - 🟡 Bug #3: NPE potensial di stopCoreLoop
   - 🟡 Bug #4: Resource leak di VPN interface close

3. **NICE TO FIX:**
   - 🟡 Bug #5: Thread.sleep di stopAllService

### 7.2 Architectural Notes
- Code generally well-structured dengan separation of concerns
- Good use of Kotlin coroutines
- Good logging untuk debugging
- Tapi ada beberapa anti-pattern (unstructured concurrency, manual locking)

### 7.3 Testing Recommendation
1. Test memory leak dengan reconnect 50+ kali
2. Test low-memory scenarios (< 100MB available)
3. Test network switch behavior (WiFi ↔ Mobile)
4. Test force-kill recovery
5. Test concurrent start attempts

---

**End of Report**
