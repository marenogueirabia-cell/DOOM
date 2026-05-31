# Requirements Document

## Introduction

O projeto atual ("DOOM: Invasão Retro") é um motor de jogo de raycasting 2.5D estilo Wolfenstein 3D / DOOM, implementado como um único ficheiro HTML autocontido (`DOOM.html`, ~2000 linhas), sem dependências de assets externos. Todo o código é procedural e monolítico: as classes `AudioEngine` e `RenderPipeline` coexistem com estado global (`player`, `gameMap`, `sprites`, `keys`), ouvintes de eventos registados diretamente no `window`/`document`, e o fluxo de jogo controlado de forma imperativa através de manipulação de `classList` CSS (`winGame`, `diePlayer`, `resetGame`, `stopGame`), sem uma Máquina de Estados Finitos.

Esta funcionalidade descreve a evolução desse monólito procedural para uma arquitetura modular profissional. O objetivo é separar responsabilidades em módulos coesos (orquestrador principal, gestão de input, renderização, áudio e um Mundo baseado em ECS), substituir a herança rígida de entidades por composição de componentes, introduzir um modelo de eventos com partição espacial para a IA, formalizar o fluxo de jogo através de uma Máquina de Estados Finitos e permitir o carregamento de mapas dinâmicos via JSON.

A restrição central e não negociável é a **preservação do comportamento e das características de desempenho existentes**: não pode haver regressões na velocidade de renderização, no tratamento de latência de áudio nem na física de colisão com deslizamento. A capacidade de distribuição como ficheiro único e estático é valorizada e deve ser mantida.

## Glossary

- **Motor**: O sistema completo de jogo após a refatoração, composto por módulos desacoplados.
- **Modulo_Principal**: O módulo orquestrador (referido pelo utilizador como `index.js`) responsável por instanciar e coordenar os restantes módulos e por hospedar o ciclo principal de jogo (game loop).
- **InputManager**: O módulo que isola os ouvintes de teclado, rato e joystick móvel e os traduz numa camada única de comandos abstratos (por exemplo, `move_forward`, `fire`, `interact`).
- **RenderEngine**: O módulo de renderização, evolução da atual classe `RenderPipeline`, responsável pelo raycasting DDA, escrita direta de píxeis (`screenBuffer32`), z-buffer, ordenação e desenho de sprites e desenho da arma.
- **AudioEngine**: O módulo de áudio, evolução da atual classe `AudioEngine`, responsável pela síntese procedural FM/subtrativa e pelo agendamento por look-ahead.
- **World**: O módulo que contém o estado do mundo de jogo (entidades, mapa, jogador) e executa os sistemas do ECS.
- **ECS**: Arquitetura Entity-Component-System leve que substitui a herança das classes `Entity`, `Demon` e `Pickup` por composição de componentes.
- **Component**: Estrutura de dados que descreve um aspeto de uma entidade (por exemplo, posição, saúde, capacidade de disparo).
- **System**: Função que opera sobre entidades que possuem um determinado conjunto de componentes.
- **EventBus**: Mecanismo de publicação/subscrição (pub-sub) usado para comunicação desacoplada entre módulos e sistemas.
- **SpatialGrid**: Estrutura de partição espacial que indexa entidades por sector do mapa para limitar o processamento de IA a sectores adjacentes ao jogador.
- **GameStateMachine**: Máquina de Estados Finitos (FSM) que governa o fluxo de jogo entre os estados Menu, Gameplay, Paused, Death e Victory.
- **MapLoader**: Componente que interpreta (faz parsing de) uma definição de mapa em formato JSON e produz um objeto de mapa interno utilizável pelo Motor.
- **MapSerializer**: Componente que converte um objeto de mapa interno de volta para a representação JSON.
- **Mapa_Interno**: A representação em memória do mapa (atualmente a matriz `gameMap` 16x16 de inteiros 0–9) consumida pelo RenderEngine e pela física.
- **Comando_Abstrato**: Um sinal lógico de input independente de hardware (por exemplo, `Input.isTriggered('move_forward')`).
- **Distribuicao_Ficheiro_Unico**: Artefacto de saída que reúne todos os módulos num único ficheiro HTML estático autocontido, sem assets externos.
- **Baseline**: O comportamento e o desempenho medidos da versão monolítica atual (`DOOM.html`), usados como referência para deteção de regressões.
- **FPS**: Fotogramas por segundo renderizados pelo Motor.
- **Look_Ahead_Scheduler**: O agendador de áudio existente que usa `setInterval` a 25 ms e uma janela de antecipação (`scheduleAheadTime`) de 0,1 s.
- **Fisica_Deslizamento**: A física de colisão atual que resolve os eixos X e Y de forma independente, permitindo ao jogador deslizar ao longo das paredes (raio de colisão de 0,22).

