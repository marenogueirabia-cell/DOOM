# DOOM: Invasão Retro

Motor de jogo de raycasting 2.5D estilo Wolfenstein 3D / DOOM (1993), implementado num **único ficheiro HTML autocontido** (`DOOM.html`), sem dependências de assets externos. Texturas, sprites e som são gerados proceduralmente em tempo de execução.

## Como jogar

Abra `DOOM.html` num navegador moderno. Não é necessário servidor — funciona diretamente via `file://`.

### Controlos

- **Mover:** `W` `A` `S` `D` ou Setas
- **Rodar câmara:** Rato (com pointer lock) ou `Q` / `E`
- **Atacar:** `Espaço` ou clique esquerdo
- **Interagir (portas):** `F` ou `E`
- **Trocar de arma:** `1` (Pistola) / `2` (Espingarda)
- **Pausa:** `P` ou `ESC`

Em ecrãs táteis, são apresentados joystick virtual e botões de ação.

### Objetivo

Sobrevive à invasão demoníaca, encontra o **Cartão de Acesso Vermelho** e alcança o portal de saída para escapar da base lunar.

## Arquitetura

O motor foi refatorado de um monólito procedural para módulos coesos, mantendo a distribuição em ficheiro único:

- **EventBus** — comunicação desacoplada (pub/sub) entre módulos.
- **InputManager** — isola teclado/rato/joystick e traduz em comandos abstratos (`Input.isTriggered('move_forward')`).
- **RenderPipeline** — raycasting DDA, escrita direta no buffer de píxeis (`Uint32Array`), z-buffer e ordenação de sprites.
- **AudioEngine** — síntese procedural FM/subtrativa com agendador look-ahead (Web Audio API).
- **World** — encapsula o estado da sessão (jogador, mapa, entidades) e a simulação; emite eventos de jogo.
- **SpatialGrid** — partição espacial que limita o processamento de IA aos inimigos próximos do jogador.
- **GameStateMachine** — fluxo formal Menu / Gameplay / Paused / Death / Victory.
- **MapLoader / MapSerializer** — carregamento e serialização de níveis em JSON (`Mapa_Interno`).

## Detalhes técnicos

- Resolução lógica retro de 320x200, escalada por CSS com `image-rendering: pixelated`.
- Deteção de endianness para escrita correta de cores RGBA no buffer de 32 bits.
- Física de colisão com deslizamento (resolução independente dos eixos X/Y).
- Game loop baseado em delta-time, com proteção contra picos de lag.

## Licença

Ver [LICENSE](LICENSE).
