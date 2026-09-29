# Engenharia Reversa de Sensor de Corrente Tuya

## Contexto

Discussão sobre sensores/medidores de corrente Tuya de baixo custo, encontrados em marketplaces como AliExpress, com o objetivo de compreender a arquitetura do equipamento, interceptar a comunicação interna e eventualmente substituir o firmware para enviar os dados a um servidor próprio.

## 1. Arquitetura provável do equipamento

Esses dispositivos Tuya frequentemente possuem duas partes:

1. **Placa/controlador de medição**
   - Faz a aquisição dos sinais elétricos.
   - Pode medir corrente, tensão, potência, energia acumulada e outros parâmetros.
   - Pode controlar relés e outras funções locais.
   - Possui um microcontrolador ou CI dedicado à medição.

2. **Módulo de comunicação Wi-Fi**
   - Recebe informações do controlador principal.
   - Faz a comunicação com a rede Wi-Fi.
   - No firmware original, normalmente encaminha informações para o ecossistema Tuya.
   - Dependendo do produto, pode utilizar chips ESP8266/ESP8285, Beken, Realtek ou outras famílias.

Arquitetura conceitual:

```text
Sensor de corrente
       |
       v
Circuito/MCU de medição
       |
       | UART
       v
Módulo Wi-Fi Tuya
       |
       v
Rede / Internet / Tuya Cloud
```

## 2. Dois conjuntos de TX/RX

Foi observado que o equipamento possui dois conjuntos aparentemente diferentes de TX/RX:

- Um conjunto na placa principal, em uma região que aparenta ser utilizada para programação, teste ou diagnóstico de fábrica.
- Outro conjunto associado à placa/módulo de comunicação Wi-Fi.

Isso pode indicar interfaces UART distintas.

Uma hipótese de arquitetura é:

```text
Pads de fábrica
      |
      v
MCU principal
      |
      | UART de comunicação
      v
Módulo Wi-Fi
```

Os pads de fábrica podem estar associados a programação, debug, calibração ou diagnóstico. A função exata só pode ser determinada após identificar os componentes e rastrear as conexões da placa.

## 3. UART, níveis TTL/CMOS e RS-232

### UART

UART (*Universal Asynchronous Receiver/Transmitter*) é o mecanismo de comunicação serial assíncrona.

Normalmente utiliza:

- TX — transmissão;
- RX — recepção;
- GND — referência elétrica.

Não existe uma linha de clock separada. Os dois equipamentos precisam utilizar parâmetros compatíveis, principalmente a taxa de transmissão (*baud rate*).

Exemplos:

- 9.600 baud
- 19.200 baud
- 115.200 baud

### TTL/CMOS

TTL/CMOS, nesse contexto, refere-se aos níveis elétricos utilizados pelos sinais digitais.

Em módulos modernos é comum encontrar UART trabalhando em 3,3 V, embora isso precise ser medido antes de conectar equipamentos externos.

Portanto:

```text
UART = forma/protocolo lógico da comunicação serial
TTL/CMOS = níveis elétricos usados nos sinais
```

### RS-232

RS-232 utiliza uma camada elétrica diferente, tradicionalmente com tensões positivas e negativas e características incompatíveis com a entrada lógica direta de muitos microcontroladores.

Assim, uma porta RS-232 clássica não deve ser conectada diretamente aos pinos RX/TX de um microcontrolador sem o circuito conversor adequado.

## 4. Interceptação da comunicação

Uma estratégia interessante é inicialmente **não modificar firmware algum**.

Pode-se observar a comunicação existente entre o controlador de medição e o módulo Wi-Fi.

Exemplo:

```text
MCU de medição ---- TX ----> módulo Wi-Fi
                        |
                        +----> analisador lógico / RX de monitoramento
```

Dessa forma é possível capturar os bytes enviados pelo controlador.

Durante a captura podem ser realizados testes controlados, como:

- ligar e desligar uma carga;
- variar a potência da carga;
- observar alterações na tensão;
- observar alterações na corrente;
- acompanhar o aumento da energia acumulada;
- acionar o relé, quando existente.

