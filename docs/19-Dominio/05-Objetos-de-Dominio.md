# Objetos de Domínio

Organização tática do modelo (DDD), permitindo rápida compreensão das estruturas imutáveis vs estruturas de ciclo de vida longo.

## 1. Aggregates (Raízes de Agregação)

A raiz de agregação (Aggregate Root) é a única porta de entrada para modificar os objetos internos.

- **`Session`**: O Agregado principal. Modificar o mundo ou o jogador fora do contexto da sessão é proibido.
- **`World`**: O Agregado geográfico. Contém chunks e entidades e garante a coerência entre eles (ex: deletar um chunk deve remover/invalidar entidades nele).
- **`Inventory`**: O Agregado de itens. Garante que qualquer tentativa de mover um item passe por regras de verificação de Slot e limitação de pilha (Stack Size).

## 2. Entities (Entidades)

Possuem identidade própria (`Id`) e seus estados mudam ao longo do tempo.

- **`Entity` / `Player` / `Mob`**: Identificadas pelo `EntityId` (gerado pelo servidor). Possuem coordenadas variadas, vida e estado de animação.
- **`Plugin`**: Identificado pelo nome/arquivo. Seu estado é Ativo/Inativo/Erro.
- **`Agent` (Command/Macro)**: Identificado pelo seu papel (ex: `MiningAgent`, `FishingAgent`). Tem estado interno mutável (Qual bloco estou olhando? Quanto tempo estou esperando?).

## 3. Value Objects (Objetos de Valor)

São imutáveis. Não possuem identidade. Se um atributo muda, você substitui o objeto inteiro. Essenciais para evitar bugs de concorrência em leitura (ex: um plugin ler a posição enquanto a IA altera).

- **`Vector3d` / `Vector3i`**: Representam posições absolutas ou deslocamentos no mundo 3D.
- **`Block` / `BlockState`**: Define que em `(10, 64, 20)` existe um "Pedra com Metadata 0". O bloco em si não tem ID única na memória, ele é apenas um tipo.
- **`ItemStack`**: Um objeto contendo `ItemType`, `Count`, `Damage/Data` e `NBT Tags`. Dois *ItemStacks* de "64 Pedras sem NBT" são matematicamente iguais.
- **`AABB` (Axis-Aligned Bounding Box)**: A caixa de colisão, definida por limites Mínimos e Máximos (MinX, MaxX...).
- **`SessionCredentials`**: Usuário, Senha (ou Access Token). Não muda depois de criado.

## 4. Domain Services (Serviços de Domínio)

Serviços que não pertencem a uma entidade específica e executam operações complexas cruzando múltiplos limites.

- **`AuthenticationService`**: Fala com a API da Mojang, recebe `SessionCredentials` e retorna tokens ou rejeições.
- **`PathfindingService (A*)`**: Recebe o `World` e duas posições `Vector3i` (Início, Fim) e retorna a rota ótima ou nula.
- **`CollisionResolver`**: Recebe o `World` e a `AABB` atual do `Player` + vetor de movimento, calculando a nova posição legal final respeitando os blocos sólidos (para gravidade e andar).
- **`ProtocolTranslatorFactory`**: Inspeciona a versão pedida na `Session` e devolve o `ProtocolTranslator` (1.5.2, 1.8, etc.) adequado para interpretar os bytes.