## Requirements

### Requirement 1: Decomposição Modular e Orquestração

**User Story:** Como arquiteto de software, quero que o motor seja decomposto em módulos coesos coordenados por um módulo principal, para que cada responsabilidade fique isolada e o código seja testável e sustentável.

#### Acceptance Criteria

1. THE Motor SHALL organizar a lógica em módulos separados — Modulo_Principal, InputManager, RenderEngine, AudioEngine e World — em que cada módulo está encapsulado na sua própria unidade e nenhum módulo acede ao estado interno de outro.
2. WHEN o Motor arranca, THE Modulo_Principal SHALL instanciar o InputManager, o RenderEngine, o AudioEngine e o World antes de iniciar o ciclo principal de jogo.
3. THE Modulo_Principal SHALL hospedar o ciclo principal de jogo (game loop) baseado em delta-time, preservando o cálculo de delta-time atual (escala relativa a 16,66 ms, limite de 50 ms para saltos e teto de escala de 3,0).
4. THE Motor SHALL eliminar por completo o estado global mutável partilhado, garantindo que o estado anteriormente exposto globalmente (`player`, `gameMap`, `sprites`, `keys`) reside encapsulado em escopo não global dentro do World e não é acessível globalmente.
5. WHEN um módulo necessita comunicar com outro módulo, THE Motor SHALL realizar essa comunicação através de interfaces explícitas ou do EventBus, em vez de variáveis globais.
6. THE Motor SHALL expor cada módulo através de fronteiras de interface explícitas que não dependam de identificadores de elementos DOM fixos dentro da lógica de simulação.
7. IF a instanciação de um módulo falha durante o arranque, THEN THE Modulo_Principal SHALL abortar o arranque, não iniciar o ciclo principal de jogo e indicar o erro identificando o módulo afetado.

### Requirement 2: Arquitetura Entity-Component-System (ECS)

**User Story:** Como arquiteto de software, quero substituir a herança rígida de entidades por composição de componentes, para que novos comportamentos (como dar capacidade de disparo a um novo inimigo) sejam adicionados sem duplicar código de movimento.

#### Acceptance Criteria

1. THE World SHALL representar cada entidade de jogo como uma identidade associada a um conjunto de Components, em substituição das classes `Entity`, `Demon` e `Pickup`.
2. THE ECS SHALL definir Components de dados que cobrem, no mínimo, as capacidades atuais: posição, saúde, estado de IA, tipo de coletável e parâmetros de combate (dano, raio, velocidade).
3. THE ECS SHALL executar a lógica de comportamento através de Systems, processando cada System exatamente as entidades que possuem todos os Components por ele exigidos e ignorando as restantes.
4. WHERE uma nova entidade deve reutilizar movimento existente, THE ECS SHALL permitir compor esse comportamento adicionando os Components correspondentes sem duplicar o código de movimento.
5. THE ECS SHALL reproduzir os comportamentos atuais dos demónios, incluindo os estados `idle`, `alert`, `attacking`, `pain` e morte, de forma equivalente à Baseline.
6. WHEN o jogador recolhe um coletável (munições, saúde, armadura, espingarda ou cartão de acesso), THE ECS SHALL aplicar ao estado do jogador o mesmo efeito da Baseline.
7. IF um System exige um Component que uma entidade não possui, THEN THE ECS SHALL excluir essa entidade do processamento desse System sem interromper o processamento das restantes entidades.
8. THE ECS SHALL suportar pelo menos 12 entidades iniciais, permitindo qualquer composição de tipos de entidade, desde que o posicionamento inicial atual seja preservado.

