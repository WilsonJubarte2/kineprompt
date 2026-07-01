# Teleprompter Inteligente — Resumo do Projeto

Documento de referência de tudo que foi construído, decidido e corrigido na
conversa sobre `index.html`. HTML + CSS + JS puro (Vanilla), sem dependências,
usando a Web Speech API (`webkitSpeechRecognition`).

---

## 1. Estrutura de dados

Roteiro é um array de objetos no topo do `<script>`:

```js
const roteiro = [
  { id: 1, text: "Olá a todos, sejam muito bem-vindos ao nosso canal." },
  ...
];
```

Cada frase é curta (uma sentença), não um bloco de texto único.

---

## 2. Interface (UI/UX)

- Dark mode: fundo escuro (`#0d0d0f`), texto claro.
- Frase ativa: branca, grande (42px), centralizada, escala 1.0.
- Frases vizinhas: opacidade reduzida (próxima 0.55, já lidas 0.25/cinza).
- Auto-scroll suave via `scrollIntoView({ behavior: 'smooth', block: 'center' })`.
- Botão iniciar/parar leitura + ponto vermelho pulsante indicando microfone
  ativo (`#mic-dot.listening`).
- Cada palavra é um `<span class="word">` individual (não a frase inteira),
  permitindo pintura palavra a palavra.
- Rótulo `.heard-label` acima de cada palavra, mostrando em tempo real o que
  o microfone está ouvindo naquela posição.
- Overlay de erro em tela cheia: emoji ⚠️ grande, fundo escurecido, contagem
  regressiva 3→2→1 antes de voltar a ouvir.

---

## 3. Lógica de reconhecimento e avanço

### Motor de matching (`matchSequence`)
Compara sequencialmente as palavras faladas com as palavras-alvo da frase
ativa, palavra por palavra (não é mais "% de acerto geral" — é posição por
posição, como fuzzy matching por palavra):

- `wordsSimilar(a, b)`: tolerância por **distância absoluta** de edição
  (Levenshtein), não percentual — evita que palavras parecidas mas diferentes
  passem como certas:
  - ≤5 letras → precisa bater exato.
  - 6-9 letras → tolera 1 letra de diferença.
  - ≥10 letras → tolera 2 letras de diferença.
- **Palavras de ligação** (`FILLER_WORDS`: o, a, de, do, em, que, e, etc.) são
  puladas automaticamente se a API "engolir" elas (comum em fala rápida) —
  não contam como erro nem travam o avanço.
- Retorna `{ t, s, mismatched }`: posição final no alvo, quantas palavras
  faladas foram consumidas, e se houve erro real.

### Dois canais de verificação (por quê)
- **Resultado final da API** (`committedSpokenWords`) → única fonte usada
  para **decidir erro real**. Resultado interino (`interim`) é instável
  (capta ruído/fala incompleta), então nunca reprova sozinho.
- **Resultado interino** → usado só para **pintar verde rápido e avançar de
  frase mais cedo** (não espera a pausa que a API precisa pra "fechar" um
  resultado final). Isso resolve o pedido de reconhecimento mais rápido sem
  sacrificar a robustez contra falso-erro.

### Feedback instantâneo
- Verde: pinta assim que a posição avança (`paintProgress`), a cada evento.
- Ao completar a frase (interim já cobre tudo): segura o verde ~250ms
  (`advancingSentence`) antes de trocar de frase — dá tempo do navegador
  desenhar o frame antes do DOM ser reconstruído.
- Ao errar: sacode a palavra errada (`wrongshake`, mais amplitude que um
  tremor simples) + dispara o overlay de alerta com contagem 3,2,1.

### Timeout de silêncio
- Se a pessoa demorar mais de 1s (`SILENCE_TIMEOUT_MS`) entre uma palavra
  certa e a próxima, conta como erro e reinicia o ciclo da frase.
- Detectado por um `setInterval` independente (200ms) — necessário porque
  silêncio puro não dispara nenhum evento da Web Speech API.
- Só ativa depois da 1ª palavra confirmada (não pune o tempo de preparo antes
  de começar a ler).

### Palavra de segurança
- "repetir" ou "errei" ditos soltos → limpa a tentativa e reinicia a leitura
  da frase do zero, sem overlay de erro (é ação intencional do usuário, não
  uma falha).

### Beeps (Web Audio API, sem arquivo externo)
- `errorBeep()`: tom grave (220Hz), toca ao detectar erro.
- `tickBeep()`: tom agudo curto (880Hz), toca a cada segundo da contagem
  regressiva (3,2,1).
- `AudioContext` é "desbloqueado" no clique do botão iniciar (gesto do
  usuário), evitando bloqueio de autoplay do navegador.

---

## 4. Bugs encontrados e corrigidos (ordem cronológica)

