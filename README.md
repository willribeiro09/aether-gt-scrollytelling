# AETHER GT — Scrollytelling Engine v1

Experiência cinematográfica conduzida pelo scroll: o vídeo é renderizado
frame a frame em um `<canvas>`, perfeitamente sincronizado com a posição do
scroll, com camadas independentes de tipografia, anotações e iluminação por
cima. Mobile-first, suave em iOS Safari (a parte difícil).

## Como rodar

```bash
npx serve .
# abra http://localhost:3000
```

**Importante:** abra sempre via servidor HTTP. Se os frames não carregarem
(canvas preto, só os textos aparecendo), a tela de loading agora exibe um
aviso com a instrução — a causa típica é abrir o `index.html` com file:// ou
por um visualizador que não serve a pasta `frames/`.

## Estrutura

```
├── index.html                  # engine + conteúdo (camadas separadas por seção)
├── frames/                     # 240 frames WebP extraídos do vídeo (720×1280)
├── Futuristic_electric_car_reveal_202607021905.mp4   # vídeo-fonte
└── .claude/launch.json         # config do servidor de preview
```

Dentro de `index.html` as responsabilidades são separadas em blocos marcados:

| Camada | Onde | O que é |
|---|---|---|
| **Tema** | `:root` (CSS vars) | Paleta extraída do vídeo (grafite, prata, ciano raio-X, laranja de energia) |
| **Storytelling** | `.beat` em `#type-layer` | Tipografia dos capítulos; `data-pos` = posição vertical em % |
| **Anotações** | `.anno` em `#overlay` | Callouts ancorados ao retângulo real do vídeo; classe `flip` espelha o cartão |
| **Coreografia** | seção `CHOREOGRAPHY` no JS | Chamadas `beat()` / `annotation()` com progresso 0–1 mapeado aos momentos reais do vídeo |
| **Site chrome** | `#site-header`, `#rail`, `#hud` | Header fixo com wordmark + CTA, trilho de capítulos (borda direita), HUD com número/nome do capítulo, CTAs finais |
| **Engine** | seção `FRAME SYNC ENGINE` | Loader, preload preditivo, cache, draw em RAF — **não alterar** |

## Como criar um novo projeto com outro vídeo

1. Extraia os frames (WebP individuais — o `-f image2 -vcodec libwebp` é obrigatório):
   ```bash
   ffmpeg -i VIDEO.mp4 -vf "scale=W:H" -f image2 -vcodec libwebp \
     -quality 80 -compression_level 4 "frames/frame_%04d.webp" -y
   ```
2. Em `index.html`, atualize `TOTAL_FRAMES`, `FRAME_W`, `FRAME_H`.
3. Assista aos frames e monte o mapa narrativo (progresso → momento → texto).
4. Troque os textos dos `.beat` e as posições/labels dos `.anno`.
5. Ajuste os valores `tIn`/`tOut` na seção de coreografia.
6. Opcional: retina a paleta nas CSS vars.

Nada na engine precisa ser tocado.

## Narrativa deste projeto (240 frames · 10 s · 24 fps)

| Progresso | Capítulo | Momento do vídeo |
|---|---|---|
| 0.00–0.08 | **Da escuridão.** | Silhueta no escuro |
| 0.08–0.20 | **Luz.** | Faróis acendem (bloom + ambiente da página clareia) |
| 0.26–0.40 | **Forma.** | Câmera orbita a carroceria |
| 0.45–0.56 | **Raio-X.** | Transição para wireframe ciano (página ganha tinta ciana) |
| 0.58–0.80 | Anotações técnicas | Bateria · 118 kWh → Motor duplo · 750 cv → Regeneração ativa |
| 0.82–0.89 | **Íntegro.** | Carro solidifica em perfil |
| 0.90–1.00 | **Conduza o futuro.** | Hero frontal final (bloom + brilho máximo) |

## Camada de intenção — Story Mode × Explore Mode

A engine não responde só a pixels de scroll: ela classifica **cada gesto** no
touch (velocidade de soltura, deslocamento, trocas de direção, oscilações) e
alterna automaticamente entre dois comportamentos:

- **Story Mode** — flick decidido (monotônico, ≥24 px, soltura ≥0,65 px/ms):
  glide cinematográfico até a **próxima âncora narrativa** (array `ANCHORS`,
  os pontos onde cada beat/anotação está plenamente visível). Duração
  proporcional à distância, easing de desaceleração. Tocar na tela durante o
  glide congela e devolve o controle imediatamente.
- **Explore Mode** — dedo lento, movimento curto ou vai-e-volta: nenhum salto.
  A inércia e o acompanhamento são **adaptativos durante o próprio gesto**
  (dedo lento → `touchInertiaMultiplier` 5 e `syncTouchLerp` 0.16 = parada
  seca e resposta direta, precisão de frame; dedo rápido → 22 / 0.07 = fluido).

No desktop, ↓/↑, PageDown/Up e espaço conduzem a apresentação pelas mesmas
âncoras. Ao trocar o vídeo, atualize `ANCHORS` junto com a coreografia.

## Performance — decisões não negociáveis

- **`syncTouch: true` no Lenis** é O fix do iOS: traz o scroll de momentum para a
  main thread, mantendo canvas e scroll em lockstep. Sem isso, o iOS engasga
  apenas durante a inércia.
- **`position: fixed` no palco em touch** (não `sticky`) — evita jank de repaint
  do sticky no iOS durante o momentum.
- **`createImageBitmap`** decodifica os frames fora da main thread; todos os 240
  frames são pré-carregados em varredura para nunca faltar frame.
- **Draw em RAF**: no máximo um `drawImage` por animation frame no touch.
- **NUNCA usar `touchMultiplier`** no Lenis — crasha o Safari. Para o swipe
  render mais, reduza `PX_PER_FRAME` (encurta a pista de scroll).
- **Orçamento de memória iOS — janela deslizante**: frames decodificados vivem
  na RAM/GPU. O set completo em resolução total (240 × 720×1280 × 4 B ≈ 885 MB)
  estoura o limite por aba do Safari e o iOS mata a página ("um problema
  ocorreu repetidamente"). Cortar frames (step 2) resolvia a memória mas
  derrubava a cadência da fonte para 12 fps — sensação de FPS baixo. A solução
  atual no touch: **todos os frames em resolução nativa** (24 fps de fonte,
  qualidade real do vídeo), com apenas uma **janela deslizante de ~70 frames
  decodificada por vez** (~260 MB — folga tripla vs. o limite): fora dela os
  `ImageBitmap` são fechados e redecodificados sob demanda — os WebP ficam no
  cache HTTP (aquecido em levas no load) e o decode do iOS é por hardware.
- **Crossfade adaptativo por velocidade**: o scroll mapeia para um slot
  fracionário; em movimento lento o canvas mescla os dois frames vizinhos
  (mata o degrau no fim da inércia), em movimento rápido mostra o frame mais
  próximo, nítido (mesclar em velocidade cria ghosting que parece FPS baixo).
  Transição por smoothstep entre `V_LO = 0.15` e `V_HI = 0.45` slots/frame,
  com média móvel para não tremular. Debug: `window.__paint` (v, w, f) e
  `window.__cacheSize()`.
- **Focus settle + curva-S**: a mescla só existe **em movimento**. Quando o
  scroll repousa (>140 ms), o resíduo fracionário é dissolvido até o frame
  inteiro mais próximo (~200 ms) — parado, a imagem é sempre 100% nítida,
  sem "vulto". Durante o movimento lento, uma curva-S na fração atravessa
  rápido a zona turva do 50/50 e demora nos frames puros.

### Knobs de sensação

| Knob | Valor atual | Efeito |
|---|---|---|
| `PX_PER_FRAME` (touch) | 9 | Menor = cada swipe percorre mais vídeo |
| `touchInertiaMultiplier` | 8 lento / 30 rápido (adaptativo) | Maior = mais deslize após soltar |
| `syncTouchLerp` | 0.16 lento / 0.06 rápido (adaptativo) | Menor = acompanhamento mais fluido |