A comparação dos frames pode revelar quais campos representam cada variável.

## 5. Protocolo Tuya e Datapoints

Muitos equipamentos Tuya utilizam comunicação serial estruturada entre o MCU principal e o módulo de comunicação.

Os dados podem ser representados através de **datapoints (DPs)**.

Dependendo do equipamento, podem existir DPs correspondentes a:

- tensão;
- corrente;
- potência;
- energia acumulada;
- estado do relé;
- alarmes;
- configurações.

O número, formato, escala e significado dos datapoints podem variar entre produtos.

Uma das etapas da engenharia reversa é construir um mapa semelhante a:

```text
DP XX -> tensão
DP YY -> corrente
DP ZZ -> potência
DP WW -> energia acumulada
```

Esses números são apenas ilustrativos até que o protocolo específico do equipamento seja capturado.

## 6. Servidor próprio

Depois de compreender a comunicação serial, é possível substituir ou complementar a função do módulo Wi-Fi.

Arquitetura desejada:

```text
MCU de medição
      |
      | UART / protocolo Tuya
      v
Firmware próprio
      |
      | Wi-Fi
      v
MQTT ou HTTP
      |
      v
Servidor próprio
```

Isso permitiria retirar a dependência da nuvem Tuya.

Um exemplo de estrutura MQTT seria:

```text
cmaker/energia/sensor01/tensao
cmaker/energia/sensor01/corrente
cmaker/energia/sensor01/potencia
cmaker/energia/sensor01/energia
```

## 7. Possibilidades de firmware

Dependendo do chip encontrado no módulo Wi-Fi, podem existir diferentes caminhos:

- firmware próprio;
- OpenBeken;
- LibreTiny;
- outras soluções compatíveis com a família específica do microcontrolador.

Antes de escolher uma solução é necessário identificar exatamente:

- modelo do módulo Wi-Fi;
- chip utilizado;
- pinagem;
- método de programação;
- tensão lógica;
- protocolo entre os dois controladores.

## 8. Estratégia recomendada de investigação

Uma sequência adequada para o projeto é:

1. Fotografar frente e verso da placa.
2. Identificar o MCU/CI de medição.
3. Identificar o chip e o modelo do módulo Wi-Fi.
4. Rastrear os dois conjuntos TX/RX.
5. Medir as tensões antes de conectar qualquer equipamento.
6. Verificar se existe isolamento galvânico em relação à rede elétrica.
7. Capturar passivamente a UART.
8. Descobrir baud rate e parâmetros da serial.
9. Registrar os frames em diferentes condições de carga.
10. Identificar o protocolo e os datapoints.
11. Construir um decodificador.
12. Publicar os dados em MQTT ou HTTP.
13. Somente depois avaliar substituição do firmware original.

## 9. Captura passiva

Para uma primeira investigação, a ideia é evitar transmitir qualquer coisa para o equipamento.

Conceitualmente:

```text
TX do Tuya --------> RX do analisador/monitor
GND ----------------> referência do analisador
```

O TX do equipamento de análise permanece desconectado nessa etapa.

Isso reduz o risco de interferência lógica na comunicação original.

## 10. Segurança elétrica

Este é um ponto crítico.

Medidores de energia e sensores de corrente conectados diretamente à rede podem utilizar fontes **não isoladas galvanicamente**.

Nesse caso, o GND da UART pode estar eletricamente relacionado à rede.

Portanto, não se deve assumir que os pads TX/RX/GND são seguros para conexão direta a:

- computador;
- notebook;
- USB-UART;
- osciloscópio aterrado;
- analisador lógico conectado via USB.

Antes da conexão é necessário verificar a arquitetura elétrica e o isolamento do equipamento.

## Próximo passo

Quando houver fotografias nítidas da placa, registrar:

- frente da placa completa;
- verso da placa completa;
- aproximação do primeiro TX/RX;
- aproximação do segundo TX/RX;
- inscrições dos circuitos integrados;
- identificação do módulo Wi-Fi.

Com essas informações será possível montar o mapa da placa e definir uma estratégia mais precisa para captura da UART e desenvolvimento do firmware/servidor próprio.
