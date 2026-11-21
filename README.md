<img width="center" height="128" alt="pokebola" src="https://github.com/user-attachments/assets/085ed913-76fe-4e87-87f9-4b6498d0262d" />

# GBA Launcher

> Emulador de Game Boy Advance em duas camadas: um launcher em Python/tkinter
> para organizar a biblioteca de ROMs (scan, capas, histórico) e um núcleo
> nativo em C++17 + SDL2 com CPU ARM7TDMI, PPU completa, áudio, saves e BIOS
> opcional.

## Como funciona

O programa é dividido em duas camadas que conversam entre si:

- **Launcher** (Python 3.11 + tkinter, sem dependências pesadas): organiza a
  biblioteca, mostra a grade de capas e abre os jogos.
- **Núcleo `gba_core`** (C++17 + SDL2): executa a ROM de verdade — CPU, vídeo,
  áudio, saves e BIOS.

O launcher chama o núcleo como subprocesso (`gba_core <rom> --backend ...`) e
mostra tudo o que o núcleo registra no painel de log da interface.

> **ROMs e BIOS:** este projeto **não inclui nem baixa** ROMs ou BIOS. Você
> fornece suas próprias ROMs `.gba` e, opcionalmente, um `bios.bin` (16 KB)
> dumped da sua própria console. **Sem `bios.bin`**, as chamadas de BIOS/SWI
> são reimplementadas em C++ (HLE) e o trampoline de IRQ é sintético — os jogos
> rodam, mas ficam sem música (o driver de som da BIOS não é emulado).
> **Com `bios.bin`**, o núcleo executa a BIOS real (boot em 0x00000000,
> SWI/IRQ reais), incluindo o driver de som MP2K original → música nas ROMs que
> o usam.

## O que o launcher faz

- **Biblioteca**: varre recursivamente as pastas escolhidas, valida o header de
  cada `.gba` (logo + checksum), lê título e game code e indexa por SHA-1 — o
  mesmo jogo em dois caminhos não entra duas vezes.
- **Atualização automática**: a cada abertura as pastas registradas são
  reescaneadas em segundo plano; jogos novos entram e arquivos apagados saem da
  lista. ROMs inalteradas (mesmo tamanho/data) são puladas sem reler o SHA-1.
- **Grade de capas**: cartões com capa, nome do arquivo, título do header, game
  code e selo (OK / header inválido / ausente). A barra de pesquisa filtra por
  nome, título ou código ignorando acentos. Sem capa, o cartão mostra um
  marcador com gradiente nas cores da logo.
- **Capas sob demanda**: nada é baixado ao abrir. O botão **Baixar capas**
  procura na pasta da ROM, em `covers/`, no cache por SHA-1, nos repositórios
  extras cadastrados e por fim no libretro-thumbnails — só PNG, sem scraping.
- **Opções por jogo e globais**: backend, escala, tela cheia, volume, velocidade
  (1x/3x/6x/8x), filtro de cor e filtro de resolução.
- **Histórico**: até 100 execuções com miniatura e **Reexecutar**; ROMs
  removidas do disco aparecem como ausentes, sem erro.
- **Configurações**: padrões globais, repositórios de capas, caminho do
  `bios.bin` e pastas da biblioteca.

Varredura, histórico e leitura de header funcionam **mesmo sem o núcleo
compilado** — só jogar exige o `gba_core`.

## O que o núcleo faz

- **CPU ARM7TDMI** completa (ARM e Thumb), modos/bancos de registrador,
  CPSR/SPSR, exceções e timing com waitstates por região (aproximado, não
  ciclo-exato).
- **Dois backends**: `interp` (referência de correção) e `dynarec` x86-64 com
  cache de blocos e invalidação; ambos compartilham os mesmos helpers de
  instrução, então os frames saem idênticos byte a byte.
- **PPU completa**: modos 0–5, sprites com rotação/escala e blend, janelas,
  mosaic, mais filtros de cor e resolução aplicados depois da renderização.
- **Áudio**: PSG (canais 1–4) + Direct Sound A/B via FIFO; áudio continua ativo
  em velocidade acelerada (comprimido N:1, estilo fast-forward).
- **Saves**: SRAM, Flash 64/128 KB e EEPROM, detectados automaticamente e
  gravados em `.sav` ao lado da ROM.