1. **Avanço não acontecia até 100% da frase / não pulava erro** — versão
   inicial usava score de "% de palavras-chave batendo" solto, sem posição.
   Trocado por matching sequencial por posição.

2. **Erro validava tarde demais** — só considerava resultado *final* da API,
   que demora (espera pausa na fala). Depois de testar o extremo oposto
   (interim puro), settled em: **avanço rápido via interim, erro só via
   final** — o melhor dos dois mundos.

3. **Acerto "pulava" sem mostrar feedback** — `goToNextSentence()` rodava no
   mesmo tick que a pintura verde, então o DOM era reconstruído antes do
   navegador desenhar o frame. Corrigido com delay de 250ms
   (`advancingSentence`) antes de trocar de frase.

4. **Erros "do nada" / falsos positivos** — quando a checagem de erro passou
   a usar resultado interino (para ser mais rápida), ruído e fala incompleta
   disparavam reset sem motivo. Corrigido: erro só considera resultado
   *final* (estável); interim vira exclusividade do caminho de acerto.

5. **Palavras parecidas passando como certas** (ex: "diretamente"/
   "diaramente", "conteúdo"/"contexto") — tolerância de 30% de distância de
   edição era frouxa demais para palavras médias/longas. Trocado por
   tolerância em faixas de tamanho com distância absoluta (ver seção 3).

6. **Palavras de ligação travando a frase** (ex: "até **o** próximo vídeo" —
   se a API não captasse o "o", a frase nunca avançava) — criado
   `FILLER_WORDS` + lógica de pular conectivo no `matchSequence`.

7. **Perda de sincronização / "falso aprovado" sem passar pelo verde** —
   durante os 250-500ms de flash (verde ou vermelho), o código ignorava
   *todos* os eventos, inclusive resultados finais da API — que uma vez
   perdidos, nunca voltam (o índice do resultado não é reenviado). Corrigido:
   acumular palavras finais **sempre**, só a reação visual (pintar/avançar/
   errar) fica pausada durante o flash.

8. **"A tecnologia" sendo lido como se "A" fosse a palavra "TECNOLOGIA"** —
   dois bugs relacionados no `matchSequence`:
   - a lógica de "pular conectivo no final da frase" disparava com o array
     de palavras faladas ainda **vazio** (0 >= 0 é verdadeiro por vacuidade),
     pulando a primeira palavra antes mesmo de qualquer fala. Corrigido:
     só pula se `spokenWords.length > 0`.
   - o rótulo "palavra ouvida" aparecia uma posição à frente quando o
     interim batia certinho (mostrava a última palavra ouvida acima da
     *próxima* palavra-alvo, em vez de acima da que realmente foi dita).
     Corrigido: mostrar em `m - 1`, não em `m`, no caso de match completo.

9. **Vazamento de contexto entre frases** — mesmo com o fix #7, um resultado
   da API ainda **não finalizado** no momento da troca/reset continua sendo
   relatado nos eventos seguintes e pode contaminar a comparação da frase
   nova (ou do reinício da mesma frase). Corrigido com `ignoreResultIndex`:
   ao trocar de frase, errar, resetar por timeout de silêncio ou por palavra
   de segurança, o índice do resultado "em aberto" naquele momento é marcado
   como contaminado (`taintCurrentResult()`) e ignorado em eventos futuros,
   até a sessão de reconhecimento reiniciar do zero (o que também reseta o
   marcador).

---

## 5. Estado atual (variáveis-chave)

| Variável | Papel |
|---|---|
| `targetWords` | palavras normalizadas da frase ativa |
| `committedSpokenWords` | palavras já finalizadas pela API nesta tentativa |
| `currentMatchCount` | posição confirmada mais recente (pra pintura e timeout de silêncio) |
| `lastProgressTime` | timestamp da última palavra certa, usado no timeout de 1s |
| `advanceGraceUntil` | ignora resultado final "atrasado" logo após avançar de frase |
| `ignoreResultIndex` / `latestResultsLength` | marca resultado da API contaminado por uma troca/reset anterior |
| `resettingSentence` / `advancingSentence` | flags que seguram a reação visual durante o flash de erro/acerto |

---

## 6. Possíveis próximos passos (não implementados)

- Testar em ambiente real (navegador, microfone) — toda a validação até
  agora foi feita simulando a lógica de matching em Node, não testando o
  reconhecimento de voz de verdade.
- Ajustar `SILENCE_TIMEOUT_MS`, tolerâncias de `wordsSimilar` e lista de
  `FILLER_WORDS` conforme uso real mostrar necessidade.
- Considerar persistir o roteiro fora do código-fonte (ex: JSON externo,
  ou campo de texto editável na UI) caso o usuário queira trocar de roteiro
  sem editar o arquivo.