### Requirement 3: Abstração de Input por Camada de Comandos

**User Story:** Como arquiteto de software, quero isolar os ouvintes de teclado, rato e joystick móvel numa única camada de tradução de comandos, para que a física leia sinais limpos e independentes do hardware.

#### Acceptance Criteria

1. THE InputManager SHALL centralizar o registo de todos os ouvintes de input de teclado, rato e joystick móvel num único módulo.
2. THE InputManager SHALL traduzir eventos de hardware em Comandos_Abstratos consultáveis, incluindo, no mínimo, `move_forward`, `move_back`, `strafe_left`, `strafe_right`, `turn_left`, `turn_right`, `fire`, `interact`, `weapon_pistol` e `weapon_shotgun`.
3. WHEN a lógica de física consulta o estado de um Comando_Abstrato através de `Input.isTriggered(<comando>)`, THE InputManager SHALL devolver um booleano verdadeiro enquanto o comando estiver ativo no fotograma atual e falso caso contrário.
4. THE InputManager SHALL preservar os mapeamentos de input atuais: WASD e Setas para movimento, Q/E para rotação por teclado, rato com pointer lock para rotação (sensibilidade de 0,003 por unidade de movimento horizontal), Espaço ou clique esquerdo para disparo, F ou E para interação, e as teclas 1 e 2 para selecionar pistola e espingarda, respetivamente.
5. THE InputManager SHALL preservar o comportamento atual do joystick virtual móvel, incluindo o limiar de ativação de movimento de 15 píxeis de deslocamento a partir do centro, o raio máximo do manípulo de 45 píxeis e os botões de fogo, abrir e trocar de arma.
6. THE RenderEngine, o AudioEngine e o World SHALL permanecer independentes das APIs de eventos de hardware do navegador, não registando diretamente ouvintes de teclado, rato ou toque, e recebendo apenas Comandos_Abstratos.

### Requirement 4: Separação do Motor de Renderização e Mapas Dinâmicos

**User Story:** Como arquiteto de software, quero que a pipeline de renderização seja um módulo independente capaz de desenhar mapas dinâmicos, para que diferentes níveis possam ser carregados sem alterar o código de renderização.

#### Acceptance Criteria

1. THE RenderEngine SHALL renderizar cada fotograma exclusivamente a partir do estado do jogador, do Mapa_Interno e da lista de entidades recebidos como entradas explícitas, sem aceder a estado global mutável nem a identificadores de elementos DOM fixos.
2. THE RenderEngine SHALL preservar o método de renderização atual à resolução lógica de 320x200: raycasting DDA, manipulação direta de píxeis via `createImageData` e `Uint32Array`, deteção de endianness, z-buffer (`Float32Array`) e ordenação de sprites por distância ao jogador.
3. THE RenderEngine SHALL desenhar qualquer Mapa_Interno definido como uma grelha retangular de L colunas por A linhas, com L e A compreendidos entre 1 e 256 tiles, em vez de assumir uma grelha fixa de 16x16.
4. THE RenderEngine SHALL desenhar os tipos de parede e elementos existentes (tijolo vermelho, painel tecnológico, pedra, porta trancada, porta comum e portal de saída) mapeando cada identificador de tile definido no Mapa_Interno para a textura correspondente.
5. THE RenderEngine SHALL aplicar sempre um limite máximo configurável de sprites renderizáveis por fotograma, com o valor por omissão `MAX_SPRITES` = 64.
6. WHEN o número de entidades visíveis num fotograma excede o limite máximo de sprites renderizáveis, THE RenderEngine SHALL renderizar apenas as entidades mais próximas do jogador até esse limite e omitir as restantes nesse fotograma.
7. THE RenderEngine SHALL preservar, a cada fotograma, a renderização do rosto animado do Doomguy e o desenho da arma do jogador.
8. WHEN o jogador sofre dano ou dispara a arma, THE RenderEngine SHALL apresentar o efeito de flash de ecrã correspondente, preservando o comportamento atual.