- **DMA, timers e IRQ**: scheduler por ciclos (1232/linha, 228 linhas/frame),
  HBlank/VBlank, timers em cascata e DMA com o caso especial do FIFO de som.
- **BIOS**: sem `bios.bin`, as chamadas SWI são reimplementadas em C++ (HLE) —
  os jogos rodam, mas sem música. Com um `bios.bin` de 16 KB fornecido pelo
  usuário, a BIOS real é executada (boot em 0x00000000, SWI/IRQ reais) e o
  driver de som MP2K original produz música.
- **Velocidade**: 1x/3x/6x/8x pelo launcher ou `Tab` durante o jogo, com o
  multiplicador medido em 8,001× a 8x.

## Requisitos e execução rápida

- **Python 3.11+** (tkinter já vem na instalação padrão do Windows).
- **SDL2** (desenvolvimento), **CMake 3.16+** e um compilador C++17: MinGW-w64
  (caminho validado) ou MSVC (requer o Windows SDK). O dynarec x86-64 requer
  Windows x64; em outras plataformas use `--backend interp`.

```bat
cd gba_launcher\core
cmake -S . -B build -G "MinGW Makefiles" -DCMAKE_BUILD_TYPE=Release -DSDL2_ROOT="C:\SDL2-2.32.10"
cmake --build build

cd ..\launcher
python launcher.py
```

Os detalhes de cada opção de compilação, da linha de comando do núcleo e da
geração do executável estão nas seções abaixo.

---

## Estrutura de pastas

```
gba_launcher/
├── README.md
├── build_exe.py              # empacota launcher + núcleo com PyInstaller
├── launcher/                 # interface Python (tkinter)
│   ├── launcher.py           # janela principal: barra lateral, grade de capas, log
│   ├── capas.py              # busca de capas (pasta local, cache, repositórios, libretro)
│   ├── config.py             # configurações globais (config.json)
│   ├── biblioteca.py         # varredura, header, SHA-1, biblioteca.json
│   ├── historico.py          # histórico de execuções (historico.json)
│   ├── nucleo.py             # localização e execução do gba_core
│   ├── pokebola_L.ico        # ícone da janela e do executável
│   └── pokebola.png          # logo exibida na barra lateral
└── core/                     # núcleo nativo C++17 + SDL2
    ├── CMakeLists.txt
    ├── include/
    │   ├── gba.hpp           # CPU, mapa de memória, IO, scheduler
    │   ├── backend.hpp       # interface IBackend + fábrica
    │   ├── interp.hpp        # backend interpretador
    │   ├── dynarec.hpp       # backend dynarec x86-64 (cache de blocos)
    │   ├── custo.hpp         # custos de ciclo por instrução (waitstates)
    │   ├── ppu.hpp           # renderização modos 0-5 + sprites
    │   ├── apu.hpp           # PSG + Direct Sound (FIFO)
    │   ├── saves.hpp         # SRAM / Flash / EEPROM
    │   ├── bios.hpp          # BIOS HLE (SWI) + vetores sintéticos
    │   └── entrada.hpp       # teclado/controle -> KEYINPUT
    └── src/
        ├── main.cpp          # loop principal, janela SDL2, argumentos
        ├── gba.cpp           # memória, IO, scheduler, IRQ, custos de acesso
        ├── backend.cpp       # fábrica + stub decomp
        ├── interp.cpp        # laço do interpretador
        ├── interp_arm.cpp    # conjunto de instruções ARM
        ├── interp_thumb.cpp  # conjunto de instruções Thumb
        ├── dynarec.cpp       # cache de blocos, invalidação, arena executável
        ├── dynarec_x64.cpp   # emissão de código x86-64 por bloco
        ├── dma.cpp           # DMA 0-3 (inclui FIFO de som)
        ├── ppu.cpp           # modos 0-5, sprites, janelas, blend, afim
        ├── apu.cpp           # PSG canais 1-4 + Direct Sound A/B
        ├── saves.cpp         # SRAM/Flash/EEPROM, detecção e .sav
        ├── bios.cpp          # SWI HLE + trampoline de IRQ sintético
        └── entrada.cpp       # mapeamento de entrada
```

