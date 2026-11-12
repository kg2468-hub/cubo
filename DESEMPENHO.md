# Sistema de Desempenho e Constância de FPS — Cubo

> Registro do trabalho realizado em `index.html` para eliminar a oscilação de FPS
> (picos altos seguidos de quedas) e substituí-la por controle organizado e previsível.
> Data: 12/11/2026. Todo o sistema vive no próprio `index.html` (jogo de arquivo único).

---

## 1. O problema que existia

O jogo tinha um bloco antigo de "resolução dinâmica" que:

- reagia a **um único quadro ruim** (FPS medido só a cada 0,5 s, sem estatística);
- só controlava **2 coisas** de forma descoordenada: escala de resolução (`rs`) e o céu (`SK.q`);
- usava contadores acoplados (`fpsPk`, `rsLow`, `rsHi`, `lowT`) espalhados pelo `frame()`;
- subia de volta muito rápido, causando o clássico **ping-pong** (sobe → cai → sobe → cai);
- alocava arrays e matrizes novas a cada quadro renderizado (`new Float32Array`,
  `[...].map(...)`, centros de câmera temporários), alimentando o coletor de lixo.

Resultado percebido: FPS alto no início, depois quedas, com a qualidade "lutando"
contra o jogador em vez de se estabilizar.

---

## 2. O que foi criado: o Governador (PERF)

Um módulo central `PERF` + escada de qualidade `LADDER`, declarados **antes** da
seção "Teclas configuráveis" (necessário porque `updDist()`/`saveSettings()` já
rodam durante a inicialização). Otimizado sob o comentário
`/* ---------- Governador de desempenho (PERF): constancia de FPS ---------- */`.

Funciona em **4 fases**:

### Fase 1 — Medição (nunca decide por 1 quadro)
- Janela rolante de **160 quadros** (~2,5 s em 60 Hz), sem alocação (`Float32Array` fixa).
- A cada avaliação calcula: **média**, **p95** (equivalente a "1% low") e **pior quadro**.
- Decisões apenas a cada **0,25 s** e somente com janela mínima de **32 amostras**.

### Fase 2 — Escada (LADDER): 9 patamares unificados
Baixar vários ajustes **juntos** evita "desligar uma coisa por vez" e alongar o sofrimento.

| Patamar | Resolução (`rs`) | Céu (`SK.q`) | Peixes (`mobScale`) | Chuva/Neve (`rainScale`) | Efeitos folhas/raios (`fxScale`) |
|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 (máx) | 100% | ligado | 100% | 100% | 100% |
| 2 | 90% | ligado | 100% | 100% | 100% |
| 3 | 85% | ligado | 80% | 100% | 100% |
| 4 | 80% | ligado | 60% | 85% | 85% |
| 5 | 75% | desligado | 60% | 85% | 85% |
| 6 | 70% | desligado | 45% | 70% | 70% |
| 7 | 60% | desligado | 35% | 55% | 55% |
| 8 | 50% | desligado | 25% | 40% | 35% |
| 9 (piso) | 42% | desligado | 15% | 30% | 20% |

### Fase 3 — Histerese (fim do ping-pong)
- **Degradar** (`step(+1)`): p95 acima de **112% do orçamento por 0,75 s seguidos**,
  ou de uma vez só se passar de **220%** (pico brutal).
- **Subir** (`step(-1)`): exige folga comprovada — p95 < 102%, média < 97% e
  pior quadro < 160% do orçamento — mantida durante um tempo de permanência `dwellUp`:
  - começa em **5 s**;
  - **dobra** (até máx. 24 s) cada vez que uma sondagem de subida falha em menos de 8 s;
  - **volta a 5 s** após 12 s de estabilidade na nova patamar.
- **Graça pós-mudança**: 1,6 s após degradar, 2,2 s após subir, 0,8 s ao entrar em
  jogo/panorama — o `resize()` do canvas gera picos falsos que não podem contaminar a janela.
- Troca de estado (pausar/retomar/entrar em mundo) **descarta a janela** e reabre a graça.

### Fase 4 — Aplicação e emergência
- Orçamentos de trabalho no fio principal por quadro:
  - Malha (`jobMesh`): **6 ms** normal → **2 ms** sob carga (p95 > 112%) → **0 ms** em freeze;
  - Luz (`jobLight`): **5 ms** normal → **2 ms** → **0 ms**.
- Streaming de chunks: intervalo no jogo passa de 0,25 s para **0,5 s** nos patamares 7–9.
- **Freeze de emergência**: já no piso (patamar 9) e ainda 220% acima do orçamento →
  congela malha/luz por **0,6 s** para o sistema respirar e se recupera sozinho.

---

## 3. Limitador de quadros (novo recurso nas Configurações)

Botão **"Limitador de quadros (constância de FPS)"** no cartão Vídeo (`#tCap`),
ciclado em: **Automático → Ilimitado → 60 → 45 → 30** (salvo no `localStorage`).

- `Automático`: sem teto forçado; o governador cuida da qualidade (orçamento de 60 fps).
- `Ilimitado`: sem ritmo e sem teto.
- `60/45/30`: pula ciclos do `requestAnimationFrame` com acumulador — **sem rajadas**
  após travadas (o crédito acumulado nunca ultrapassa um intervalo).
- O orçamento do governador se adapta ao teto escolhido (`intervalo × 1,8`), para o
  ritmo do rAF em telas de alta taxa (ex.: 144 Hz) **não** ser confundido com lentidão
  e degradar qualidade à toa.

---

## 4. Mudanças de engenharia que sustentam a constância

1. **Pausa e inventário não renderizam mais.** Cena estática = menos calor = clocks
   estáveis. Continuam funcionando: decaimento do flash/rewind, miniatura e salvamento,
   e a detecção de troca de estado do governador.
