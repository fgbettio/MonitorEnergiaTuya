# Documentação Técnica e Engenharia Reversa: Medidor de Corrente Tuya (PJ-1103C)

---

## 1. Visão Geral do Dispositivo

* **Modelo:** PJ-1103C (versão para medição com transformadores de corrente / clamps A e B).
* **Alimentação:** AC 100–240V 50/60Hz.
* **Faixa de Corrente Suportada:** 0.2A a 80A.
* **Conectividade:** Wi-Fi 802.11 b/g/n + Bluetooth LE (BLE v5.2).
* **Bornes de Conexão:**
* `L` e `N`: Entrada de alimentação AC (Fase e Neutro).
* `S1` e `S2` (Canal A): Leitura do transformador de corrente (TC/Clamp) do canal 1.
* `S1` e `S2` (Canal B): Leitura do transformador de corrente (TC/Clamp) do canal 2.

---

## 2. Levantamento e Mapeamento de Componentes

### 2.1. Processamento e Medição

* **Microcontrolador Principal (MCU):** `Nation N32G430C8L7`.
* Arquitetura: ARM Cortex-M4F de 32 bits (frequência de até 64 MHz, 64 KB Flash, 16 KB SRAM).
* Função: Gerencia o fluxo de trabalho local, controla periféricos, lê o CI de medição e comunica com o módulo de rádio via UART.
* **CI de Medição de Energia:** `HLW8112`.
* Função: Front-end analógico/digital para medição de grandezas elétricas (Tensão RMS, Corrente RMS nos canais A e B, Potência Ativa, Potência Reativa, Fator de Potência).
* Interface: Comunicação com a MCU Nations via SPI ou UART.

### 2.2. Módulo de Rádio / Conectividade

* **Módulo:** `T1-M 101` (Placa filha vertical conectada via conector SMD/solda).
* **SoC Integrado:** Beken BK7238 (Wi-Fi + BLE) sob blindagem metálica.
* **Pinagem exposta na serigrafia traseira do módulo:**
* `3V3`: Alimentação positiva de 3.3V DC.
* `GND`: Referência de terra.
* `RX1` / `TX1` (ou `XTX1`): Barramento UART primário (comunicação com a MCU Nations via TuyaMCU).
* `TX2`: UART secundária / depuração de firmware.
* `P24`: GPIO de uso geral / modo de boot.

### 2.3. Alimentação, Filtragem e Proteção

* **Varistor (MOV):** `JK-ET 7S471K` (470V, 7 mm) para proteção contra surtos e transientes na linha AC.
* **Indutor de Potência:** Bobina indutora marcada como `681` (680 µH) do conversor DC-DC interno.
* **Capacitor Primário de Alta Tensão:** Eletrolítico 400V 4.7µF para retificação primária da rede.
* **Capacitores de Filtragem DC:** Dois capacitores eletrolíticos SMD de 10V 220µF.
* **Alarme Sonoro:** Buzzer piezoelétrico para avisos de sobrecarga e status.

---

## 3. Interfaces de Depuração e Programação Expostas na PCB

No verso da placa de circuito impresso (PCB) principal encontram-se expostos os seguintes pontos de teste (*test points*):

1. **Barramento SWD (Gravação da MCU ARM Cortex-M4):**

* `+` : VCC (3.3V)
* `D` : SWDIO (Serial Wire Data Input/Output)
* `C` : SWCLK (Serial Wire Clock)
* `G` : GND

2. **Barramento UART da PCB Principal:**

* Pads `TX`, `RX` e `GND` para monitoramento serial de dados entre a MCU e o módulo de rádio.

---

## 4. Desvinculação da Nuvem Tuya e Firmware Customizado

A placa não utiliza microcontrolador ESP8266/ESP32, mas a sua arquitetura modular permite autonomia total por diferentes caminhos:

### Opção A: Regravação do SoC Beken BK7238 (Sem modificação física)

* **Conceito:** O chip Nation N32G430 continua operando o código original, amostrando o HLW8112 e transmitindo os dados via protocolo TuyaMCU pela serial.
* **Ação:**

1. Conectar um conversor USB-Serial (nível lógico 3.3V) nos pads `3V3`, `GND`, `TX1` e `RX1` do módulo vertical T1-M.
2. Gravar um firmware alternativo de código aberto, como o **OpenBeken** (OpenBK7231 / BK7238), ou um binário C próprio baseado no SDK FreeRTOS da Beken.
3. Configurar a interface serial para decodificar os pacotes TuyaMCU e publicar as métricas diretamente via **MQTT** ou **HTTP** local para servidores próprios ou Home Assistant.

### Opção B: Substituição do Módulo de Rádio por ESP32 / ESP8266

* **Conceito:** Aproveitar códigos pré-existentes no ecossistema Espressif (ESP-IDF, Arduino Core, PlatformIO).
* **Ação:**

1. Dessoldar a placa filha vertical `T1-M`.
2. Soldar nos mesmos pontos um módulo compacto (ex.: ESP32-C3 SuperMini ou ESP-12F) interligando apenas **3.3V**, **GND**, **TX** e **RX**.
3. O ESP lê o tráfego serial da MCU principal e publica os dados na rede local.

### Opção C: Regravação Bare-Metal da MCU Nations N32G430

* **Conceito:** Controle integral de ponta a ponta sem qualquer resquício de código de fábrica.
* **Ação:**

1. Ligar um gravador **ST-Link v2**, **J-Link** ou **DAPLink** nos pads `+`, `D`, `C` e `G` da placa principal.
2. Apagar a flash original e gravar um firmware próprio compilado com a *toolchain* `arm-none-eabi-gcc`.
3. Implementar diretamente o driver de leitura dos registradores de amostragem do chip `HLW8112` via SPI/UART.

---

## 5. Por Que o Hardware é Tão Acessível?

* **Linha de Montagem Industrial (ICT/FCT):** Os pontos de teste (SWD, UART) são mantidos abertos para permitir a gravação de lote em gabaritos de agulhas (*bed-of-nails*) e testes de calibração automática de tensão/corrente na fábrica.
* **Otimização de Custos:** A remoção de serigrafia a laser (*IC de-marking*) ou a aplicação de resina epóxi aumentaria o tempo e os custos do ciclo de fabricação em um produto de margem financeira muito reduzida.
* **Modelo de Negócio e Ameaça:** O foco de segurança dos dispositivos domésticos é concentrado em proteger conexões remotas via internet. A indústria assume que o acesso físico com solda na bancada descaracteriza a premissa de violação em escala. A memória flash da MCU costuma conter apenas *Readout Protection (RDP)* ativado (que impede a extração do código proprietário, mas não bloqueia o apagamento total e a regravação).

---

## 6. Procedimentos Obrigatórios de Segurança de Bancada

> ⚠️ **PERIGO: RISCO DE CHOQUE ELÉTRICO E DESTRUIÇÃO DE EQUIPAMENTO**
>
> * A fonte primária desta placa utiliza arquitetura chaveada/buck **não isolada galvanicamente** da rede elétrica alternada.
> * **JAMAIS conecte** gravadores ST-Link, conversores USB-Serial, cabos de osciloscópio ou PCs aos pads de programação enquanto os bornes `L` e `N` estiverem conectados à tomada AC (110V/220V).
> * Durante qualquer processo de depuração, teste ou gravação serial/SWD, a placa deve estar **completamente desconectada da rede AC** e ser alimentada estritamente por uma fonte externa de **3.3V regulada**.