`config.json`, `biblioteca.json` e `historico.json` são criados
automaticamente no primeiro uso (não fazem parte do código). Em execução
normal eles ficam em `launcher/`; no executável empacotado, em `dados/` ao
lado do `.exe`. As pastas opcionais `covers/` (capas fornecidas por você) e
`covers_cache/` (capas baixadas + índice do libretro) ficam na mesma raiz de
dados.

---

## Requisitos

- **Python 3.11+** (tkinter já vem na instalação padrão do Windows).
- **SDL2** (desenvolvimento): headers + biblioteca. Pode ser obtido em
  <https://github.com/libsdl-org/SDL/releases> (pacote `SDL2-devel-2.x-VC.zip`
  para Windows) ou via vcpkg / pacman / apt.
- **CMake 3.16+** e um compilador C++17:
  - **MinGW-w64** (GCC) — caminho validado nesta entrega; ou
  - **MSVC** (Visual Studio 2019/2022) — requer o **Windows SDK** instalado
    (sem ele o MSVC falha com `C1083: stddef.h`).
- O **dynarec x86-64** requer Windows x64 (usa `VirtualAlloc` com memória
  executável). Em outras plataformas/compiladores, use `--backend interp`.

---

## Compilar o núcleo (Windows)

### Opção A — MinGW-w64 (validada)

