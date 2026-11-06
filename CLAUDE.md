# CLAUDE.md — Gerber para CopperCAM

Ferramenta web de um arquivo só (`index.html`) que converte o Gerber exportado pelo **EasyEDA Pro** para um formato que o **CopperCAM** lê sem erro, e confere a placa para fresagem (folgas, borda, anel dos furos, cobre solto, jumpers, curto e rede aberta pela netlist). Publicada no GitHub Pages. Interface e textos em **português do Brasil**.

## Regras do projeto

- Tudo fica em `index.html`: HTML, CSS e JS. Sem build, sem dependências externas em tempo de execução, sem servidor. Tem que funcionar **offline** abrindo o arquivo com dois cliques.
- A biblioteca **JSZip 3.10.1** está embutida num `<script>` próprio (o bloco grande minificado). Não edite esse bloco.
- Os arquivos da placa nunca saem do navegador. Não adicione analytics, fetch ou upload.
- Fontes do Google são opcionais: sem internet, a página cai nas fontes de sistema.
- Cores só por tokens CSS em `:root`, com tema claro e escuro (`prefers-color-scheme` e `[data-theme]`). A página precisa funcionar em 400 px de largura sem rolagem horizontal.
- Textos da interface: frases curtas e diretas, voz ativa, nomeando as coisas como o usuário de EasyEDA/CopperCAM conhece.

## Estrutura do código (dentro de `index.html`)

1. **Núcleo puro** (primeiro `<script>` depois do JSZip): `tokenize`, `processGerber`, `parseDrill`, `buildDrill`, `bboxOf`, `parseNetlist`. Não tocam no DOM.
   - `processGerber(texto, rótulo)` → `{ out, objs, aps, warns }`. `out` é o Gerber convertido; `objs` é a geometria em mm (`flash`, `draw` com arco opcional, `reg` com contornos); `aps` são as aberturas em mm.
   - `parseDrill` lê Excellon (METRIC/INCH, LZ/TZ, com ou sem ponto decimal, slots `G85`). `buildDrill` junta furos de vários arquivos, remove repetidos, troca slots por furos redondos e grava o Excellon convertido.
   - `parseNetlist(texto)` lê o `FlyingProbeTesting.json` → `{ pins, nets }`, um pino por posição física, em mm: `{ name, ref, num, x, y, net, padX, pad, hole }`.
2. **Interface** (segundo `<script>`): estado global `st`, parâmetros (`params()`, salvos em `localStorage`), análise (`analyzeLayer`, `connectivity`, `analyzeAll`), imagens base por camada (`buildBases`), visualização com zoom/pan (`draw`, `zoomAt`, `centerOn`, `fitPoints`, eventos de ponteiro e roda), lista de avisos (`renderUI`, `select`), destaque de furos por diâmetro (`showDia`), relatório (`reportText`) e montagem do .zip (`buildZip`).
   - `analyzeLayer(nome, camada, G, furos, P, NL)` devolve também `lab` (rótulo da ilha de cada pixel), `nl` e `pinPix` (pino → pixel de cobre). Com a netlist (`NL`), põe nomes de pino nos avisos.
   - `connectivity(...)` → `{ nets, shorts, opens }`; `meetInIsland` acha o ponto do curto dentro de uma ilha.
   - Textos de pino e rede: `pinTxt` ("C6 pino 2"), `pinNet` ("C6 pino 2 (GND)"), `netTxt` ("rede sem nome ($2N8)"), `netIn` ("rede sem nome $2N8", para dentro de parênteses).
   - Um aviso pode ter `pts: [[x, y, k], ...]`: ao clicar, a prévia enquadra esses pontos e desenha cada um na cor `PARTCOL[k]` (pedaço da rede aberta ou rede do curto). Com `center: true`, o clique centraliza em `(x, y)` em vez de enquadrar os pontos.

## Netlist: FlyingProbeTesting.json do EasyEDA Pro

Vem dentro do .zip do Gerber. É opcional: sem ele, a página funciona igual, sem nomes nos avisos e sem checagem de curto/aberto, e mostra a nota "Sem netlist no .zip: …".

