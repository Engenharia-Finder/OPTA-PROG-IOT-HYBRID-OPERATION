# Documentação do Código - Sistema IoT Cloud Offline

Este documento explica as principais partes do código, com foco na função de thread e na lógica de funcionamento offline.

---

## 📋 Índice
1. [Função de Thread (cloudThreadFunc)](#função-de-thread-cloudthreadfunc)
2. [Lógica de Funcionamento Offline do Loop](#lógica-de-funcionamento-offline-do-loop)
3. [Sistema de Callbacks de Conectividade](#sistema-de-callbacks-de-conectividade)
4. [Variáveis de Controle Offline](#variáveis-de-controle-offline)
5. [Setup e Inicialização](#setup-e-inicialização)

---

## 🔄 Função de Thread (cloudThreadFunc)

### Localização: Linhas 35-46

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

### Explicação Detalhada:

**Propósito:**
- Esta função roda em uma **thread separada** (thread secundária), independente do `loop()` principal
- Executa continuamente o `ArduinoCloud.update()`, que mantém a comunicação com a Arduino IoT Cloud

**Por que usar uma thread separada?**
1. **Não bloqueia o loop principal**: Se a conexão com a cloud travar ou demorar (ex: problemas de DHCP, timeout de rede), o `loop()` principal continua executando normalmente
2. **Funciona sem USB**: A thread funciona mesmo quando não há cabo USB conectado, permitindo operação totalmente autônoma
3. **Isolamento de problemas**: Problemas de rede não afetam a lógica principal do dispositivo

**Como funciona:**
- `while (true)`: Loop infinito que mantém a thread sempre ativa
- `ArduinoCloud.update()`: Atualiza a comunicação com a cloud, enviando e recebendo dados
- `rtos::ThisThread::sleep_for(10)`: Pausa de 10ms para ceder CPU para outras threads, evitando consumo excessivo de recursos

**Inicialização da Thread:**
- A thread é criada na linha 33: `rtos::Thread cloudThread(osPriorityNormal, 8192);`
  - `osPriorityNormal`: Prioridade normal da thread
  - `8192`: 8KB de memória stack para a thread
- A thread é iniciada na linha 95: `cloudThread.start(cloudThreadFunc);`

---

## 🔁 Lógica de Funcionamento Offline do Loop

### Localização: Linhas 102-146

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

### Explicação Detalhada:

#### 1. **Verificação Periódica de Conectividade (Linhas 104-108)**

```cpp
static unsigned long ultimaVerificacao = 0;
if ((millis() - ultimaVerificacao) >= 10000) {
  verificarStatusConectividade();
  ultimaVerificacao = millis();
}
```

**O que faz:**
- Verifica o status de conexão com a cloud a cada **10 segundos**
- Usa `millis()` para não bloquear o código (não usa `delay()`)
- Chama `verificarStatusConectividade()` que verifica se houve mudança no status de conexão

**Por que é importante:**
- Serve como **backup** caso os callbacks não sejam chamados
- Garante que o sistema detecte mudanças de conectividade mesmo se os callbacks falharem
- Não bloqueia o loop, permitindo que o código principal continue executando

#### 2. **Proteção de Atualização de Variáveis (Linhas 120-143)**

```cpp
if(cloudConectado && cloudSincronizado) {
  // Atualiza variáveis da cloud apenas aqui
}
```

**O que faz:**
- **Só atualiza variáveis da cloud** quando AMBAS as condições são verdadeiras:
  - `cloudConectado == true`: Dispositivo está conectado à cloud
  - `cloudSincronizado == true`: Sincronização inicial foi concluída

**Por que essa proteção é crítica:**
1. **Evita travamentos**: Se tentar atualizar variáveis antes da sincronização, o dispositivo pode travar ao reiniciar
2. **Garante integridade**: Só envia dados quando a conexão está totalmente estabelecida
3. **Modo offline-first**: O código principal continua funcionando mesmo sem conexão

**Exemplo de uso (Linhas 122-127):**
```cpp
//FEEDBACK DE DISJUNTOR ABERTO
if(fbAbertura == HIGH){
  feedbackabertura = HIGH;
}else{
  feedbackabertura = LOW;
}
```
- Essas atualizações só acontecem quando `cloudConectado && cloudSincronizado` são verdadeiros
- Fora dessa condição, o dispositivo funciona normalmente, mas não tenta enviar dados para a cloud

---

## 📡 Sistema de Callbacks de Conectividade

### Localização: Linhas 161-190

O sistema usa três callbacks principais para monitorar o status de conexão:

#### 1. **onCloudConnect() - Linhas 162-172**

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

**O que faz:**
- Chamado quando o dispositivo **conecta** à Arduino IoT Cloud
- Marca `cloudConectado = true` mas **ainda não sincronizado** (`cloudSincronizado = false`)
- Atualiza LEDs: LED azul ligado, LED vermelho desligado
- Reseta contador de tentativas

#### 2. **onCloudSync() - Linhas 174-177**

```cpp
void onCloudSync() {
  // Sincronização concluída - agora é seguro atualizar variáveis da cloud
  cloudSincronizado = true;
}
```

**O que faz:**
- Chamado quando a **sincronização inicial** com a cloud é concluída
- Marca `cloudSincronizado = true`, permitindo que o loop atualize variáveis
- **Crítico**: Só após este callback é seguro atualizar variáveis da cloud

#### 3. **onCloudDisconnect() - Linhas 179-190**

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

**O que faz:**
- Chamado quando o dispositivo **desconecta** da cloud
- Marca `modoOffline = true` e reseta flags de conexão
- Atualiza LEDs: LED azul desligado, LED vermelho ligado (indica modo offline)
- O dispositivo continua funcionando normalmente, mas não envia dados para a cloud

---

## 🔧 Variáveis de Controle Offline

### Localização: Linhas 25-30

```cpp
bool modoOffline = false;           // Indica se está em modo offline
bool cloudConectado = false;        // Indica se está conectado à cloud
bool cloudSincronizado = false;    // Indica se sincronização inicial foi concluída
unsigned long ultimaConexaoCloud = 0; // Timestamp da última conexão
int tentativasConexao = 0;          // Contador de tentativas de conexão
```

**Explicação:**
- **modoOffline**: Flag principal que indica modo offline (true) ou online (false)
- **cloudConectado**: Indica se há conexão ativa com a cloud
- **cloudSincronizado**: Indica se a sincronização inicial foi concluída (crítico para atualizar variáveis)
- **ultimaConexaoCloud**: Armazena quando foi a última conexão bem-sucedida
- **tentativasConexao**: Contador para gerenciar tentativas de reconexão

---

## ⚙️ Setup e Inicialização

### Localização: Linhas 48-100

#### Principais etapas:

1. **Inicialização Serial (Linha 51)**
   ```cpp
   Serial.begin(9600);
   ```

2. **Configuração de LEDs (Linhas 54-57)**
   - LED azul (LED_USER): Indica conexão com cloud
   - LED vermelho (LEDR): Indica modo offline
   - Inicialmente: LED azul desligado, LED vermelho ligado (modo offline)

3. **Inicialização do Cloud (Linha 71)**
   ```cpp
   ArduinoCloud.begin(ArduinoIoTPreferredConnection, false);
   ```
   - `false`: Desabilita watchdog interno que causa reinícios

4. **Registro de Callbacks (Linhas 82-84)**
   ```cpp
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::CONNECT, onCloudConnect);
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::SYNC, onCloudSync);
   ArduinoCloud.addCallback(ArduinoIoTCloudEvent::DISCONNECT, onCloudDisconnect);
   ```

5. **Inicialização Offline-First (Linhas 88-90)**
   ```cpp
   modoOffline = true;
   cloudConectado = false;
   cloudSincronizado = false;
   ```
   - Sistema inicia em modo offline por padrão
   - Garante que o dispositivo funcione mesmo sem conexão

6. **Início da Thread (Linha 95)**
   ```cpp
   cloudThread.start(cloudThreadFunc);
   ```
   - Inicia a thread que gerencia a comunicação com a cloud

---

## 🎯 Resumo do Fluxo de Funcionamento

1. **Inicialização**: Sistema inicia em modo offline, thread da cloud é iniciada
2. **Thread da Cloud**: Executa `ArduinoCloud.update()` continuamente em background
3. **Loop Principal**: 
   - Verifica conectividade a cada 10 segundos
   - Executa lógica principal do dispositivo
   - Só atualiza variáveis da cloud se `cloudConectado && cloudSincronizado`
4. **Callbacks**: Monitoram mudanças de status e atualizam flags e LEDs
5. **Modo Offline**: Dispositivo continua funcionando normalmente mesmo sem conexão

---

## ⚠️ Pontos Importantes

1. **Nunca atualize variáveis da cloud fora da condição `if(cloudConectado && cloudSincronizado)`**
   - Isso pode causar travamentos ao reiniciar o dispositivo

2. **A thread da cloud funciona independente do loop principal**
   - Problemas de rede não afetam a lógica principal

3. **O sistema é offline-first**
   - Funciona normalmente mesmo sem conexão com a cloud
   - Apenas não envia dados quando offline

4. **A sincronização é crítica**
   - Só atualize variáveis após `cloudSincronizado == true`
   - Isso garante que a conexão está totalmente estabelecida

---

## 📝 Notas Finais

Este código implementa um sistema robusto de IoT que:
- ✅ Funciona offline-first (não depende de conexão)
- ✅ Usa threads para não bloquear o loop principal
- ✅ Protege atualizações de variáveis da cloud
- ✅ Fornece feedback visual através de LEDs
- ✅ Funciona sem cabo USB conectado

Qualquer dúvida sobre o funcionamento, consulte esta documentação ou os comentários no código fonte.