Com SDL2 extraído em `C:\SDL2-2.32.10` (contendo `include\` e `lib\x64\`):

```bat
cd gba_launcher\core
cmake -S . -B build -G "MinGW Makefiles" ^
      -DCMAKE_BUILD_TYPE=Release ^
      -DSDL2_ROOT="C:\SDL2-2.32.10"
cmake --build build
```

Se tiver Ninja, troque `-G "MinGW Makefiles"` por `-G Ninja` (mais rápido).
O `CMakeLists.txt` embute o runtime C++ (`-static`) no MinGW, então o
executável só precisa de `SDL2.dll` ao lado (copiado automaticamente quando o
SDL2 é localizado via `SDL2::SDL2`; com a busca manual, copie `SDL2.dll` para
`build\` você mesmo).

Compilação direta sem CMake (alternativa, também validada):

```bat
cd gba_launcher\core
g++ -std=c++17 -O2 -Wall -Wextra -Iinclude -IC:\SDL2-2.32.10\include ^
    src\*.cpp -o build\gba_core.exe C:\SDL2-2.32.10\lib\x64\SDL2.dll -static
```

### Opção B — MSVC

Requer o Windows SDK. No "x64 Native Tools Command Prompt":

```bat
cd gba_launcher\core
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 -DSDL2_ROOT="C:\SDL2-2.32.10"
cmake --build build --config Release
```

O executável fica em `build\Release\gba_core.exe` (o launcher procura lá).

### Linux / macOS

```sh
cd gba_launcher/core
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release   # SDL2 via pkg-config/apt
cmake --build build
```

---

## Executar

### Launcher (interface gráfica)

```bat
cd gba_launcher\launcher
python launcher.py
```

A janela tem uma barra lateral escura (Jogos, Recentes, Configurações), com a
logo no topo, e uma área clara com a lista de jogos em grade; o log do núcleo
fica no rodapé. As cores seguem a logo (`pokebola_L.ico`): preto, vermelho,
branco e amarelo; o mesmo `.ico` é o ícone da janela e do executável.

- **Jogos**: grade de cartões com capa, título e selo de situação (OK, header
  inválido ou ROM ausente). O título do cartão é o **nome do arquivo** da ROM
  (sem `.gba`) e o detalhe mostra título do header · game code · backend; a
  lista é ordenada pelo nome do arquivo. A grade se ajusta de 2 a 4 colunas
  conforme a largura da janela e rola até o fim do conteúdo (sem espaço vazio
  extra). Clique seleciona (borda azul), duplo clique executa. Jogos sem capa
  exibem um marcador com gradiente e iniciais. A **barra de pesquisa** filtra a
  grade ao digitar (ignorando acentos e maiúsculas) pelo nome do arquivo,
  título do header ou game code; `×` ou `Esc` limpa o campo, e a contagem de
  jogos visíveis aparece no rodapé. O botão **Baixar capas** procura as capas
  ausentes só quando clicado (nada é baixado na abertura).
- **Indicador de scan**: enquanto as pastas são varridas (inclusive na
  abertura), uma barra animada aparece na barra lateral, logo acima do status
  “Escaneando pastas…”, e some junto com o fim da varredura.
- **Abrir ROM**: adiciona um `.gba` avulso à biblioteca.
- **Escanear pasta**: varre recursivamente `.gba`, valida o header (logo +
  checksum), lê título/game code, indexa por SHA-1 (sem duplicatas) e registra
  a pasta para as próximas atualizações.
- **Opções do jogo**: backend, escala, tela cheia, volume, velocidade
  (1x/3x/6x/8x), filtro de cor e filtro de resolução, por jogo.
- **Recentes**: histórico (até 100 entradas) com miniatura da capa e
  **Reexecutar**; ROMs que sumiram do disco aparecem como ausentes, sem erro.
- **Configurações**: padrões globais (backend, escala, tela cheia, volume,
  velocidade, filtros), capas (**Baixar capas ausentes** e **Repositórios de
  capas…**), BIOS opcional (`Escolher…`/`Limpar`) e as pastas da biblioteca
  (`Adicionar pasta…`, `Remover`, `Reescanear agora`).

**Atualização automática:** em toda inicialização as pastas registradas em
`config.json` (`pastas_biblioteca`) são reescaneadas em segundo plano; jogos
novos entram na lista e jogos cujo arquivo não existe mais (inclusive ROMs
avulsas) são removidos da biblioteca, com o resumo no log. A interface não
trava durante a varredura.

**Abertura rápida:** a varredura registra tamanho e data de modificação de
cada ROM e pula a leitura do arquivo quando nada mudou (nada de recalcular o
SHA-1 de centenas de ROMs a cada abertura). A grade é montada em lotes de 12
cartões, então a janela aparece e responde enquanto o restante é desenhado; a
grade só é reconstruída se algo visível mudou (jogo novo, removido ou
metadados diferentes).

**Rolagem fluida:** pedidos da barra e da roda do mouse são agrupados em um
alvo por quadro (~16 ms) antes de repintar a grade, o recálculo de colunas é
ignorado durante a rolagem (só a largura da janela o dispara) e a região de
rolagem só é reescrita quando o conteúdo muda; o realce de hover é adiado
para quando a rolagem para. Um arrasto contínuo na biblioteca completa gasta
cerca de 57% menos CPU e a tela acompanha o cursor.

### Capas dos jogos

Cada jogo procura a capa nesta ordem (o primeiro resultado vence):

1. **Pasta da ROM**: `capa.png`, `cover.png`, `boxart.png` ou
   `<nome-da-rom>.png`/`.gif`.
2. **`covers/`** (raiz de dados): `<game_code>.png`, `<sha1>.png` ou
   `<título>.png`.
3. **Cache** `covers_cache/<sha1>.png` (capas já baixadas).
4. **Repositórios adicionais** cadastrados em **Repositórios de capas…**
   (`sites_capas` no `config.json`), na ordem em que aparecem: tenta
   `<url>/Named_Boxarts/<nome-do-arquivo>.png` e depois
   `<url>/<nome-do-arquivo>.png` (somente PNG, nome exato do arquivo).
5. **libretro-thumbnails** (`Nintendo_-_Game_Boy_Advance`): baixa apenas o
   *boxart* em PNG correspondente ao nome da ROM/título. O índice do repositório
   fica em cache por 7 dias; falhas por jogo são silenciosas e o resumo aparece
   no log ("Capas: N obtida(s) de M...").
6. **Marcador**: gradiente com as iniciais do título.

**Nada é baixado na abertura.** O download só acontece ao clicar em
**Baixar capas** (página Jogos) ou **Baixar capas ausentes** (Configurações);
a busca roda em segundo plano e os botões ficam desabilitados até terminar.
A busca no libretro é conservadora: nomes normalizados (acentos e pontuação
descartados) precisam bater quase exatamente; em caso de empate ou dúvida o
jogo fica com o marcador. Por isso hacks/rótulos personalizados podem não
receber capa (ou herdar a do jogo-base). A capa local sempre tem prioridade.

> **Somente imagens:** o launcher usa o suporte nativo do Tk (PNG e GIF), sem
> Pillow. Capas em JPG/WebP são ignoradas — converta para PNG. O download busca
> **apenas boxart**; ROMs e BIOS nunca são baixados. Repositórios adicionais
> precisam servir arquivos por essa convenção de nomes; páginas HTML não são
> interpretadas (não há raspagem).

O launcher procura o `gba_core` em `core\build\Release`, `core\build\Debug`,
`core\build\`, na raiz do projeto, na pasta do launcher e por fim no `PATH`.
É possível fixar um caminho manual em `config.json` (`caminho_nucleo`).

A página **Configurações** define o caminho global do `bios.bin` (16 KB),
gravado em `config.json` como `caminho_bios`. O launcher repassa
`--bios <caminho>` ao núcleo em todas as execuções. Deixe vazio para usar o HLE
(sem música). Varredura, histórico e header continuam funcionando sem o núcleo
compilado.

### Núcleo (linha de comando)

```bat
gba_core.exe caminho\para\jogo.gba --backend dynarec --scale 3 --volume 80
```

Argumentos: `--backend interp|dynarec|decomp`, `--bios <bios.bin>`, `--scale N`
(1–8), `--fullscreen`, `--volume 0-100`, `--speed 1|3|6|8`,
`--filtro-cor normal|lcd|vibrante|pb`, `--filtro-res nearest|bilinear|lcd`.

- **`--speed`**: multiplica a velocidade executando N quadros emulados por
  quadro apresentado (60 Hz). O áudio continua ativo: a APU comprime N
  amostras em uma média, então o som acompanha a velocidade (tom mais
  agudo) sem acumular atraso na fila. `Tab` alterna 1x→3x→6x→8x durante o
  jogo, com aviso no log.
- **`--filtro-cor`**: `normal` (sem alteração), `lcd` (dessatura levemente,
  aproxima o painel sem retroiluminação), `vibrante` (aumenta a saturação) ou
  `pb` (escala de cinza por luminância BT.601).
- **`--filtro-res`**: `nearest` (pixels nítidos), `bilinear` (suavização na
  ampliação) ou `lcd` (grade LCD: upscale ×3 com linhas de grade, **exige
  escala ≥3** — abaixo disso o núcleo usa 3 e avisa no log).
- **`--frames N`** (oculto, diagnóstico): encerra após N quadros apresentados
  e registra o total de ciclos emulados; é o que permite medir a velocidade
  real (ex.: 8x medido = 8,001× o de 1x).

**BIOS real opcional:** `--bios <caminho>` carrega um `bios.bin` de exatamente
16 KB. Sem o argumento, o núcleo procura `bios.bin` ao lado do executável e no
diretório corrente. Se a BIOS não for encontrada ou tiver tamanho inválido, cai
no HLE com um aviso no log. Com a BIOS real, o boot é em 0x00000000 (modo SVC) e
SWI/IRQ são exceções reais; o driver de som MP2K da BIOS roda e produz música.

- **interp**: interpretador ARM/Thumb completo (referência de correção).
- **dynarec**: tradução de blocos para x86-64 com cache e invalidação.
- **decomp**: ponto de extensão para builds nativas de projetos *pret*; ainda
  não disponível — cai para `interp` com um aviso.

---

## Controles

| GBA      | Teclado     | Controle (SDL) |
|----------|-------------|----------------|
| A        | Z           | A              |
| B        | X           | B              |
| L        | A           | LB             |
| R        | S           | RB             |
| Start    | Enter       | Start          |
| Select   | Backspace   | Back           |
| D-pad    | Setas       | D-pad          |

`Esc` ou fechar a janela encerra o núcleo. `Tab` alterna a velocidade
(1x → 3x → 6x → 8x) durante o jogo.

---

## Gerar o executável (Windows)

Com o núcleo compilado e o PyInstaller instalado:

```bat
cd gba_launcher
python build_exe.py
```

O script gera `dist\GbaLauncher\` com o `GbaLauncher.exe` (Python e tkinter
embutidos, sem console), `gba_core.exe`, `SDL2.dll`, a arte da logo
(`pokebola_L.ico` como ícone do executável e da janela + `pokebola.png`) e a
pasta `dados/` semeada com a biblioteca e o histórico atuais. A BIOS não é
embutida nem copiada — você a aponta na página Configurações; ROMs também não
são incluídas.

---

## O que está implementado

- **Launcher completo:** interface com barra lateral e grade de capas, abrir
  ROM, escanear pasta, biblioteca por SHA-1, pesquisa, opções por jogo,
  histórico com reexecução, página de configurações (padrões, capas, BIOS,
  pastas) e painel de log; a lista é atualizada em toda inicialização
  (reescaneia as pastas e remove jogos ausentes); header funciona sem o núcleo
  compilado.
- **CPU ARM7TDMI:** interpretadores ARM e Thumb completos, modos/bancos de
  registradores (USR/SYS/FIQ/IRQ/SVC/ABT/UND), CPSR/SPSR, flags N/Z/C/V,
  exceções (SWI, IRQ, indefinida) e retorno com restauração de CPSR. Escritas no
  PC via DP/LDM preservam o bit 1 (endereços Thumb semi-alinhados), como no
  ARMv4T — necessário para o retorno de IRQ do trampoline da BIOS.
- **Dynarec x86-64:** traduz blocos ARM/Thumb para código nativo em tempo de
  execução, com **cache de blocos** (chave = PC + estado Thumb) e **invalidação**
  em escritas na memória de código (EWRAM/IWRAM); mudança de WAITCNT limpa o
  cache. A semântica de cada instrução usa **micro-helpers compartilhados com o
  interpretador**, garantindo resultado idêntico (validado: frames byte a byte
  iguais entre `interp` e `dynarec` numa ROM real).
- **Vídeo (PPU completa):** modos 0/1/2 (BG texto e afim), modos 3/4/5 (bitmap),
  sprites/OAM (afins, tamanho dobrado, semi-transparência, prioridade), janelas
  (WIN0/WIN1/OBJ), blend alpha e brilho, mosaic.
- **Áudio (APU):** PSG canais 1-4 (quadrada com sweep, wave, ruído) + Direct
  Sound A/B (FIFO alimentado por DMA1/DMA2, amostrado por timer), mixagem SDL.
- **Saves:** SRAM, Flash 64/128 KB e EEPROM 512 B/8 KB, com detecção automática
  do tipo e persistência em `.sav` ao lado da ROM.
- **DMA, timers e IRQ:** scheduler de eventos (1232 ciclos/linha, 228
  linhas/frame), HBlank/VBlank, timers 0-3 (cascata e IRQ), DMA 0-3 (incluindo o
  caso especial de FIFO de som), IRQ de teclado.
- **BIOS HLE (SWI):** SoftReset, RegisterRamReset, Halt/Stop, IntrWait,
  VBlankIntrWait, Div/DivArm, Sqrt, ArcTan/ArcTan2, CpuSet/CpuFastSet,
  GetBiosChecksum, BgAffineSet/ObjAffineSet, BitUnPack, descompressão
  LZ77/Huff/RL/Diff, SoundBias, MidiKey2Freq. Trampoline de IRQ sintético.
- **BIOS real opcional:** com um `bios.bin` (16 KB) fornecido pelo usuário, o
  núcleo executa a BIOS real — boot em 0x00000000 (SVC), SWI como exceção ao
  vetor 0x08 e IRQ real (handler do usuário em 0x03FFFFFC). O driver de som MP2K
  original roda dentro da BIOS e escreve nos registradores que a APU já emula →
  **música** nas ROMs que o usam. O dynarec herda o modo real pelos helpers
  compartilhados (paridade byte a byte mantida). Sem `bios.bin`, o HLE permanece
  como fallback.
- **Timing por instrução:** custos de ciclo com waitstates por região conforme
  WAITCNT, detecção de acesso sequencial/não-sequencial, custos de MUL/MLA e
  transferências em bloco (compartilhado entre interp e dynarec).
- Entrada por teclado e controle; janela SDL2 com escala e tela cheia.
- **Velocidade 3x/6x/8x:** N quadros emulados por quadro apresentado, com
  áudio ativo comprimido N:1 (tom acompanha a velocidade); atalho `Tab` em jogo.
- **Filtros de cor** (normal/LCD/vibrante/P&B) por pixel e **filtros de
  resolução** (nearest/bilinear/grade LCD ×3) aplicados após a PPU, sem
  alterar o timing emulado.

## Limitações conhecidas (declaradas)

Os itens abaixo são limitações **declaradas**, não comportamentos inventados:

- **Timing aproximado:** os custos modelam waitstates por região e os principais
  efeitos por instrução, mas **não são ciclo-exatos** (pipelines, prefetch buffer
  e detalhes finos de waitstate não são reproduzidos). VBlank/HBlank e VCOUNT
  derivam do contador de ciclos (1232/linha, 228 linhas/frame).
- **Dynarec com micro-helpers:** a tradução nativa cobre o laço de bloco
  (fetch/avanço de PC/contagem de ciclos/encadeamento), mas a semântica de cada
  instrução ainda passa pelos helpers C++ compartilhados. Isso garante paridade
  exata com o interpretador, porém o ganho de desempenho é **moderado** — não é
  um recompilador nativo completo de instruções. Requer Windows x64.
- **Música sem `bios.bin`:** no fallback HLE, os SWI do driver de som da BIOS
  (0x1A–0x2F, exceto 0x1F) são no-op (retornam 0), então as ROMs que dependem do
  driver MP2K ficam **sem música**; o PSG/Direct Sound por registradores funciona
  normalmente. Com `bios.bin` real, o driver roda e a música é produzida.
- **Open-bus da BIOS não modelado:** leituras da região 0x00000000 são sempre
  permitidas (não há emulação de open-bus quando a BIOS está "protegida"). Isso
  não afeta o boot nem a execução das ROMs da biblioteca.
- **`bios.bin` de exatamente 16 KB:** tamanho inválido é rejeitado com aviso e o
  núcleo cai no HLE. O projeto não fornece nem baixa a BIOS.
- **Stop (SWI 0x03):** tratado como Halt (não há modo de parada real de clock).
- **Decomp:** stub (`disponivel() = false`); ponto de extensão para builds
  nativas de projetos pret, ainda não implementado.
- **Escritas em IO não mapeado:** ignoradas com um aviso (uma vez por endereço),
  em vez de comportamento inventado.
- **Precisão de saves/EEPROM:** o protocolo serial é modelado de forma funcional;
  temporizações exatas de EEPROM não são reproduzidas.
- **Filtro de cor `lcd` é aproximação:** dessatura misturando 3/4 da cor com 1/4
  da luminância — reproduz o aspecto geral do painel GBA sem retroiluminação,
  não uma calibração colorimétrica exata.
- **Áudio em velocidade >1x é comprimido, não time-stretched:** cada N
  amostras emuladas viram a média de 1, então todas as notas são ouvidas e
  sem atraso, mas o tom sobe junto com a velocidade (estilo fast-forward).
  Em 8x a taxa efetiva cai para ~4 kHz e o som fica bastante agudo/abafado;
  não há preservação de tom original nem qualidade de rádio-esporte.
- **Grade LCD exige escala ≥3:** em modo janela a grade é desenhada no upscale
  ×3 interno (720×480); com `--scale 1` ou `2` o núcleo força 3 e registra um
  aviso no log.
- **Capas apenas em PNG/GIF:** o Tk 8.6 lê nativamente só esses formatos (sem
  Pillow), então JPG/WebP na pasta da ROM ou em `covers/` são ignorados. O
  libretro-thumbnails só fornece PNG, nunca JPG.
- **Busca de capa conservadora:** nomes com acentos/pontuação são normalizados,
  mas é exigida correspondência quase exata (e sem empate). Hacks, traduções e
  rótulos personalizados podem ficar sem capa ou receber a capa do jogo-base;
  nenhuma arte é escolhida por aproximação frouxa.
- **Repositórios adicionais são por convenção de URL:** não há leitura de
  páginas HTML nem escolha por aproximação; o servidor precisa entregar o PNG
  no caminho exato derivado do nome do arquivo da ROM.

## Validação

- Testes por subsistema (CPU, dynarec×interp, invalidação de cache, HW/SWI, PPU,
  saves, APU, BIOS real, leitura de registradores DMA) e diferencial
  interp×dynarec: todos passando. Regressões de PPU: mapa dos registradores de
  janela (WIN0H/WIN1H/WIN0V/WIN1V) e OBJ semi-transparente (alpha forçado,
  opaco sem 2nd target, brilho não se aplica). Regressão de paleta: `LDR` word
  desalinhado rotaciona o dado lido (`ROR (addr AND 3)*8`, conforme GBATEK) —
  sem isso, laços de fade que avançam a fonte de halfword em halfword
  duplicavam cores e tingiam sprites de magenta em todos os jogos; o teste
  cobre a leitura rotacionada e um laço de fade Thumb real de 16 cores.
- **Launcher:** testes Python cobrem scan/header/SHA-1, persistência de
  config/biblioteca/histórico, pastas da biblioteca e remoção de jogos
  ausentes, re-scan que ignora ROMs inalteradas (sem reler SHA-1), duplicatas
  (mesmo SHA em dois caminhos) sem duplicar a entrada e com caminho estável
  entre scans, a lógica de capas (normalização, prioridade pasta da ROM →
  `covers` → cache, repositórios adicionais antes do libretro, validação de URL,
  pontuação conservadora, índice e download com validação PNG, nada baixado
  com a busca desligada) e um smoke test da GUI (construção, grade de cartões,
  ordenação pelo nome do arquivo, título/detalhe em uma linha, região de
  rolagem acompanhando o conteúdo, roda do mouse liberada ao sair, navegação
  entre páginas, salvar configurações, aplicação de capa pela fila, atualização
  em segundo plano, nenhum download na abertura, botão **Baixar capas**,
  janela de repositórios adicionando/removendo/persistindo sites e rejeitando
  URL inválida, grade reconstruída só quando algo visível muda, seleção
  repintando apenas os cartões afetados, busca por nome/título/game code
  ignorando acentos, limpeza da busca, aviso de “nenhum jogo”, filtro mantido
  ao reconstruir a grade e o indicador de scan aparecendo/sumindo). Validado
  com as ROMs reais: a atualização de inicialização adicionou a ROM nova e
  removeu da lista a entrada cujo arquivo havia sido apagado; na grade real,
  FireRed/LeafGreen/Mario receberam o boxart correto (3 de 5, com os 2 hacks
  ficando no marcador) e a janela refluiu de 4 para 3 colunas.
- **Fallback HLE (sem BIOS):** *Pokémon FireRed* inicia, roda o laço principal e
  renderiza a tela de título de forma estável por **20000 frames consecutivos**
  (~5,5 min de emulação), sem exceções; o ponto antigo de crash (~frame 2990) é
  atravessado sem corrupção de pilha. Frames `interp` e `dynarec` idênticos byte a
  byte.
- **BIOS real (com `bios.bin` de 16 KB):** as **5 ROMs** então na biblioteca rodam
  estáveis por 3000 frames cada, sem exceções indefinidas (UND=0) e com
  `bios_real=1`: *Pokémon FireRed*, *LeafGreen*, *Psychic Adventure v3.5*,
  *Pigment Ruby v1.0* e *Girls Hunter 3*. O boot real inicia em 0x00000000 (SVC).
- **Música:** o áudio da APU capturado com a BIOS real é **não-silencioso**
  (WAV PCM16 estéreo 32768 Hz; RMS ≈ 1484, pico ≈ 14000 em 3000 frames de
  FireRed) — o driver MP2K original produz música.
- **Paridade com BIOS real:** frames despejados por `interp` e `dynarec` são
  idênticos byte a byte (mesmo contador de ciclos), confirmando que o dynarec
  herda SWI/IRQ reais sem divergência.
- **Saves:** FireRed é detectado como **Flash 128K**; o teste de roundtrip
  (program/erase, troca de banco, IDs Sanyo/Panasonic, EEPROM e persistência
  `.sav`) passa.
- **Velocidade (medida):** com `--frames 120`, o total de ciclos emulados a 8x
  é **8,001×** o de 1x e a 6x é **6,0×** exato — o multiplicador entrega a
  velocidade pedida, não um valor aproximado.
- **Áudio acelerado:** teste unitário confirma que a 3x o ring recebe ~1/3
  das amostras (sem backlog), a média preserva o valor de um FIFO constante
  e voltar a 1x restaura a taxa cheia; valores fora de 3/6/8 caem para 1x.
- **Filtros (visual):** despejos de FireRed (frame 3400) confirmam P&B em
  escala de cinza, vibrante saturado, LCD dessaturado e a grade LCD em
  720×480 com scanlines; o contador de ciclos é idêntico ao do modo normal,
  provando que os filtros são pós-processamento puro. O teste unitário cobre
  identidade, saturação, cinza neutro, grade ×3 e o parsing dos nomes.