- `"lengthUnit": "mil"`. Coordenadas em mil, com a mesma origem do Gerber (canto inferior esquerdo do contorno): mm = mil × 0,0254. Conferido: os 528 registros de pino da placa de áudio caem em cima de flashes do Bottom.
- `"pins": { fields: [PIN_NO, PIN_NAME, PIN_X, PIN_Y, LAYER, PIN_TYPE, NET_NAME, NET_TYPE, PAD_SHAPE, PAD_SIZEX, PAD_SIZEY, HOLE_SIZE, HOLE_LEN, PAD_ANGLE], rows: [...] }`.
  - **Cada pino físico aparece repetido na mesma coordenada** (4 linhas na placa de áudio): duas com o nome do componente (`C6_2`) e duas com um apelido `PADn_m` (`PAD8_2`). Agrupe por coordenada e fique com o nome que não começa com `PAD`. `PIN_NAME` = `<designador>_<pino>`.
  - **`LAYER` vem `"T"` em todos, inclusive nos PTH.** Não use para decidir a camada: o pino entra na ilha da camada que tem cobre no centro do pad. `HOLE_SIZE > 0` indica furo passante.
  - `NET_NAME` começando com `$` é rede sem nome (`$2N8`): mostre como "rede sem nome ($2N8)".
  - Pad do tamanho do furo (ex.: CN1 pino 3, pad = furo = 1,2 mm) não tem cobre; se o centro não tiver cobre, procure no raio `PAD_SIZEX/2`.
- `"components"`: centro e ângulo de cada componente; entradas `PADn` são pads soltos e vias. Não é usado hoje.
- **Vias não estão na lista de pinos.** As pontas dos jumpers da Top caem em furos de via.

## Como funciona a checagem de redes

- **Nomes nos avisos:** na folga, um pino de cada lado, o mais perto do pixel de origem de cada frente *dentro da própria ilha* (o mais perto no geral pode ser da outra ilha e daria "entre C6 pino 2 e C6 pino 2"). Pino a mais de 3 mm (`NEAR`) não entra no texto; se um lado tiver pino perto e o outro não, o outro aparece como "cobre da rede X". No anel e no furo sem cobre, o pino é o do próprio furo. Borda e cobre solto citam o pino mais perto, se houver a menos de 3 mm.
- **Union-find** com as ilhas do Bottom, as ilhas da Top (dupla face) e os pontos das trilhas da Top (face simples).
  - **Face simples:** as trilhas da Top são jumpers. Primeiro una TODOS os pontos das trilhas (cada trecho une suas duas pontas; uma cadeia de vários trechos vira um nó só). Só depois ligue ao Bottom as pontas de cadeia (grau 1) e os pontos que estão em cima de um furo. **Ligar trecho por trecho direto às ilhas dá rede aberta falsa**, porque os pontos do meio da cadeia não caem em cobre do Bottom. Ligar dobras sem furo daria curto falso quando a dobra passa por cima de um plano do Bottom.
  - **Dupla face:** cada furo passante une a ilha do Bottom e a da Top naquele ponto.
- **Curto:** grupo conectado com pinos de redes diferentes. O ponto do aviso é onde as frentes que crescem de cada pino pelo cobre se encontram (`meetInIsland`); se o curto passa por jumper ou furo, é a ligação entre nós de redes diferentes; se nada disso der, o primeiro pino.
- **Rede aberta:** pinos da mesma rede em grupos diferentes. De cada pedaço, o aviso cita o pino mais perto de outro pedaço (é por ali que falta ligar).
- Na placa de áudio, várias ilhas de GND são ligadas por mais de um jumper. Apagar um desses jumpers **não** abre o GND, e está certo.

## O que o CopperCAM exige (aprendido na prática — não mude sem testar no CopperCAM)

- **Formato de saída:** `%MOIN*%`, `%FSLAX24Y24*%`, `%IPPOS*%`, coordenadas em polegada com 4 decimais. O Gerber original do EasyEDA Pro (`%MOMM*%`, `%FSLAX45Y45*%`) abre com escala errada no CopperCAM.
- **Aberturas renumeradas a partir de D20**, com `D10` e `D11` definidos como "fantasma" (`C,0.0010`) e sem uso. O CopperCAM lia errado os pads que caíam no código **D11**: eles apareciam sem cobre e ele avisava "Some holes are bigger than their pads". Foi confirmado em duas versões da placa (o D11 era o pad de 1,5 mm numa e o de 1,6 mm na outra).
- **Comentários `G04` e atributos X2 (`%TF`, `%TA`, `%TO`) são removidos.**
- **Furação:** o EasyEDA Pro exporta `Drill_PTH_Through.DRL` e `Drill_PTH_Through_Via.DRL`, e o arquivo de vias repete furos que já estão no PTH. Por isso tudo é juntado e deduplicado. Saída: `INCH,LZ`, ferramentas com 4 decimais em polegada, coordenadas com 6 dígitos (formato 2.4).
- **Slots (G85)** não são aceitos pela furação da fresa: viram furo redondo com diâmetro = comprimento total do slot, e a página avisa.
- **Nada é espelhado**: o CopperCAM espelha na hora de fresar.

