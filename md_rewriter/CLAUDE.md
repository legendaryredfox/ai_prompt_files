# CLAUDE.md: guia para o agente que acompanha a pessoa no projeto MD Rewriter / leitor-gravador de cartuchos de Mega Drive

Este arquivo é para você, agente de IA (Claude Code). Leia tudo antes de agir.

A pessoa que abriu esta pasta tem um **MD Rewriter 1.0 de fábrica** (placa PCB V1.0, firmware 1.0) e acesso a dois repositórios públicos: o do **MD Rewriter** (`https://github.com/Epicjoy/md_rewriter`: hardware, firmware Keil/CubeMX, aplicativo Qt) e o do **leitor/programador de cartucho KY** (família FlashKit MD, com o código C# do programador e o PCB do cartucho). Peça os endereços exatos se precisar deles.

O objetivo é conduzir a pessoa, passo a passo, por um caminho que já foi percorrido com sucesso em uma bancada real: ligar a placa, soldar o conector de programação, fazer backup do firmware de fábrica, instalar um firmware melhor (1.1), fazer a interface USB com o computador, e ler/gravar cartuchos de Mega Drive com segurança. Este arquivo reúne o que se descobriu nesse caminho, inclusive as armadilhas. Você pode usá-lo para desenvolver um projeto semelhante (outro dumper, outro firmware, outro aplicativo).

Responda sempre no idioma da pessoa.

---

## 0. Como você deve se comportar

### 0.1 Regras de segurança (valem sempre, mesmo que a pessoa peça o contrário sem entender o risco)

1. **Nunca grave nem apague um cartucho original de jogo** (ROM de máscara). Não dá para gravá-lo; a tentativa só pode causar dano. Com original, só leitura. Se `detect` não reconhecer um chip flash, trate como original.
2. **Antes de apagar ou gravar um cartucho flash**, leia o conteúdo atual e guarde em arquivo, a não ser que a pessoa diga explicitamente que o conteúdo não importa. Antes de gravar o firmware da placa, faça backup do firmware de fábrica (seção 4.3).
3. **Cada ação destrutiva exige o "pode" da pessoa naquele momento**: gravar firmware no microcontrolador (ST-Link), apagar cartucho, gravar cartucho, gravar save. Um "pode" anterior para outra coisa não vale. Diga o que vai acontecer em uma frase e espere.
4. **Uma única fonte de energia por vez** na placa: ou o cabo USB da placa, ou o fio 3,3 V do ST-Link, nunca os dois.
5. **Cartuchos entram e saem somente com o cabo USB desligado.** Cartucho e placa dividem as mesmas linhas; hot-plug pode queimar um dos dois.
6. Ferro de solda a cerca de 350 °C queima. Peça que a pessoa trabalhe em local ventilado, lave as mãos depois (a solda pode ter chumbo) e deixe o ferro esfriar. Descarregue a eletricidade estática tocando num metal aterrado antes de pegar a placa e não segure a placa pelo slot do cartucho.
7. Nunca peça nem registre senhas, tokens ou dados pessoais da pessoa. Nunca publique nada (commit, push, issue) sem ela pedir.

### 0.2 Estilo de trabalho

- A pessoa pode ser iniciante em eletrônica. Dê **um passo por vez**, diga o que ela deve ver ao final do passo e só avance quando ela confirmar. Use listas curtas e comandos prontos para copiar.
- Peça para ela descrever ou fotografar o que vê (a imagem fala mais que o texto) e meça com o multímetro antes de concluir que uma ligação está certa.
- Distinga sempre **fato medido** de **hipótese**. Este arquivo marca o que foi provado em bancada e o que não foi. Se algo na placa dela divergir deste arquivo, **o que ela medir vale mais que o que está escrito aqui**.
- Se um passo falhar, pare e diagnostique (seção 10). Não improvise gravações no microcontrolador para "ver se funciona".
- Escreva testes e scripts de teste que **não possam** gravar ou apagar (ver 7.4). Operações destrutivas só atrás de uma confirmação explícita (`--yes`).

### 0.3 Perguntas iniciais (faça antes de tudo)

1. Sistema operacional do computador (Linux foi o testado; Windows funciona, com comandos manuais).
2. Quer só **usar** a placa como veio (firmware 1.0, não precisa soldar nada) ou **instalar o firmware 1.1** (precisa soldar 4 pinos e ter um ST-Link V2)?
3. O que tem em mãos: ST-Link V2 (original ou clone "pendrive"), ferro de solda, multímetro, barra de 4 pinos 2,54 mm, jumpers Dupont fêmea, cartuchos (quais? flash? originais?).
4. Qual cartucho flash: marca, rótulo, foto da placa do cartucho. Lembre da seção 8.

---

## 1. O que é a placa e o que ela faz

O MD Rewriter é um dumper/gravador de cartuchos de Mega Drive (Genesis): um microcontrolador STM32 aciona diretamente as linhas do slot do cartucho (endereço, dados, controle) e conversa com o computador por USB como porta serial virtual (CDC). O aplicativo no PC manda comandos; o firmware executa ciclos de barramento.

Fatos da placa V1.0 (projeto no EasyEDA Pro, 2 camadas, 108,4 × 56,5 mm, sem esquemático publicado; a rede foi lida do PCB):

| Item | Fato |
|------|------|
| Microcontrolador | os arquivos de projeto dizem STM32F103VB (128 KiB de flash, 20 KiB de RAM). **A primeira unidade medida tinha um STM32F103xE** (ID `0x414`, 512 KiB de flash, 64 KiB de RAM). O firmware 1.1 foi ligado para 128 KiB/20 KiB e roda no chip maior. **Sempre sonde o chip da pessoa** (4.2) e não assuma. |
| Cristal | 8 MHz, PLL ×9 = 72 MHz, USB derivado da PLL |
| USB | PA11 (D-) e PA12 (D+) com 22 Ω em série; pull-up de 1,5 kΩ em D+. Enumera como `0483:5740` "STM32 Virtual ComPort" (CDC). **Não há DFU.** |
| Energia | VBUS → PTC (polifusível) → 5 V → regulador AMS1117 → 3,3 V (MCU) e 5 V no slot do cartucho |
| LEDs | POWER (3,3 V) e ACT (PC13) |
| Botão | RESET (NRST com pull-up) |
| Conector SWD | **4 furos soltos sem serigrafia**, em coluna, sob o capacitor C10, à direita do chip U1 e acima do conector USB-C |
| NRST/BOOT0 | não há pad de NRST; não há pad de BOOT0 (BOOT0 vai a GND por R3, 10 kΩ); PA9/PA10 (UART do bootloader de ROM) não têm pad |

### Mapa de pinos MCU → slot do cartucho (confirmado pelo firmware nos pinos que ele usa)

| Pinos do MCU | Sinal |
|--------------|-------|
| `PE0..PE15` | D0..D15 (entrada com pull-up) |
| `PD0..PD7` | A1..A8 |
| `PD8..PD15` | A9..A16 |
| `PA0..PA6` | A17..A23 |
| `PB4` | /CAS0 (leitura, `MD_RD`) |
| `PB6` | /CE0 (`MD_CS`) |
| `PB14` | /LWR (escrita, `MD_WR`) |
| `PB15` | /VRES (reset do cartucho) |
| `PB12`, `PB13` | /TIME, M3 |
| `PB5`, `PB7`, `PB8`, `PB9`, `PB11` | /ASEL, /AS, VCLK, /MRES, /CART |
| `PA13`, `PA14` | SWDIO, SWCLK |
| `PC13` | LED ACT |

Não ligados ao MCU: `/UWR`, `/CAS2`, `/DTACK`, EDCLK, YS, HSYNC, VSYNC, SL1, SR1. Consequências: o dumper nunca aciona /UWR (cartuchos flash do tipo FlashKit decodificam só /LWR para escrita de palavra, então funciona; cartuchos que dependem de /UWR ou /CAS2, como EEPROM serial I2C, não são atendidos por esta placa).

Sem conversor de nível: o MCU é 3,3 V e o cartucho é 5 V. Funcionou em todos os cartuchos testados (entradas de dados FT e saídas de endereço), mas é um risco de projeto conhecido.

### Defeitos conhecidos da placa V1.0 (que você deve avisar)

- **USB-C sem resistores Rd de 5,1 kΩ nas linhas CC1/CC2.** Um cabo USB-C para USB-C ligado a um carregador ou notebook moderno pode **não fornecer energia**. **Use cabo USB-A para USB-C com fios de dados.** Correção avançada: soldar dois resistores 0603 de 5,1 kΩ entre os pads A5 e B5 do conector e GND (só para quem sabe soldar 0603).
- O polifusível pode ser de 0,1 A em alguns lotes (o BOM diz 0,5 A): confirme na placa; 0,1 A pode ser pouco para MCU + cartucho.
- Sem TVS/ESD no USB nem no slot, sem resistores em série nas linhas do cartucho, sem desacoplamento junto aos pinos de 5 V do slot. Manuseie com cuidado.
- O LED ACT está em PC13 (pino que a folha de dados não recomenda para acionar LED): funciona, fraco.

---

## 2. Material necessário (confira com a pessoa)

| Item | Para quê | Obrigatório? |
|------|----------|--------------|
| Placa MD Rewriter V1.0 | o dispositivo | sim |
| Cabo USB com fios de dados, de preferência A→C | ligar ao computador | sim |
| Computador (Linux testado; Windows possível) | rodar o aplicativo | sim |
| Cartucho flash (S29GL064N, tipo KY/FlashKit) | gravar jogos | para gravar |
| Cartucho de jogo original | fazer backup | para ler |
| Programador ST-Link V2 (original ou clone USB) | instalar o firmware 1.1 | opcional |
| Barra de pinos 1×4, 2,54 mm | soldar nos pads SWD | com o ST-Link |
| 4 jumpers Dupont fêmea | ligar o ST-Link | com o ST-Link |
| Ferro de solda 25–40 W, ponta fina, solda 0,6–0,8 mm com fluxo | soldar | com o ST-Link |
| Malha dessoldadora, caneta de fluxo, álcool isopropílico, pincel | corrigir e limpar | ajuda muito |
| Multímetro com teste de continuidade | checar pads e soldas | com o ST-Link |
| Fita ou "terceira mão" | segurar a placa | ajuda |

Sem ST-Link e sem solda a placa **já funciona** com o firmware 1.0 para ler cartuchos e gravar alguns flash (com limitações, seção 6.1).

---

## 3. Soldagem do conector SWD (somente se for instalar o firmware 1.1)

### 3.1 Onde ficam os 4 pads

Vista de cima, USB-C embaixo da placa. De cima para baixo:

```
   ●  3V3
   ●  SWCLK   (PA14, pino 76 do U1)
   ●  SWDIO   (PA13, pino 72 do U1)
   ●  GND
 [USB-C]
```

Vistos por trás a ordem vertical é a mesma (3V3 em cima), só parece espelhada. A ordem dos pads foi lida do arquivo de projeto; **confirme com o multímetro antes de soldar e antes de ligar o ST-Link**:

1. Placa sem energia, modo de continuidade: uma ponta no pad de baixo e a outra na carcaça metálica do conector USB-C. Deve apitar (é o GND). Se não apitar, o pad ou a placa (de cabeça para baixo?) está errado.
2. Mais tarde, com o USB ligado, o pad de cima mede 3,3 V contra GND.
3. Sinais SWCLK e SWDIO: foram inferidos do arquivo e depois aceitos pelo ST-Link na bancada. Se o `probe` (4.2) não achar nada, **troque SWCLK e SWDIO** antes de suspeitar de outra coisa.

### 3.2 Passo a passo da solda

1. Prenda a placa com fita, lado de cima para cima.
2. Encaixe a barra de 4 pinos pelo lado de cima, pinos curtos para baixo, plástico apoiado na placa, pinos retos e perpendiculares.
3. Vire a placa. Ferro a 330–350 °C, ponta limpa e estanhada.
4. Toque a ponta no pino e no pad ao mesmo tempo (cerca de 1 s); alimente a solda na junção (não na ponta) por 1–2 s; tire a solda e depois o ferro. **Solde só um pino primeiro.**
5. A junta boa é brilhante, em forma de pequeno vulcão, cobrindo pad e pino. Fosca, cinza, em bola ou com vão: reaqueça com um pouco de solda nova, ou retire com a malha e refaça.
6. Confira se a barra continua reta. Corrija reaquecendo o primeiro pino enquanto empurra. Depois solde os outros três (cerca de 3 s cada; calor demais descola o pad).
7. Limpe com álcool isopropílico e pincel; inspecione com lupa ou câmera do celular: sem ponte de solda entre vizinhos e sem vão.
8. **Multímetro, placa desligada:**
   - GND: apita até a carcaça do USB-C (como antes).
   - Entre pinos vizinhos (3V3–SWCLK, SWCLK–SWDIO, SWDIO–GND): **não** pode apitar. Apito = ponte de solda; remova com a malha.
   - Empurre cada pino com a unha: nenhum pode balançar. **Um pino de GND frouxo custou horas na bancada**: a ligação com o ST-Link caía a cada poucos minutos e parecia defeito de software. Se algo balançar, reaqueça aquele pino.

### 3.3 Correção opcional do USB-C (resistores CC)

Dois resistores 0603 de 5,1 kΩ, um de A5 a GND e outro de B5 a GND. Só se a pessoa souber soldar 0603. Caso contrário, cabo A→C resolve.

### 3.4 Remover o conector depois (se quiser)

Reaqueça um pino e puxe com alicate, ou corte o plástico com alicate de corte e retire os pinos um a um, limpando os furos com a malha. Nunca aqueça um pad por mais que poucos segundos: ele descola. O conector pode ficar na placa sem problema.

---

## 4. Instalar o firmware 1.1 por SWD (cerca de 1 hora)

### 4.1 Por que SWD é o único caminho

Na bancada ficou provado que **não há atualização por USB**: a placa é CDC puro, sem interface DFU; o bootloader de ROM do STM32F103 não fala USB (só UART em PA9/PA10, que não têm pad), e o firmware 1.0 não tem comando "entrar no bootloader". Logo é preciso um ST-Link e o conector da seção 3. (Quem quiser atualização por USB no futuro teria de gravar um bootloader uma vez por SWD: não foi feito.)

### 4.2 Ligar o ST-Link e sondar

**Desligue tudo antes**: cabo USB da placa desconectado e slot **vazio**. Ligue os fios **pelo nome impresso no ST-Link**, nunca por cor ou posição (clones diferem; o ST-Link "pendrive" de 4/6/10 pinos não segue a numeração dos 20 pinos do original):

| Pad (de cima para baixo) | Sinal | Fio do ST-Link |
|--------------------------|-------|----------------|
| 1 | 3V3 | `3.3V` (só se a placa não puder ir ao USB; ver abaixo) |
| 2 | SWCLK | `SWCLK` |
| 3 | SWDIO | `SWDIO` (alguns clones escrevem `SWIO`) |
| 4 | GND | `GND` |
| nenhum | NRST | deixe `RST` solto (a placa não tem pad) |

O `5V` do ST-Link fica sem ligar. No ST-Link original de 20 pinos: SWCLK = pino 9, SWDIO = pino 7, GND = pino 4 (e 6, 8, 10...), 3V3 = pino 1 (só referência de tensão: não alimenta a placa).

**Energia:** o jeito preferido, que ficou estável: só `GND`, `SWCLK` e `SWDIO` no ST-Link, **sem** o fio 3,3 V, e a placa alimentada pelo próprio cabo USB. Se a placa não puder ir ao USB, use o fio `3.3V` do ST-Link com USB desligado e slot vazio; a saída 3,3 V dos clones é fraca, mantenha testes curtos. Nunca os dois.

Aperte cada conector Dupont com o dedo: ligação frouxa é o problema mais comum. Só então espete o ST-Link no computador.

Sonde (Linux: `sudo apt install stlink-tools`):

```
st-info --probe
```

Esperado na primeira unidade: `chipid 0x414`, `F1xx_HD`, flash de 512 KiB (`0x0200` no registrador `0x1FFFF7E0`), SRAM de 64 KiB. Se for `0x410` com 128 KiB, também é válido. Se aparecer outro ID ou outro tamanho, **pare e entenda antes de continuar**.

Se não achar nada (`chipid 0x000`, `Failed to connect`, "0 programmers"):

1. **GND no conector primeiro** (pino frouxo foi a causa real de horas de falhas intermitentes).
2. Troque SWCLK e SWDIO.
3. Reencoste cada Dupont; teste outro cabo/porta USB do ST-Link.
4. Se a ligação cai de vez em quando, repita a leitura/gravação em laço com nova tentativa, mas **sempre exigindo a leitura de conferência**.

### 4.3 Backup do firmware de fábrica (obrigatório antes de gravar)

```
st-flash read fw-factory.bin 0x08000000 0x80000     # 512 KiB; use 0x20000 se o st-info disser 128 KiB
sha256sum fw-factory.bin > fw-factory.bin.sha256
```

Leia **duas vezes** e compare. Valide:

- Leitura **toda 0xFF ou toda 0x00** significa proteção de leitura: **pare**. Não apague nem grave. Apagar a proteção apaga o chip e o firmware de fábrica seria perdido; a decisão é da pessoa, com o aviso claro disso.
- Um firmware sadio tem vetor de interrupção válido: o primeiro word (SP inicial) aponta para a RAM (`0x2000xxxx`) e o segundo (reset) para a flash (`0x0800xxxx`, ímpar).

Na primeira unidade: leituras idênticas, sem proteção, 524 288 bytes lidos, cerca de 18 KiB úteis, SP `0x20003C30`, reset `0x08000105`. Guarde o backup em **dois lugares** (outro disco, pendrive).

### 4.4 Obter o firmware 1.1

O repositório do MD Rewriter só traz o firmware **1.0** (código-fonte Keil em `Firmware`/`legacy_firmware` e `.hex` na página de releases). Duas rotas:

- **Rota A, recomendada:** pegar um binário 1.1 pronto, se a pessoa tiver (com `.sha256`; confira com `sha256sum -c`).
- **Rota B:** construir o firmware 1.1 a partir do código do repositório original aplicando as mudanças da seção 6.2, com `arm-none-eabi-gcc` (`sudo apt install gcc-arm-none-eabi make`). Cuidados:
  - O 1.0 foi compilado com Keil ARMCC5 `-O3`. As esperas do barramento são feitas por atraso em software (`Delay_nop`), então **o tempo muda com o compilador e a otimização**. O build GCC com `-O1` funcionou na bancada para ler e gravar; `-O3` não foi medido. Meça (analisador lógico) se mudar a otimização, e valide lendo o mesmo cartucho duas vezes.
  - O linker deve respeitar a flash do chip (128 KiB se `0x410`; o arquivo de 30 KiB do 1.1 cabe em qualquer um), a RAM de 20 KiB e começar em `0x08000000` sem bootloader.
  - Não regenere o código pelo CubeMX sobre o código alterado (apaga as mudanças).

### 4.5 Gravar

Só depois do backup validado, do "pode" da pessoa e com o USB da placa ligado (energia só do USB):

```
st-flash write md-rewriter-firmware-1.1.0.bin 0x08000000
st-flash read check.bin 0x08000000 <tamanho do .bin>      # e compare com o .bin
```

Regras que valeram na bancada:

- **Verifique relendo** a flash e comparando com a imagem. Só então reinicie.
- Como não há NRST, o reset é feito por software (AIRCR). **Depois de gravar, desligue e religue o cabo USB da placa**: o reset por software nem sempre faz a placa enumerar no USB.
- Se o chip contiver algo diferente do backup que você fez (outra unidade? outra imagem?), **pare e faça novo backup** antes de gravar. Nunca grave sem backup do que está lá.
- Para voltar ao firmware de fábrica: gravar `fw-factory.bin` de volta e reler/comparar.
- Se a placa deixar de responder por SWD (não aconteceu), o caminho é o bootloader de ROM com BOOT0 em nível alto (levar o lado BOOT0 de R3 a 3,3 V), que só fala UART em PA9/PA10, sem pads (pinos 68/69 do LQFP100): exige fios em passo fino. Por isso o backup e a verificação são inegociáveis.

### 4.6 Conferir

```
lsusb | grep 0483:5740           # a placa aparece
```

e use o comando de conexão do aplicativo/CLI (seção 7). O firmware 1.1 imprime `FWVER:1.1.0` logo depois do banner de conexão.

---

## 5. Interface USB com o computador

### 5.1 Permissões e portas

- Linux: a placa vira `/dev/ttyACM0` (ou parecido). Dê acesso: `sudo usermod -aG dialout $USER` e **saia e entre de novo na sessão**, ou instale uma regra udev (VID `0483`, PID `5740`, `MODE="0660"`, `GROUP="dialout"`, e `ENV{ID_MM_DEVICE_IGNORE}="1"` para o ModemManager não mexer na porta). Depois re-espete o cabo.
- Windows: aparece como porta COM. Instale o driver "STM32 Virtual ComPort" se o sistema pedir.
- Se aparecer "permission denied": é permissão de serial. Se a placa não aparece: outro cabo (com fios de dados) e outra porta; o LED POWER deve estar aceso.

### 5.2 Protocolo serial (firmware 1.0, e o que o 1.1 acrescenta)

Observado no firmware de fábrica e mantido no 1.1 (o 1.1 só acrescenta). Pacote de comando do PC: `[cmd] AA 55 AA BB [argumentos...]`.

| Comando | Significado |
|---------|-------------|
| `0C` | conectar. O 1.1 acrescenta uma segunda linha `FWVER:<versão>` ao banner |
| `0D` | detectar: o firmware imprime `FLASHID:` e uma linha com o chip (1.1: `FLASHID:%02X%02X`, `CHIPINFO:`) |
| `0A` | ler a ROM (argumento = tamanho: `01` 512K, `02` 1M, `03` 2M, `04` 4M; 1.1 também `05` = 8M) |
| `1A` | ler a SRAM de save (argumento `01` = 32K, senão 8K) |
| `0B` | gravar um bloco de ROM (argumentos: índice do bloco no banco, banco) |
| `1B` | gravar um bloco de save |
| `0E` | apagar tudo (chip e SRAM) |
| `1E` | apagar por tamanho (argumento `01`..`06`, ou `00` mais endereço de 3 bytes) |
| `2E` | apagar um setor |
| `0F` | limpar buffers |
| `3A` | só 1.1: leitura pelo mapper SSF2 (argumento = número de páginas de 512 KiB); **não testado em cartucho** |

Formatos:

- **Leitura:** o firmware manda texto `... DUMP START!!!\r\n`, depois N blocos brutos de 1024 bytes (512, 1024, 2048 ou 4096 blocos para 512K a 4M; 32 ou 8 para save; 8192 para 8M no 1.1), e por fim `DUMPER ROM FINISH!!!\r\n` (ou `DUMPER RAM FINISH!!!\r\n`).
- **Gravação de bloco:** o PC manda **1024 bytes brutos e depois o comando `0B`/`1B` de 7 bytes**. O firmware responde `ADD:0x... WRITE OK\r\n` (ROM) ou `... WRITE GK\r\n` (save). O 1.1 responde `ADD:0x<endereço de palavra> WRITE TIMEOUT` quando uma palavra não foi programada.
- **Fonte da verdade da semântica fina** (exato significado de cada argumento, tamanho do banco, códigos de `1E`): o código do firmware 1.0 no repositório original (`main.c` e `MD.c`). **Confirme lá antes de implementar**; esta tabela vem da observação em bancada.

### 5.3 Manias do firmware (aprendidas em bancada, valem para 1.0 e 1.1)

1. **Depois de responder `detect`, o firmware espera 100 ms e só então limpa o buffer de entrada.** Bytes enviados nesse intervalo se perdem. Na prática: o primeiro bloco de gravação enviado logo após o `detect` some e o PC espera para sempre uma confirmação que não vem (`The device did not confirm block 1`). **Solução no PC: segurar o próximo comando 250 ms depois de `detect`.** Qualquer ferramenta que fale com o dumper sofre disso.
2. **Um `connect` que chega logo depois de a porta ter sido reaberta pode ser ignorado.** Faça uma segunda tentativa de `connect`, ou espere 0,3 s entre execuções de processos seguidos.
3. Depois de gravar firmware, religue o USB (seção 4.5).
4. Velocidade medida: leitura de 2 MiB em 10–12,5 s (cerca de 166–214 KiB/s); 8 MiB em cerca de 48 s.

---

## 6. O que mudou do firmware 1.0 para o 1.1 e por quê

### 6.1 O defeito do 1.0 com cartuchos flash rápidos

Na bancada, com o cartucho S29GL064N: **o firmware 1.0 programa só uma palavra sim, outra não**, porque não espera o chip terminar de gravar cada palavra. Nenhuma palavra sai errada; metade fica em branco (`0xFFFF`). Sintoma no aplicativo: a gravação termina e a verificação falha (`Verification failed: ... bytes differ`). Dá para contornar repetindo a gravação só das palavras faltantes (8 passagens, cerca de 2 minutos para 2 MiB), mas o correto é consertar o firmware.

### 6.2 Mudanças do 1.1 (o mínimo para um bom resultado)

Parta do código-fonte 1.0 e aplique:

1. **Polling de dados DQ7 depois de cada palavra programada.** Sequência AMD de programação: `AA`@`555`, `55`@`2AA`, `A0`@`555`, depois dado@endereço (endereços de palavra). Depois de cada palavra, leia o endereço programado até DQ7 igualar o bit 7 do dado gravado; se DQ5 subir (e DQ7 ainda não bater), o chip desistiu: envie reset (`F0`), pare e responda `WRITE TIMEOUT`. Limite o número de leituras. Troque o barramento entre saída e entrada rapidamente (acesso direto a registradores) para o polling não ficar lento. Resultado medido: gravação correta em uma única passagem.
2. **Identificação do chip (autoselect)**: ler as palavras `00`, `01`, `0E`, `0F` numa sessão de autoselect, imprimir `FLASHID:` com 4 dígitos hex (`017E` no S29GL064N) e mapear numa tabela de nomes.
3. **Apagar por setores pequenos**: apagar em passos de 8 KiB (comando `30` por setor) abaixo de 64 KiB e depois em setores de 64 KiB. Veja a razão em 8.3.
4. Leitura de 8 MiB (código `05`), `FWVER:`, saída `WRITE TIMEOUT` e, se quiser, consulta CFI para chips desconhecidas.
5. Opcional: gravação por buffer de escrita (`25`/`29`, 16 palavras por operação) ficou implementada mas **não foi testada em cartucho** (desligada por padrão).
6. Nada além disso: manter o restante igual ao 1.0 para o aplicativo antigo continuar funcionando.

Tempos medidos com o 1.1 (S29GL064N): apagar 2 MiB em 6–9 s; gravar 2 MiB em cerca de 79 s (cerca de 90 s com a leitura de conferência).

### 6.3 Teste do firmware sem placa

Vale a pena compilar o `MD.c` no PC contra um **modelo de chip flash AMD** (autoselect, programação com DQ7/DQ6/DQ5, apagamento por setor, chip que ignora comandos enquanto ocupado, chip que mente sobre o layout). Isso pegou erros reais antes de a placa chegar (por exemplo, DQ5 lido errado em `0xFF` quando o apagamento terminava entre duas leituras de status). Mas o modelo é só hipótese: **só vale o que a placa provar**.

---

## 7. O aplicativo no PC

### 7.1 O que existe

O MD Rewriter original traz um aplicativo Qt (`Software/`). O projeto de referência melhorou-o: aceita ROMs pares de qualquer tamanho até 4 MiB, converte `.smd`, verifica a gravação relendo, mostra o nome do chip, trata `WRITE TIMEOUT`, oferece 8 MiB só quando o firmware diz `FWVER:`, e tem uma **linha de comando** (`mdrewriter-cli`) feita para scripts e agentes.

Linux, Qt 6 (testado no Linux Mint 22 com Qt 6.4):

```
sudo apt install build-essential qt6-base-dev qt6-serialport-dev qt6-tools-dev qt6-l10n-tools
cd Software && qmake6 MDRewriter.pro && make -j"$(nproc)" && ./MDRewriter
```

Windows: instale o Qt e abra o `.pro` no Qt Creator, ou use o executável pronto da página de releases.

### 7.2 Se a pessoa não quiser Qt

Um cliente simples em Python com `pyserial` basta para ler: abrir a porta, mandar `0C AA 55 AA BB` (conectar), esperar 0,3 s, mandar `0D AA 55 AA BB` (detect), **esperar 250 ms**, mandar `0A AA 55 AA BB 03` (2 MiB), consumir até `DUMP START!!!`, ler `2048 × 1024` bytes e procurar `DUMPER ROM FINISH!!!`. Descubra o banner e o texto exato imprimindo o que a placa responde (não suponha). Para gravar, siga o formato do 5.2 e **só depois de apagar**.

### 7.3 Contrato recomendado para uma CLI (use no projeto da pessoa)

- Comandos somente leitura: `ports`, `info`, `detect`, `read-rom`, `read-save`, `verify`, `compare`, `rominfo`.
- Comandos que alteram o cartucho (`write-rom`, `write-save`, `erase`) **recusam** sem `--yes` (código de saída 4) e têm `--dry-run`. Antes de gravar, a CLI confere o chip (`connect`, `detect`, aí grava): recusa se o firmware não imprimiu ID de chip, se a ROM é maior que o chip, ou se o chip precisa de um conjunto de comandos que só firmware mais novo envia.
- `read-*` não sobrescreve arquivo existente sem `--yes`; grava de forma atômica.
- Saída `--json` e códigos de saída distintos: 0 ok, 1 operação falhou, 2 uso, 3 sem dispositivo, 4 falta `--yes`, 5 erro de arquivo.
- Gravar por padrão **lendo de volta e comparando**. Na bancada a verificação pegou um bit lido errado uma vez (byte `0xAAB`, `0x04` lido como `0x00`, não reproduzido): nunca desligue a verificação sem motivo.
- Um **firmware falso** (simulador) para testes automáticos, com opções para reproduzir as manias da seção 5.3.

### 7.4 Suíte de bancada somente leitura

Escreva (ou peça ao agente para escrever) um script que chame a CLI **só por uma lista branca de comandos de leitura** e recuse `--yes`. Verificações úteis que valeram a pena:

- Sem cartucho: a placa enumera como `0483:5740`; 10 reconexões seguidas; `detect` responde `FFFF` sem nomear chip; duas leituras de 512K do slot vazio são idênticas e **todas `0xFF`** (prova que os pull-ups de D0–D15 funcionam).
- Com cartucho: duas leituras idênticas; tamanho; cabeçalho `SEGA`; tamanho de ROM do cabeçalho; checksum do cabeçalho (informação, não falha); leitura menor igual ao prefixo da maior; bits de dados (cada bit deve ser 1 em ~30–40% das palavras: nenhum travado); blocos de 32 KiB distintos (endereço travado dá repetição); duas leituras de save idênticas.

Resultado de referência: em todas as unidades o slot vazio deve dar uma leitura com todos os bytes `0xFF`. Se vier outra coisa, desconfie dos pull-ups, de solda em curto ou de um cartucho esquecido no slot.

---

## 8. Descobertas sobre os cartuchos (leia antes de gravar qualquer coisa)

Foram testados: um cartucho original de máscara (Rolling Thunder 2, Namco), dois cartuchos flash S29GL064N (um deles da KY Technology) e o conteúdo de um jogo de 2 MiB (Metal Gear) gravado e verificado nos dois. A pessoa que fez os testes confirmou que **os cartuchos gravados rodaram num Mega Drive real**.

### 8.1 Cartuchos originais (ROM de máscara)

- `detect` mostra `FLASHID` `0000`/`00`/`FFFF` e nenhum chip: é o normal. **Nunca grave.**
- Uma leitura maior que o cartucho é um **espelho**: lendo 2 MiB de um jogo de 1 MiB vem a mesma imagem duas vezes (as linhas altas do endereço não são decodificadas). O aplicativo não detecta o tamanho: a pessoa escolhe; leitura maior repete o conteúdo. Use o tamanho do cabeçalho (`ROM size`) como guia.
- O checksum do cabeçalho pode não bater mesmo em original (aconteceu em Rolling Thunder 2: cabeçalho `BEB6`, soma `11B1`). Duas leituras idênticas, bits de dados sadios e nenhum alias de endereço indicam que a leitura está certa, mas só um dump de referência (No-Intro) fecha a dúvida: **ficou em aberto**.

### 8.2 Cartuchos flash tipo KY / FlashKit: **o limite de 2 MiB**

**Resultado mais importante:** os dois cartuchos flash testados guardam **no máximo 2 MiB de jogo**, apesar de o chip (S29GL064N) ter 8 MiB. Sintomas:

- Apagar ou gravar acima de 2 MiB falha: `erase 4M` para com `A sector did not finish erasing in time` depois de cerca de 23 s; apagar um setor em endereço de byte ≥ 2 MiB dá `ERASE TIMEOUT!`.
- Ler acima de 2 MiB mostra a **SRAM de save**, não a flash: cada segundo byte é `0xFF` e os outros bytes repetem o texto do save (no cartucho KY apareceu o nome do jogo: `ff 52 ff 41 ff 47 ff 4e ...`). Ler 8 MiB mostra os mesmos 4 MiB duas vezes.

**Causa (do projeto de referência do cartucho FlashKit, `cart-md-v1.2.pcb`, e confirmada pelo comportamento):** o cartucho usa um decodificador 74HC139 em que o /CE da flash só é ativado quando a linha de endereço de 2 MiB está baixa; com ela alta o mesmo decodificador seleciona a SRAM. O pino A21 da flash está ligado a GND e as linhas A21/A22 do conector não vão a lugar nenhum. **É a construção do cartucho, não defeito do dumper nem do firmware.**

Tentou-se mapear a flash sobre a janela da SRAM com as escritas de registrador do programador original (`$A13000` = `0x0000`/`0x0001`/`0xFFFF`, `$A130F1` = `0`/`1`), com um ciclo de escrita de mapper (`/TIME` e `/LWR` baixos, `/CE0` alto): **nada mudou** (só um byte da RAM trocou, como uma escrita de RAM faria). Portanto não existe registrador nesses cartuchos, e **2 MiB é o limite**.

O que cabe: quase todos os jogos de Mega Drive (a maioria tem de 512 KiB a 2 MiB). O que não cabe: jogos maiores que 2 MiB (por exemplo Super Street Fighter II, 4 MiB, com mapper). Para eles é preciso outro cartucho: flash de 4 MiB sem esse decodificador (por exemplo o projeto aberto `MD-Flash-Cart-4MB`, com MX29LV320 ou MX29L3211) ou cartucho com mapper. **Seu agente deve avisar antes de gravar ou apagar acima de 2 MiB nesses cartuchos.**

### 8.3 Dois cartuchos com o mesmo ID e layouts de setor diferentes

Os dois cartuchos respondem `FLASHID:017E` e a mesma consulta CFI, e se parecem em tudo, mas:

- Um tem o **primeiro setor de 64 KiB uniforme** (um único comando de apagar setor em `0x1000` palavras zera os 64 KiB).
- O outro (KY) tem **oito setores de 8 KiB** no início (boot sectors): para zerar os mesmos 64 KiB é preciso apagar oito vezes. Se o firmware apagar só 64 KiB de uma vez, ficam 56 544 bytes programados, e a gravação seguinte dá `WRITE TIMEOUT` já no primeiro setor sujo.
- Os códigos de autoselect extras (ciclos `0E/0F`) deram `017E1001` e `017E1000` nos dois, mas a regra baseada neles errou, e as palavras de setor do CFI (`2C..34h`) deram o mesmo valor nos dois e **mudaram entre sessões** (`...7F...` e depois `...7E...`). **Conclusão prática: não confie em ID, autoselect nem CFI para escolher o layout de apagamento. Apague sempre em passos de 8 KiB nos primeiros 64 KiB.** Funciona nos dois, ao custo de um pouco de tempo.
- Mostre ao usuário os códigos extras do chip (`Chip codes 017E1001`) e a consulta CFI como **informação**, nunca como decisão.

### 8.4 Regras de gravação em flash

- Flash só muda de 1 para 0. **Apague antes de gravar.** Gravar sobre dados sem apagar dá `WRITE TIMEOUT` em cerca de 0,4 s e a placa continua respondendo: é a falha segura e esperada.
- Gravar **dados idênticos** sobre dados já gravados não dá erro (não há nada a mudar). Para provar a detecção de timeout, grave dados diferentes (por exemplo a imagem invertida).
- Um apagamento de 2 MiB não apaga o resto do chip. O que estiver acima fica lá (num dos cartuchos havia dados antigos).
- O S29GL064N dos cartuchos testados tem **save RAM sem bateria** em um deles (e FRAM no PCB 1.4): `read-save` funciona; **não teste `write-save` em flash cartucho** sem entender o mapa, porque a janela do save pode coincidir com espaço de flash e uma escrita ali seria um ciclo de barramento dentro do chip.
- Um dump do cartucho antes de apagar é a rede de segurança. Guarde.

### 8.5 Chips e comandos (resumo útil para ampliar o suporte)

| Chip | ID (fabricante+dispositivo) | Comandos |
|------|----------------------------|----------|
| S29GL064N (Spansion/Cypress) | `01 7E` | AMD, desbloqueio `555h/2AAh` |
| MX29LV640EB/ET | `C2 CB` / `C2 C9` (o `C9` não foi confirmado) | AMD 555h |
| M29W640D/G | `20 DF`, `20 DE`, `20 7E` | AMD 555h |
| SST39VF6401B/6402B | `BF 236D` | AMD 555h, apagar bloco com `50h` |
| M29W320DT/DB, M29W320ET/EB | `20 CA/CB/56/57` | AMD 555h; "T" = boot no topo (últimos 64 KiB em oito setores de 8 KiB) |
| MX29LV320T/B | `C2 A7/A8` | AMD 555h |
| MX29L3211 | `C2 F9` | AMD com desbloqueio `5555h/2AAAh`; sem programação por página |
| Intel PA28F400 | `89 70/71` | família Intel/Sharp (`FFh`, `90h`, `40h`, `20h`+`D0h`, `70h`/`50h`) |

Só o S29GL064N foi **provado em hardware**. O resto está no modelo em PC e na leitura de folhas de dados: **leia a folha de dados do chip antes da primeira gravação**.

### 8.6 Não testado (diga isso à pessoa)

Leitura do mapper SSF2 (comando `3A`), Sega Lock-on, Open EverDrive, EverDrive v2, Magistr-16, MX26L6420, saves seriais I2C/SPI (faltam `/UWR` e `/CAS2` na placa), Master System/Game Gear (conector diferente), gravação com falta de energia no meio, gravação por buffer, cartuchos flash de outras marcas, `write-save` em cartucho flash, a gravação acima de 2 MiB em cartuchos que não tenham o decodificador do FlashKit.

---

## 9. Roteiro completo para o agente conduzir

Avance só quando a pessoa confirmar cada etapa.

**Etapa 1. Conhecer (0.3).** Faça as perguntas iniciais.

**Etapa 2. Usar como veio (sem solda).**
1. Instale o aplicativo (7.1) e a permissão da serial (5.1).
2. Ligue a placa (cabo A→C) sem cartucho: o LED POWER acende; `lsusb` mostra `0483:5740`.
3. `Connect` no aplicativo (ou `mdrewriter-cli info`): "Device identified" e firmware `1.0`.
4. **Slot vazio:** `detect` não acha chip; leitura do slot vazio dá só `0xFF`.
5. Desligue o USB, coloque um cartucho original (rótulo virado para a seta "CART FRONT", reto, sem balançar), religue, `detect`, leia 2 MiB duas vezes e compare. Resultados diferentes = mau contato: limpe os contatos do cartucho com álcool isopropílico e reencaixe.

**Etapa 3. Solda e conferência (seção 3)**, só se for instalar o firmware 1.1.

**Etapa 4. SWD: sondar, fazer backup, gravar 1.1, conferir (seção 4).**

**Etapa 5. Testar a unidade (tabela abaixo), nesta ordem.** Se falhar, pare e vá à seção 10.

| # | Teste | O que deve aparecer |
|---|-------|---------------------|
| 1 | Versão, sem cartucho | `Device identified.` e firmware `1.1.0` (ou `1.0`) |
| 2 | Slot vazio | nenhum chip; leitura só com `0xFF` |
| 3 | Original, só leitura, 2 MiB | dump feito em cerca de 12,5 s; `detect` mostra ID `0000`/desconhecido (não é flash) |
| 4 | Ler de novo e comparar | sem diferenças |
| 5 | Cartucho flash: detect | `S29GL064N ... Supported` e `Chip codes ...` |
| 6 | **Backup do que está no flash** | ROM 2 MiB e save 32K em arquivo |
| 7 | Apagar 2 MiB | `Erase complete` em 6–9 s; nova leitura só `0xFF` |
| 8 | Gravar ROM (≤ 2 MiB) com verificação | `Game written` e `Verified` (cerca de 90 s para 2 MiB) |
| 9 | Ler e comparar com o arquivo original | idênticos |
| 10 | Save: ler duas vezes e comparar | idênticos |

**Etapa 6. Testar num Mega Drive real** (é a prova final).

**Etapa 7. Registrar.** Ajude a pessoa a escrever o que mediu (backup, SHA-256, tempos, o que falhou), no estilo de relatório de bancada, para ela compartilhar.

---

## 10. Diagnóstico

| Sintoma | O que fazer |
|---------|-------------|
| A placa não aparece no computador | Cabo USB com fios de dados (A→C); outra porta; LED POWER aceso? Sem Rd no USB-C, cabos C→C podem não alimentar (seção 1) |
| `permission denied` | Grupo `dialout` + novo login, ou regra udev (5.1) |
| `st-info` não acha o chip (`chipid 0x000`, `Failed to connect`) | GND do conector primeiro; trocar SWCLK/SWDIO; reencostar os Dupont; trocar cabo/porta; ver continuidade e pontes de solda (3.2) |
| Leitura de backup toda `0xFF`/`0x00` | Proteção de leitura. **Pare.** Não apague nem grave (4.3) |
| Depois de gravar a placa não aparece no USB | Desligue e religue o cabo USB da placa |
| Primeiro bloco de gravação não é confirmado | Segurar 250 ms depois de `detect` (5.3) |
| `connect` sem resposta logo depois de outro comando | Repetir o `connect` uma vez / esperar 0,3 s |
| `detect` diz chip desconhecido | Provavelmente original (ROM de máscara) ou chip fora da tabela: **não grave** |
| `did not finish erasing` / `ERASE TIMEOUT` | Não é flash, está danificado, **ou foi pedido mais de 2 MiB** (8.2) ou o apagamento não percorreu os setores de 8 KiB (8.3) |
| `WRITE TIMEOUT` | A área não foi apagada antes; apague e grave de novo (8.4) |
| Verificação falha | Limpar contatos, outro cabo USB, repetir apagar+gravar. Com firmware 1.0 num S29GL064N é o defeito da seção 6.1 |
| Duas leituras do mesmo cartucho diferem | Mau contato: limpe com álcool isopropílico e reencaixe |
| Leitura maior que o cartucho repete conteúdo | É espelho (8.1) ou a janela de SRAM do cartucho flash (8.2): normal |
| Jogo gravado não inicia no console | Confirme que é uma ROM válida (cabeçalho `SEGA`), que tem no máximo 2 MiB e que a verificação passou |

---

## 11. Se for desenvolver um projeto semelhante (outro dumper, outro firmware)

Lições que valem independente do hardware:

1. **Primeiro leia, depois escreva.** Faça o dumper ler com 100% de repetibilidade (duas leituras idênticas, SHA-256 igual, slot vazio todo `0xFF`) antes de qualquer gravação.
2. **Backup do firmware antes de tudo**, leitura dupla, verificação de proteção, cópia em dois lugares.
3. **A bancada manda.** O modelo em PC pega muita coisa, mas errou o layout dos setores, o limite de 2 MiB e o chip real da placa. Anote "não testado em hardware" até medir.
4. **Nunca decida pelo ID** o que pode ser feito de forma segura em qualquer caso (como apagar em setores de 8 KiB).
5. **Verifique sempre relendo.** É barato e pegou erros reais.
6. **Sondar antes de supor** (ID do MCU, tamanho de flash, versão do firmware).
7. **Separe o que lê do que escreve**, nas ferramentas e nos scripts de teste.
8. **Ligação intermitente quase sempre é GND ou solda.** Antes de culpar o software, balance os pinos.
9. Quando o hardware não permite (limite de 2 MiB, USB-C sem Rd), **documente o limite** em vez de insistir no software.
10. Guarde cada medida (tempos, hashes, ID) num relatório. Quem vier depois economiza semanas.
11. Se for projetar uma placa nova: ponha resistores Rd de 5,1 kΩ no USB-C, pads rotulados e pad de NRST e BOOT0, TVS no USB, resistores em série nas linhas do cartucho, desacoplamento junto aos 5 V do slot, /UWR e /CAS2 conectados, um conversor de nível 5 V/3,3 V se o cartucho exigir, e um bootloader USB (DFU) para dispensar o ST-Link.

---

## 12. Repositórios de referência que foram úteis

Clone os que precisar (em pasta própria, fora do projeto) e **leia o código antes de confiar neste arquivo**: onde houver divergência, vale o que a bancada da pessoa mostrar. Cada licença é diferente (GPL-3.0, MIT, CERN OHL): respeite-as ao reutilizar código ou projeto.

| Repositório | Para que serviu |
|-------------|-----------------|
| `https://github.com/Epicjoy/md_rewriter` | O MD Rewriter original: projeto da placa (EasyEDA), firmware 1.0 (Keil/CubeMX), aplicativo Qt, release 1.0 com o `.hex` de fábrica. Ponto de partida |
| `https://github.com/krikzz/flashkit` | Programador FlashKit MD original (Krikzz): fonte C# (`Cart.cs`, `Device.cs`, `Form1.cs`), PCB do cartucho (`cart-md-v1.2.pcb`, P-CAD) e CPLD. Foi a fonte do decodificador 74HC139 que explica o limite de 2 MiB e do registrador `$A13000` de habilitação da SRAM |
| `https://github.com/MiGeRA/FlashKit-MD-Plus` | Fork do FlashKit com mais chips (Intel PA28F400, MX29LV320, MX29L3211 com desbloqueio `5555h/2AAAh`), tempos e mappers (Magistr-16, EverDrive-MD v2). Base das rotinas de comandos alternativos |
| `https://github.com/MiGeRA/MD-Flash-Cart-4MB` | Cartucho flash aberto de 4 MiB (KiCad, MX29L3211/MX29LV320, CERN OHL): alternativa sem o decodificador de 2 MiB e cartucho de teste |
| `https://github.com/joeyparrish/flashkit-md-py` | Cliente Python (pyserial) para o FlashKit: separa tamanho da imagem do tamanho do chip, usa CFI para validar a comunicação. Bom modelo para uma CLI e para a consulta CFI |
| `https://github.com/grantek/fkmd` | Implementação independente em Go (leitura, escrita de ROM, RAM): contraprova de sequências e protocolo |
| `https://github.com/X-death25/MD_Dumper` | Dumper aberto USB-C com STM32 (firmware, software GUI/CLI, `gameslist.csv`, tabelas de chips): a melhor fonte de sequências por chip (SST `50h`, Sharp, Toshiba), mappers (SSF2, Lock-on, WaterMelon, Open-EverDrive) e saves seriais. Variantes antigas: `https://github.com/X-death25/MD-Dumper_old_stm32F4`, `https://github.com/leolobato/MD_Dumper_V2`; interface gráfica: `https://github.com/X-death25/PyMD-Dumper` |
| `https://github.com/krikzz/open-ed` | Cartucho Open EverDrive: mapper (`A130E0h`, `CTRL_REG` em `A13000h`), e o aviso de que o programador precisa de pull-up de 10 kΩ em /VRES (pino B27) |
| `https://gitlab.com/doragasu/mw-prog` (mais `mw-mdma-bl`, `mw-mdma-fw`, `mw-mdma-cli`) | Programador dos cartuchos MegaWiFi: referência para cartuchos com mapper próprio. Não foi clonado nem testado |
| `https://github.com/sanni/cartreader` (Open Source Cartridge Reader) | Leitor de cartuchos de vários consoles em Arduino Mega; base do projeto irmão abaixo e fonte de ideias para detecção de chip flash e banco de dados de jogos |
| `https://github.com/legendaryredfox/oscr_minimal` | Projeto irmão do autor deste material: leitor/gravador de cartuchos de SNES, NES, Famicom, Mega Drive e GBA em shield para Arduino Mega, baseado no OSCR; reúne as descobertas de bancada sobre cartuchos de Mega Drive e planeja suporte a cartuchos de até 8 MiB |

Sobre o cartucho KY: a Krikzz vende o FlashKit; a KY Technology vende um programador e cartuchos compatíveis. Nenhuma fonte mostrou protocolo exclusivo da KY: "suporte a KY" significa gravar e ler os chips e mappers que aparecem nesses cartuchos. O programador da KY em si (FT245R + CPLD) é outro dispositivo e está fora do escopo.

## 13. O que este projeto melhorou em relação ao MD Rewriter original

Se a pessoa perguntar o que muda em relação ao `Epicjoy/md_rewriter`, resuma assim (detalhes, com o que foi testado na unidade e o que não foi: `https://github.com/legendaryredfox/md_rewriter/blob/main/docs/improvements/pt-BR.md`):

- **Aplicativo** reescrito em camadas: ROM de qualquer tamanho par, conversão `.smd`, conferência por leitura de volta depois de gravar, nome do chip, botão Cancelar, avisos antes de operações destrutivas, tema claro ou escuro e guia embutido (F1).
- **Linha de comando `mdrewriter-cli`**: saída `--json`, códigos de saída fixos, `--yes` obrigatório para gravar ou apagar.
- **Firmware 1.1** (GCC e `make`): grava o S29GL064N em uma só passada (o 1.0 gravava uma palavra sim, outra não), apaga em setores de 8 KiB, nomeia mais chips, prepara 8 MiB.
- **Testes**: cerca de 150 no aplicativo, o `MD.c` real contra um modelo de chip flash e uma suíte de bancada somente leitura.
- **Ferramentas**: SWD com backup e verificação, reparo de cartuchos gravados pelo firmware 1.0, pacote `.deb`.
- **Documentação** em seis idiomas (pt-BR, en, ru, zh-Hans, hi, ur): README, guia passo a passo, protocolo, CLI, suporte a cartuchos, resultados de bancada.
- **Descobertas**: o chip é um STM32F103xE, não há atualização por USB, cartuchos flash KY / FlashKit comportam no máximo 2 MiB.
- **Não mudou**: a placa é a V1.0; vários chips extras só passaram em testes no PC (marca ◇ na lista completa).

Bom trabalho. Em caso de dúvida, pergunte à pessoa e meça.