2. **Zero alocação por quadro no render.** `persp/lookAt/mul` escrevem em matrizes
   reutilizadas (out-params); `eyePos/lookDir/aimDir` e a cor do céu usam vetores
   compartilhados (`_eye`, `_dir`, `_aim`, `_e2`, `_ctr`, `_sky`, `UPV`, `_view`,
   `_proj`, `_mvpM`). A matemática foi verificada **bit a bit idêntica** à original
   (500 casos aleatórios).
3. **Relógio correto com limitador.** A física usa `dtP` = tempo real desde o último
   quadro *processado* (o `dt` puro do rAF ignoraria os ciclos pulados e deixaria o
   jogo em câmera lenta com teto ativo).
4. **Tela de carregamento privilegiada.** Não alimenta estatísticas (picos esperados)
   e, ao entrar nela, os orçamentos voltam a 6/5 ms para montar o mundo em velocidade total.
5. **Conteúdo escalável**: peixes (`mobsMaintain`), chuva/neve (`drawRain`),
   folhas (`updateLeaves`) e raios de sol (`updateBeams`) leem as escalas do patamar.

---

## 5. Como o sistema se comporta agora (cenários)

| Situação | Comportamento |
|---|---|
| Mundo carregado, máquina dando conta | Fica no patamar 1; janela cheia, sem mudanças (silêncio = sucesso) |
| Queda repentina (ex.: olhar para área nova) | Um pico isolado **não** faz nada; sustentado por 0,75 s → desce 1 patamar + graça de 1,6 s |
| Queda brutal (> 2,2× orçamento) | Desce um patamar **imediatamente** |
| Máquina fraca contínua | Desce até o piso em degraus (~1 s por degrau, espaçados pelas graças) |
| Até o piso não basta | **Freeze** de 0,6 s zera malha/luz; repete se continuar acima de 2,2× |
| Situação melhora | Só sobe após 5 s de folga comprovada (ou mais, se `dwellUp` cresceu) + graça de 2,2 s |
| Tentou subir e falhou rápido | `dwellUp` dobra → desiste de tentar de novo tão cedo (máx. 24 s) |
| Estável por 12 s no novo patamar | `dwellUp` volta a 5 s |
| Pausa / inventário | Cena congela; nada é medido; ao voltar, janela nova + graça |
| Entrar em mundo / trocar de mundo | Janela descartada; orçamentos de carregamento em velocidade total |
| Teto manual (ex.: 60 em monitor 144 Hz) | Limitador pula ciclos; governador tolera o padrão esticado do rAF e não degrada por causa dele |
| Menu (panorama) | Mede e se antecipa: ao entrar no jogo, o patamar já está ajustado |

---

## 6. Onde cada coisa está em `index.html`

| Parte | Localização (por seção de comentário) |
|---|---|
| Botão `#tCap` | HTML, cartão **Vídeo** de `#s-settings`, logo abaixo do `#tDebug` |
| `LADDER` + `PERF` (com cabeçalho explicando as 4 fases) | Antes de `/* Teclas configuráveis (computador) */` |
| `rs`, `resize()`, `PERF.apply()` inicial | Seção **Loop principal** |
| Orçamentos de malha/luz + `streamT` | `update()` |
| `PERF.rainScale` | `drawRain()` |
| `PERF.fxScale` | `updateLeaves()` / `updateBeams()` |
| `PERF.mobScale` | `mobsMaintain()` |
| Vetores/matrices reutilizáveis | Seção **Matematica** |
| Portão de estado vivo + `pace` + `dtP` + HUD | `frame()` |
| Ciclo do botão, salvamento (`cap`), sincronia de UI | Seção de **sliders/interruptores** (`CAP_CYCLE`) |
| Estatísticas no chat | `execCommand`, caso **`/status`** |

---

## 7. Como conferir na prática

1. Abra `index.html` e ative o HUD de depuração (Configurações → "FPS e XYZ").
   O HUD agora mostra, por exemplo:
   ```
   FPS 60   1% 58   quadro 16.9/21.3 ms
   qualidade 2/9   res 90%   peixes alvo 1200   teto Automático
   chunks 98   visao 6   fila malha 3   fila luz 1
   XYZ ...
   ```
2. Comando `/status` no chat (T): traz as mesmas estatísticas + filas e alvo de peixes.
3. Clique em "Limitador de quadros" para ciclar o teto; o rótulo e o `localStorage` atualizam.
4. Para ver o governador agir de propósito: `/visao 12` (distância máxima) e
   `/peixe 4000` — o patamar deve descer em degraus no HUD até estabilizar, e subir
   de volta devagar quando a carga aliviar.
5. Verificações já executadas em Node: sintaxe OK, equivalência numérica das matrizes
   (500 casos) e 6 cenários do governador (degradação até o piso, freeze de emergência,
   recuperação completa, anti-ping-pong, teto manual, reset por troca de estado).

---

## 8. Limitações conhecidas (de projeto, não defeitos)

- Com teto manual em telas de alta taxa, o navegador só entrega FPS em múltiplos do
  refresh (ex.: 144 Hz → ~48–72 com jitter); é propriedade do rAF/sincronia vertical.
  O orçamento folgado (×1,8) existe justamente para isso não derrubar a qualidade.
- No piso absoluto com hardware insuficiente, o freeze alivia por pulsos de 0,6 s, mas
  nenhum ajuste de software garante 60 fps onde o hardware não chega — o sistema apenas
  garante a *constância* do melhor patamar possível.
- O panorama (menus) amostra e ajusta o patamar; spikes curtos de troca de mirante
  podem provocar no máximo um degrau, que se recupera sozinho no jogo.