## Armadilhas já resolvidas

- **Sentido dos arcos na tela:** a imagem é desenhada com o Y invertido para ficar "de cima", então um arco G03 (anti-horário no Gerber) continua anti-horário na tela: `ctx.arc(..., anticlockwise = (dir === 3))`. Com o sinal trocado, os arcos saem pelo lado maior e aparecem folgas falsas.
- **Regiões (pour):** preencher cada contorno com `fill('evenodd')`.
- **Folgas:** a análise rasteriza a camada (até 25 px/mm), rotula as ilhas de cobre e propaga frentes a partir de cada ilha guardando o pixel de origem. Quando duas frentes de ilhas diferentes se encontram, a folga é a distância entre as origens. Corte em `largura da ferramenta + 0,15 mm`.
- **Coordenadas dos avisos** são em mm a partir do canto inferior esquerdo do contorno (a origem do EasyEDA).

## Como validar mudanças

1. **Conversão:** renderize o Gerber/Excellon original e o convertido com o **gerbv** (`gerbv -x png -D 200 -B 0 ...`) e compare as imagens pixel a pixel. Devem ser praticamente idênticas (só arredondamento de borda). Qualquer mudança em `processGerber`, `parseDrill` ou `buildDrill` precisa passar por isso.
2. **Página:** teste com Playwright abrindo `index.html` via `file://` e com a rede bloqueada (garante que funciona offline). Carregue um .zip do EasyEDA Pro com `setInputFiles('#file', ...)` e espere `window.__result`. Confira o veredito, a lista de avisos, o clique num aviso (o marcador tem que ficar no centro da prévia) e o download do pacote.
3. **Referências conhecidas** (placa de áudio do autor, parâmetros padrão, face simples):
   - `Gerber_PCB2_2026-V3.zip` e `Gerber_PCB2_2026-V6.zip`: "Pronto, com avisos", 165 furos, 16 jumpers, menor folga ≥ 0,45 mm, 33 redes, zero curtos e zero abertos. O único aviso é o furo sem cobre do CN1 pino 3.
   - `Gerber_PCB2_2026-11-06.zip`: "Pronto, com avisos", 163 furos, 15 jumpers, menor folga 0,40 mm, 33 redes sem curto nem aberto.
   - Em dupla face, a V6 também dá 33 redes sem curto nem aberto.
   - Referências antigas, de versões cujo .zip não foi guardado: trilha a 0,16 mm do pad de GND do C6 em (59,3; 29,7) dava "Corrija antes de fresar"; a versão corrigida dava 157 furos e 12 jumpers.
   - Se der rede aberta no GND nessas placas, o problema é a ligação das cadeias de jumper (veja acima).
4. **Testes sintéticos** (gere os .zip modificados no próprio teste, sem tocar nos originais). Coordenadas do C6 na V3/V6: pino 1 ($2N8) em (2455; 1229,21) mil, pino 2 (GND) em (2455; 1130,79) mil, pads de 1,8 mm. O C6 pino 2 fica dentro de um plano de GND: trilha passando ao lado dele encosta no plano e dá curto (correto), não folga.
   - V3 + toco de trilha de 0,5 mm (`D10`) no Bottom saindo do C6 pino 1 para baixo até 0,16 mm do pad do pino 2 → "Corrija antes de fresar" e "Folga de 0,16 mm entre C6 pino 1 (rede sem nome $2N8) e C6 pino 2 (GND)". O clique põe o marcador no centro da prévia.
   - V6 + traço no Bottom do C6 pino 1 ao pino 2 → um curto entre $2N8 e GND, ponto entre os dois pads.
   - V6 sem a linha `G01X780554Y2984500D01*` no `Gerber_TopLayer.GTL` (jumper de um trecho que leva EQ_IN ao U2 pino 5) → EQ_IN aberta em 2 partes.
   - V6 sem a linha `G01X4305300Y7683500D01*` (trecho do meio de uma cadeia de 3) → EQ_FB aberta em 2 partes.
   - V6 sem o `FlyingProbeTesting.json` → mesmo veredito, furos, jumpers e avisos, sem nomes de pino, com a nota de netlist ausente.
5. Antes de publicar, teste no próprio CopperCAM: a placa tem que abrir no tamanho certo e sem o aviso de furo maior que o pad.

## Ideias que ainda não foram feitas

- Suporte a macros de abertura (`%AM`), como pads de retângulo arredondado (hoje gera erro pedindo para trocar o pad).
- Checagem de borda para contorno não retangular (hoje usa o retângulo envolvente).
- Furos de alinhamento automáticos para dupla face.
