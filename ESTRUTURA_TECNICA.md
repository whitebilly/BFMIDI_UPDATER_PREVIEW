# Estrutura e Funcionamento do BFMidi2 Web Updater

Este documento detalha a arquitetura técnica e o fluxo de funcionamento do gravador web (Web Flasher) desenvolvido para a BFMidi2 (baseada no ESP32-S2).

## 1. Visão Geral
O sistema é uma página única (`index.html`) que utiliza a **Web Serial API** do navegador para se comunicar diretamente com a porta USB do computador, e a biblioteca **esptool-js** para realizar o protocolo de gravação da Espressif.

## 2. Componentes Principais

### Interface (UI)
- **HTML5/CSS3**: Design moderno com tema escuro, responsivo e feedback visual em tempo real.
- **Elementos Chave**:
  - `baudSelect`: Seleção de velocidade (padrão 921600 bps).
  - `fileList`: Lista dinâmica dos binários a serem gravados.
  - `log`: Console de saída para depuração.
  - `progressBar`: Indicadores visuais de progresso para cada arquivo.

### Bibliotecas Externas
- **esptool-js**: Implementação em JavaScript do esptool.py. Responsável pelo protocolo de baixo nível (SLIP, comandos de flash, etc.).
- **crypto-js**: Usada para calcular o hash MD5 dos arquivos locais e verificar a integridade após a gravação.

## 3. Fluxo de Conexão e Gravação (Lógica Crítica)

O ESP32-S2 possui uma peculiaridade: ele usa **USB Nativo (OTG)**. Isso significa que, ao entrar em modo de bootloader (download mode), ele se desconecta fisicamente do USB e se reconecta com um novo Product ID (PID).

### O Algoritmo de Conexão (`connect()`)
Para lidar com essa reconexão, implementamos a seguinte máquina de estados:

1.  **Solicitação de Porta**: O usuário seleciona a porta COM.
2.  **Verificação de Estado (PID)**:
    *   Se o PID for `0x0002`: O dispositivo já está em modo Bootloader. **Pula para o passo 5**.
    *   Se o PID for outro (ex: `0x80c2`): O dispositivo está em modo App (rodando firmware). **Prossegue para o passo 3**.
3.  **Pulso de Boot (Manual)**:
    *   Conectamos brevemente a 115200 bps.
    *   Enviamos a sequência DTR/RTS específica para colocar o chip em modo download.
    *   **Importante**: Fechamos a porta imediatamente (`closeTransportSafe`).
4.  **Espera de Re-enumeração**:
    *   Aguardamos **2 segundos** (`sleep(2000)`). Isso dá tempo para o Windows detectar a desconexão e a nova conexão do dispositivo USB.
    *   Tentamos reconectar automaticamente à nova porta.
5.  **Conexão Final (esptool)**:
    *   Iniciamos o `ESPLoader` com a flag `'no_reset'`.
    *   *Por que 'no_reset'?* Porque já fizemos o reset manualmente no passo 3 (ou o usuário já estava em bootloader). Se o esptool tentar resetar de novo, o USB nativo cairia novamente, quebrando o processo.

### Mapa de Memória (Partition Map)
Os endereços de gravação foram ajustados especificamente para o ESP32-S2:

| Arquivo | Endereço (Offset) | Descrição |
| :--- | :--- | :--- |
| `bootloader.bin` | **0x1000** | Bootloader de segundo estágio (S2/S3 usam 0x0000 ou 0x1000, S2 é 0x1000). |
| `partition-table.bin` | **0x8000** | Tabela de partições. |
| `app.bin` | **0x10000** | O firmware principal da BFMidi2. |

## 4. Finalização e Reinício

Após a gravação bem-sucedida de todos os binários:
1.  O `esptool-js` verifica o MD5 do flash.
2.  Chamamos `esploader.hardReset()`: Envia um comando para reiniciar o chip via software/hardware.
3.  O dispositivo reinicia e carrega o novo firmware.
4.  A conexão USB é encerrada automaticamente (pois o dispositivo reiniciou).

## 5. Resolução de Problemas Comuns (Troubleshooting)

*   **"Failed to open serial port"**: Geralmente ocorre se tentar conectar rápido demais após o pulso de boot. O delay de 2s resolveu isso.
*   **Dispositivo não inicia**: Geralmente causado pelo offset incorreto do bootloader (estava 0x0000, corrigido para 0x1000).
*   **Porta presa**: Se o script falhar no meio do caminho, a porta pode ficar "alocada". O `try/finally` no código garante que `closeTransportSafe` seja chamado mesmo em caso de erro.