### Requirement 5: Carregamento e Serialização de Mapas em JSON

**User Story:** Como criador de níveis, quero definir mapas em JSON e carregá-los no motor, para que novos níveis sejam adicionados sem editar o código-fonte.

#### Acceptance Criteria

1. WHEN um mapa em JSON estruturalmente válido é fornecido — uma grelha retangular com dimensões entre 1 e 256 tiles por lado, contendo apenas identificadores de tile reconhecidos (0 a 9) e uma posição de spawn válida —, THE MapLoader SHALL interpretá-lo e produzir um Mapa_Interno consumível pelo RenderEngine e pela física.
2. IF um mapa em JSON é estruturalmente inválido (não é uma grelha retangular, excede os limites de dimensão, contém identificadores de tile não reconhecidos ou está sintaticamente malformado), THEN THE MapLoader SHALL rejeitar o mapa, não produzir um Mapa_Interno, manter o mapa atualmente carregado inalterado e devolver uma mensagem de erro descritiva que identifica a causa.
3. THE MapLoader SHALL validar que o mapa importado define uma posição de spawn do jogador dentro dos limites da grelha e situada num tile transitável (que não seja parede, porta nem portal).
4. WHEN o MapSerializer recebe um Mapa_Interno, THE MapSerializer SHALL produzir um documento JSON sintaticamente válido que o MapLoader consiga voltar a interpretar.
5. WHEN o MapLoader interpreta o resultado de serializar um Mapa_Interno válido com o MapSerializer, THE Motor SHALL produzir um Mapa_Interno equivalente ao original — com as mesmas dimensões, o mesmo identificador de tile em cada célula e a mesma posição de spawn (propriedade de ida-e-volta).
6. WHEN o jogo arranca sem um mapa importado, THE MapLoader SHALL carregar o mapa incorporado por omissão equivalente ao `gameMap` atual — com dimensões 16x16, o mesmo identificador de tile em cada célula e a mesma posição de spawn.

### Requirement 6: IA por Eventos e Partição Espacial

**User Story:** Como arquiteto de software, quero substituir os ciclos de distância por fotograma da IA por um modelo de eventos com partição espacial, para que o desempenho não se degrade com muitas entidades.

#### Acceptance Criteria

1. WHEN a coordenada inteira (parte inteira de x ou de y) da posição do jogador muda entre dois fotogramas consecutivos, THE World SHALL publicar a nova posição de sector do jogador no EventBus, sendo o sector definido pela célula do mapa correspondente à parte inteira das coordenadas.
2. WHILE a coordenada inteira da posição do jogador permanece inalterada, THE World SHALL evitar a reindexação espacial das entidades.
3. THE SpatialGrid SHALL indexar as entidades pela célula do mapa que ocupam, permitindo a consulta das entidades de um sector e dos sectores adjacentes.
4. WHEN uma atualização de sector do jogador é publicada, THE World SHALL processar a lógica de linha de visão e perseguição apenas para os inimigos situados nos sectores adjacentes ao jogador, devendo o conjunto de sectores adjacentes cobrir integralmente o raio de aquisição de alvo de 8,0 unidades em redor do jogador, recortado aos limites do mapa.
5. THE World SHALL preservar as transições de estado de combate atuais dos demónios: transição de `idle` para `alert` quando o jogador está a menos de 8,0 unidades, e de `alert` para `attacking` quando o jogador está a menos de 0,7 unidades.
6. WHEN ocorre o impacto de um ataque de demónio com o jogador a menos de 0,8 unidades, THE World SHALL aplicar 15 de dano ao jogador.
7. WHEN o jogador sofre dano e possui armadura, THE World SHALL absorver 40% do dano pela armadura (arredondado por defeito) e aplicar o restante à saúde.
8. IF um inimigo está fora dos sectores adjacentes ao jogador, THEN THE World SHALL omitir o cálculo de linha de visão para esse inimigo nesse fotograma, mantendo inalterado o estado de IA desse inimigo.
9. IF nenhum inimigo está presente nos sectores adjacentes ao jogador quando uma atualização de sector é publicada, THEN THE World SHALL omitir por completo o processamento de linha de visão e perseguição nesse fotograma.

