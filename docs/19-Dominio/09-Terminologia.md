# Terminologia e Glossário (Linguagem Ubíqua)

Diferente do glossário técnico ligado à implementação (ex: "WinForms", "PacketStream"), este documento foca nos **Termos de Negócio** universais do domínio, permitindo que a equipe fale a mesma língua independente da tecnologia base.

### Agregados e Entidades
- **Session (Sessão)**: Uma instância ativa de um bot jogando em um servidor.
- **World (Mundo)**: A percepção local (matriz 3D de blocos) que o bot possui do jogo em um dado momento.
- **Chunk**: Um pilar geográfico de blocos (16x16 horizontal) até o limite de altura do mundo. Unidade fundamental de carregamento/descarregamento.
- **Entity (Entidade)**: Qualquer ator dinâmico no Mundo: Monstros (Mobs), outros Jogadores, Animais, Itens no chão, Boias de pesca.
- **Player (O Bot)**: A entidade que representa o próprio operador local, sujeita à manipulação de ações (Input).
- **Agent / Macro**: Um controlador de software reativo ou proativo que dita as ações do Jogador no lugar de um ser humano.

### Rede e Fluxo
- **Tick**: A unidade atômica de tempo de processamento. O bot vive em ciclos de 50ms (20 TPS). Cada ciclo é um Tick onde a IA pensa.
- **Protocol Translator / Handler**: A peça que sabe como falar um "dialeto" específico de Minecraft (ex: Protocolo 47 para a versão 1.8) e transformá-lo nas ações universais do Bot.
- **Anti-Cheat**: Ferramentas usadas pelos servidores (ex: Sunshine, NoCheatPlus) para detectar robôs. O Domínio se molda (ex: limitando Pathfinding, respeitando delays físicos) em torno do conceito de *Bypass/Legit*.
- **Proxy**: Uma conexão terceirizada usada para mascarar a origem da Sessão. Para o Domínio puro, é irrelevante, apenas uma propriedade de roteamento.

### Geometria e Física
- **AABB (Axis-Aligned Bounding Box)**: Retângulo 3D que define o volume físico exato que uma Entidade ocupa. Usado para colisões de paredes e cálculos de tiro/hit.
- **RayCast (RayTrace)**: Ato de projetar uma linha virtual a partir do olho do Jogador em uma direção de Pitch/Yaw específica para descobrir em qual bloco/entidade a linha colide. Usado pela IA para saber o que o bot está "olhando".
- **Yaw**: Rotação horizontal da câmera (0 a 360, Sul, Oeste, Norte, Leste).
- **Pitch**: Rotação vertical da câmera (-90 para cima, 90 para baixo).
- **Air**: A abstração da "ausência" de um bloco sólido. Usado quando o bot indaga blocos em espaços vazios ou descarregados.

### Ações (Comandos Embutidos)
- **Drop / Descarte**: Ato de jogar um item do Inventário no chão, transformando-o de `ItemStack` para `EntityItem`.
- **Equip**: Mover um item para a Hotbar e ativá-lo como item em mão.
- **Spoof / Fraud**: Quando a infraestrutura mente deliberadamente para o servidor (ex: altera pacotes no nível da rede) para enganar checagens, ignorando o estado real do Domínio interno.
