# HISTORY.md: como este projeto foi desenvolvido

Resumo do processo de desenvolvimento do projeto de suporte ao **MD Rewriter** (dumper/gravador de cartuchos de Mega Drive com STM32) e a cartuchos flash do tipo **KY / FlashKit**, de 31 de outubro a 14 de novembro de 2026. O trabalho foi feito por uma pessoa com a ajuda de um agente de IA (Claude Code): ela decidia, o agente analisava, escrevia código e documentos, e a bancada julgava. As datas vêm do histórico do repositório e das anotações de cada sessão.

Os detalhes técnicos e as instruções práticas estão no `CLAUDE.md` que acompanha este arquivo. Aqui está a história: o que foi feito, em que ordem, o que deu errado e o que se aprendeu.

---

## Ponto de partida (outubro de 2025)

O projeto original (`Epicjoy/md_rewriter`) foi publicado em 10 de outubro de 2025: projeto da placa no EasyEDA, firmware 1.0 (STM32F103VB, projeto Keil/CubeMX), aplicativo Qt e um release com o `.hex` de fábrica. Tinha pouca documentação, nenhum teste, nenhum esquemático, e o aplicativo e o firmware só gravavam bem alguns poucos chips flash.

A pessoa que conduziu este trabalho comprou os cartuchos e o dumper e, enquanto a placa não chegava, decidiu preparar todo o software e o firmware **no papel e no PC**, para que a bancada servisse só para confirmar.

## Fase 1: revisão e base de qualidade (31 de outubro de 2026)

- Revisão do firmware: limites de vetores, `volatile` onde faltava, esperas sem limite de tempo trocadas por esperas limitadas, avisos do compilador eliminados, script de checagem estática (`arm-none-eabi-gcc` com avisos e `cppcheck`).
- Aplicativo refeito em camadas: protocolo, sessão com o dispositivo, interface, traduções (português, inglês, russo, chinês) e uma suíte de testes (unitários, de integração contra um firmware simulado e de ponta a ponta com porta serial pseudo-terminal).
- Guia de usuário embutido no aplicativo, README em quatro idiomas, especificação e plano, licença MIT.
- Testes de cancelamento, desconexão e "ocupado" tornados determinísticos.

**Lição:** um firmware simulado dentro dos testes permitiu exercitar falhas reais (resposta fragmentada, cabo desconectado) muito antes de existir hardware.

## Fase 2: congelar o antigo, construir o novo (2 e 3 de novembro)

- `Firmware/` virou `legacy_firmware/` (congelado, só correções de revisão) e nasceu um firmware novo com compilação por GCC e `Makefile`, que depois foi nomeado 1.1. O de Keil não foi carregado adiante.
- Revisão do hardware da PCB V1.0, só leitura, com extração da rede a partir do arquivo `.eprj` (um banco SQLite com dados em base64+gzip). Resultado: o defeito principal é a falta dos resistores de 5,1 kΩ no USB-C; também metadados errados, BOM desatualizada, polifusível divergente, SWD sem rótulo e sem NRST, sem proteção ESD.
- Aplicativo: ROMs de qualquer tamanho par até 4 MiB, conversão `.smd`, verificação por releitura, nomes de chips a partir do `FLASHID`, tratamento de `WRITE TIMEOUT`, leitura de 8 MiB oferecida só quando o firmware se identifica com `FWVER:`. Versão 1.2.0.
- Firmware: identificação do S29GL064N a partir da folha de dados da Cypress, leitura do autoselect, apagamento por tamanho, **polling DQ7 após cada palavra programada**, e a decisão de **testar o código do firmware no PC**: o `MD.c` real compilado contra um modelo de chip flash AMD (a suíte chegou a mais de 200 verificações). Também entraram chips SST39VF e uma gravação por buffer (opcional, desligada por padrão).
- O modelo de PC pegou um erro real de apagamento (DQ5 lido errado em `0xFF` quando o apagamento terminava entre duas leituras de status).

**Lição:** o modelo em PC é uma hipótese bem testada, não um fato. Tudo o que dependia dele foi marcado "não testado em hardware" até a bancada confirmar.

## Fase 3: cartuchos KY / FlashKit (4 de novembro)

- Pesquisa em fontes abertas (os repositórios listados na seção 12 do `CLAUDE.md`): ficou claro que não existe protocolo exclusivo da KY. "Suporte a KY" passou a significar "gravar e ler os chips e mappers que aparecem em cartuchos FlashKit/KY com o nosso dumper".
- Firmware 1.1 para a PCB atual: chips de 32 Mbit (M29W320, MX29LV320), e dois novos conjuntos de comandos (AMD com desbloqueio `5555h/2AAAh` e Intel/Sharp).
- `mdrewriter-cli`: a mesma sessão do aplicativo, para scripts e agentes, com `--yes` obrigatório para tudo o que altera o cartucho, `--json` e códigos de saída definidos.
- Suíte de bancada **somente leitura** (`hwtest.py`): lista branca de comandos de leitura, recusa `--yes`.
- **Decisão:** a reforma da PCB do MD Rewriter foi cancelada. As ideias de hardware migraram para o projeto irmão `oscr_minimal` (shield para Arduino Mega); o MD Rewriter V1.0 continua sendo usado como veio, com firmware melhor.