### Requirement 7: Máquina de Estados Finitos para o Fluxo de Jogo

**User Story:** Como jogador, quero um fluxo de jogo formal com menu, jogo, pausa, morte e vitória, para que existam pausa real, transições suaves de ecrã e suporte a múltiplos níveis.

#### Acceptance Criteria

1. THE GameStateMachine SHALL definir os estados Menu, Gameplay, Paused, Death e Victory e permitir exclusivamente as transições Menu→Gameplay, Gameplay→Paused, Paused→Gameplay, Gameplay→Victory, Gameplay→Death, Death→Gameplay e Victory→Gameplay, ignorando qualquer pedido de transição não listado e mantendo o estado atual.
2. WHILE a GameStateMachine está no estado Paused, THE Modulo_Principal SHALL suspender as atualizações de simulação e o avanço do tempo de jogo, mantendo o último fotograma renderizado visível.
3. WHEN o jogador retoma a partir do estado Paused, THE Modulo_Principal SHALL continuar a simulação de modo que o primeiro delta-time após a retoma desconsidere o tempo decorrido durante a pausa e não exceda o limite de 50 ms de salto de delta-time.
4. WHEN o jogador alcança o tile de saída, THE GameStateMachine SHALL transitar para o estado Victory, reproduzindo o comportamento atual de vitória.
5. IF a saúde do jogador é igual ou inferior a zero, THEN THE GameStateMachine SHALL transitar para o estado Death, reproduzindo o comportamento atual de derrota.
6. WHEN ocorre uma transição entre estados, THE GameStateMachine SHALL aplicar uma transição visual de ecrã com duração não superior a 1,0 segundo e, ao concluí-la, garantir que a simulação está suspensa sempre que o estado de destino é Menu, Paused, Death ou Victory e que nenhum efeito sonoro do estado de origem permanece a tocar de forma incoerente com o estado de destino.
7. WHEN o estado Gameplay é iniciado ou reiniciado, THE GameStateMachine SHALL permitir o carregamento de um nível através do MapLoader, suportando pelo menos 2 níveis distintos.
8. WHEN o jogador reinicia a partir do estado Death ou Victory, THE GameStateMachine SHALL repor o estado do jogador, o mapa e as entidades para as condições iniciais do nível.
9. WHEN o jogador aciona o comando de pausa enquanto a GameStateMachine está no estado Gameplay, THE GameStateMachine SHALL transitar para o estado Paused.
10. WHEN o jogador inicia o jogo a partir do estado Menu, THE GameStateMachine SHALL transitar para o estado Gameplay.

### Requirement 8: Preservação de Comportamento e Desempenho (Sem Regressões)

**User Story:** Como arquiteto de software, quero que a refatoração preserve o comportamento e o desempenho atuais, para que não existam regressões percetíveis na jogabilidade.

#### Acceptance Criteria

1. THE Motor SHALL preservar a Fisica_Deslizamento atual, resolvendo os eixos X e Y de forma independente com raio de colisão de 0,22 unidades.
2. WHEN executado no mesmo hardware e na mesma resolução lógica (320x200) que a Baseline e percorrendo o mesmo cenário de referência durante uma janela contínua de, no mínimo, 60 segundos, THE RenderEngine SHALL manter uma taxa média de FPS não inferior a 95% da taxa média de FPS da Baseline medida nas mesmas condições.
3. THE AudioEngine SHALL preservar o Look_Ahead_Scheduler atual, mantendo o intervalo de agendamento de 25 ms e a janela de antecipação de 0,1 s.
4. THE AudioEngine SHALL preservar a pré-alocação estática dos buffers de ruído da pistola e da espingarda para evitar interrupções causadas pelo coletor de lixo durante o disparo.
5. THE Motor SHALL preservar a deteção e o tratamento de endianness na escrita do buffer de píxeis para ambas as ordens de bytes (little-endian e big-endian).
6. WHEN o jogador interage com uma porta comum destrancada, THE World SHALL abrir a porta, preservando a mecânica de abertura atual.
7. WHEN decorrerem 5 segundos após a abertura de uma porta e o bloco da porta não estiver ocupado pelo jogador nem por um demónio, THE World SHALL fechar a porta automaticamente.
8. WHILE o jogador ou um demónio ocupam o bloco da porta, THE World SHALL adiar o fecho automático dessa porta até que o bloco fique desocupado.
9. IF o jogador interage com a porta trancada sem o cartão de acesso, THEN THE World SHALL manter a porta fechada, impedir a sua abertura e emitir a mensagem de bloqueio atual.

