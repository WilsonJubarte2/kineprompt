# 🎬 KinePrompt

Teleprompter inteligente controlado por **voz**, com **gravação de vídeo por trecho** e **alinhamento de pose (onion skin)** para cortes contínuos. HTML + CSS + JavaScript puro, **um único arquivo**, sem dependências.

👉 **App:** abra o `index.html` (ou a página do GitHub Pages).

## O que faz

- **Avança por voz:** o texto destaca a frase atual e só avança quando você a lê corretamente (Web Speech API + fuzzy matching enviesado pelo roteiro via `maxAlternatives`).
- **Confirmação manual:** ao concluir uma frase, diga **PRONTO** (ou aperte **Espaço**) — com contagem 3-2-1 antes de seguir.
- **Comandos de voz na espera:** **REPLAY** (rever o trecho), **REGRAVAR** (refazer), **PRONTO** (continuar).
- **Correção:** errou ou pausou demais → sinaliza, descarta o trecho e espera você refazer.
- **Pose fantasma (onion skin):** captura um **contorno** (filtro Sobel) da sua pose no fim de cada frase e sobrepõe na próxima/refação, pra o corte emendar.
- **Gravação em vídeo por frase:** cada frase vira um segmento de vídeo (só os aprovados entram na track).
- **Carrossel de trechos:** rever, **refazer** ou **baixar** cada trecho; e **baixar tudo num vídeo só**.
- **Fonte de câmera:** escolha webcam ou celular (ver abaixo).
- **Atalhos:** `Espaço` iniciar/continuar · `←/→` navegar · `R` reiniciar frase · `Esc` cancelar o take atual.

## Usar o celular como câmera

O seletor de câmera lista qualquer dispositivo de vídeo do sistema. Pra usar o celular com lente boa, exponha ele como webcam via um app (ex.: **Camo**, **Iriun**, **DroidCam**, **EpocCam**, ou **Continuity Camera** no Mac). Ele aparece na lista e é só selecionar.

## Requisitos

- Navegador **Chrome** ou **Edge** (Web Speech API `webkitSpeechRecognition`).
- **HTTPS** ou `localhost` (necessário para câmera/microfone). GitHub Pages serve HTTPS ✅.
- Permitir **câmera + microfone**.

## Rodar local

Abra `index.html` no Chrome/Edge. Se a câmera não liberar via `file://`, sirva local:

```bash
python -m http.server 8000
# abre http://localhost:8000
```

## Deploy (GitHub Pages)

`index.html` está na raiz, então o Pages serve direto. Veja os passos no fim deste README ou nas instruções do repositório.

---

Feito com HTML/CSS/JS vanilla. Sem build, sem dependências.
