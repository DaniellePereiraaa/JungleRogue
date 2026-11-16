# JungleRogue — Action Platformer 2D em Unity

**JungleRogue** é um jogo de plataforma 2D de ação desenvolvido em **Unity (C#)**, com mecânicas de combate corpo-a-corpo, sistema de níveis progressivos, loja, chefes, wall jump/wall slide e sistema de vida baseado em corações.

---

## Visão Geral

| Aspecto | Detalhes |
|---------|----------|
| **Engine** | Unity (2D) |
| **Linguagem** | C# (.NET) |
| **Gênero** | Action Platformer / Roguelite elements |
| **Perspectiva** | Side-scroller 2D |
| **Sistema de Build** | Unity Package Manager (Packages/) |

---

## Estrutura do Projeto

```
JungleRogue/
├── Assets/
│   ├── Scripts/                    # Código principal (C#)
│   │   ├── Player.cs               # Player: movimento, pulo, wall slide/jump, vida, transições
│   │   ├── playerMelee.cs          # Combate: soco, chute, bloqueio, ultimate, knockback
│   │   ├── Enemy.cs                # Inimigo básico: perseguição, vida, dano, knockback
│   │   ├── Boss.cs                 # Chefe: movimento lateral, dash aleatório, vida
│   │   ├── Frutas.cs               # Coletáveis: pontuação
│   │   ├── GameController.cs       # Singleton: pontuação global, UI
│   │   ├── UIManager.cs            # UI: corações de vida (6 slots)
│   │   ├── Lojinha.cs              # Loja: itens (Picanga, Chave), voltar ao nível
│   │   ├── entradaLoja.cs          # Trigger: colisão → carrega cena Loja
│   │   ├── Spikes.cs               # Espinhos: dano por colisão
│   │   ├── CameraController.cs     # Camera follow simples
│   │   ├── MainMenu.cs             # Menu principal: Jogar, Config, Sair
│   │   └── Touch_gameOver.cs       # Tela de morte: Reiniciar, Menu
│   ├── Scenes/                     # Cenas do jogo
│   │   ├── Menu.unity              # Menu principal
│   │   ├── Config.unity            # Configurações
│   │   ├── DeathScreen.unity       # Game Over
│   │   ├── Loja.unity              # Loja entre níveis
│   │   └── Levels/                 # 4 níveis progressivos
│   │       ├── Level1.unity
│   │       ├── Level2.unity
│   │       ├── Level3.unity
│   │       └── Level4.unity
│   ├── Pixel Adventure 1/          # Asset pack "Pixel Adventure 1" (tiles, chars, scenes)
│   ├── Sprites/                    # Sprites adicionais
│   ├── Anim/                       # Animações
│   ├── Images/                     # Imagens/UI
│   ├── Musics/                     # Trilha sonora e SFX
│   └── Prefab/                     # Prefabs reutilizáveis
├── ProjectSettings/                # Configurações do Unity
├── Packages/                       # Dependências UPM
├── UserSettings/                   # Configurações de usuário (layout do editor)
├── .collabignore                   # Unity Collaborate ignore
└── .gitignore                      # Unity standard gitignore
```

---

## Principais Mecânicas

### 1. Movimento do Player (`Player.cs`)
- **Movimento horizontal**: `Input.GetAxis("Horizontal")` + `Speed` (transform.position)
- **Pulo**: `JumpForce` via `Rigidbody2D.AddForce(Impulse)` — apenas se `!isJumping`
- **Wall Slide**: Ao colidir com tag `"Wall"` no ar → `wallSliding = true`, velocidade Y clampada em `-wallSlidingSpeed`
- **Wall Jump**: Durante wall slide, seta L/R → `AddForce(wallJump, wallJump*4)` impulsiona para fora
- **Flip automático**: Inverte `localScale.x` conforme direção
- **Mola** (tag `"mola"`): Multiplica `JumpForce` por `molaForce` temporariamente

### 2. Sistema de Combate (`playerMelee.cs` + `Player.cs`)
| Ação | Tecla | Alcance | Dano | Efeito |
|------|-------|---------|------|--------|
| **Soco** | Q | `punchRange` | `punchDamage` | Trigger `PlayerPunch` |
| **Chute** | W | `kickRange` | `kickDamage` | Trigger `PlayerKick` |
| **Bloqueio** | E | — | — | Trigger `PlayerBlock` (anula dano) |
| **Ultimate** | R | `ultRange` | `ultDamage` | (Planejado) `PlayerUlt` |

- **Detecção**: `Physics2D.OverlapCircleAll(transform.position, range, enemyLayers)`
- **Knockback**: `pushBack(hitted, hitter, rig)` aplica força `pushForce` na direção oposta ao inimigo
- **Estados**: Booleans `isPunch`, `isKick`, `isBlock`, `isUlt` resetados após animação via `reset()`

### 3. Sistema de Vida (`Player.cs` + `UIManager.cs`)
- `Health` / `maxHealth` (float) — 6 corações máximos (`Hearts[0..5]`)
- `TakeDamage(damage, enemyPos)`:
  - Se **não bloqueando** (`!isBlock`): reduz vida, knockback, atualiza UI
  - Se **bloqueando**: anula dano completamente
- `UIManager.reduceHearts(current, max)`: Desativa corações da direita para esquerda
- **Game Over**: `Health <= 0` → carrega cena `DeathScreen`

### 4. Inimigos

#### Básico (`Enemy.cs`)
- **Perseguição**: `Vector3.MoveTowards` se distância ≤ `within_range`
- **Vida**: `Health` → `TakeDamage()` reduz, knockback via `playerMelee.pushBack()`
- **Morte**: `Health <= 0` → `Destroy(gameObject)`
- **Colisão com Player**: Causa `damage` + `Afasta(afastaForce)` (impulso vertical)

#### Chefe (`Boss.cs`)
- **Movimento lateral**: Direção `direcao` inverte ao colidir com layer 8 (ground)
- **Dash aleatório**: `Random.Range(0,1000) < 1` → `mergulha()` → `AddForce(-1, 0)` impulsiona para esquerda
- **Colisão com Player**: Dano + `Afasta` + desabilita dash temporariamente (`podeDash = false`)
- **Vida/Morte**: Igual ao inimigo básico

### 5. Coletáveis & Pontuação
- **Frutas** (`Frutas.cs`): `OnTriggerEnter2D` com tag `"Player"` → `GameController.instance.totalScore += Score` → `UpdateScoreText()` → auto-destrói
- **GameController** (Singleton): `totalScore` + `scoreText` (UI Text)

### 6. Progressão de Níveis
- **Checkpoints** (tag `"CheckPoint"`): Incrementa `nextLevel` → carrega `sceneNameList[nextLevel]` (`Level1` → `Level2` → `Level3`)
- **Portão/Loja** (tag `"Gate"`): `isOnGate = true` → `UpArrow` carrega cena `Loja`
- **Loja** (`Lojinha.cs`): Botões `Picanga`, `Chave` (planejados), `Voltar` → `Level1`
- **Entrada Loja** (`entradaLoja.cs`): Colisão direta carrega `Loja`

### 7. Perigos Ambientais
- **Espinhos** (`Spikes.cs`): Colisão → `TakeDamage(damage)` sem knockback horizontal

### 8. Câmera (`CameraController.cs`)
- Follow simples: copia X, Y do player, Z = -10 (fixo)

### 9. Menus & UI
- **MainMenu**: `PlayGame()` → `Level1`, `OpenConfig()` → `Config`, `End()` → `Application.Quit()`
- **DeathScreen** (`Touch_gameOver.cs`): `reiniciaClick()` → `Level1`, `fechaAplicação()` → `Menu`
- **UIManager**: Array `Hearts[6]` — inicia com 3 ativos (índices 3,4,5 desativados)

---

## Assets Utilizados

| Pacote | Conteúdo |
|--------|----------|
| **Pixel Adventure 1** | Tilesets, personagens, animações, cenários (pixel art estilo aventura) |
| **Sprites/** | Sprites adicionais customizados |
| **Anim/** | Animation Controllers / clips |
| **Musics/** | BGM + SFX (pulo, ataque, dano, coleta) |
| **Prefab/** | Prefabs de inimigos, player, itens, UI |
| **TextMesh Pro** | UI de texto (pontuação) |

---

## Cenas & Fluxo

```
Menu.unity
    │
    ├─ PlayGame → Level1.unity
    │               │
    │               ├─ CheckPoint → Level2.unity
    │               │               │
    │               │               ├─ CheckPoint → Level3.unity
    │               │               │               │
    │               │               │               ├─ CheckPoint → Level4.unity
    │               │               │               │
    │               │               │               └─ Gate/Loja → Loja.unity → Voltar → Level1
    │               │               │
    │               │               └─ Gate/Loja → Loja.unity
    │               │
    │               └─ Gate/Loja → Loja.unity
    │
    ├─ OpenConfig → Config.unity
    │
    └─ End → Quit

DeathScreen.unity (quando Health <= 0)
    ├─ Reiniciar → Level1.unity
    └─ Menu → Menu.unity
```

---

## Como Executar

### Pré-requisitos
- **Unity Editor** 2021.3 LTS ou superior (recomendado 2022.3+)
- **.NET Standard 2.1** / .NET 4.x
- **TextMeshPro** (incluído via UPM)

### Passos
1. Clone o repositório:
   ```bash
   git clone https://github.com/DaniellePereiraaa/JungleRogue.git
   ```
2. Abra no **Unity Hub** → "Add project" → selecione a pasta `JungleRogue`
3. Aguarde importação de pacotes (Packages/) e compilação de scripts
4. Abra a cena `Assets/Scenes/Menu.unity`
5. Pressione **Play** no Editor

### Build
- File → Build Settings → Add Open Scenes (Menu, Level1-4, Loja, Config, DeathScreen)
- Target Platform: **Windows / Mac / Linux** (Standalone) ou **WebGL**
- Build and Run

---

## Controles Padrão

| Ação | Tecla |
|------|-------|
| Mover esquerda/direita | ← / →  ou  A / D |
| Pular | Espaço (ou `Jump` input) |
| Soco | Q |
| Chute | W |
| Bloqueio | E |
| Ultimate | R |
| Interagir (Portão/Loja) | ↑ (Seta cima) |
| Pausar / Menu | Esc (se implementado) |

---

## Arquitetura & Padrões

| Padrão | Onde Aplicado |
|--------|---------------|
| **Singleton** | `GameController.instance` (pontuação global) |
| **Component Pattern** | Cada comportamento em `MonoBehaviour` separado |
| **State Machine (Animator)** | Triggers: `PlayerPunch`, `PlayerKick`, `PlayerBlock`, `PlayerUlt` |
| **Observer (Unity Events)** | `OnCollisionEnter2D`, `OnTriggerEnter2D` para colisões/coletáveis |
| **Object Pooling (planejado)** | Frutas, inimigos, partículas |

---

## Roadmap / Melhorias Futuras

- [ ] Ultimate funcional (área de efeito, dano massivo)
- [ ] Itens da loja funcionais (Picanga = cura, Chave = baús)
- [ ] Mais tipos de inimigos (ranged, voador, tanque)
- [ ] Sistema de save/load (PlayerPrefs ou JSON)
- [ ] Pooling de objetos (frutas, inimigos, partículas)
- [ ] Novo Input System (Unity Input System package)
- [ ] Cinemachine para câmera suave (shake, dead zone)
- [ ] Partículas / juice (hit pause, screen shake, dust)
- [ ] Localização (PT/EN/ES)
- [ ] Testes automatizados (Unity Test Framework)
- [ ] Mais níveis / geração procedural (roguelite)

---

## Licença

Projeto educacional / portfolio — livre para estudo e referência.

---

## Autor

Desenvolvido como projeto de **desenvolvimento de jogos em Unity (Action Platformer 2D)**.

Demonstra:
- Programação C# em Unity (MonoBehaviours, Rigidbody2D, física 2D)
- Sistema de combate com múltiplos ataques, bloqueio, knockback
- Movimento avançado: wall slide, wall jump, mola
- IA simples de inimigos (perseguição por distância) e chefe (patrol + dash aleatório)
- Progressão de níveis com checkpoints e loja intermediária
- UI de vida (corações) e pontuação
- Organização de cenas e fluxo de jogo (Menu → Níveis → Loja → Game Over)
- Uso de asset packs (Pixel Adventure 1) e pipeline 2D Unity