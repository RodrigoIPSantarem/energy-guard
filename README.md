# energy-guard

Medidor de energia para o quadro elétrico com ESP32 e PZEM-004T. Mede o consumo total da casa, publica os valores por MQTT e avisa os dispositivos antes de se atingir a potência contratada, para evitar que o quadro dispare.

Projeto desenvolvido no âmbito da unidade curricular de Internet das Coisas (Licenciatura em Engenharia Informática), com base num medidor de energia de baixo custo descrito na literatura (ver [Referências](#referências)).

## Aviso de segurança

Este projeto envolve a rede elétrica de 230 V. Uma ligação mal feita pode causar choque elétrico, curto-circuito ou incêndio.

- A instalação no quadro elétrico deve ser feita ou validada por um eletricista certificado.
- Corta sempre a alimentação no disjuntor antes de mexer em qualquer ligação.
- Todas as partes em tensão de rede devem ficar dentro de uma caixa fechada e isolada. Nunca deixes fase ou neutro expostos.
- Este sistema não substitui o disjuntor, que continua a ser a proteção real da instalação.
- O autor não se responsabiliza por danos resultantes da reprodução deste projeto.

## Objetivo

Em instalações com potência contratada limitada, ligar vários equipamentos em simultâneo (por exemplo, o carregador do carro e o forno) pode fazer disparar o disjuntor geral. O objetivo deste projeto é medir o consumo total em tempo real e, antes de o limite ser atingido, enviar mensagens aos dispositivos para que reduzam ou desliguem cargas de forma ordenada.

## Funcionalidades

| Funcionalidade | Estado |
|---|---|
| Leitura de tensão, corrente, potência ativa, frequência e fator de potência | Planeado |
| Publicação dos valores por MQTT | Planeado |
| Níveis de alerta (aviso, crítico, reposição) com base na potência contratada | Planeado |
| Envio de comandos aos dispositivos por ordem de prioridade | Planeado |
| Painel de monitorização no Node-RED | Planeado |
| Limite local no ESP32, que atua mesmo sem Wi-Fi ou broker | Planeado |
| Página web de configuração (Wi-Fi, MQTT, limites) | A avaliar |

Atualizar esta tabela à medida que cada parte for concluída.

## Arquitetura

```
Quadro elétrico
   |
   |  pinça (corrente) + terminais de tensão
   v
PZEM-004T  --UART-->  ESP32  --Wi-Fi/MQTT-->  Broker Mosquitto
                                                    |
                                                    v
                                                Node-RED
                                          (limites, painel, comandos)
                                                    |
                                                    v
                                      Dispositivos (tomadas, carregador, ...)
```

- **ESP32:** lê o PZEM por UART, publica os valores e aplica um limite local de segurança.
- **Mosquitto:** broker MQTT que liga o medidor, o Node-RED e os dispositivos.
- **Node-RED:** recebe os valores, compara com os limites e publica os comandos.

## Material

| Componente | Função |
|---|---|
| ESP32 (clássico, WROOM-32, 38 pinos) | Microcontrolador com Wi-Fi |
| PZEM-004T V3.0 com pinça aberta (100 A) | Medição de tensão, corrente, potência e fator de potência |
| HLK-5M05 (Hi-Link, 5 V, 1 A) | Fonte isolada que converte 230 V AC em 5 V DC |
| Fusível de 1 A, ação lenta, 250 V, com porta-fusíveis | Proteção do circuito de alimentação |
| Varistor S14K275 (275 V AC) | Proteção contra picos na rede |
| Placa perfurada de fibra de vidro (FR4), passo 2,54 mm | Montagem do módulo de alimentação |
| Blocos de terminais de parafuso (passo 5,08 mm) | Ligações de entrada e saída |
| 2 resistências para divisor de tensão (por exemplo, 1 kΩ e 2 kΩ) | Adaptação do nível lógico do TX do PZEM, se necessário |
| Caixa isolada fechada (calha DIN ou ABS) | Proteção contra contacto com a parte de 230 V |
| Disjuntor pequeno (2 A ou 6 A) | Alimentação dedicada ao medidor, no quadro |

Este projeto não usa ecrã (OLED). O estado é consultado por MQTT e no painel.

## Ligações

### Lado da rede (230 V)

1. A fase passa pelo fusível e segue para um dos pinos AC do HLK-5M05.
2. O neutro segue para o outro pino AC do HLK-5M05.
3. O varistor liga entre fase e neutro, depois do fusível.
4. Os terminais de tensão do PZEM ligam à mesma fase e ao mesmo neutro.
5. A pinça abraça **apenas o condutor de fase** da entrada geral, respeitando o sentido da seta marcada. Se abraçar fase e neutro em simultâneo, a leitura é zero.

### Lado de 5 V

| Origem | Destino |
|---|---|
| HLK-5M05 +Vo | VIN do ESP32 e 5 V do PZEM |
| HLK-5M05 -Vo | GND do ESP32 e GND do PZEM |
| TX do PZEM | GPIO16 (RX2) do ESP32 |
| RX do PZEM | GPIO17 (TX2) do ESP32 |

O PZEM comunica na UART2 do ESP32, deixando a UART0 livre para programação e depuração.

### Nível lógico do TX do PZEM

O TX do PZEM pode sair com nível próximo de 5 V, e os pinos do ESP32 não são oficialmente tolerantes a 5 V. Antes de ligar, medir com o multímetro o nível do TX do PZEM (apenas com os 5 V ligados e sem rede no módulo). Se estiver perto de 5 V, usar um divisor de tensão no fio TX do PZEM para o RX do ESP32.

### Pinos a evitar no ESP32

- GPIO6 a GPIO11 (memória flash).
- GPIO0, 2, 5, 12 e 15 para sinais externos, porque influenciam o arranque.
- RX0 e TX0 (UART de programação).

## Limites de alerta

Os limites devem ser definidos em corrente (ampere), porque é a corrente que faz disparar o disjuntor. A conversão a partir da potência contratada é:

```
I = P / U
```

Exemplo, para 6,9 kVA a 230 V (cerca de 30 A):

| Nível | Percentagem do limite | Corrente (exemplo) | Ação |
|---|---|---|---|
| Aviso | cerca de 80 % | cerca de 24 A | Pedir aos equipamentos menos importantes que reduzam ou desliguem |
| Crítico | cerca de 90 % | cerca de 27 A | Desligar mais cargas |
| Reposição | abaixo de cerca de 70 %, durante alguns minutos | cerca de 21 A | Voltar a ligar as cargas |

Notas:

- A reposição só deve ocorrer depois de o consumo estar abaixo do limite durante algum tempo, para evitar que os equipamentos liguem e desliguem em ciclo (histerese).
- As leituras devem ser filtradas com uma média de alguns segundos, porque os picos de arranque de motores e compressores são curtos e não devem disparar o alerta.
- Os valores acima são um exemplo. Ajustar à potência contratada e à instalação em causa.

## Comunicação MQTT

Proposta de tópicos (a confirmar durante o desenvolvimento):

| Tópico | Sentido | Conteúdo |
|---|---|---|
| `energia/quadro/estado` | ESP32 para broker | Tensão, corrente, potência, fator de potência e nível de alerta |
| `energia/quadro/alerta` | Node-RED para dispositivos | Nível de alerta atual (aviso, crítico, normal) |
| `energia/dispositivos/<nome>/comando` | Node-RED para dispositivo | Reduzir, desligar ou repor |

- Usar QoS 1 nos comandos aos dispositivos.
- Usar a mensagem de última vontade (Last Will) para detetar quando o medidor fica offline.
- Prioridade de corte sugerida: primeiro o carregador do carro, depois o termoacumulador, depois a máquina de lavar loiça.

## Estado do projeto

- [x] Escolha dos componentes
- [x] Compra do material
- [ ] Teste da fonte de 5 V (apenas com o HLK-5M05, sem o resto ligado)
- [ ] Leitura do PZEM no ESP32 (monitor série)
- [ ] Ligação Wi-Fi e publicação por MQTT
- [ ] Fluxo no Node-RED com limites e painel
- [ ] Comandos aos dispositivos
- [ ] Limite local no ESP32
- [ ] Montagem em caixa e instalação no quadro

## Estrutura do repositório

Proposta, a ajustar ao longo do projeto:

```
energy-guard/
├── firmware/        código do ESP32
├── node-red/        fluxos exportados
├── docs/            esquemas de ligação e fotografias
├── README.md
└── LICENSE
```

## Configuração

As credenciais (Wi-Fi, MQTT) não devem ser publicadas no repositório. Usar um ficheiro de configuração de exemplo, com valores fictícios, e manter o ficheiro real fora do controlo de versões.

## Referências

- J. Rosado, F. Cardoso e M. Silva, "A Low Cost and Highly Parameterizable Energy Meter", International Conference on Electrical, Computer, Communications and Mechatronics Engineering (ICECCME), Tenerife, 2023.
- Documentação do PZEM-004T V3.0.
- Documentação do ESP32 (Espressif).
- Mosquitto e Node-RED.

## Licença

A definir (por exemplo, MIT), de acordo com as regras da instituição de ensino.

## Autor

A preencher.