## Fase 4: mais cobertura no papel (5 a 10 de novembro)

- Apagamento de bloco `50h` para SST39VF320x e gravações de até 8 MiB.
- Consulta CFI para chips fora da tabela (apenas informação) e leitura de ROM com mapper SSF2 (comando `3A`, só no modelo).
- README e ajuda do aplicativo atualizados; documentação de como ligar o ST-Link V2 aos pads SWD (incluindo o mapa de pinos do ST-Link original e dos clones).
- Itens que **foram recusados de propósito** por falta de dados confirmados: Sharp LH28F320BJE e Toshiba TC58FVT160F.
- Plano global com registro (`GLOBAL_PLAN.md`): cada passo feito, cada passo adiado, para a próxima sessão não refazer análise.

## Fase 5: a placa chega (13 de novembro)

O dia mais cheio. Em ordem:

1. **Primeiro contato (firmware 1.0, slot vazio).** A placa enumera como `0483:5740`, `hwtest.py host` e `nocart` passam. Slot vazio lê só `0xFF`. Achado 1: o firmware 1.0 ignora um `connect` que chega logo depois de a porta ser reaberta (a CLI passou a tentar duas vezes). Achado 2: **não há atualização por USB** (CDC puro, sem DFU, bootloader de ROM só por UART): é preciso SWD.
2. **Cartucho original (Rolling Thunder 2), só leitura.** Leituras repetíveis, bits sem travamento, leitura maior que o cartucho vira espelho. O checksum do cabeçalho não bate; sem dump de referência a dúvida ficou em aberto. A pessoa decidiu: **este cartucho nunca será gravado**.
3. **Cartucho flash S29GL064N com firmware 1.0.** Dois achados que mudaram o rumo: (a) o firmware descarta bytes enviados nos 100 ms depois de `detect` (o aplicativo passou a segurar 250 ms); (b) o 1.0 **programa só uma palavra sim, outra não**: nenhuma palavra errada, metade em branco. Contorno: uma ferramenta de reparo que grava só as palavras faltantes, em 8 passagens. Metal Gear (2 MiB) gravado e verificado desse modo.
4. **Sessão SWD.** O conector de 4 pinos foi soldado. A ligação com o ST-Link caiu a cada poucos minutos por horas; a causa era **um pino de GND frouxo**. O chip da unidade é um **STM32F103xE** (512 KiB), não o F103VB dos arquivos de projeto. Backup do firmware de fábrica (sem proteção de leitura), firmware 1.1 gravado, conferido e rodando. O script `swd.sh` ganhou novas tentativas para elos que caem, sempre exigindo a leitura de conferência.
5. **Firmware 1.1 nos cartuchos.** Gravação correta em **uma única passagem** (cerca de 79 s para 2 MiB). A falha de gravação sem apagar dá `WRITE TIMEOUT` em 0,4 s e a placa continua respondendo.
6. **O cartucho KY pregou uma peça.** Mesmo ID (`017E`) que o outro cartucho, mas o apagamento de 2 MiB deixou 56 544 bytes sujos: ele tem **oito setores de 8 KiB** no início, enquanto o outro tem o primeiro setor uniforme. Nem autoselect nem CFI distinguem os dois (e o CFI mudou de valor entre sessões). Decisão de projeto: **apagar sempre em passos de 8 KiB e tratar ID, autoselect e CFI como informação**.
7. **O limite de 2 MiB.** Apagar ou gravar acima de 2 MiB dá timeout; ler acima de 2 MiB mostra a SRAM de save; ler 8 MiB repete 4 MiB. O diagnóstico veio de ler o projeto do cartucho FlashKit (74HC139: flash só abaixo de 2 MiB, SRAM acima, A21 da flash em GND). Experimentos com escritas em `$A13000` e `$A130F1` (por um comando de firmware experimental, depois removido) não mapearam a flash sobre a SRAM. Conclusão: **2 MiB é o limite desses cartuchos**; é a construção deles, não do dumper.
8. **Os cartuchos gravados rodaram num Mega Drive real.**
9. **Guia passo a passo** em quatro idiomas (solda, montagem, firmware 1.1, testes, limite de 2 MiB), aplicativo 1.3.0 e firmware 1.1.0 publicados (30 112 bytes) em um release, com a imagem na unidade de bancada.
10. Último commit do dia: aviso antes de gravar ou apagar acima de 2 MiB nesses cartuchos, título de janela limpo, documentação enxugada (ainda não lançado: seria a 1.3.1).

## Fase 6: passar adiante (14 de novembro)

Preparação deste material para uma pessoa que só tem o MD Rewriter de fábrica e os repositórios públicos: o `CLAUDE.md` (instruções para um agente conduzir o processo, com as descobertas dos cartuchos), este `HISTORY.md`, as traduções dos dois (inglês, russo, chinês simplificado, hindi e urdu) e o guia passo a passo em hindi e urdu.

---

## Números do projeto

