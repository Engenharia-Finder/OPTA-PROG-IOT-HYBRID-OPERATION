# Code Documentation - IoT Cloud Offline System

This document explains the main parts of the code, focusing on the thread function and the offline operation logic.

---

## 📋 Index
1. [Thread Function (cloudThreadFunc)](#thread-function-cloudthreadfunc)
2. [Loop Offline Operation Logic](#loop-offline-operation-logic)
3. [Connectivity Callbacks System](#connectivity-callbacks-system)
4. [Offline Control Variables](#offline-control-variables)
5. [Setup and Initialization](#setup-and-initialization)

---

## 🔄 Thread Function (cloudThreadFunc)

### Location: Lines 35-46

```cpp
void cloudThreadFunc() {
  while (true) {
    // Executa o update da nuvem em uma thread separada
    // Se bloquear aqui (ex: DHCP), não afeta o loop principal
    // IMPORTANTE: Esta thread funciona independente da presença de USB
    // O NetworkConfigurator com SerialAgent não deve bloquear esta thread
    ArduinoCloud.update();
    
    // Pequeno delay para ceder CPU - otimizado para não causar delays desnecessários
    rtos::ThisThread::sleep_for(10);
  }
}
```

### Detailed Explanation:

**Purpose:**
- This function runs in a **separate thread** (secondary thread), independent of the main `loop()`
- Continuously executes `ArduinoCloud.update()`, which maintains communication with the Arduino IoT Cloud

**Why use a separate thread?**
1. **Does not block the main loop**: If the cloud connection hangs or takes time (e.g., DHCP issues, network timeout), the main `loop()` continues executing normally
2. **Works without USB**: The thread works even when no USB cable is connected, allowing for fully autonomous operation
3. **Isolation of issues**: Network problems do not affect the main logic of the device

**How it works:**
- `while (true)`: Infinite loop that keeps the thread always active
- `ArduinoCloud.update()`: Updates communication with the cloud, sending and receiving data
- `rtos::ThisThread::sleep_for(10)`: 10ms pause to yield CPU to other threads, avoiding excessive resource consumption

**Thread Initialization:**
- The thread is created on line 33: `rtos::Thread cloudThread(osPriorityNormal, 8192);`
  - `osPriorityNormal`: Normal thread priority
  - `8192`: 8KB stack memory for the thread
- The thread is started on line 95: `cloudThread.start(cloudThreadFunc);`

---

## 🔁 Loop Offline Operation Logic

### Location: Lines 102-146

```cpp
void loop() {
  // Verificação de conectividade (executada periodicamente, não bloqueia)
  static unsigned long ultimaVerificacao = 0;
  if ((millis() - ultimaVerificacao) >= 10000) { // Verifica a cada 10 segundos
    verificarStatusConectividade();
    ultimaVerificacao = millis();
  }

  // Só atualiza variáveis da cloud se estiver conectado E sincronizado
  if(cloudConectado && cloudSincronizado) {
    // Atualizações das variáveis de feedback...
  }
}
```

### Detailed Explanation:

#### 1. **Periodic Connectivity Check (Lines 104-108)**

```cpp
static unsigned long ultimaVerificacao = 0;
if ((millis() - ultimaVerificacao) >= 10000) {
  verificarStatusConectividade();
  ultimaVerificacao = millis();
}
```

**What it does:**
- Checks connection status with the cloud every **10 seconds**
- Uses `millis()` so as not to block the code (does not use `delay()`)
- Calls `verificarStatusConectividade()` which checks if there was a change in connection status

**Why it is important:**
- Serves as a **backup** in case callbacks are not called
- Ensures the system detects connectivity changes even if callbacks fail
- Does not block the loop, allowing the main code to continue executing

#### 2. **Variable Update Protection (Lines 120-143)**

```cpp
if(cloudConectado && cloudSincronizado) {
  // Atualiza variáveis da cloud apenas aqui
}
```

**What it does:**
- **Only updates cloud variables** when BOTH conditions are true:
  - `cloudConectado == true`: Device is connected to the cloud
  - `cloudSincronizado == true`: Initial synchronization was completed

**Why this protection is critical:**
1. **Avoids crashes**: If you try to update variables before synchronization, the device may crash upon restart
2. **Ensures integrity**: Only sends data when the connection is fully established
3. **Offline-first mode**: The main code continues functioning even without connection

**Usage Example (Lines 122-127):**
```cpp
//FEEDBACK DE DISJUNTOR ABERTO
if(fbAbertura == HIGH){
  feedbackabertura = HIGH;
}else{
  feedbackabertura = LOW;
}
```
- These updates only happen when `cloudConectado && cloudSincronizado` are true
- Outside this condition, the device works normally but does not try to send data to the cloud

---

## 📡 Connectivity Callbacks System

### Location: Lines 161-190

The system uses three main callbacks to monitor connection status:

#### 1. **onCloudConnect() - Lines 162-172**

```cpp
void onCloudConnect() {
  cloudConectado = true;
  modoOffline = false;
  cloudSincronizado = false; // Reset: aguarda sincronização
  ultimaConexaoCloud = millis();
  tentativasConexao = 0;
  
  // Atualiza LEDs de status
  digitalWrite(LED_USER, HIGH); // LED azul ligado
  digitalWrite(LEDR, LOW);       // LED vermelho desligado
}
```

**What it does:**
- Called when the device **connects** to the Arduino IoT Cloud
- Marks `cloudConectado = true` but **not yet synchronized** (`cloudSincronizado = false`)
- Updates LEDs: Blue LED on, Red LED off
- Resets attempt counter

#### 2. **onCloudSync() - Lines 174-177**

```cpp
void onCloudSync() {
  // Sincronização concluída - agora é seguro atualizar variáveis da cloud
  cloudSincronizado = true;
}
```

**What it does:**
- Called when **initial synchronization** with the cloud is completed
- Marks `cloudSincronizado = true`, allowing the loop to update variables
- **Critical**: Only after this callback is it safe to update cloud variables

#### 3. **onCloudDisconnect() - Lines 179-190**

```cpp
void onCloudDisconnect() {
  cloudConectado = false;
  cloudSincronizado = false;
  modoOffline = true;
  
  // Atualiza LEDs de status
  digitalWrite(LED_USER, LOW);  // LED azul desligado
  digitalWrite(LEDR, HIGH);     // LED vermelho ligado (modo offline)
  
  tentativasConexao = 0;
}
```

**What it does:**
- Called when the device **disconnects** from the cloud
- Marks `modoOffline = true` and resets connection flags
- Updates LEDs: Blue LED off, Red LED on (indicates offline mode)
- The device continues working normally, but does not send data to the cloud

---

## 🔧 Offline Control Variables

### Location: Lines 25-30

```cpp
bool modoOffline = false;           // Indica se está em modo offline
bool cloudConectado = false;        // Indica se está conectado à cloud
bool cloudSincronizado = false;    // Indica se sincronização inicial foi concluída
unsigned long ultimaConexaoCloud = 0; // Timestamp da última conexão
int tentativasConexao = 0;          // Contador de tentativas de conexão
```

**Explanation:**
- **modoOffline**: Main flag indicating offline mode (true) or online (false)
- **cloudConectado**: Indicates if there is an active connection to the cloud
- **cloudSincronizado**: Indicates if initial synchronization was completed (critical for updating variables)
- **ultimaConexaoCloud**: Stores when the last successful connection occurred
- **tentativasConexao**: Counter to manage reconnection attempts

---

## ⚙️ Setup and Initialization

### Location: Lines 48-100

#### Main steps:

1. **Serial Initialization (Line 51)**
   ```cpp
   Serial.begin(9600);
   ```

2. **LED Configuration (Lines 54-57)**
   - Blue LED (LED_USER): Indicates connection with cloud
   - Red LED (LEDR): Indicates offline mode
   - Initially: Blue LED off, Red LED on (offline mode)

3. **Cloud Initialization (Line 71)**
   ```cpp
   ArduinoCloud.begin(ArduinoIoTPreferredConnection, false);
   ```
   - `false`: Disables internal watchdog that causes restarts

4. **Callback Registration (Lines 82-84)**
   ```cpp
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::CONNECT, onCloudConnect);
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::SYNC, onCloudSync);
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::DISCONNECT, onCloudDisconnect);
   ```

5. **Offline-First Initialization (Lines 88-90)**
   ```cpp
   modoOffline = true;
   cloudConectado = false;
   cloudSincronizado = false;
   ```
   - System starts in offline mode by default
   - Ensures the device works even without connection

6. **Thread Start (Line 95)**
   ```cpp
   cloudThread.start(cloudThreadFunc);
   ```
   - Starts the thread that manages communication with the cloud

---

## 🎯 Operation Flow Summary

1. **Initialization**: System starts in offline mode, cloud thread is started
2. **Cloud Thread**: Executes `ArduinoCloud.update()` continuously in background
3. **Main Loop**: 
   - Checks connectivity every 10 seconds
   - Executes main device logic
   - Only updates cloud variables if `cloudConectado && cloudSincronizado`
4. **Callbacks**: Monitor status changes and update flags and LEDs
5. **Offline Mode**: Device continues working normally even without connection

---

## ⚠️ Important Points

1. **Never update cloud variables outside the `if(cloudConectado && cloudSincronizado)` condition**
   - This can cause crashes when restarting the device

2. **The cloud thread works independently of the main loop**
   - Network problems do not affect the main logic

3. **The system is offline-first**
   - Works normally even without cloud connection
   - It simply does not send data when offline

4. **Synchronization is critical**
   - Only update variables after `cloudSincronizado == true`
   - This ensures the connection is fully established

---

## 📝 Final Notes

This code implements a robust IoT system that:
- ✅ Works offline-first (does not depend on connection)
- ✅ Uses threads to not block the main loop
- ✅ Protects cloud variable updates
- ✅ Provides visual feedback through LEDs
- ✅ Works without a USB cable connected

Any questions about the operation, consult this documentation or the source code comments.
