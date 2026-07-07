# SEL0433 - Projeto 3: Controle PWM e Comunicação

**Integrantes:** Vinicius Gryczak Domingos - 15698383

## 1. Visão geral

Este repositório contém as três aplicações do Projeto 3: controle PWM de um LED RGB com a biblioteca LEDC, controle manual de um servomotor por potenciômetro com a biblioteca ESP32Servo e uma aplicação própria com a biblioteca nativa MCPWM - uma cancela automática de estacionamento.

As três partes compartilham a mesma base: a geração de PWM por periféricos dedicados do ESP32, comunicação serial UART a 115200 para monitoramento e validação integral no simulador Wokwi.

## 2. Parte 1 - PWM em LED RGB (LEDC)

**Simulação:** https://wokwi.com/projects/468840762843191297

Um LED RGB de catodo comum é ligado às GPIOs 32 (R), 33 (G) e 25 (B) por resistores de 220 Ω. Cada cor usa um canal PWM independente do periférico LEDC, configurado com resolução de 8 bits (0-255) e frequência de 5 kHz. O duty cycle de cada canal varia de 0 a 100 % em loop, com taxas independentes por cor. Verde +4 %, azul +8 % e vermelho +12 % por passo, a cada 150 ms.

Trecho central: no núcleo 3.x, `ledcAttach(pino, freq, res)` substitui o par `ledcSetup`/`ledcAttachPin` das versões 2.x, e `ledcWrite` passa a ser endereçado por pino:

```cpp
ledcAttach(PIN_R, PWM_FREQ, PWM_RES);   // 5 kHz, 8 bits
ledcWrite(PIN_R, pctParaRaw(dutyR));    // duty em % convertido para 0..255
dutyR += INC_R; if (dutyR > 100) dutyR = 0;
```

A frequência de 5 kHz fica acima da percepção visual, e com 8 bits o LEDC comporta até ~312 kHz.

<img width="871" height="678" alt="docsparte1_circuito" src="https://github.com/user-attachments/assets/d0fef18b-c99d-4dd7-9acb-c7e48398cb88" />

<img width="1913" height="728" alt="docsparte1_demo" src="https://github.com/user-attachments/assets/9245936b-f41c-476c-b61c-88a5b129d7a1" />

## 2. Parte 2a - Servomotor + potenciômetro (ESP32Servo)

**Simulação:** https://wokwi.com/projects/468841748992306177

O ADC de 12 bits lê a tensão do potenciômetro na GPIO35 (ADC1, somente entrada) e o valor 0-4095 é mapeado para o ângulo do servo na GPIO23. A biblioteca ESP32Servo gera o PWM de 50 Hz com pulsos de 0,5-2,4 ms. Internamente ela também usa o periférico LEDC.

```cpp
adcFiltrado = (3 * adcFiltrado + adc) / 4;        // media movel simples
int angulo = map(adcFiltrado, 0, 4095, 0, 180);   // ADC -> posicao
servo.write(angulo);
```
<img width="1190" height="712" alt="docsparte2a_circuito" src="https://github.com/user-attachments/assets/5ae2272f-7644-4991-bd64-4dff38c689fb" />

<img width="1913" height="728" alt="docsparte2a_demo" src="https://github.com/user-attachments/assets/85c8409f-58ef-42a1-82ff-0fc7c5ce9586" />

## 4. Parte 2b - Aplicação própria com MCPWM: cancela automática

**Simulação:** https://wokwi.com/projects/468841733639052289

Uma cancela de estacionamento em que o periférico MCPWM gera o sinal de 50 Hz que posiciona o servo do braço (GPIO19). O sistema integra HC-SR04 para detecção do veículo (TRIG 17 / ECHO 16), botão com interrupção externa e debounce por software (GPIO25, modo manual), buzzer via LEDC (GPIO27, sinalização de movimento), LEDs de estado (32/33), display OLED SSD1306 via I2C (0x3C, SDA 21 / SCL 22) com estado, distância, ângulo e contagem, e UART para telemetria e registro de eventos.

Máquina de estados: `FECHADA → ABRINDO → ABERTA → FECHANDO`, com dois comportamentos de segurança/consistência. Se um veículo é detectado durante o fechamento, a cancela reabre; e apenas aberturas disparadas por veículo (distância < 45 cm) incrementam o contador. A cancela permanece aberta enquanto houver veículo na área e fecha 4 s após ela ficar livre.

No MCPWM, o timer conta em ticks de 1 µs com período de 20 000 ticks (50 Hz). O comparador define o instante de descida do sinal.

```cpp
tcfg.resolution_hz = 1000000;  // 1 tick = 1 us
tcfg.period_ticks  = 20000;    // 20 ms -> 50 Hz
// sobe no inicio do periodo, desce ao atingir o comparador:
mcpwm_generator_set_action_on_timer_event(gen, MCPWM_GEN_TIMER_EVENT_ACTION(
    MCPWM_TIMER_DIRECTION_UP, MCPWM_TIMER_EVENT_EMPTY, MCPWM_GEN_ACTION_HIGH));
mcpwm_generator_set_action_on_compare_event(gen, MCPWM_GEN_COMPARE_EVENT_ACTION(
    MCPWM_TIMER_DIRECTION_UP, comp, MCPWM_GEN_ACTION_LOW));
mcpwm_comparator_set_compare_value(comp, pulso_us); // largura do pulso
```

<img width="798" height="747" alt="docsparte2b_circuito" src="https://github.com/user-attachments/assets/4a4d39cd-2818-424e-9111-14659fd635be" />

<img width="1913" height="876" alt="docsparte2b_demo" src="https://github.com/user-attachments/assets/fafd6901-3e57-4760-9ae3-164ed7f2f0b1" />

## 5. Bibliotecas empregadas

| Biblioteca | Uso | Observação |
|---|---|---|
| LEDC (núcleo Arduino-ESP32) | PWM do LED RGB e do buzzer | API 3.x por pino (`ledcAttach`) |
| ESP32Servo | Servo da Parte 2a | Usa o LEDC internamente |
| MCPWM (`mcpwm_prelude.h`, ESP-IDF) | Servo da cancela (Parte 2b) | Periférico dedicado ao controle de motores |
| Adafruit SSD1306 + Adafruit GFX | Display OLED via I2C | Endereço 0x3C, framebuffer enviado no `display()` |

O LEDC foi adotado onde o objetivo é intensidade média (LED e buzzer); o MCPWM, periférico voltado ao controle de motores, ficou responsável pelo servo da cancela, o que permite demonstrar os dois blocos de PWM do ESP32 operando em conjunto.

## 6. Discussão - quando um microcontrolador de 32 bits se justifica

A utilização do ESP32 mostrou-se justificada neste projeto principalmente pelo conjunto de periféricos dedicados, e não pela capacidade de processamento. A geração de PWM ficou a cargo dos periféricos LEDC e MCPWM, a aquisição analógica do ADC de 12 bits e a comunicação com o display do barramento I2C, e o processador só a lógica de controle.

A escolha da plataforma deve decorrer dos requisitos da aplicação. Para sistemas de acionamento simples, microcontroladores de 8 bits atendem com custo e consumo inferiores. No caso deste projeto, em que display, sensor, comunicação sem fio e controle de motor operam simultaneamente, o ESP32 representa a solução de menor complexidade que cumpre os requisitos com margem adequada.