| Item | Valor |
|------|-------|
| Período do trabalho | 31 de outubro a 14 de novembro de 2026 |
| Commits do trabalho (depois do original) | 43 (de 31 de outubro a 13 de novembro) |
| Primeiro contato com a placa | 13 de novembro de 2026 |
| Firmware 1.1.0 | 30 112 bytes |
| Aplicativo | 1.3.0 (1.2.0 em 3 de novembro) |
| Leitura de 2 MiB | 10 a 12,5 s |
| Apagar 2 MiB | 6 a 9 s |
| Gravar 2 MiB com firmware 1.1 | cerca de 79 s (cerca de 90 s com a releitura de conferência) |
| Cartuchos usados | 1 original de máscara (só leitura), 2 flash S29GL064N (um deles KY) |

## Como o trabalho foi conduzido

- **Primeiro no papel, depois na bancada.** Quase todo o código nasceu antes de a placa chegar, com firmware simulado, modelo de chip e documentação de cada decisão. Quando a placa chegou, os testes foram feitos em ordem crescente de risco: sem cartucho, original só leitura, flash com firmware 1.0, só então gravar firmware novo.
- **Cada ação destrutiva exigiu o "pode" naquele momento**: gravar o microcontrolador, apagar ou gravar um cartucho. Nenhum teste automático podia gravar ou apagar.
- **Backup antes de tudo**: firmware de fábrica lido duas vezes e guardado em dois lugares; conteúdo do cartucho flash lido antes de ser sobrescrito.
- **Cada achado virou documento e teste**: relatório de bancada com hashes e tempos, regressão no modelo de PC, linha no plano global. O que não foi testado em hardware ficou marcado como tal.
- **Hipóteses conferidas com experimento**: o limite de 2 MiB tinha uma explicação provável (projeto do cartucho) e foi conferido com leitura, escritas de registrador e apagamentos antes de ser afirmado.
- **Autoria e publicação**: a pessoa é a única autora dos commits; publicação só quando ela pediu.

## Erros e correções que valem lembrar

| O que deu errado | Como foi resolvido |
|------------------|--------------------|
| `connect` logo após outro comando ignorado | A CLI repete o `connect` uma vez; o teste espera 0,3 s |
| Primeiro bloco de gravação perdido depois de `detect` | O aplicativo segura 250 ms; correção no firmware proposta (NFW-19), não aplicada |
| Firmware 1.0 grava meia palavra em S29GL064N | Firmware 1.1 faz polling DQ7 após cada palavra |
| Apagar 64 KiB de uma vez deixa setores de 8 KiB sujos no KY | Apagar sempre em 8 KiB abaixo de 64 KiB |
| A regra por autoselect e por CFI errou o layout | Autoselect e CFI viraram só informação |
| A ligação SWD caía de minuto em minuto | Pino de GND frouxo no conector; o `swd.sh` repete a operação e exige releitura |
| Chip da unidade diferente do projeto (F103xE em vez de VB) | `swd.sh` aceita os dois IDs, faz backup da flash inteira |
| Verificação "espelho" da suíte reprovava cartucho flash com dados antigos acima do jogo | O teste só falha quando o período do espelho é menor que o tamanho do jogo |
| Leitura em branco contada como "imagem repetida" | Imagem apagada nunca conta como espelho |
| Um bit lido errado uma vez (não reproduzido) | A verificação por releitura pegou; ela fica sempre ligada por padrão |
| Suposição de que o registrador de mapper liberaria 4 MiB | Experimento mostrou que não; o limite de 2 MiB foi documentado e o aplicativo avisa |

## O que ficou em aberto

- Gravar ou apagar acima de 2 MiB: só em cartucho sem o decodificador do FlashKit (por exemplo o `MD-Flash-Cart-4MB`).
- Leitura de mapper SSF2, Lock-on, Open EverDrive, EverDrive v2, Magistr-16, MX26L6420: nunca rodaram em cartucho.
- Gravação por buffer (`FLASH_FAST`), gravação de save em cartucho flash, queda de energia no meio de uma gravação, desligar o cabo durante a gravação, chips de outras marcas.
- Correção do descarte de bytes após `detect` no firmware (NFW-19).
- Checksum do cabeçalho do Rolling Thunder 2 (precisa de um dump de referência).
- Atualização de firmware por USB (exige um bootloader gravado uma vez por SWD).
- Uma revisão da PCB (resistores Rd no USB-C, pads rotulados, NRST e BOOT0, ESD, `/UWR` e `/CAS2`) ficou fora: o trabalho de hardware novo foi para o projeto irmão `oscr_minimal`.

## Para quem for repetir

1. Comece sem soldar e sem gravar: leia o slot vazio, depois um original, duas vezes cada.
2. Só depois solde o conector, sonde, faça backup, grave. Nessa ordem.
3. Meça sempre; este histórico mostra que a unidade real contradisse os arquivos de projeto (o chip), o modelo (o layout de setores) e a primeira hipótese (o limite de 2 MiB) pelo menos uma vez cada.
4. Registre o que medir, com hashes e tempos. É o que torna o próximo passo seguro.