### Requirement 9: Preservação da Distribuição em Ficheiro Único

**User Story:** Como mantenedor, quero continuar a poder distribuir o jogo como um único ficheiro HTML estático, para que a implantação permaneça simples e sem dependências de assets externos.

#### Acceptance Criteria

1. THE Motor SHALL produzir uma Distribuicao_Ficheiro_Unico composta por exatamente 1 ficheiro HTML estático, reunindo todos os módulos (Modulo_Principal, InputManager, RenderEngine, AudioEngine e World), com 0 assets externos.
2. WHILE a Distribuicao_Ficheiro_Unico está em execução, THE Distribuicao_Ficheiro_Unico SHALL operar a lógica do jogo com 0 pedidos de rede.
3. WHEN a Distribuicao_Ficheiro_Unico arranca, THE Distribuicao_Ficheiro_Unico SHALL gerar as texturas e os sprites proceduralmente em tempo de execução, com 0 ficheiros de imagem externos.
4. THE Distribuicao_Ficheiro_Unico SHALL ser considerada válida desde que não dependa de ficheiros externos, independentemente de a geração procedural ter ou não sucesso em tempo de execução.
5. WHERE um passo de empacotamento (build) é utilizado para combinar os módulos, THE Motor SHALL gerar a Distribuicao_Ficheiro_Unico (exatamente 1 ficheiro HTML) a partir dos módulos separados de forma automatizada e reproduzível, com 0 alterações manuais do resultado.
6. WHEN nenhum mapa externo é fornecido, THE Distribuicao_Ficheiro_Unico SHALL reproduzir o mapa incorporado por omissão.

### Requirement 10: Preservação da Síntese de Áudio Procedural

**User Story:** Como jogador, quero que todos os efeitos sonoros e a música se mantenham idênticos após a refatoração, para que a atmosfera do jogo seja preservada.

#### Acceptance Criteria

1. THE AudioEngine SHALL preservar a síntese procedural FM/subtrativa dos seis efeitos atuais (disparo de pistola, disparo de espingarda, dano, morte de inimigo, recolha de item e porta), reproduzindo, para cada efeito, a forma de onda, o envelope de amplitude e a duração equivalentes à Baseline.
2. THE AudioEngine SHALL preservar o sequenciador da linha de baixo (bassline E1M1) com o tempo de 135 BPM, a resolução de semicolcheia e o ciclo de 32 passos atuais.
3. WHEN o utilizador ativa o som, THE AudioEngine SHALL retomar o `AudioContext`, iniciar a música e refletir o estado ativo no botão de áudio, preservando o comportamento atual.
4. WHEN o utilizador desativa o som, THE AudioEngine SHALL parar a música e refletir o estado inativo no botão de áudio, preservando o comportamento atual.
5. WHEN ocorre a primeira interação do utilizador, THE AudioEngine SHALL criar no máximo um `AudioContext`, preservando o comportamento atual de criação do `AudioContext` apenas em resposta a interação do utilizador.
6. WHEN um nó de áudio termina a reprodução, THE AudioEngine SHALL desligar (`disconnect`) todos os nós criados para esse som (osciladores, filtros e ganhos), preservando o comportamento atual de prevenção de fugas de memória.
7. IF o som está desativado ou o `AudioContext` ainda não foi criado, THEN THE AudioEngine SHALL não produzir qualquer efeito sonoro nem música.
