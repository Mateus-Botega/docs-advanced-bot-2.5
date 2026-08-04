# 11 - Estado Atual da Migração

> Última atualização: 2026-07-23
>
> Documento responsável por registrar o estado oficial da migração do AdvancedBot C# para Java.
>
> Este documento deve ser atualizado ao final de cada milestone concluída.

---

# 1. Objetivo

Este documento representa a fotografia oficial do projeto.

Toda IA, desenvolvedor ou colaborador deve consultá-lo antes de iniciar novas implementações.

Seu objetivo é evitar perda de contexto entre sessões e impedir regressões arquiteturais.

---

# 2. Status Geral

Projeto: AdvancedBot Java

Status:

☑ Planejamento

☑ Governança

☑ Fundação Arquitetural

☑ Baseline Tecnológica

☐ Migração do Domínio

☐ Infraestrutura Completa

☐ Interface React

☐ Testes Integrados

☐ Release

---

# 3. Stack Oficial

## Backend

- Java 21 LTS
- Spring Boot 3.2.5
- Maven

## Frontend

- React

## Banco

- PostgreSQL

## Arquitetura

- Clean Architecture
- Hexagonal Architecture
- DDD (quando aplicável)
- SOLID

---

# 4. Estado da Governança

| Documento | Status |
|-----------|--------|
| 00 | ✔ |
| 01 | ✔ |
| 02 | ✔ |
| 03 | ✔ |
| 04 | ✔ |
| 05 | ✔ |
| 06 | ✔ |
| 07 | ✔ |
| 08 | ✔ |
| 09 | ✔ |
| 10 | ✔ |

---

# 5. Milestones

## Milestone 1

Status

Concluído

Objetivos

- Estrutura Maven
- Spring Boot
- Organização inicial
- Fundação arquitetural

Resultado

Aprovado

---

## Milestone 2

Status

Concluído

Objetivos

- Consolidação da baseline Java 21
- Ajustes de documentação

Resultado

Aprovado

---

## Milestone 3

Status

Concluído

Objetivo

Início da migração incremental do domínio C# — núcleo do domínio do Bot (entidades, Value Objects e Use Cases iniciais)

Resultado

Aprovado

Entregue

- Pacotes `domain.bot` e `application.usecase` criados (DEC-12).
- Entidade: `Bot` (agregado raiz, identidade + configuração de conexão + sessão).
- Value Objects: `IdentificadorBot`, `EnderecoServidor`, `CredenciaisBot`, `SessaoBot`.
- Enum de domínio: `EstadoSessao` (`DISCONNECTED`, `CONNECTING`, `CONNECTED`).
- Use Cases: `CasoDeUsoCriarBot`, `CasoDeUsoConectarBot`, `CasoDeUsoDesconectarBot`.
- Sem regras de negócio completas do legado (rede, protocolo, macros, reconexão automática, persistência) — deliberadamente fora do escopo desta milestone.

Observação de escopo

"Migração do Domínio" (item da Seção 2) permanece parcialmente incompleta — esta milestone cobriu apenas o núcleo (kernel) do agregado Bot. Demais entidades do jogo (Jogador, Mundo, Bloco, Item, Inventário) não foram modeladas e ficam para milestones futuras, quando houver necessidade funcional real (ex.: ao integrar a camada de protocolo).

---

## Milestone 4

Status

CONCLUÍDA

Objetivo

Sistema de conexão Minecraft — Protocol Layer 1.8 e Handshake (Fase 3 do [07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md](07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md))

### Incremento 1 — Arquitetura da Camada de Comunicação

Status

Concluído

Objetivo

Projetar a infraestrutura de comunicação entre domínio e protocolo (Ports da Application, arquitetura de rede, Packet/Codec/Serializer/Deserializer/Registry/Handler) antes de implementar qualquer packet Minecraft concreto. C# consultado apenas para entender responsabilidades (`Client/PacketStream.cs`, `Client/ReadBuffer.cs`/`WriteBuffer.cs`, `Client/Handler/Handler_v18.cs`, `Client/IPacket.cs`), sem migração de código.

Resultado

Aprovado (ver [DEC-13](01-Decisoes-Arquiteturais.md))

Entregue

- `domain.protocol`: `Packet`, `Codec<T>`, `LeitorDePacote`, `EscritorDePacote`, `EstadoConexao`, `VersaoProtocolo`, `PacketHandler<T>`, `EventoDeProtocolo` (marcadora, EventBus fica para milestone futura).
- `domain.network`: `ConexaoMinecraft` (port), `SessaoDeRede` (Value Object imutável).
- `application.port`: `ConexaoBotPort`.
- `infrastructure.protocol`: `RegistroDePacotes` (mapeamento ID/EstadoConexao/Codec), `ProtocolDispatcher` (localiza e encaminha pacotes aos `PacketHandler`s; os Handlers só traduzem, não roteiam).
- Removidos os diretórios legados pré-DEC-12 (`com.advancedbot.core`, `.bot`, `.network`, `.protocol`, `.pathfinding` — vazios, apenas `.gitkeep`), substituídos pela estrutura em camadas.
- Decisão confirmada: sem Netty — `java.net.Socket` bloqueante + Virtual Threads (DEC-03) quando a conexão real for implementada.

Observação de escopo

Handshake, Login, pacotes concretos (1.5.2/1.8) e o adapter de Socket real **não foram implementados** — ficam para o próximo incremento da Milestone 4.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — compilação e suíte de testes existente passaram sem regressões.

### Incremento 2 — Packets Concretos de Handshake e Login (Protocolo 1.8)

Status

Concluído

Objetivo

Modelar os primeiros packets concretos do protocolo Minecraft 1.8 (Handshake e Login Start) sobre os contratos criados no incremento 1, usando o C# (`Client/Packets/PacketHandshake.cs`, `Client/Packets/PacketLoginStart.cs`, `Client/Handler/Handler_v18.cs`, `Client/ReadBuffer.cs`/`WriteBuffer.cs`) apenas como fonte da verdade do protocolo, sem migração de código.

Resultado

Aprovado (ver [DEC-14](01-Decisoes-Arquiteturais.md))

Entregue

- `domain.protocol.v1_8`: `HandshakePacket` e `LoginStartPacket` (Records imutáveis, sem conhecimento de Socket/Streams/Registry/Dispatcher), `HandshakeCodec` e `LoginStartCodec` (implementam `Codec<T>`, apenas serialização/desserialização, sem regra de negócio).
- `infrastructure.protocol`: `BufferLeitorDePacote`/`BufferEscritorDePacote` — implementações concretas de `LeitorDePacote`/`EscritorDePacote` sobre buffer em memória (sem Socket/TCP), necessárias para exercitar os Codecs nos testes.
- `infrastructure.protocol.v1_8`: `RegistroDePacotesV1_8` — implementação concreta de `RegistroDePacotes` registrando `HandshakePacket` (`EstadoConexao.HANDSHAKING`, id `0x00`) e `LoginStartPacket` (`EstadoConexao.LOGIN`, id `0x00`), sem switch monolítico e sem reflexão (mapas por `EstadoConexao`+id e por `Class`).
- DEC-14: extensão aditiva de `LeitorDePacote`/`EscritorDePacote` com `readUnsignedShort`/`writeUnsignedShort`, necessária porque `ServerPort` é `ushort` no protocolo (0–65535), o que não cabia em `short` assinado.
- 14 testes unitários novos (JUnit 5 + AssertJ) cobrindo encode, decode e round-trip de `HandshakeCodec` e `LoginStartCodec`, mais localização no `RegistroDePacotesV1_8`.

Observação de escopo

Não foram implementados: Socket/TCP/conexão real, Login completo (Encryption Request/Login Success), KeepAlive, Chat, Compression/Encryption/SetCompression, Play State, EventBus, Scheduler, Bots, Macros, ou qualquer adapter de infraestrutura que conecte `ConexaoMinecraft` a I/O real. `ProtocolDispatcher` e `PacketHandler`s concretos também não foram implementados nesta etapa (não fazem parte do escopo do incremento 2).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 16 testes executados (2 pré-existentes + 14 novos), 0 falhas.

### Incremento 3 — Conclusão da Modelagem do Estado LOGIN (Protocolo 1.8)

Status

Concluído

Objetivo

Concluir a modelagem do estado LOGIN do protocolo Minecraft 1.8 (Records, Codecs e Registro — sem Socket/TCP real), usando o C# (`AdvancedBot.Client.MinecraftClient.cs`, `AdvancedBot.Client.Packets.PacketEncryptionResponse.cs`, `AdvancedBot.Client.ReadBuffer.cs`/`WriteBuffer.cs`) apenas como fonte da verdade do protocolo, sem migração de código.

Resultado

Aprovado (ver [DEC-15 e DEC-16](01-Decisoes-Arquiteturais.md))

Entregue

- `domain.protocol.v1_8`: `EncryptionRequestPacket`, `EncryptionResponsePacket`, `SetCompressionPacket`, `LoginSuccessPacket` (Records imutáveis) e seus respectivos Codecs (`EncryptionRequestCodec`, `EncryptionResponseCodec`, `SetCompressionCodec`, `LoginSuccessCodec`), todos restritos a serialização/desserialização, sem regra de negócio.
- `EncryptionRequestPacket`/`EncryptionResponsePacket` sobrescrevem `equals`/`hashCode` (comparação por conteúdo via `Arrays.equals`/`Arrays.hashCode`), já que os componentes `byte[]` de um Record usam igualdade por referência por padrão.
- DEC-15: extensão aditiva de `LeitorDePacote`/`EscritorDePacote` com `readByteArray`/`writeByteArray`, necessária para os campos criptográficos crus (`publicKey`, `verifyToken`, `sharedSecret`) prefixados por comprimento `VarInt`.
- DEC-16: `domain.protocol.SentidoDoPacote` (novo enum `CLIENTBOUND`/`SERVERBOUND`) — `RegistroDePacotes.registrar`/`localizarCodec` passam a exigir a direção do pacote, corrigindo uma colisão real de ID (`EncryptionRequestPacket` e `EncryptionResponsePacket` usam ambos o id `0x01` no estado LOGIN, em direções opostas).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8`: os 4 novos packets registrados (`EncryptionRequestPacket` CLIENTBOUND id `0x01`, `EncryptionResponsePacket` SERVERBOUND id `0x01`, `LoginSuccessPacket` CLIENTBOUND id `0x02`, `SetCompressionPacket` CLIENTBOUND id `0x03`), mantendo os registros existentes de `HandshakePacket`/`LoginStartPacket` (ambos SERVERBOUND).
- 26 testes unitários novos (JUnit 5 + AssertJ) cobrindo encode, decode, round-trip dos 4 novos Codecs e localização no `RegistroDePacotesV1_8` (incluindo teste dedicado provando que a colisão de ID 0x01 é resolvida corretamente pela direção).

Observação de escopo

Não foram implementados: Socket/TCP/conexão real, Virtual Threads, criptografia AES/RSA real, geração de Shared Secret, compressão funcional (zlib), Session Server/Mojang Authentication, KeepAlive, Chat, Play State, Status State, EventBus, Scheduler, Macros, Bot Engine, `ProtocolDispatcher` e `PacketHandler`s concretos. Os 4 novos Codecs manipulam apenas a forma serializada dos campos (bytes crus, inteiros e strings), sem qualquer lógica de negócio associada.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 42 testes executados (16 pré-existentes + 26 novos), 0 falhas.

### Incremento 4 — Infraestrutura de Transporte TCP

Status

Concluído

Objetivo

Implementar a infraestrutura responsável por transportar bytes entre a aplicação e o protocolo: adapter Socket (`java.net.Socket`), Virtual Threads (DEC-03), FrameEncoder e FrameDecoder (length-prefix VarInt), integração com `LeitorDePacote`/`EscritorDePacote`. C# consultado (`AdvancedBot.Client.PacketStream.cs`, `AdvancedBot.Client.MinecraftStream.cs`) para entender o formato de framing do protocolo Minecraft, sem migração de código.

Entregue

- `infrastructure.protocol.CodificadorDeFrame`: codifica um packet ID + payload em um frame com prefixo VarInt de comprimento (`[VarInt length][VarInt packetId][payload]`), reutilizando `BufferEscritorDePacote` internamente. Stateless, sem I/O.
- `infrastructure.protocol.DecodificadorDeFrame`: lê um frame completo de um `InputStream`, parseando o VarInt de comprimento byte a byte, validando tamanho máximo (2 MiB, conforme C# legado), e retornando o conteúdo do frame (packetId + payload) sem o prefixo de comprimento.
- `infrastructure.network.TransporteSocket`: implementação concreta de `ConexaoMinecraft` (`domain.network`). Recebe um `java.net.Socket` (ou `InputStream`/`OutputStream` para testes), utiliza `CodificadorDeFrame`/`DecodificadorDeFrame` para framing, `BufferLeitorDePacote`/`BufferEscritorDePacote` para serialização via Codecs, e `RegistroDePacotes` para lookup de ID/Codec. Escrita sincronizada (`writeLock`). Leitura em Virtual Thread (`Thread.ofVirtual()`, DEC-03) via `startReading()`, com loop que decodifica frames, resolve Codecs (CLIENTBOUND) e notifica o handler registrado.
- 20 testes unitários novos (JUnit 5 + AssertJ): 5 para `CodificadorDeFrame` (frame simples, payload vazio, packet ID multi-byte, compatibilidade com decodificador, múltiplos frames), 8 para `DecodificadorDeFrame` (frame simples, payload vazio, múltiplos frames consecutivos, stream vazio, stream truncado, VarInt inválido, VarInt de 2 bytes, compatibilidade com codificador), 7 para `TransporteSocket` (envio serializado, payload compatível com codec, recepção via read loop com Virtual Thread, múltiplos pacotes consecutivos, sessão de rede, encerramento gracioso, round-trip completo send+receive).

Observação de escopo

Não foram implementados: conexão real com servidor Minecraft, Handshake/Login flow, compressão de frames (formato comprimido com Data Length), criptografia AES-CFB8, `ProtocolDispatcher`/`PacketHandler`s concretos, Play State, Status State, EventBus, Scheduler, Macros, Bot Engine. O `TransporteSocket` suporta apenas o formato de frame não-comprimido (compressionThreshold < 0), que é o formato usado durante Handshake e Login antes de `SetCompression` ser recebido.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 62 testes executados (42 pré-existentes + 20 novos), 0 falhas.

### Incremento 5 — Validação Integrada da Pipeline de Protocolo (PacketHandlers e ProtocolDispatcher)

Status

Concluído

Objetivo

Validar a integração entre `RegistroDePacotes`, `Codec`, `CodificadorDeFrame`/`DecodificadorDeFrame`, `ProtocolDispatcher` e `PacketHandler`, implementando os primeiros `PacketHandler`s concretos do protocolo 1.8 e testando a pipeline completa de ponta a ponta — sem conectar a um servidor Minecraft real. C# consultado apenas para confirmar a semântica dos campos já modelados (`AdvancedBot.Client.MinecraftClient.cs`, método `HandlePacket`), sem migração de código e sem replicar a lógica de negócio associada (RSA/Mojang/AES, ativação de compressão, transição de estado de conexão).

Entregue

- `domain.protocol.v1_8`: 6 Records de Evento (`EventoHandshake`, `EventoLoginStart`, `EventoEncryptionRequest`, `EventoEncryptionResponse`, `EventoLoginSuccess`, `EventoSetCompression`), espelhando 1:1 os campos — e a mesma postura de validação — dos Packets correspondentes. Nomenclatura com prefixo "Evento" segue o precedente já estabelecido em código por `EventoDeProtocolo` (DEC-13), não a seção "Nomeação de Eventos" do [12-Guia-de-Nomenclatura.md](12-Guia-de-Nomenclatura.md) (anterior à DEC-11/pt-BR e nunca atualizada); ajuste de nomenclatura, não decisão arquitetural (não abre DEC, conforme [07-Controle-de-Decisoes.md](07-Controle-de-Decisoes.md)).
- `domain.protocol.v1_8`: 6 `PacketHandler`s concretos (`HandshakeHandler`, `LoginStartHandler`, `EncryptionRequestHandler`, `EncryptionResponseHandler`, `LoginSuccessHandler`, `SetCompressionHandler`), cada um restrito a traduzir Packet em Evento — sem lógica de negócio, sem side-effects (nenhuma criptografia, nenhuma mutação de `EstadoConexao`/`SessaoDeRede`, nenhuma ativação de compressão).
- `ProtocolDispatcher` (já implementado desde o Incremento 1, sem alteração) validado por `ProtocolDispatcherTest` (novo): roteamento correto por tipo de Packet e erro `IllegalStateException` para tipo sem handler registrado.
- `PipelineDeProtocoloV1_8Test` (novo): 6 cenários de integração (um por Packet), sem mocks, exercitando a cadeia real completa `Packet → Codec.encode → CodificadorDeFrame → DecodificadorDeFrame → RegistroDePacotesV1_8.localizarCodec → Codec.decode → Packet (round-trip) → ProtocolDispatcher.dispatch → PacketHandler → EventoDeProtocolo`, incluindo o cenário de colisão de id 0x01 (Encryption Request/Response, DEC-16) processado corretamente de ponta a ponta.
- Nenhuma interface existente foi alterada (`Packet`, `Codec`, `PacketHandler`, `EventoDeProtocolo`, `RegistroDePacotes`, `ProtocolDispatcher`, `TransporteSocket`) — incremento 100% aditivo, sem DEC nova.
- 15 testes novos (6 de Handler + 3 de `ProtocolDispatcher` + 6 de integração da pipeline), JUnit 5 + AssertJ.

Observação de escopo

Não foram implementados: orquestração real do fluxo Handshake→Login via `TransporteSocket`/Socket (o teste de integração exercita a mesma lógica de forma síncrona/determinística, sem I/O real — `TransporteSocket` não foi alterado), conexão com servidor Minecraft, autenticação Mojang, criptografia AES-CFB8/RSA real, compressão funcional (zlib), Play State, Status State, KeepAlive, Chat, entidades, inventário, mundo, scheduler, macros, EventBus, mutação de `EstadoConexao`/`SessaoDeRede`.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 77 testes executados (62 pré-existentes + 15 novos), 0 falhas.

### Incremento 6 — Integração Application ↔ Infraestrutura de Comunicação

Status

Concluído

Objetivo

Conectar as peças arquiteturais já existentes — `CasoDeUsoConectarBot`, `ConexaoBotPort`, `ConexaoMinecraft`, `TransporteSocket`, `ProtocolDispatcher`, `PacketHandlers`, `EventoDeProtocolo` e `SessaoBot` — fechando o achado principal da auditoria da Milestone 4 Incremento 5 (`ConexaoBotPort` declarado desde a DEC-13 mas nunca implementado nem chamado). Sem conectar a um servidor Minecraft real; toda validação usa infraestrutura local (socket loopback). C# consultado apenas para confirmar a semântica do fluxo de conexão (`AdvancedBot.Client.MinecraftClient.cs`, método `ConnectAndHandshake()`), sem migração de código.

Resultado

Aprovado (ver [DEC-17](01-Decisoes-Arquiteturais.md))

Entregue

- `domain.network.ConexaoMinecraft`: novo método `avancarEstado(EstadoConexao)` (extensão aditiva, DEC-17) — corrige um bloqueador real descoberto durante o design: `TransporteSocket` não tinha nenhuma forma de sair de `HANDSHAKING` após a construção, o que faria qualquer tentativa de enviar `LoginStartPacket` após `HandshakePacket` falhar sempre com `IllegalArgumentException`.
- `infrastructure.network.TransporteSocket`: implementa `avancarEstado` reatribuindo `SessaoDeRede` via `comEstado` (método já existente desde o Incremento 1, sem chamador até agora).
- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8`: primeira implementação de `ConexaoBotPort` (DEC-13). Autocontido — monta seu próprio `ProtocolDispatcher` com os 6 `PacketHandler`s v1_8 no construtor (mesmo padrão de `RegistroDePacotesV1_8`). `connect()` envia `HandshakePacket`+`LoginStartPacket`, avança `EstadoConexao` para LOGIN entre os dois envios, e bloqueia em um `CompletableFuture` com timeout configurável até `EventoLoginSuccess` chegar (sucesso) ou `EventoEncryptionRequest`/`EventoSetCompression` chegar (falha rápida e explícita — não suportados nesta milestone) ou o timeout estourar. Recebe a fábrica de conexão (`Function<EnderecoServidor, ConexaoMinecraft>`) via construtor — nenhuma fábrica de produção (Socket real) é fornecida nesta etapa. `disconnect()` lança `UnsupportedOperationException` explícita (não integrado nesta milestone).
- `application.usecase.CasoDeUsoConectarBot`: passa a depender de `ConexaoBotPort` via construtor; marca `SessaoBot` como CONNECTING, chama a porta, aplica o resultado preservando `autoReconnect`, e reverte para DISCONNECTED relançando a exceção em caso de falha. Zero import de tipos de protocolo/pacote.
- `domain.bot.Bot`: campo `session` passa a ser `volatile` — primeira vez que uma thread de fundo (leitura do `TransporteSocket`) influencia causalmente esse valor.
- 12 testes novos (JUnit 5 + AssertJ, sem mocks): 2 de regressão em `TransporteSocketTest` (prova direta do bug do `avancarEstado`), 4 em `AdaptadorConexaoBotV1_8Test` (sucesso, bytes enviados corretos, timeout, fechamento best-effort na falha — usando `ServerSocket` loopback), 5 em `CasoDeUsoConectarBotTest` (gap de cobertura da Milestone 3 fechado, usando um `ConexaoBotPort` fake escrito à mão), 1 em `FluxoConexaoBotV1_8Test` (ponta a ponta real: `CasoDeUsoConectarBot` → `AdaptadorConexaoBotV1_8` → `TransporteSocket` → `ProtocolDispatcher` → Handlers → `SessaoBot`, sobre socket loopback).

Observação de escopo

Não foram implementados: conexão com servidor Minecraft real, fábrica de produção de `Socket` real, autenticação Mojang, Session Server, criptografia AES-CFB8/RSA, compressão funcional (zlib) — o Adapter falha rápido e explicitamente se o servidor exigir qualquer um dos dois últimos, em vez de negociá-los. `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` continuam não integrados (limitação aceita e documentada: uma conexão bem-sucedida não é retida em nenhum lugar, então a Virtual Thread de leitura de uma conexão com sucesso roda indefinidamente sem ninguém para fechá-la — inofensivo com streams/sockets locais de teste, real quando houver socket de produção). Nenhuma transição para `EstadoConexao.PLAY` (sem Packets/Handlers de Play State ainda). Nenhum wiring Spring/`@Configuration` — a fábrica de conexão continua injetada manualmente nos testes.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 89 testes executados (77 pré-existentes + 12 novos), 0 falhas.

### Incremento 7A — Fábrica de Produção e Primeira Comunicação Real (Handshake)

Status

Concluído

Objetivo

Realizar a primeira comunicação real com um servidor Minecraft 1.8: implementar a fábrica de produção de `ConexaoMinecraft` sobre `java.net.Socket`, integrá-la ao `AdaptadorConexaoBotV1_8`, e validar o envio de um `HandshakePacket` de verdade, com encerramento limpo da conexão. C# consultado apenas para confirmar o formato de abertura de conexão (`AdvancedBot.Client.MinecraftClient.cs`, `ConnectAndHandshake()`), sem migração de código.

Entregue

- `infrastructure.network.v1_8.FabricaDeConexaoMinecraftV1_8`: primeira implementação de produção de `Function<EnderecoServidor, ConexaoMinecraft>` (o seam definido na DEC-17/Incremento 6). Abre um `java.net.Socket` real (com timeout de conexão explícito via `Socket.connect(SocketAddress, int)`), constrói `TransporteSocket` sobre ele com `RegistroDePacotesV1_8`/`SessaoDeRede.inicial(V1_8)`, e inicia a leitura (`startReading()`). Falhas de conexão (host inalcançável, porta fechada) resultam em `IllegalStateException` explícita.
- Nenhuma alteração em `AdaptadorConexaoBotV1_8`, `ConexaoMinecraft`, `ConexaoBotPort`, `TransporteSocket` ou `CasoDeUsoConectarBot` — 100% aditivo, sem DEC nova.
- 3 testes novos (JUnit 5 + AssertJ, sem mocks) em `FabricaDeConexaoMinecraftV1_8Test`, usando `ServerSocket` loopback local (mesmo padrão do Incremento 6): abertura de conexão real via `Socket` + envio de `HandshakePacket` com verificação byte a byte do que o "servidor" recebeu; encerramento limpo (`close()`) comprovado pelo lado servidor observando EOF; falha explícita e previsível quando a conexão não pode ser estabelecida.
- **Validação contra servidor Minecraft 1.8 real**, fornecido pelo responsável do projeto (rede Craftlandia, servidor Olimpo): `infrastructure.network.v1_8.HandshakeServidorRealTest`, anotado `@Disabled` (não roda em `mvn clean test`/CI — é uma validação manual pontual contra infraestrutura de terceiros, não um teste automatizado repetível). Executado manualmente uma vez em 2026-07-16 contra `olimpo.clmc.com.br:3737`: conexão TCP real aceita, `HandshakePacket(protocolVersion=47, nextState=LOGIN)` enviado sem erro, nenhuma resposta inesperada em 2s de observação (comportamento correto — o protocolo não define resposta a um Handshake isolado). `olimpo.craftlandia.com.br` (a outra URL fornecida) não resolveu por DNS neste ambiente (sem registro A/AAAA — apenas SOA); `olimpo.clmc.com.br` resolveu normalmente para `91.190.102.46`. Reportado ao responsável do projeto como divergência a acompanhar, não corrigido nem contornado.

Observação de escopo — decisões explícitas do responsável pelo projeto

Duas questões foram levadas ao responsável pelo projeto antes de tocar em rede real:

1. Como obter um servidor Minecraft 1.8 real para validação — decisão inicial: pendente, resolvida nesta mesma sessão quando o responsável forneceu o endereço do servidor Olimpo/Craftlandia.
2. Como reconciliar "integrar o `AdaptadorConexaoBotV1_8` com a fábrica" e "não implementar LoginStart", dado que `AdaptadorConexaoBotV1_8.connect()` (aprovado no Incremento 6) já envia Handshake **e** LoginStart juntos — decisão: a validação ocorre na camada `ConexaoMinecraft` diretamente (`fabrica.apply(endereco)` + `conexao.send(handshake)` + `conexao.close()`), sem chamar `connect()`, inclusive contra o servidor real. A integração do Adapter com a fábrica de produção está pronta por construção (`AdaptadorConexaoBotV1_8` aceita qualquer `Function<EnderecoServidor, ConexaoMinecraft>`, incluindo a nova fábrica), mas não foi exercitada ponta a ponta (isto é, `connect()` nunca foi chamado contra o servidor real) — fica para quando LoginStart voltar a estar em escopo.

Não foram implementados (permanecem fora de escopo, conforme instruído): LoginStart, autenticação Mojang, Encryption, Compression, KeepAlive, Chat, Play State, World, Inventory, Entity, Scheduler, Macros. Nenhum desses foi enviado ao servidor real em nenhum momento.

Divergência registrada em relação ao legado/esperado: `olimpo.craftlandia.com.br` não resolveu por DNS neste ambiente, apenas `olimpo.clmc.com.br` — ambos descritos pelo responsável do projeto como o mesmo servidor; a causa (filtragem de DNS local vs. ausência real do registro público) não foi investigada, por estar fora do escopo desta validação.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 93 testes executados (92 pré-existentes + 1 novo), 1 skipped (`HandshakeServidorRealTest`, deliberadamente), 0 falhas. Bytes do `HandshakePacket` validados tanto sobre `java.net.Socket` local (loopback) quanto, manualmente, sobre uma conexão real ao servidor Olimpo/Craftlandia — aceita sem erro nos dois casos.

### Incremento 7B — Fluxo Real de LOGIN até a Primeira Resposta do Servidor

Status

Concluído

Objetivo

Implementar o fluxo real de LOGIN (Handshake → LoginStart) até a primeira resposta do servidor, usando `AdaptadorConexaoBotV1_8` como ponto oficial de entrada, e validar o comportamento contra o mesmo servidor Minecraft 1.8 real do Incremento 7A (Olimpo/Craftlandia).

Entregue

- **Correção em `AdaptadorConexaoBotV1_8`**: o caminho de sucesso (`EventoLoginSuccess`) não avançava `EstadoConexao` da conexão para `PLAY` — ficava preso em `LOGIN` mesmo após o login confirmado. Bug pré-existente do Incremento 6, não coberto por nenhum teste até agora (nenhum teste verificava `EstadoConexao` após `connect()`, só `SessaoBot`). Corrigido adicionando `conexao.avancarEstado(EstadoConexao.PLAY)` em `reagirAoPacote` antes de completar o login — reaproveita o método `avancarEstado` já existente desde a DEC-17, nenhuma interface mudou, sem DEC nova.
- **2 testes novos** em `AdaptadorConexaoBotV1_8Test`: confirma que `EstadoConexao` avança para `PLAY` ao receber `LoginSuccess` (regressão direta do bug acima); confirma que, ao receber `EncryptionRequest`, a exceção lançada é clara (mensagem contém "autenticação/criptografia") e `EstadoConexao` permanece corretamente em `LOGIN` (não avança) — cenário que nenhum teste anterior exercitava com um `EncryptionRequestPacket` de verdade vindo de um servidor fake.
- **Validação manual contra o servidor real** (`infrastructure.network.v1_8.LoginServidorRealTest`, `@Disabled`, mesmo padrão do Incremento 7A): `AdaptadorConexaoBotV1_8` real + `FabricaDeConexaoMinecraftV1_8` real, contra `olimpo.clmc.com.br:3737`. Resultado observado: o servidor respondeu com **`SetCompressionPacket`** — um terceiro cenário real, não descrito no escopo original (que citava apenas Encryption Request/modo online e Login Success/modo offline), mas já coberto pelo design existente desde o Incremento 6 (mesmo tratamento de falha explícita e clara dado a `EncryptionRequest`). Não foi possível determinar se o servidor está em modo online ou offline: como ele habilita compressão antes do próximo pacote (seja `EncryptionRequest` ou `LoginSuccess`), e o projeto não implementa parsing de frames comprimidos (fora de escopo), não há como decodificar o que viria a seguir.

Observação de escopo — decisões e achados

Nenhum dos itens fora de escopo (AES-CFB8, criptografia, Session Server, autenticação Mojang, compressão, Play State, KeepAlive, Chat, Entity, Inventory, World, Scheduler, React) foi implementado. `TransporteSocket` permanece completamente agnóstico de protocolo (nenhum novo import de tipo v1.8) e os `PacketHandler`s continuam apenas traduzindo Packet → Evento — nenhuma das duas invariantes foi violada.

**Risco real encontrado, não corrigido nesta etapa:** após o servidor enviar `SetCompressionPacket`, o próximo pacote que ele envia já está no formato comprimido (que não implementamos). A thread de leitura (`TransporteSocket.readLoop`, Virtual Thread) tenta decodificá-lo como não-comprimido, `RegistroDePacotesV1_8.localizarCodec` lança `IllegalArgumentException` (id não registrado), e essa exceção **não é capturada** pelo `catch (IOException e)` do `readLoop` — a Virtual Thread morre com uma exceção não tratada (visível como stack trace em stderr). Isso não afeta a corretude do `connect()` em si (que já havia falhado corretamente e fechado a conexão antes disso), mas é um encerramento não inteiramente limpo da thread de leitura. A causa raiz é diretamente compressão (fora de escopo) — por isso a correção não foi feita agora; ficou registrada para quando compressão for implementada, ou como um ajuste pontual e pequeno de robustez (`readLoop` capturar `RuntimeException` de forma equivalente ao `IOException`) a ser autorizado separadamente.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 96 testes executados (93 pré-existentes + 3 novos: 2 de `AdaptadorConexaoBotV1_8Test`, 1 de `LoginServidorRealTest`), 2 skipped (`HandshakeServidorRealTest` e `LoginServidorRealTest`, deliberadamente), 0 falhas.

### Incremento 7C — Robustez do TransporteSocket contra Falhas de Decodificação

Status

Concluído

Objetivo

Fortalecer `TransporteSocket.readLoop` para um encerramento seguro diante de falhas de decodificação durante a leitura do protocolo, fechando o risco de robustez registrado no Incremento 7B (exceção não tratada ao processar um pacote não-decodificável, ex.: bytes já comprimidos após `SetCompression`) antes de iniciar qualquer implementação de compressão. C# não consultado — robustez de Virtual Threads (DEC-03) é uma preocupação específica da infraestrutura Java, sem equivalente no legado.

Entregue

- `infrastructure.network.TransporteSocket.readLoop`: novo bloco `catch (RuntimeException e)`, adicionado ao lado do `catch (IOException e)` já existente (não alterado). Qualquer `RuntimeException` originada durante a decodificação ou o despacho de um pacote (ex.: `IllegalArgumentException` de `RegistroDePacotes.localizarCodec` para um id não registrado — o mesmo tipo de falha esperado ao interpretar bytes comprimidos como não-comprimidos) agora encerra a Virtual Thread de forma controlada (`active = false` + `close()`, liberando `input`/`output`) em vez de propagar como exceção não tratada. Nenhuma interface alterada (`ConexaoMinecraft`/`TransporteSocket` mantêm a mesma superfície pública) — mudança 100% interna, sem DEC nova.
- 1 teste novo em `TransporteSocketTest` (`deveEncerrarReadLoopGraciosamenteEFecharRecursosAoReceberPacoteNaoRegistrado`): alimenta o `readLoop` real com um frame de id não registrado (`0x7F` em LOGIN/CLIENTBOUND) e comprova, via streams de rastreamento de fechamento, que `input`/`output` são fechados após a falha — prova indireta de que a exceção foi capturada e a thread encerrou graciosamente (sob o comportamento antigo, o teste trava até o timeout, pois `close()` nunca seria chamado). Confirmado adicionalmente por inspeção do log do `mvn test`: nenhuma ocorrência de "Exception in thread" no cenário.

Observação de escopo

Não foram implementados: compressão, criptografia, `LoginSuccess`/Play State, `KeepAlive` — conforme instruído. O `catch (IOException e)` pré-existente não foi alterado (permanece sem fechar `input`/`output` quando a falha não é causada por um `close()` explícito); esse gap é anterior a este incremento e não fazia parte do risco documentado no Incremento 7B (que era especificamente sobre `RuntimeException` não capturada), então não foi tocado — registrado aqui apenas como observação, não como pendência formal.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 97 testes executados (96 pré-existentes + 1 novo), 2 skipped (`HandshakeServidorRealTest` e `LoginServidorRealTest`, deliberadamente), 0 falhas.

### Incremento 8A — Fundação de Compressão zlib (Framing, Isolado do Fluxo de LOGIN)

Status

Concluído

Objetivo

Implementar o framing com suporte a compressão zlib (formato do protocolo Minecraft 1.7+) de forma isolada e testada — sem integrar ainda ao fluxo de LOGIN — resolvendo o bloqueador identificado no Incremento 7B (servidor real Olimpo/Craftlandia exige compressão). Precedido de estudo técnico do C# (`AdvancedBot.Client.PacketStream.cs`, `MinecraftStream.cs`, `Utils.ZlibCompress`, `Ionic.Zlib/ZlibStream.cs`, `CompressionLevel.cs`) e de uma revisão arquitetural crítica do plano antes de qualquer código (ver [DEC-18](01-Decisoes-Arquiteturais.md)).

Entregue

- `domain.network.ConexaoMinecraft`: novo método `ativarCompressao(int threshold)` (DEC-18, extensão aditiva, mesmo padrão da DEC-17). `infrastructure.network.TransporteSocket` implementa reatribuindo `sessao` via `SessaoDeRede.comCompressao` (método existente desde o Incremento 1, sem chamador até agora).
- `infrastructure.protocol.CodificadorDeFrame`/`DecodificadorDeFrame`: novas sobrecargas aditivas `encode(int,byte[],int)`/`decode(InputStream,int)` implementando o formato de frame com compressão (`VarInt(dataLength)` + payload cru quando abaixo do threshold, ou `VarInt(dataLength)` + zlib quando no threshold ou acima), usando `java.util.zip.Deflater`/`Inflater` (nível `BEST_SPEED`, formato zlib — equivalente exato ao `Ionic.Zlib.ZlibStream`/`CompressionLevel.BestSpeed` do C#, sem dependência nova). Sobrecargas antigas (`encode(int,byte[])`/`decode(InputStream)`) inalteradas. `TransporteSocket.send`/`readLoop` passam a consultar `sessao.compressionThreshold()` a cada chamada.
- `dataLength` tratado como dica de pré-alocação do buffer de descompressão, não como validação rígida (replica o comportamento do C#, que também não valida esse campo), com teto de 2 MiB aplicado ao valor declarado (mesmo guard já existente para o frame externo). `DataFormatException` (checked) do `Inflater` é embrulhada como `IOException`, preservando o tratamento já existente desde o Incremento 7C.
- `AdaptadorConexaoBotV1_8` **não foi alterado** — `EventoSetCompression` continua falhando explicitamente. Integração ao fluxo de LOGIN fica para o Incremento 8B, deliberadamente separado para manter este incremento pequeno e revisável isoladamente.
- 10 testes novos (JUnit 5 + AssertJ, sem mocks): 3 em `CodificadorDeFrameTest` (threshold negativo idêntico ao comportamento atual, payload abaixo do threshold com sentinel `dataLength=0`, payload comprimido com round-trip completo), 5 em `DecodificadorDeFrameTest` (threshold negativo idêntico, sentinel de não-comprimido, descompressão de zlib construído independentemente do `CodificadorDeFrame`, dado corrompido levantando `IOException`, compatibilidade round-trip com compressão ativa), 2 em `TransporteSocketTest` (regressão de `ativarCompressao`, envio+recepção completos com compressão ativa sobre pipe loopback).

Observação de escopo

Não implementados: criptografia AES-CFB8 real, Session Server/Mojang Authentication, Play State, Status State, KeepAlive. Revisão arquitetural crítica do plano (violações de DEC, camadas, SOLID, lifecycle de `Deflater`/`Inflater`, Virtual Threads, compatibilidade multi-versão) executada antes da implementação — nenhuma violação encontrada; três refinamentos de implementação incorporados (nome `ativarCompressao`, `dataLength` como dica em vez de validação rígida, instância nova de `Deflater`/`Inflater` por chamada). Registrado como observação para o futuro (não pendência formal): se a criptografia também precisar de um método de ativação análogo em `ConexaoMinecraft`, será o terceiro caso do mesmo padrão — ponto de reconsiderar um mecanismo mais genérico em vez de repetir.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 107 testes executados (97 pré-existentes + 10 novos), 2 skipped (`HandshakeServidorRealTest` e `LoginServidorRealTest`, deliberadamente), 0 falhas.

### Incremento 8B — Integração da Compressão ao Fluxo de LOGIN

Status

Concluído

Objetivo

Integrar a fundação de compressão construída no Incremento 8A (DEC-18) ao fluxo real de LOGIN em `AdaptadorConexaoBotV1_8`, para que `EventoSetCompression` deixe de ser tratado como falha e passe a ativar a compressão da conexão, permitindo que o login prossiga normalmente até `LoginSuccess` (ou falhe explicitamente em `EncryptionRequest`, que continua fora de escopo).

Entregue

- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8.reagirAoPacote`: o ramo de `EventoSetCompression` deixa de completar `loginConfirmado` com falha — passa a extrair o `threshold` (pattern matching `instanceof`) e chamar `conexao.ativarCompressao(threshold)`, deixando a future em aberto para continuar aguardando o próximo pacote (LoginSuccess ou EncryptionRequest). Nenhuma outra linha do método foi alterada.
- Nenhum contrato tocado nesta etapa: `ConexaoMinecraft`, `ConexaoBotPort`, `TransporteSocket`, `CodificadorDeFrame`/`DecodificadorDeFrame`, `RegistroDePacotes`, `ProtocolDispatcher`, `PacketHandler`s, Codecs, Packets e Eventos permanecem exatamente como o Incremento 8A os deixou — reutilizados, não modificados. Sem DEC nova (mudança de comportamento interno de um adapter já aprovado, não um contrato).
- 2 testes novos em `AdaptadorConexaoBotV1_8Test` (servidor fake via `ServerSocket` loopback, mesmo padrão dos testes existentes): `deveAtivarCompressaoEConcluirLoginAoReceberLoginSuccessComprimido` (SetCompression → LoginSuccess comprimido → `connect()` retorna sessão `CONNECTED`, `EstadoConexao.PLAY`, `compressionThreshold` da sessão de rede confere) e `deveFalharExplicitamenteQuandoEncryptionRequestChegaComprimidoAposSetCompression` (SetCompression → EncryptionRequest comprimido → falha explícita idêntica à já existente, com `EstadoConexao` permanecendo em LOGIN) — provando que a ativação de compressão e a rejeição de criptografia continuam corretamente desacopladas mesmo com o canal comprimido. Helpers de teste (`responderComLoginSuccess`, `responderComEncryptionRequest`) ganharam sobrecargas aceitando `compressionThreshold` (reutilizando a sobrecarga de 3 argumentos de `CodificadorDeFrame.encode` do Incremento 8A); novo helper `responderComSetCompression` envia o pacote em si sem compressão, replicando a mesma regra de ovo-e-galinha do protocolo já documentada na DEC-18.

Observação de escopo

`TransporteSocket` continua completamente agnóstico de protocolo (nenhum novo import de tipo v1.8). `PacketHandler`s continuam apenas traduzindo Packet → Evento — nenhuma lógica de negócio foi adicionada a `SetCompressionHandler` ou a qualquer outro handler. `CasoDeUsoConectarBot` permanece sem nenhum conhecimento de protocolo/compressão — a decisão de quando ativar compressão continua isolada no adapter v1.8, exatamente como `avancarEstado` desde a DEC-17. Não implementados: criptografia AES-CFB8 real, Session Server/Mojang Authentication, Play State, Status State, KeepAlive.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 109 testes executados (107 pré-existentes + 2 novos), 2 skipped (`HandshakeServidorRealTest` e `LoginServidorRealTest`, deliberadamente), 0 falhas.

### Incremento 8C — Validação da Compressão contra Servidor Minecraft Real

Status

Concluído

Objetivo

Validar manualmente, uma única vez, o caminho oficial completo (`CasoDeUsoConectarBot → ConexaoBotPort → AdaptadorConexaoBotV1_8 → TransporteSocket`, sem atalhos) contra o servidor Minecraft 1.8 real Olimpo/Craftlandia, fechando o ciclo aberto no Incremento 7B (servidor respondia com `SetCompressionPacket`, então não suportado).

Entregue

- `infrastructure.network.v1_8.LoginComCompressaoServidorRealTest` (novo, `@Disabled`, mesmo padrão de `HandshakeServidorRealTest`/`LoginServidorRealTest`): constrói um `Bot`, um `CasoDeUsoConectarBot` e um `AdaptadorConexaoBotV1_8` reais, com a fábrica de conexão real (`FabricaDeConexaoMinecraftV1_8`) envolvida apenas por um decorator transparente (`ConexaoObservavel`, classe privada do teste, implementa `ConexaoMinecraft` delegando 100% do comportamento) que loga cada evento observável (`send`, pacote recebido, `ativarCompressao`) sem alterar nada — nenhuma camada do caminho oficial foi pulada ou substituída por um fake.
- **Executado manualmente uma única vez em 2026-07-16** contra `olimpo.clmc.com.br:3737` (a anotação `@Disabled` foi temporariamente suspensa só para essa execução isolada via `mvn test -Dtest=LoginComCompressaoServidorRealTest`, e restaurada imediatamente depois com o resultado real incorporado ao texto da anotação). Resultado observado, na ordem:
  1. Conexão TCP estabelecida com sucesso.
  2. `HandshakePacket` enviado.
  3. `LoginStartPacket` enviado.
  4. `SetCompressionPacket` recebido (o mesmo comportamento observado no Incremento 7B).
  5. Compressão ativada — `threshold=1024`.
  6. Próximo pacote recebido: **`LoginSuccessPacket`** (já no formato comprimido, decodificado corretamente pela infraestrutura do Incremento 8A) — não `EncryptionRequestPacket`.
  7. `connect()` concluiu com sucesso: `SessaoBot[state=CONNECTED, autoReconnect=false]`.
- **Achado que resolve a ambiguidade dos Incrementos 7B/7C**: o servidor Olimpo/Craftlandia opera em **modo OFFLINE** (não exigiu autenticação Mojang/Encryption Request) com `compressionThreshold=1024`. Esta é a **primeira conclusão bem-sucedida de login ponta a ponta contra um servidor Minecraft real** neste projeto — `CasoDeUsoConectarBot` retornou uma sessão `CONNECTED` de verdade, pela primeira vez, através do caminho oficial completo.

Observação de escopo

Não implementados nesta etapa: AES-CFB8, autenticação Mojang, Play State, KeepAlive, Chat, Entity, compressão adicional, `disconnect()`. Nenhum contrato foi alterado — sem DEC nova. O achado de modo OFFLINE resolve o bloqueio *deste* servidor específico, mas não elimina a necessidade de suporte a modo online (Encryption Request) para compatibilidade geral com outros servidores Minecraft 1.8 — permanece uma decisão em aberto para a próxima milestone.

Validação executada

Execução manual isolada (fora de `mvn clean test`): 1 teste executado, 0 falhas, resultado real documentado acima. Em seguida, com `@Disabled` restaurado: `mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 110 testes executados (109 pré-existentes + 1 novo, permanentemente `@Disabled`), 3 skipped (`HandshakeServidorRealTest`, `LoginComCompressaoServidorRealTest`, `LoginServidorRealTest`, deliberadamente), 0 falhas.

### Encerramento da Milestone 4

Status final

**CONCLUÍDA**

Objetivos atingidos

- Arquitetura completa da camada de comunicação Minecraft 1.8 (Packet/Codec/Registro/Dispatcher/Handler/Evento), em camadas (DEC-12) e hexagonal, com Ports estáveis desde a DEC-13 (`ConexaoBotPort`/`ConexaoMinecraft` não quebraram nenhuma assinatura em 12 incrementos).
- Transporte TCP real (`java.net.Socket` + Virtual Threads, DEC-03), com framing VarInt, compressão zlib e robustez contra falhas de decodificação (Incremento 7C).
- Fluxo completo de LOGIN — Handshake → LoginStart → SetCompression → LoginSuccess — através do caminho oficial `CasoDeUsoConectarBot → ConexaoBotPort → AdaptadorConexaoBotV1_8 → TransporteSocket`, sem atalhos.
- Validação real contra um servidor Minecraft 1.8 de terceiros (Olimpo/Craftlandia): Handshake (7A), fluxo de LOGIN completo com detecção de SetCompression (7B) e login concluído com sucesso após a compressão ser tratada (8C).
- Auditoria arquitetural completa sem bloqueadores: 100% de aderência às DEC-01–DEC-18, Clean/Hexagonal e SOLID confirmados por leitura direta de todo o código de `domain`, `application` e `infrastructure`.

Critérios de aceite

- [x] Bot conecta com sucesso no servidor Minecraft 1.8 alvo — validado contra servidor real (Incremento 8C, modo OFFLINE, `SessaoBot CONNECTED`).
- [x] Arquitetura em camadas (Clean + Hexagonal, DEC-12) preservada em todos os 12 incrementos.
- [x] Nenhuma DEC congelada violada; nenhuma interface já aprovada quebrada sem justificativa registrada.
- [x] `TransporteSocket` permanece agnóstico de protocolo; `PacketHandler`s restritos a tradução Packet→Evento, sem lógica de negócio.
- [x] Suíte de testes automatizados sem falhas ao final da milestone.
- [ ] Autenticação Mojang/modo online, Play State, reconexão automática, múltiplos bots simultâneos, proxy, `disconnect()` — deliberadamente fora do escopo desta milestone (ver "Base pronta para a Milestone 5" e pendências abaixo).

Resumo das decisões arquiteturais da milestone (DEC-13 a DEC-18)

- **DEC-13** — arquitetura da camada de comunicação: `Packet`/`Codec`/`LeitorDePacote`/`EscritorDePacote`/`PacketHandler`/`EventoDeProtocolo` no domínio; `RegistroDePacotes`/`ProtocolDispatcher` na infraestrutura; sem Netty.
- **DEC-14** — extensão aditiva de `LeitorDePacote`/`EscritorDePacote` com unsigned short (porta do servidor, `ushort` no protocolo).
- **DEC-15** — extensão aditiva com byte array de comprimento explícito (campos criptográficos crus do Encryption Request/Response).
- **DEC-16** — `SentidoDoPacote` (CLIENTBOUND/SERVERBOUND): única DEC da milestone que alterou uma assinatura já aprovada, por necessidade real (colisão de id `0x01` no estado LOGIN).
- **DEC-17** — `ConexaoMinecraft.avancarEstado` (extensão aditiva) e `ConexaoBotPort.connect()` síncrono (divergência deliberada e documentada do modelo assíncrono do C#).
- **DEC-18** — `ConexaoMinecraft.ativarCompressao` (extensão aditiva) para a compressão zlib.

Das 6 DECs da milestone, 5 foram puramente aditivas (nenhum teste ou assinatura existente quebrado); apenas a DEC-16 exigiu alterar uma assinatura já aprovada, e por uma causa raiz que não tinha solução aditiva.

Principais riscos remanescentes (nenhum bloqueia o início da Milestone 5)

- Suporte a modo ONLINE (Encryption Request/AES-CFB8) não implementado nem validado contra servidor real — o único servidor de teste disponível (Olimpo/Craftlandia) opera em modo OFFLINE.
- Nenhum mecanismo definido para pacotes recebidos depois que o LOGIN termina — pré-requisito de design para Play State, não uma falha do que já existe.
- `ConexaoMinecraft` acumulando métodos de ativação por propriedade de sessão (`avancarEstado`, `ativarCompressao`); reconsiderar um mecanismo mais genérico se um terceiro caso análogo aparecer (ex.: ativação de criptografia).
- `SessaoDeRede.cifrada`/`comCifra()` existem desde o Incremento 1 sem nenhum chamador e provavelmente não são a forma certa para o estado real de um `Cipher` (que é mutável e vivo, não um valor imutável de sessão).
- `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` não integrados; uma conexão bem-sucedida não é rastreada por Bot em lugar nenhum.
- Nenhum wiring Spring/DI para a fábrica de conexão em produção (`infrastructure.config` permanece vazio).

Estatísticas finais da Milestone 4

- **Incrementos concluídos:** 12 (1, 2, 3, 4, 5, 6, 7A, 7B, 7C, 8A, 8B, 8C).
- **DECs criadas:** 6 (DEC-13 a DEC-18).
- **Testes automatizados ao final da milestone:** 110 (0 falhas, 3 skipped deliberadamente).
- **Testes adicionados durante a milestone:** 108 (de 2 pré-existentes ao final da Milestone 3 para 110).
- **Validações manuais contra servidor Minecraft real:** 3 (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`), todas `@Disabled` por design, uma execução manual real registrada em cada.
- **Primeira conexão ponta a ponta bem-sucedida contra servidor real:** Incremento 8C — modo OFFLINE, `compressionThreshold=1024`, `SessaoBot CONNECTED`.

### Lições Aprendidas

Decisões que se mostraram corretas ao longo da implementação:

- **Packet/Codec simétricos** (DEC-13): manter `encode`/`decode` no mesmo `Codec<T>`, sem I/O, permitiu testar cada packet isoladamente por round-trip antes de qualquer Socket existir, e absorveu a compressão (Incremento 8A) sem que um único Codec precisasse mudar.
- **Registro por EstadoConexao + SentidoDoPacote** (DEC-16): a colisão real do id `0x01` (EncryptionRequest/EncryptionResponse) só apareceu ao implementar o registro — corrigir a chave de busca em vez de mascarar um dos dois Codecs foi a decisão certa, e o mesmo tipo de colisão não voltou a aparecer em nenhum incremento posterior.
- **Virtual Threads por conexão** (DEC-03): uma Virtual Thread dedicada por `TransporteSocket.readLoop()` manteve o modelo de concorrência simples — sem pool de threads, sem callbacks encadeados como no C# — durante toda a migração do transporte, sem nenhuma race condition nova introduzida.
- **TransporteSocket desacoplado do protocolo**: manter `TransporteSocket` sem nenhum import de tipo `v1_8`, mesmo depois de Handshake, Login, compressão e hardening, significou que a compressão zlib (8A) e a robustez contra falhas de decodificação (7C) puderam ser implementadas e testadas isoladamente, sem tocar em nenhum Packet ou Codec concreto.
- **Extensões aditivas antes de reversões** (DEC-14, DEC-15, DEC-17, DEC-18): sempre que um gap de contrato apareceu, a extensão aditiva resolveu sem quebrar nenhum teste existente; só a DEC-16 exigiu alterar uma assinatura já aprovada, e apenas porque a causa raiz não tinha outra solução correta.
- **Validações contínuas contra servidor real** (Incrementos 7A, 7B, 8C): cada validação manual contra Olimpo/Craftlandia revelou algo que nenhum teste local previa — resposta a Handshake isolado, SetCompressionPacket antes de LoginSuccess, modo OFFLINE do servidor. Sem essas validações, a compressão teria sido implementada especulativamente, sem confirmação real de que resolvia o bloqueio.
- **Evolução incremental pequena e revisável** (7A/7B/7C, 8A/8B/8C): separar "fundação", "integração" e "validação real" em incrementos distintos, cada um com sua própria bateria de testes, permitiu revisar e aprovar cada etapa isoladamente, sem acumular risco.
- **Planejamento e revisão arquitetural antes de codificar** (estudo do C# como fonte da verdade antes do Incremento 8A, revisão crítica antes de implementar): corrigiu uma escolha de design (validação rígida de `dataLength`) antes que virasse código.

### Base Pronta para a Milestone 5

Componentes reutilizáveis integralmente, sem nenhuma alteração, por qualquer frente futura (criptografia, Play State, múltiplas versões de protocolo):

- `domain.protocol` completo (`Packet`, `Codec<T>`, `PacketHandler<T>`, `EventoDeProtocolo`, `LeitorDePacote`, `EscritorDePacote`, `EstadoConexao`, `VersaoProtocolo`, `SentidoDoPacote`) — contratos puros, sem I/O.
- `infrastructure.protocol.RegistroDePacotes` (interface) e o padrão de implementação por versão (`RegistroDePacotesV1_8`) — uma nova versão de protocolo só precisa de uma nova implementação da mesma interface.
- `infrastructure.protocol.ProtocolDispatcher` — roteamento por `Class<? extends Packet>`; novos packets só adicionam entradas ao mapa de handlers, sem mudança estrutural.
- `infrastructure.protocol.CodificadorDeFrame`/`DecodificadorDeFrame` — framing e compressão zlib comprovadamente agnósticos de versão (o próprio C# compartilha esse framing entre 1.7/1.8/1.9); Play State usará exatamente os mesmos métodos.
- `infrastructure.network.TransporteSocket` — transporte TCP + Virtual Threads + compressão, testado e validado contra servidor real; qualquer novo Packet/Codec/Handler passa por ele sem exigir alteração.
- `application.port.ConexaoBotPort` e `application.usecase.CasoDeUsoConectarBot` — não mudaram uma linha desde a DEC-13/DEC-17 apesar de cinco incrementos de complexidade nova por baixo; devem continuar estáveis para Play State e criptografia.
- `domain.network.ConexaoMinecraft` — reaproveitado integralmente; só deve ganhar métodos aditivos novos (como nas DEC-17/DEC-18) se a próxima frente realmente precisar, com o cuidado adicional já registrado de avaliar um mecanismo mais genérico caso um terceiro método do mesmo tipo seja necessário.
- Os 6 Packets/Codecs/Handlers/Eventos do estado LOGIN v1.8 — completos, testados, não precisam de nenhuma mudança para Play State, que introduzirá seus próprios seguindo o mesmo padrão.
- Padrão de validação manual `@Disabled` contra servidor real (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`) — mesmo formato deve validar Play State ou criptografia contra o mesmo servidor.

Não reaproveitável sem desenho novo: `SessaoDeRede.cifrada`/`comCifra()` (forma provavelmente inadequada para o estado real de um `Cipher`) e o mecanismo de reação a pacotes em `AdaptadorConexaoBotV1_8` (hoje modelado só para o terminal do LOGIN, sem rota para pacotes pós-PLAY).

---

## Milestone 5

Status

Em andamento — Incrementos 1 a 5 concluídos (retenção de sessão/roteamento, Keep Alive/Join Game/Player Position And Look/Disconnect, Chat Message, Update Health/Respawn, Inventário do Jogador); Incrementos 6.1 a 6.4 concluídos (ciclo de vida completo de entidades: fundação, movimentação, velocidade e estado visual — equipamento, animação, status e efeitos)

Objetivo

Play State — implementar o estado PLAY do protocolo Minecraft 1.8 (mundo, entidades, jogador, inventário, chat) sobre a base de comunicação entregue na Milestone 4. Escolhida pelo responsável do projeto entre as três frentes candidatas registradas ao encerrar a Milestone 4 (ver Seção 10).

### Fase de Planejamento — Desenho Arquitetural do Play State

Status

Concluída

Objetivo

Produzir o desenho arquitetural completo do Play State antes de qualquer implementação — análise do domínio existente, das DECs vigentes, da arquitetura construída até a Milestone 4, e do legado C# (`AdvancedBot.Client.MinecraftClient.cs`, `AdvancedBot.Client.Handler.Handler_v18.cs`, `ProtocolHandler.cs`) exclusivamente como fonte de regras de negócio, nunca como referência arquitetural — e formalizar como DECs as decisões arquiteturais identificadas. Nenhum código, classe ou teste foi criado nesta fase, conforme instrução explícita do responsável do projeto.

Resultado

Aprovado (ver [DEC-19 e DEC-20](01-Decisoes-Arquiteturais.md))

Entregue

- Documento de arquitetura e roadmap de incrementos da Milestone 5, apresentado ao responsável do projeto como material de discussão e aprovado. Cobriu: o que ocorre arquiteturalmente após `EstadoConexao` atingir `PLAY`; responsabilidade pelo consumo de `EventoDeProtocolo`; integração `ProtocolDispatcher` → Eventos → Domínio → `Bot` → Casos de Uso; bounded contexts do estado PLAY (Mundo, Entidade, Jogador, Inventário, Chat — Movimento/Combate/Containers tratados como capacidades/serviços de aplicação, não contextos próprios); componentes 100% reutilizáveis sem alteração; novos Ports e Eventos necessários; primeiros pacotes PLAY recomendados e ordem de implementação (Keep Alive → Join Game → Player Position And Look → Chat → Disconnect → Update Health/Respawn → Chunk/Block → Entidades → Inventário); roadmap de 10 incrementos pequenos e independentes.
- **DEC-19** — Retenção da Sessão de Jogo e Roteamento de Eventos no Estado PLAY: novo agregado `SessaoDeJogo` (`domain.bot`), retendo a conexão viva após o login; `ReceptorDeEvento<T>` (`domain.protocol`) e `RoteadorDeEventos` (`infrastructure.protocol`) formalizando o roteamento evento→domínio deferido desde a DEC-13; `ConexaoBotPort.connect()` passa a retornar `SessaoDeJogo` em vez de `SessaoBot` — única mudança não aditiva desta DEC, no mesmo espírito da DEC-16.
- **DEC-20** — Política de Tolerância a Pacotes PLAY Não Registrados: HANDSHAKING/STATUS/LOGIN permanecem estritos (comportamento inalterado desde o Incremento 7C); em PLAY, um pacote sem Codec registrado passa a ser descartado com log em nível WARN em vez de encerrar a conexão, distinguido via a nova `PacoteNaoRegistradoException` (`infrastructure.protocol`, subtipo de `IllegalArgumentException`).

Observação de escopo

Exclusivamente documental — nenhuma classe, teste ou linha de código de produção foi criada ou alterada nesta fase. `SessaoDeJogo`, `ReceptorDeEvento`, `RoteadorDeEventos` e `PacoteNaoRegistradoException` existem, neste momento, apenas como decisão registrada em DEC-19/DEC-20; sua implementação é o Incremento 1 do roadmap aprovado. Nenhum commit foi realizado como parte desta fase.

Validação executada

Não aplicável a código (fase exclusivamente documental, sem `mvn clean test`). Validação da fase: revisão e aprovação do desenho arquitetural e das duas DECs pelo responsável do projeto, conforme processo do CLAUDE.md.

Próximo passo autorizado (histórico)

Incremento 1 — retenção da Sessão de Jogo e ciclo de vida da conexão após LOGIN, conforme DEC-19 — foi autorizado a iniciar (ver Incrementos 1 a 3 abaixo).

### Incremento 1 — Retenção da Sessão de Jogo e Roteamento de Eventos (DEC-19)

Status

Concluído

Objetivo

Implementar exatamente o que a DEC-19 formalizou: `SessaoDeJogo` (`domain.bot`) retendo a `ConexaoMinecraft` viva após o LOGIN; `ReceptorDeEvento<T>` (`domain.protocol`) e `RoteadorDeEventos` (`infrastructure.protocol`) formalizando o roteamento evento→domínio; `ConexaoBotPort.connect()` passando a retornar `SessaoDeJogo` em vez de `SessaoBot`.

Entregue

- `domain.bot.SessaoDeJogo`: novo agregado, retendo `SessaoBot` + `ConexaoMinecraft`; primeiros métodos de intenção (`responderKeepAlive`, `encerrarPorDesconexaoDoServidor`, `registrarEntradaNoJogo`, `atualizarPosicao`), cada um introduzido junto do incremento que o consumiu (ver Incremento 2).
- `domain.protocol.ReceptorDeEvento<T extends EventoDeProtocolo>` e `infrastructure.protocol.RoteadorDeEventos` (mapa `Class<? extends EventoDeProtocolo> → ReceptorDeEvento`, mesmo estilo de `ProtocolDispatcher`).
- `application.port.ConexaoBotPort.connect(...)` alterado para retornar `SessaoDeJogo` (única mudança não aditiva da DEC-19); `application.usecase.CasoDeUsoConectarBot` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` ajustados de acordo — `AdaptadorConexaoBotV1_8` passa a construir `SessaoDeJogo` ao confirmar `LoginSuccess` e a montar o `RoteadorDeEventos` com os receptores disponíveis.
- `domain.bot.Bot`: campo `sessaoDeJogo` e método `iniciarSessaoDeJogo(SessaoDeJogo)`.

Observação de escopo

Nenhum Packet/Handler/Evento novo de PLAY foi introduzido nesta etapa — exclusivamente a infraestrutura de retenção e roteamento definida pela DEC-19.

Validação executada

`mvn clean test` sem regressões (suíte herdada da Milestone 4 mantida verde).

### Incremento 2 — Primeiros Pacotes do Estado PLAY (Keep Alive, Join Game, Player Position And Look, Disconnect)

Status

Concluído

Objetivo

Implementar os primeiros pacotes PLAY recomendados pelo roadmap da fase de planejamento, validando o roteamento evento→domínio definido pelo Incremento 1 com casos reais, seguindo a política de tolerância da DEC-20 para pacotes PLAY ainda não cobertos.

Entregue

- `domain.protocol.v1_8`: `KeepAlivePacket`/`KeepAliveCodec`/`KeepAliveHandler` (CLIENTBOUND, id `0x00`) e `RespostaKeepAlivePacket`/`RespostaKeepAliveCodec`/`RespostaKeepAliveHandler` (SERVERBOUND, id `0x00` — mesma colisão de id por direção já resolvida pela DEC-16); `JoinGamePacket`/`JoinGameCodec`/`JoinGameHandler` (CLIENTBOUND, id `0x01`); `PlayerPositionAndLookPacket`/`PlayerPositionAndLookCodec`/`PlayerPositionAndLookHandler` (CLIENTBOUND, id `0x08`) e `ConfirmacaoDePosicaoPacket`/`ConfirmacaoDePosicaoCodec`/`ConfirmacaoDePosicaoHandler` (SERVERBOUND, id `0x06` — confirmação obrigatória do protocolo); `DisconnectPlayPacket`/`DisconnectPlayCodec`/`DisconnectPlayHandler` (CLIENTBOUND, id `0x40`) — `motivo` mantido como o JSON cru do ChatComponent, sem parsing para texto formatado.
- `domain.protocol.v1_8`: Eventos correspondentes (`EventoKeepAlive`, `EventoRespostaKeepAlive`, `EventoJoinGame`, `EventoPlayerPositionAndLook`, `EventoConfirmacaoDePosicao`, `EventoDisconnectPlay`) e Receptores (`ReceptorKeepAlive`, `ReceptorJoinGame`, `ReceptorPlayerPositionAndLook`, `ReceptorDisconnectPlay`), cada um traduzindo exatamente um Evento em exatamente uma chamada sobre `SessaoDeJogo`.
- `domain.bot.SessaoDeJogo` cresce de forma aditiva, incremento a incremento: `responderKeepAlive` (ecoa o id automaticamente), `registrarEntradaNoJogo` (entityId/gamemode/dimension), `atualizarPosicao` (resolve flags relativas do protocolo e ecoa `ConfirmacaoDePosicaoPacket`), `encerrarPorDesconexaoDoServidor` (fecha a conexão e registra o motivo).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` (mapas `handlersV1_8()`/`receptoresV1_8()`) atualizados com todos os pacotes acima.

Observação de escopo

Nenhum Port novo, nenhuma interface já aprovada alterada. Comandos, macros, automações, inventário, entidades, combate e movimentação continuam fora de escopo.

Validação executada

`mvn clean test` sem regressões, incluindo cenário de integração real via `AdaptadorConexaoBotV1_8Test` (Keep Alive respondido automaticamente, Join Game isolado corretamente entre duas conexões do mesmo adaptador).

### Incremento 3 — Chat Recebido do Servidor (Chat Message)

Status

Concluído

Objetivo

Implementar o primeiro pacote PLAY de conteúdo (não apenas plumbing de protocolo): o Chat Message recebido do servidor, percorrendo toda a arquitetura já validada pelos Incrementos 1 e 2 (`Servidor → TransporteSocket → Codec → Handler → Evento → Receptor → SessaoDeJogo → Bot`). C# consultado como fonte de regras de negócio (`AdvancedBot.Client.Handler.Handler_v18.cs`, case `2`; `AdvancedBot.Client.Packets.PacketChatMessage.cs`; `AdvancedBot.Client.ChatParser.cs`; `AdvancedBot.Client.MinecraftClient.cs`, `HandlePacketChat`), sem migração de código.

Análise do legado

O Chat Message clientbound (id `0x02` em `Handler_v18.HandlePacket`, `case 2`) carrega dois campos: uma string JSON representando um `ChatComponent` do protocolo, e um byte de posição (0 = chat box, 1 = system message, 2 = above hotbar). O C# decodifica o JSON para texto formatado (`ChatParser.ParseText`, incluindo cores/estilos via `§`, resolução de `translate`/`chat.type.*` e até download de arquivos de idioma do Mojang) antes de repassar a `HandlePacketChat`, que por sua vez despacha para plugins e contém regras de negócio específicas do servidor Craftlandia (detecção de VIP, fluxo de login automático via `/vip`/AuthMe) — inteiramente fora do escopo autorizado (comandos, automações, parsing de texto de exibição).

Decisão de escopo (não é uma DEC — não altera nenhum Port/contrato)

O campo de texto é mantido como o JSON cru do `ChatComponent`, exatamente como o precedente já aprovado e testado em `DisconnectPlayPacket.motivo` (Incremento 2) — nenhum parser de `ChatComponent` (cores, `translate`, extras) foi reconstruído nesta etapa. Interpretação/formatação de texto de exibição fica para quando houver necessidade funcional real (ex.: UI, comandos), consultando o C# (`ChatParser.cs`) como fonte de verdade quando esse incremento futuro for autorizado.

Determinação: nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova

Analisado explicitamente antes de implementar, conforme processo do CLAUDE.md:

- **Caso de Uso**: nenhum criado. A DEC-19 define a responsabilidade do `ReceptorDeEvento` como "traduzir exatamente um `EventoDeProtocolo` em exatamente uma chamada sobre exatamente um agregado de domínio" — um Caso de Uso é um tipo da camada de aplicação, e o Receptor vive em `domain.protocol.v1_8` (camada de domínio); uma chamada de um Receptor a um Caso de Uso inverteria a direção de dependência Clean/Hexagonal (domínio não pode depender de aplicação) e contradiria o contrato já congelado do Receptor. Os quatro pacotes do Incremento 2 (Keep Alive, Join Game, Player Position And Look, Disconnect) já estabeleceram o mesmo tratamento — Receptor chamando `SessaoDeJogo` diretamente, sem Caso de Uso — e o Chat Message segue exatamente o mesmo padrão, sem assimetria injustificada.
- **Port**: nenhum novo. `ConexaoMinecraft`, `ConexaoBotPort` e `SessaoDeJogo` já existentes cobrem integralmente a necessidade.
- **DEC**: nenhuma nova. Mudança 100% aditiva em todas as camadas — nenhuma interface já aprovada (`Codec`, `PacketHandler`, `ReceptorDeEvento`, `RegistroDePacotes`, `EventoDeProtocolo`) foi alterada; o crescimento de `SessaoDeJogo` (dois campos + um método novos) segue exatamente o padrão já pré-autorizado pela DEC-19 ("cada incremento futuro deve adicionar apenas a referência que efetivamente precisa").

Entregue

- `domain.protocol.v1_8.ChatMessagePacket(String mensagem, byte posicao)`: Record imutável, valida `mensagem` não nula (mesmo padrão de `DisconnectPlayPacket`).
- `domain.protocol.v1_8.ChatMessageCodec`: `decode`/`encode` simétricos, apenas `readString`/`writeString` seguido de `readByte`/`writeByte`, sem lógica de negócio.
- `domain.protocol.v1_8.ChatMessageHandler`: traduz `ChatMessagePacket` em `EventoChatMessage`, sem lógica de negócio.
- `domain.protocol.v1_8.EventoChatMessage(String mensagem, byte posicao)`: espelha os campos do Packet 1:1, mesmo padrão dos demais Eventos da Milestone 5.
- `domain.protocol.v1_8.ReceptorChatMessage`: traduz `EventoChatMessage` em uma única chamada, `sessaoDeJogo.registrarMensagemDeChat(mensagem, posicao)`.
- `domain.bot.SessaoDeJogo`: dois novos campos (`ultimaMensagemDeChat`, `ultimaPosicaoDeChat`) e o método `registrarMensagemDeChat` — mesmo padrão de "último valor conhecido" já usado por `registrarEntradaNoJogo`/`atualizarPosicao`, resolvendo a questão de que o Chat deve atualizar estado interno observável em vez de ser apenas propagado e descartado (sem essa retenção, não haveria como a mensagem "chegar até o Bot" de forma observável, já que Chat — ao contrário de Keep Alive/Player Position And Look — não gera nenhuma resposta automática de protocolo).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8`: `ChatMessagePacket` registrado em `EstadoConexao.PLAY`, id `0x02`, `SentidoDoPacote.CLIENTBOUND`.
- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8`: `ChatMessagePacket`/`ChatMessageHandler` adicionados a `handlersV1_8()`; `EventoChatMessage`/`ReceptorChatMessage` adicionados a `receptoresV1_8()` — fecha o fluxo completo `Servidor → TransporteSocket → Codec → Handler → Evento → Receptor → SessaoDeJogo → Bot`.
- 10 testes novos (JUnit 5 + AssertJ, sem mocks): 4 em `ChatMessageCodecTest` (encode com ordem de campos do protocolo, decode, round-trip, validação de mensagem nula), 1 em `ChatMessageHandlerTest`, 1 em `ReceptorChatMessageTest`, 1 em `SessaoDeJogoTest` (`deveRegistrarMensagemDeChat`), 1 em `RegistroDePacotesV1_8Test` (localização por estado/id/direção, mais asserção de id no teste já existente de localização por tipo), 1 em `PipelineDeProtocoloV1_8Test` (pipeline completa: encode → frame → decode → dispatch → Evento), 1 em `AdaptadorConexaoBotV1_8Test` (`deveRegistrarMensagemDeChatNaSessaoDeJogoAoReceberChatMessage` — cenário de integração real ponta a ponta: servidor fake → `TransporteSocket` → `RegistroDePacotesV1_8` → `ProtocolDispatcher` → `RoteadorDeEventos` → `ReceptorChatMessage` → `SessaoDeJogo`).

Restrições respeitadas

Não foram implementados: comandos do bot, parser de comandos, automações, inventário, entidades, combate, movimentação — nem o parser de `ChatComponent` do C# (`ChatParser.cs`), deliberadamente adiado (ver "Decisão de escopo" acima).

Riscos e observações

- O texto do chat permanece como JSON cru até que um incremento futuro precise de texto formatado — consumidores atuais (nenhum ainda) devem estar cientes disso.
- `SessaoDeJogo` mantém apenas a última mensagem recebida (sem histórico) — consistente com o mesmo tratamento dado a posição/entrada no jogo, e com a advertência da DEC-19 contra estado especulativo; um histórico/buffer de mensagens deve ser desenhado apenas quando um consumidor real (ex.: macros) precisar dele.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 184 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`).

Próximo passo sugerido (histórico)

Próximo pacote PLAY recomendado pelo roadmap da fase de planejamento: Update Health/Respawn (validar ambiguidade de id 0x06 vs 0x08 registrada no roadmap) ou seguir para Chunk/Block, a critério do responsável do projeto — nenhuma DEC pendente bloqueia o início (ver Incremento 4 abaixo).

### Incremento 4 — Gerenciamento do Estado de Vida do Jogador (Update Health, Respawn)

Status

Concluído

Objetivo

Implementar exclusivamente o gerenciamento do estado de vida do jogador: vida, fome e saturação (Update Health) e atualização de dimensão/gamemode ao renascer (Respawn). C# consultado como fonte de regras de negócio (`AdvancedBot.Client.Handler.Handler_v18.cs`, cases `6` e `7`; `AdvancedBot.Client.MinecraftClient.cs`, campos `health`/`foodlevel` e `HandlePacketRespawn`), sem replicar a arquitetura do C# (campos soltos em `MinecraftClient`, ausência de um agregado `Player`).

Análise do legado

`Handler_v18.HandlePacket` case `6` (Update Health) só lê e usa o campo `health` (`float`) — não lê `food`/`saturation`, que também existem na formação do pacote pelo protocolo 1.8 (Health: Float, Food: VarInt, Food Saturation: Float); esse é um gap do handler 1.8, não uma regra de negócio deliberada (o campo `foodlevel` existe em `MinecraftClient` mas só é populado pelo handler do protocolo 1.5.2, `Handler_v152.cs`, nunca pelo 1.8). Regra de negócio observada e **não replicada** (automação fora de escopo): ao `health <= 0`, o C# zera a motion do jogador e, após 300ms, envia `PacketClientStatus(0)` solicitando respawn automaticamente. Case `7` (Respawn) lê `Dimension` (Int, 4 bytes — diferente do Byte usado pelo Join Game), descarta `Difficulty` (Byte) e lê `Gamemode` (Byte), sem ler o campo final `Level Type` (String) também previsto pelo protocolo; `HandlePacketRespawn` reatribui `Dimension`/`Gamemode` e limpa `TheWorld`/`PlayerManager`/`ActivePotions` (mundo/entidades/poções — fora de escopo deste incremento).

Determinação de arquitetura (sem nova DEC)

- **Vida/fome/saturação pertencem a `SessaoDeJogo`, não a um novo agregado.** O próprio legado já trata esse estado como campos escalares diretos do "cliente"/sessão (`MinecraftClient.health`/`foodlevel`), não como um agregado `Player` separado — não existe nenhuma classe `Player` com essa responsabilidade no C#. `SessaoDeJogo` já é, desde a DEC-19, o agregado que representa o estado conhecido do jogador durante PLAY (entityId, gamemode, dimensão, posição); vida/fome/saturação são mais três atributos escalares do mesmo conceito, sem identidade, invariantes ou ciclo de vida próprios que justifiquem extração. A DEC-19 já antecipava exatamente esse crescimento ("Jogador" citado como vetor de crescimento natural de `SessaoDeJogo`) e alertava contra estado especulativo — introduzir um agregado `Jogador` agora, sem nenhum comportamento/invariante associado, seria abstração prematura.
- **Nenhum Caso de Uso novo** — mesma determinação já aplicada aos incrementos anteriores (Keep Alive, Join Game, Player Position And Look, Disconnect, Chat): Receptor (domínio) chamando `SessaoDeJogo` (domínio) diretamente, sem atravessar a camada de aplicação.
- **Nenhum Port novo.**
- **Nenhuma DEC nova.** Mudança 100% aditiva; a colisão de id `0x06` entre Update Health (CLIENTBOUND) e Confirmação de Posição (SERVERBOUND) é resolvida pelo mesmo mecanismo já congelado na DEC-16 (`SentidoDoPacote`), sem exigir nenhuma extensão.
- **Ambiguidade de id `0x06` vs `0x08` (registrada como risco na fase de planejamento) resolvida por leitura direta do C#**: Update Health é `0x06` CLIENTBOUND (confirmado por `Handler_v18`, `case 6`), sem colidir com `0x08` (Player Position And Look, já registrado). O nome do teste pré-existente `deveProcessarConfirmacaoDePosicaoAtravesDaPipelineCompletaMesmoComIdColidindoComUpdateHealth` (Incremento 2) já antecipava corretamente essa colisão por direção em `0x06`.
- **Simplificação registrada (não é DEC — não altera nenhum Port/contrato)**: `RespawnPacket.dimension` é `int` (fiel ao formato de fio, 4 bytes); `SessaoDeJogo.dimension` continua `byte` (padrão já aprovado desde Join Game). `ReceptorRespawn` estreita o valor via cast explícito antes de chamar `SessaoDeJogo.registrarRespawn(byte, byte)` — seguro para todo valor real de dimensão em uso (-1/0/1), mas registrado como uma perda de precisão deliberada e localizada no Receptor, não no Codec/Evento (que permanecem fiéis ao protocolo).

Entregue

- `domain.protocol.v1_8.UpdateHealthPacket(float health, int food, float saturation)` + `UpdateHealthCodec` (`readFloat`/`readVarInt`/`readFloat`, sem lógica de negócio) + `UpdateHealthHandler` (traduz para `EventoUpdateHealth`) + `EventoUpdateHealth` (espelha os 3 campos) + `ReceptorUpdateHealth` (chama `sessaoDeJogo.atualizarVida(health, food, saturation)`).
- `domain.protocol.v1_8.RespawnPacket(int dimension, byte difficulty, byte gamemode, String levelType)` + `RespawnCodec` (`readInt`/`readByte`/`readByte`/`readString`) + `RespawnHandler` (traduz para `EventoRespawn`, todos os 4 campos) + `EventoRespawn` + `ReceptorRespawn` (chama `sessaoDeJogo.registrarRespawn((byte) dimension, gamemode)` — `difficulty`/`levelType` conscientemente descartados, fora do escopo funcional "vida, fome e saturação").
- `domain.bot.SessaoDeJogo`: três novos campos (`health`, `food`, `saturation`) + método `atualizarVida(float, int, float)`; método `registrarRespawn(byte dimension, byte gamemode)` reaproveitando os campos `dimension`/`gamemode` já existentes desde Join Game (sem novos campos — Respawn semanticamente "reentra" no jogo em nova dimensão/gamemode, exatamente como o C# reatribui os mesmos campos de `MinecraftClient` em vez de criar novos).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8`: `UpdateHealthPacket` registrado em `EstadoConexao.PLAY`, id `0x06`, `SentidoDoPacote.CLIENTBOUND` (convivendo com `ConfirmacaoDePosicaoPacket` no mesmo id, SERVERBOUND); `RespawnPacket` registrado em `EstadoConexao.PLAY`, id `0x07`, `SentidoDoPacote.CLIENTBOUND`.
- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8`: `UpdateHealthPacket`/`UpdateHealthHandler` e `RespawnPacket`/`RespawnHandler` adicionados a `handlersV1_8()`; `EventoUpdateHealth`/`ReceptorUpdateHealth` e `EventoRespawn`/`ReceptorRespawn` adicionados a `receptoresV1_8()`.
- 22 testes novos (JUnit 5 + AssertJ, sem mocks): 4 em `UpdateHealthCodecTest`, 1 em `UpdateHealthHandlerTest`, 2 em `ReceptorUpdateHealthTest` (atualização normal e vida zerada), 4 em `RespawnCodecTest`, 1 em `RespawnHandlerTest`, 1 em `ReceptorRespawnTest`, 2 em `SessaoDeJogoTest` (`deveAtualizarVidaFomeESaturacao`, `deveRegistrarRespawnAtualizandoDimensaoEGamemode`), 3 em `RegistroDePacotesV1_8Test` (localização de Update Health, distinção de Update Health/Confirmação de Posição no mesmo id, localização de Respawn), 2 em `PipelineDeProtocoloV1_8Test` (pipeline completa de cada packet), 2 em `AdaptadorConexaoBotV1_8Test` (cenários de integração real ponta a ponta: servidor fake → `TransporteSocket` → `RegistroDePacotesV1_8` → `ProtocolDispatcher` → `RoteadorDeEventos` → Receptor → `SessaoDeJogo`).

Restrições respeitadas

Não foram implementados: inventário, entidades, chunks, blocos, IA, parser de comandos, automações — incluindo, deliberadamente, o envio automático de `PacketClientStatus(0)` que o C# dispara ao detectar `health <= 0` (essa é a única peça de `HandlePacketChat`/Update Health do legado que é uma automação, não apenas reflexo de estado). Nenhum outro bounded context (World, Entity, Inventory, Chunk, Block) foi iniciado.

Riscos e observações

- `SessaoDeJogo` não deriva nenhum estado de "morte" (`isDead`/`estaMorto`) — apenas expõe `health()`/`food()`/`saturation()` como valores brutos; qualquer decisão sobre o que fazer com `health <= 0` (inclusive respawn automático) fica para quando automações forem autorizadas.
- `RespawnPacket.dimension` (int, fiel ao protocolo) é estreitado para `byte` em `ReceptorRespawn` antes de chegar a `SessaoDeJogo` — seguro para os valores reais do protocolo 1.8 (-1/0/1), mas é uma simplificação deliberada a ter em mente se uma versão futura do protocolo usar a faixa completa de um Int.
- `difficulty` e `levelType` do Respawn são capturados no Packet/Evento (fidelidade ao protocolo) mas descartados no Receptor — não fazem parte do escopo funcional deste incremento; ficam disponíveis em `EventoRespawn` para quando (se) forem necessários.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 206 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`). Nenhuma interface já aprovada foi alterada (`Codec`, `PacketHandler`, `ReceptorDeEvento`, `RegistroDePacotes`, `EventoDeProtocolo`, `ConexaoBotPort`, `ConexaoMinecraft` permanecem exatamente como antes) — mudança 100% aditiva.

Próximo passo sugerido (histórico)

World/Chunk/Block, Entidades, ou Inventário — a critério do responsável do projeto — ou consolidar Player com XP/Experience (pacote PLAY relacionado ainda não implementado). Nenhuma DEC pendente bloqueia o início (ver Incremento 5 abaixo).

### Incremento 5 — Modelagem do Inventário do Jogador (Window Items, Set Slot, Held Item Change)

Status

Concluído

Objetivo

Iniciar a modelagem do inventário do jogador: refletir o conteúdo completo do inventário (Window Items), atualizar slots individuais (Set Slot) e acompanhar o slot ativo da hotbar (Held Item Change). C# consultado como fonte de regras de negócio (`AdvancedBot.Client.Handler.Handler_v18.cs`, cases `9`, `47`, `48`; `AdvancedBot.Client.Inventory.cs`; `AdvancedBot.Client.ItemStack.cs`; `ReadBuffer.cs`/`WriteBuffer.cs`, `ReadItemStack`/`WriteItemStack`/`ReadOptionalNBT`), sem replicar a arquitetura do C# (campos soltos em `MinecraftClient`, ausência de um agregado de inventário).

Análise do legado

Held Item Change clientbound (case `9`, id `0x09`) é um único byte (slot da hotbar selecionado, 0–8). Set Slot (case `47`, id `0x2F`) e Window Items (case `48`, id `0x30`) carregam a estrutura "Slot" do protocolo (Item ID Short, se ≥0: Count Byte, Damage Short, NBT opcional) — o inventário do próprio jogador é sempre `windowId=0`, fixo em 45 slots (`new Inventory(45, this)` no C#); `windowId≠0` (baús, mesas de trabalho, o resultado de um Open Window nunca implementado) cai nos ramos que o C# só executa quando há uma janela não-jogador rastreada — como esse rastreamento (Open Window, case `45`) está fora do escopo autorizado, esses ramos tornam-se no-op por construção. NBT (encantamentos, nome customizado, lore) é um subsistema inteiro à parte no C# (`AdvancedBot.Client.NBT`, com `CompoundTag`/`ListTag`/etc.) — fora de escopo; porém como o Window Items transporta múltiplos slots em sequência no mesmo pacote, o NBT de um slot com dado extra precisa ser corretamente **consumido** (não necessariamente interpretado), sob risco de desalinhar a leitura dos slots seguintes do mesmo pacote.

Determinação de arquitetura (sem nova DEC, sem parar para propor bounded context)

- **Inventário justifica um agregado dedicado — `domain.bot.InventarioDoJogador`** — mas não um bounded context próprio. Critério aplicado: um bounded context exigiria persistência própria, Ports/Casos de Uso distintos, ou interação real com outros contextos (baú, crafting, trade) — nenhum desses existe neste incremento (todos explicitamente fora de escopo pelas restrições). `InventarioDoJogador` é estado transiente com o mesmo ciclo de vida de `SessaoDeJogo` (nasce e morre com a conexão), sem persistência, Port ou Caso de Uso próprios. A DEC-19 já pré-autorizou exatamente esse tipo de crescimento ("Jogador, Inventário... irão associá-la [SessaoDeJogo]... sem exigir nova DEC"), então a implementação prosseguiu sem interromper para apresentar uma proposta de bounded context (não haveria o que propor).
- **`ItemStack` (domain.protocol.v1_8)**: Record mínimo (`itemId`, `count`, `damage`), sem nome de exibição, encantamentos ou NBT estruturado — nenhum desses é necessário para os objetivos funcionais ("refletir corretamente o conteúdo", "atualizar slots individualmente", "acompanhar o slot selecionado"). Vive em `domain.protocol.v1_8` (não em `domain.bot`) por ser um valor puramente derivado da decodificação do protocolo, sem comportamento — mesmo tratamento dado aos demais campos de Packet; `domain.bot` já depende de tipos `v1_8` desde a DEC-19 (`SessaoDeJogo` construindo `RespostaKeepAlivePacket`/`ConfirmacaoDePosicaoPacket`), então essa dependência não é uma direção nova.
- **NBT**: um parser mínimo, interno ao pacote (`ItemStackCodec`, não público), consome (sem expor) qualquer NBT presente usando exclusivamente primitivas já existentes e aprovadas de `LeitorDePacote` (`readByte`/`readShort`/`readInt`/`readLong`/`readFloat`/`readDouble`/`readUnsignedShort`/`readByteArray`) — nenhum método novo em `LeitorDePacote`/`EscritorDePacote`, nenhuma DEC.
- **Nenhum Caso de Uso novo, nenhum Port novo** — mesmo tratamento já aplicado a todos os incrementos anteriores da Milestone 5: Receptor (domínio) chama `SessaoDeJogo`/`InventarioDoJogador` (domínio) diretamente.
- **Nenhuma DEC nova.** Mudança 100% aditiva; ids `0x09`/`0x2F`/`0x30` (PLAY, CLIENTBOUND) não colidem com nenhum id já registrado.

Entregue

- `domain.protocol.v1_8.ItemStack(short itemId, byte count, short damage)` e `ItemStackCodec` (pacote-privado, compartilhado por Window Items e Set Slot) — decodifica a estrutura "Slot" completa, incluindo consumo seguro de NBT eventual.
- `domain.protocol.v1_8.WindowItemsPacket(byte windowId, ItemStack[] items)` (equals/hashCode via `Arrays`, mesmo padrão de `EncryptionRequestPacket`) + `WindowItemsCodec` + `WindowItemsHandler` + `EventoWindowItems` + `ReceptorWindowItems` (aplica a `InventarioDoJogador` apenas quando `windowId == 0`).
- `domain.protocol.v1_8.SetSlotPacket(byte windowId, short slot, ItemStack item)` + `SetSlotCodec` + `SetSlotHandler` + `EventoSetSlot` + `ReceptorSetSlot` (mesma restrição a `windowId == 0`).
- `domain.protocol.v1_8.HeldItemChangePacket(byte slot)` + `HeldItemChangeCodec` + `HeldItemChangeHandler` + `EventoHeldItemChange` + `ReceptorHeldItemChange`.
- `domain.bot.InventarioDoJogador`: novo agregado, 45 slots fixos (`ItemStack[]`) + slot ativo (byte); `definirConteudo(ItemStack[])` (Window Items), `atualizarSlot(int, ItemStack)` (Set Slot, com bounds-check), `selecionarSlotAtivo(byte)` (Held Item Change). `domain.bot.SessaoDeJogo` ganha o campo final `inventario` (construído eagerly, sempre presente) e o acessor `inventario()`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8`: `HeldItemChangePacket` (`0x09`), `SetSlotPacket` (`0x2F`), `WindowItemsPacket` (`0x30`), todos PLAY/CLIENTBOUND.
- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8`: os três novos Handlers/Receptores adicionados a `handlersV1_8()`/`receptoresV1_8()`.
- 35 testes novos (JUnit 5 + AssertJ, sem mocks): 6 em `InventarioDoJogadorTest`, 4 em `ItemStackCodecTest` (slot vazio, sem NBT, e um teste dedicado provando que um NBT sintético é consumido sem corromper a leitura do slot seguinte no mesmo pacote), 3 em `WindowItemsCodecTest`, 1 em `WindowItemsHandlerTest`, 2 em `ReceptorWindowItemsTest` (windowId 0 e ≠0), 3 em `SetSlotCodecTest`, 1 em `SetSlotHandlerTest`, 2 em `ReceptorSetSlotTest`, 3 em `HeldItemChangeCodecTest`/`HeldItemChangeHandlerTest`, 1 em `ReceptorHeldItemChangeTest`, 3 em `RegistroDePacotesV1_8Test`, 3 em `PipelineDeProtocoloV1_8Test`, 2 em `AdaptadorConexaoBotV1_8Test` (integração real ponta a ponta: Window Items + Set Slot refletidos no inventário; Held Item Change atualizando o slot ativo).

Restrições respeitadas

Não foram implementados: clique em inventário, uso de itens, crafting, interação com baús/janelas não-jogador (Open Window/Close Window/Confirm Transaction — nem sequer os Packets foram criados), entidades, chunks, blocos, automações. `windowId != 0` é tratado como no-op por construção (nenhum estado de "janela aberta" é rastreado).

Riscos e observações

- **NBT é descartado, não preservado.** Itens com encantamentos, nome customizado ou lore são refletidos com `itemId`/`count`/`damage` corretos, mas sem esses metadados — `ItemStackCodec.encode` sempre escreve "sem NBT". Isso deve ser revisitado antes de qualquer funcionalidade que dependa de NBT (exibição de encantamentos, validação de crafting).
- **`InventarioDoJogador.slot(int)` lança `ArrayIndexOutOfBoundsException` para índices fora de 0–44** (leitura não faz bounds-check, ao contrário de `atualizarSlot`) — aceitável neste incremento pois todo índice usado internamente vem de `Set Slot`/`Window Items` já filtrados por `windowId == 0` (0–44 por definição do protocolo), mas qualquer futuro consumidor externo de `slot(int)` deve respeitar esse contrato.
- Determinação explícita de que o inventário **não** exige bounded context próprio neste momento — se um incremento futuro precisar de interação com baús/crafting/trade (que cruzam para outros bounded contexts), essa determinação deve ser reavaliada.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 241 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`). Nenhuma interface já aprovada foi alterada (`Codec`, `PacketHandler`, `ReceptorDeEvento`, `RegistroDePacotes`, `EventoDeProtocolo`, `ConexaoBotPort`, `ConexaoMinecraft`, `LeitorDePacote`, `EscritorDePacote` permanecem exatamente como antes) — mudança 100% aditiva.

Próximo passo sugerido

World/Chunk/Block, Entidades, ou consolidar Player com XP/Experience — a critério do responsável do projeto. Nenhuma DEC pendente bloqueia o início.

### Fase de Planejamento — Incremento 6: Modelagem das Entidades do Mundo

Status

Concluída

Objetivo

Produzir o desenho arquitetural do ciclo de vida de entidades (jogadores remotos e mobs) antes de qualquer implementação — análise completa do legado C# (`AdvancedBot.Client.Entity.cs`, `AdvancedBot.Client.Entitybase` — `IEntity`, `MPPlayer`, `EntityMob`, `EntityManager`, `EntityProperty` —, `AdvancedBot.Client.MinecraftClient.cs`, `AdvancedBot.Client.PlayerManager.cs`, `AdvancedBot.Client.Handler.Handler_v18.cs`), exclusivamente como fonte de regras de negócio, nunca como referência arquitetural. Nenhum código foi escrito nesta fase, conforme instrução explícita do responsável do projeto.

Resultado

Aprovado pelo responsável do projeto, com adição de escopo (Incremento 6.4 — ver abaixo)

Entregue

- Documento de arquitetura apresentado como material de discussão (não commitado a `docs-reescrita`, mesmo tratamento do documento de planejamento da própria Milestone 5). Cobriu: correção estrutural sobre a nomenclatura enganosa do legado (`Entity.cs` é a física do próprio bot, não uma base de entidades remotas; `IEntity`/`MPPlayer`/`EntityMob` é a hierarquia real; `EntityManager` é um catálogo estático de metadados de tipo de mob, não uma coleção viva); mapeamento completo dos pacotes PLAY de ciclo de vida de entidade no `Handler_v18` (implementados com fidelidade parcial: Spawn Player `0x0C`, Spawn Mob `0x0F`, Entity Velocity `0x12`, Destroy Entities `0x13`, Entity Relative Move `0x15`, Entity Look `0x16`, Entity Look And Relative Move `0x17`, Entity Teleport `0x18`, Entity Head Look `0x19`; nunca implementados no C#: Entity Equipment `0x04`, Use Bed `0x0A`, Animation `0x0B`, Collect Item `0x0D`, Spawn Object `0x0E`, Spawn Painting `0x10`, Spawn Experience Orb `0x11`, Entity `0x14`, Entity Status `0x1A`, Attach Entity `0x1B`, Entity Metadata contínua `0x1C`, Entity Properties `0x20`); distinção entre espelhamento puro de estado (100% do `Handler_v18`) e automação real (inteiramente na camada `AdvancedBot.Client.Commands` — `CommandKillAura`, `CommandMob`/`CommandMobPlus`/`CommandMobTeleport` —, nunca portada); modelagem proposta (`domain.bot.EntidadeRemota` sealed, `EntidadeJogadorRemoto`/`EntidadeMob`, `EntidadesDoMundo` como coleção única pendurada em `SessaoDeJogo`, sem novo agregado raiz, sem `EntityManager`, sem novo bounded context — mesmo critério já aplicado ao Inventário no Incremento 5); roteiro em sub-incrementos (6.1 fundação, 6.2 movimentação, 6.3 velocity); conclusão de que nenhuma DEC nova é necessária (mudança 100% aditiva, mesmo padrão pré-autorizado pela DEC-19).
- **Achado de auditoria não-arquitetural, registrado por transparência**: `docs-reescrita/docs/20-Rastreabilidade/06-Mapa-de-Pacotes.md` e `04-Mapa-de-Eventos.md` contêm anotações de automação de combate ("Killaura", "AntiKnockback", risco de banimento) — esses documentos já eram não-autoritativos (a DEC-19 rejeitou a proposta arquitetural do mesmo diretório) e não influenciaram a modelagem proposta; usados apenas como referência cruzada de IDs de pacote, validada contra o C# real.
- **Escopo adicional aprovado pelo responsável do projeto**: Incremento 6.4 — Entity Metadata, Entity Equipment, Entity Effect, Animation, Entity Status, agrupados por natureza semelhante (enriquecimento de entidade já existente, sem interferir em movimentação, independentes do sistema de mundo/chunks). Ressalva registrada: apenas Entity Effect tem qualquer implementação no C# (exclusiva do próprio bot, nunca de outras entidades) — os outros 4 pacotes não existem no legado, então a fonte da verdade para eles passa a ser a especificação do protocolo vanilla, não o C#; o detalhamento campo a campo desse sub-incremento fica para quando ele for iniciado.

Observação de escopo

Exclusivamente documental nesta fase — nenhuma classe, teste ou linha de código de produção foi criada. `EntidadeRemota`, `EntidadeJogadorRemoto`, `EntidadeMob`, `EntidadesDoMundo` existiam, neste momento, apenas como proposta; sua implementação é o Incremento 6.1 (ver abaixo).

Validação executada

Não aplicável a código (fase exclusivamente documental). Validação da fase: revisão e aprovação do desenho arquitetural pelo responsável do projeto, com adição de escopo (Incremento 6.4).

Próximo passo autorizado (histórico)

Incrementos 6.1, 6.2 e 6.3 aprovados sem ressalvas; Incremento 6.4 aprovado com a ressalva de ausência de precedente no C# para 4 dos 5 pacotes. Início pelo Incremento 6.1 (ver abaixo).

### Incremento 6.1 — Fundação do Ciclo de Vida de Entidades (Spawn Player, Spawn Mob, Destroy Entities)

Status

Concluído

Objetivo

Implementar a fundação do rastreamento de entidades remotas (jogadores e mobs) definida na fase de planejamento: `EntidadeRemota`/`EntidadeJogadorRemoto`/`EntidadeMob`/`EntidadesDoMundo`, e os três primeiros pacotes do ciclo de vida (nascimento de duas formas, morte).

Determinação de arquitetura (sem nova DEC)

- **`EntidadeRemota` é uma Entity no sentido DDD (identidade via `entityId`, estado mutável), não um Aggregate Root** — vive dentro de `SessaoDeJogo` via uma única coleção (`EntidadesDoMundo`), mesmo padrão de `InventarioDoJogador` (Incremento 5): sem persistência, sem Port, sem Caso de Uso próprio.
- **Coleção única para jogadores e mobs**, ao contrário do C# (`MinecraftClient.Entities` + `PlayerManager.Players` mantidos manualmente em paralelo) — corrige por construção o bug real encontrado no legado (Entity Look busca só no dicionário de jogadores, então nunca encontra um mob). Filtragem por tipo (`instanceof`), quando necessária, substitui a segunda coleção sem perda de capacidade.
- **Divergências deliberadas, registradas conforme a Política de Compatibilidade com o Legado**:
  - Rotação (yaw/pitch/head-yaw) aplicada simetricamente no Spawn Mob e no Spawn Player — o C# lê esses campos no Spawn Mob e os descarta (`case 15`), deixando todo mob nascer olhando para 0/0; comparado ao tratamento simétrico dado ao Spawn Player no mesmo handler, é uma omissão, não uma regra de negócio deliberada.
  - Campo `headYaw` dedicado em `EntidadeRemota` — o C# reaproveita o mesmo campo `Yaw` para corpo e cabeça (não existe campo próprio em `IEntity`); o protocolo real já distingue os dois conceitos (pacotes `0x16` e `0x19` são pacotes diferentes).
  - Posição do Spawn Mob decodificada com divisão double correta (`÷32.0`) — o C# usa divisão inteira (`/32`) e trunca a posição fracionária só para mobs (`Entity Teleport`, no mesmo handler, já usa a divisão correta).
  - `SpawnPlayerCodec` modela apenas até `pitch`, fiel ao `Handler_v18` (`case 12`, que também para nesse ponto) — Current Item e o bloco de Entity Metadata do pacote real ficam sem leitura; seguro porque o frame já foi isolado por comprimento antes do Codec rodar (mesma garantia de isolamento que sustenta a DEC-20), então os bytes restantes simplesmente sobram sem risco de desalinhamento.
  - `SpawnMobCodec` consome o bloco de Entity Metadata (formato pré-1.9, terminador `0x7F`) apenas para alinhamento do frame, via novo utilitário `MetadataDeEntidadeCodec` — nenhum campo de metadata é exposto como dado de domínio nesta etapa (mesmo tratamento já dado ao NBT de item no Incremento 5); extração do nome de exibição do mob fica para um incremento futuro com consumidor real, evitando a heurística "última string encontrada vence" do `getEntityName` do C#, que descarta o índice do campo.
  - Velocidade do Spawn Mob (3 shorts) é lida para manter o alinhamento do frame mas não é modelada — sem consumidor nesta etapa (Entity Velocity como pacote próprio fica para o Incremento 6.3).
- **Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova** — mesma determinação já aplicada a todos os incrementos da Milestone 5: Receptor (domínio) chama `SessaoDeJogo`/`EntidadesDoMundo` (domínio) diretamente; mudança 100% aditiva (`SessaoDeJogo` ganha um campo e um acessor, mesmo padrão de `inventario`).

Entregue

- `domain.bot`: `EntidadeRemota` (sealed abstract, `entityId`/`x`/`y`/`z`/`yaw`/`pitch`/`headYaw` + mutadores `moverRelativo`/`definirRotacao`/`moverEGirar`/`teleportar`/`atualizarHeadYaw`), `EntidadeJogadorRemoto` (+ `uuid`), `EntidadeMob` (+ `tipo`), `EntidadesDoMundo` (`Map<Integer, EntidadeRemota>` com `registrar`/`remover`/`porId`/`todas`). `SessaoDeJogo` ganha o campo final `entidades` e o acessor `entidades()`.
- `domain.protocol.v1_8`: `SpawnPlayerPacket`/`SpawnPlayerCodec`/`SpawnPlayerHandler`/`EventoSpawnPlayer`/`ReceptorSpawnPlayer` (PLAY, id `0x0C`, CLIENTBOUND); `SpawnMobPacket`/`SpawnMobCodec`/`SpawnMobHandler`/`EventoSpawnMob`/`ReceptorSpawnMob` (PLAY, id `0x0F`, CLIENTBOUND); `DestroyEntitiesPacket`/`DestroyEntitiesCodec`/`DestroyEntitiesHandler`/`EventoDestroyEntities`/`ReceptorDestroyEntities` (PLAY, id `0x13`, CLIENTBOUND, `int[] entityIds`). Utilitários novos, package-privados: `AnguloCodec` (conversão do tipo "Angle" do protocolo, 1 byte = 1/256 de volta, compartilhado pelos dois Codecs de spawn), `MetadataDeEntidadeCodec` (consumo do bloco de Entity Metadata pré-1.9).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` (`handlersV1_8()`/`receptoresV1_8()`) atualizados com os três novos pacotes.
- 40 testes novos (JUnit 5 + AssertJ, sem mocks): 8 em `EntidadeRemotaTest`, 7 em `EntidadesDoMundoTest`, 3+1+1 em `SpawnPlayerCodecTest`/`SpawnPlayerHandlerTest`/`ReceptorSpawnPlayerTest`, 4+1+1 em `SpawnMobCodecTest`/`SpawnMobHandlerTest`/`ReceptorSpawnMobTest` (incluindo teste dedicado provando que metadata sintética com múltiplos tipos é consumida sem alterar os campos anteriores, mesmo espírito do teste de NBT do Incremento 5), 4+1+2 em `DestroyEntitiesCodecTest`/`DestroyEntitiesHandlerTest`/`ReceptorDestroyEntitiesTest`, 6 em `RegistroDePacotesV1_8Test`, 3 em `PipelineDeProtocoloV1_8Test`, 1 em `AdaptadorConexaoBotV1_8Test` (integração real ponta a ponta: Spawn Player + Spawn Mob registrados, Destroy Entities removendo um dos dois).

Restrições respeitadas

Não foram implementados: Entity Velocity como pacote próprio (Incremento 6.3), movimentação de entidade já conhecida — Relative Move/Look/Look+Move/Teleport/Head Look (Incremento 6.2), Entity Metadata/Equipment/Effect/Animation/Status (Incremento 6.4), Player List Item/tab list, qualquer automação de combate ou farm (explicitamente fora de escopo — ver fase de planejamento).

Riscos e observações

- Diferente dos incrementos anteriores da Milestone 5, os Receptores de movimentação (a partir do Incremento 6.2) vão operar sobre uma coleção que cresce incrementalmente — um pacote para um `entityId` ainda não registrado é possível durante o rollout. `EntidadesDoMundo.porId` retorna `null` para id desconhecido; Receptores futuros devem tratar isso como no-op, não exceção (nenhum caso assim existe ainda neste incremento, já que Spawn/Destroy não dependem de estado anterior).
- Ambiguidade resolvida por leitura direta do C#, não por suposição sobre o protocolo vanilla: a hierarquia `Entity`/`IEntity`/`EntityManager` do legado não corresponde ao que o nome sugere (ver fase de planejamento) — qualquer sessão futura que volte a este código deve consultar `EntidadeRemota`/`EntidadesDoMundo`, não tentar mapear 1:1 contra as classes C# de mesmo nome aparente.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 281 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`). Nenhuma interface já aprovada foi alterada (`Codec`, `PacketHandler`, `ReceptorDeEvento`, `RegistroDePacotes`, `EventoDeProtocolo`, `ConexaoBotPort`, `ConexaoMinecraft`, `LeitorDePacote`, `EscritorDePacote` permanecem exatamente como antes) — mudança 100% aditiva.

Próximo passo sugerido

Incremento 6.2 — Movimentação de Entidades (Entity Relative Move `0x15`, Entity Look `0x16`, Entity Look And Relative Move `0x17`, Entity Teleport `0x18`, Entity Head Look `0x19`). Já aprovado pelo responsável do projeto; nenhuma DEC pendente bloqueia o início.

### Incremento 6.2 — Movimentação de Entidades (Entity Relative Move, Entity Look, Entity Look And Relative Move, Entity Teleport, Entity Head Look)

Status

Concluído

Objetivo

Implementar a movimentação de entidades já conhecidas (registradas pelo Incremento 6.1), cobrindo os 5 pacotes de posição/rotação restantes do ciclo de vida mapeados na fase de planejamento.

Determinação de arquitetura (sem nova DEC)

- Mapeamento confirmado por leitura direta do `Handler_v18` (cases 21–25): Entity Relative Move (`0x15`) `entityId`+`dx`/`dy`/`dz` (SByte/32.0); Entity Look (`0x16`) `entityId`+`yaw`/`pitch` (ângulo-byte); Entity Look And Relative Move (`0x17`) combina os dois; Entity Teleport (`0x18`) `entityId`+`x`/`y`/`z` (Int/32.0, já com a divisão double correta no próprio C#) +`yaw`/`pitch`; Entity Head Look (`0x19`) `entityId`+`headYaw`.
- **Divergência do Incremento 6.1 corrigida por construção**: o C# busca o Entity Look só em `PlayerManager.Players` (nunca encontra um mob). Como `EntidadesDoMundo` já é uma coleção única desde o Incremento 6.1, esse bug simplesmente não existe aqui — qualquer entidade registrada, jogador ou mob, é encontrada da mesma forma.
- Campo "On Ground" do protocolo real (presente nos 4 pacotes de posição, se existir) não é modelado — mesmo raciocínio de segurança já usado para o Current Item do Spawn Player (Incremento 6.1): o frame já foi isolado por comprimento antes do Codec rodar, então ignorar um campo final sem uso funcional é seguro por construção, não uma lacuna arriscada.
- **Primeira vez que um Receptor pode receber um evento para uma entidade não registrada** em `EntidadesDoMundo` (rollout incremental, ou pacote de movimento chegando antes do spawn correspondente ser processado) — resolvido exatamente como o risco já registrado no Incremento 6.1 antecipava: `porId` retorna `null`, o Receptor trata como no-op silencioso, sem exceção. Os 5 novos Receptores seguem esse contrato de forma consistente.
- Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova — mesmo padrão de todos os incrementos anteriores.

Entregue

- `domain.protocol.v1_8`: `EntityRelativeMovePacket`/Codec/Handler/Evento/Receptor (`0x15`), `EntityLookPacket`/Codec/Handler/Evento/Receptor (`0x16`), `EntityLookAndRelativeMovePacket`/Codec/Handler/Evento/Receptor (`0x17`), `EntityTeleportPacket`/Codec/Handler/Evento/Receptor (`0x18`), `EntityHeadLookPacket`/Codec/Handler/Evento/Receptor (`0x19`) — todos PLAY/CLIENTBOUND, reaproveitando `AnguloCodec` (Incremento 6.1) sem alteração.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com os 5 pacotes.

Restrições respeitadas

Não foram implementados: Entity Velocity (`0x12`, ver Incremento 6.3 a seguir), Entity Metadata/Equipment/Effect/Animation/Status (Incremento 6.4), Player List Item, qualquer automação de combate/movimento.

Riscos e observações

Campo "On Ground" não modelado nos 4 pacotes de posição — se um consumidor futuro precisar dele (ex.: física/colisão local de outra entidade), retomar a decodificação nesses Codecs; hoje é seguro ignorá-lo, sem risco de desalinhamento de frame.

### Incremento 6.3 — Entity Velocity

Status

Concluído

Objetivo

Completar a lista de pacotes de ciclo de vida de entidade tratados com fidelidade parcial pelo C# (`Handler_v18`, case 18), refletindo velocidade em qualquer entidade conhecida.

Determinação de arquitetura (sem nova DEC)

- `EntidadeRemota` ganha `velocityX`/`velocityY`/`velocityZ` (volatile double) + `definirVelocidade(double, double, double)` — mesmo padrão dos demais mutadores; valor absoluto (substituição a cada pacote), não delta acumulado, fiel à semântica do campo no protocolo.
- **Regra de negócio do C# explicitamente não portada**: aplicação ao próprio bot (`Player.MotionX`/`Y`/`Z`) sob a flag estática `MinecraftClient.Knockback` (bypass de conhecimento de recuo). Não existe motor de física no lado Java — `SessaoDeJogo` só reflete a posição já confirmada pelo servidor, não simula movimento — e o bypass é automação de combate, fora de escopo desde a fase de planejamento do Incremento 6.
- Um pacote de Entity Velocity com o `entityId` do próprio bot é absorvido de forma inofensiva pelo mesmo contrato "entidade desconhecida → no-op" do Incremento 6.2: o bot nunca é registrado em `EntidadesDoMundo` (só entidades remotas o são, via Spawn Player/Mob, e o servidor nunca envia um desses pacotes sobre a própria conexão do bot) — `porId(idDoBot)` já retorna `null` por construção, sem exigir nenhuma checagem especial de "isto sou eu". Testado explicitamente.
- Conversão `÷8000.0` (Short) aplicada a todas as entidades — é formato de fio (mesma constante usada pelo C#), não regra de negócio.

Entregue

- `domain.bot.EntidadeRemota`: campos e mutador de velocidade (extensão aditiva).
- `domain.protocol.v1_8.EntityVelocityPacket`/`EntityVelocityCodec`/`EntityVelocityHandler`/`EventoEntityVelocity`/`ReceptorEntityVelocity` (PLAY, id `0x12`, CLIENTBOUND).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados.
- 46 testes novos no total para 6.2+6.3 (JUnit 5 + AssertJ, sem mocks): 25 nos 5 pacotes de movimentação (2 Codec + 1 Handler + 2 Receptor cada, incluindo o caso de entidade desconhecida), 6 no Entity Velocity (2 Codec + 1 Handler + 3 Receptor — conhecida, desconhecida, e `entityId` do próprio bot), 2 em `EntidadeRemotaTest` (velocidade), 7 em `RegistroDePacotesV1_8Test`, 11 em `PipelineDeProtocoloV1_8Test`, 1 em `AdaptadorConexaoBotV1_8Test` (integração real: mob nasce, move-se, gira, teleporta e recebe velocidade, tudo refletido em `EntidadesDoMundo`).

Restrições respeitadas

Não foi replicado o bypass de knockback nem qualquer aplicação automática de física ao próprio bot.

Riscos e observações

Velocidade é reflexo puro de estado, sem consumidor funcional ainda — nenhuma automação de movimento/física existe no lado Java. Mantido por consistência com o restante da modelagem de entidades (mesmo tratamento dado a outros campos "espelhados, sem uso imediato" ao longo da Milestone 5), não por necessidade concreta já comprovada.

Validação executada (6.2 + 6.3, sessão combinada)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 327 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`). Nenhuma interface já aprovada foi alterada — mudança 100% aditiva. Durante a implementação, 1 teste de integração falhou inicialmente por usar `yaw=180f` no Entity Teleport — mesma armadilha de precisão do ângulo-byte já observada no Incremento 6.1 (180° mapeia para o valor de byte 128, fora do intervalo assinado, e decodifica de volta como -180°); corrigido trocando para um múltiplo seguro de 45°.

Próximo passo sugerido

Incremento 6.4 — Entity Metadata, Entity Equipment, Entity Effect, Animation, Entity Status. Ressalva já registrada na fase de planejamento: apenas Entity Effect tem qualquer precedente no C# (exclusivo do próprio bot); os outros 4 pacotes nunca foram implementados no legado, então a fonte da verdade passa a ser a especificação do protocolo vanilla, não o C#. Já aprovado pelo responsável do projeto; detalhamento campo a campo fica para quando este sub-incremento iniciar.

### Incremento 6.4 — Estado Visual das Entidades (Entity Equipment, Animation, Entity Status, Entity Effect, Remove Entity Effect)

Status

Concluído

Objetivo

Completar a modelagem das entidades com o estado "visual/enriquecimento" identificado na fase de planejamento do Incremento 6: equipamento, animação, status e efeitos de status, aplicados a qualquer entidade remota conhecida.

Determinação de arquitetura (sem nova DEC)

- Confirmado por leitura direta do `Handler_v18.cs`: não existe `case` no switch de pacotes PLAY para `0x04` (Entity Equipment), `0x0B` (Animation) e `0x1A` (Entity Status) — nunca implementados no legado. Apenas `0x1D`/`0x1E` (Entity Effect/Remove Entity Effect) têm precedente, e só para o próprio bot (`Client.PlayerID`, gravando em `Player.ActivePotions`). Para os 4 pacotes sem precedente, o formato de fio foi validado contra a especificação oficial do protocolo 47 (cruzada com o schema machine-readable do `minecraft-data`), conforme já sinalizado ao responsável do projeto na fase de planejamento — nenhuma regra de negócio nova foi inventada, apenas o formato de fio, que é dado objetivo e verificável.
- `EntidadeRemota` recebe quatro extensões aditivas, todas seguindo o padrão já estabelecido (mutador + getter, sem novo agregado/Port/Caso de Uso): `Map<Integer, ItemStack> equipamento` (chave = slot, valor removido do mapa em vez de armazenar `null` — `ConcurrentHashMap` não aceita valores nulos, e semanticamente um slot vazio é a ausência da entrada, não uma entrada nula), `int ultimaAnimacao` (sentinel `-1`), `int ultimoStatus` (sentinel `-1`), `Map<Integer, EfeitoAtivo> efeitosAtivos` (`EfeitoAtivo` — novo record aninhado, VO nativo do domínio do bot, não um tipo do protocolo). `ItemStack` (já existente em `domain.protocol.v1_8`, usado por Set Slot/Window Items) é reaproveitado para o slot de equipamento — mesmo cruzamento de pacote já praticado por `InventarioDoJogador`.
- Entity Equipment (`0x04`): `entityId` (VarInt) + `slot` (Short, 0 = mão principal, 1-4 = armadura nesta versão pré-1.9 — sem mão secundária, que só existe a partir do 1.9) + `item` (Slot, reaproveitando `ItemStackCodec`).
- Animation (`0x0B`, clientbound): `entityId` (VarInt) + `animation` (Unsigned Byte). Modelado como estado transiente espelhado na entidade (mesmo tratamento dado à velocidade no Incremento 6.3: reflexo de protocolo, sem consumidor funcional ainda).
- Entity Status (`0x1A`): único pacote deste sub-incremento cujo `entityId` é `Int` puro de 4 bytes, não VarInt — confirmado pela especificação como resquício do protocolo anterior à adoção generalizada de VarInt para identificadores de entidade nesta versão específica de pacote.
- Entity Effect (`0x1D`) / Remove Entity Effect (`0x1E`): campos idênticos entre C# e especificação para `entityId`/`effectId` (Remove Entity Effect é fiel 1:1 ao C#). Para Entity Effect, dois pontos de divergência deliberada e documentada em relação ao C#: (1) o C# só aplica o efeito ao próprio bot (`entityId == Client.PlayerID`) e descarta silenciosamente qualquer efeito de outra entidade — aqui, como `EntidadesDoMundo` é uma coleção única para qualquer entidade remota (mesma filosofia que já corrigiu o bug de Entity Look no Incremento 6.1), o efeito é aplicado a qualquer entidade rastreada; o efeito do próprio bot continua fora deste pipeline, pelo mesmo motivo estrutural já usado para velocidade no Incremento 6.3 (o bot nunca está em `EntidadesDoMundo`) — sem motor de física/status no lado Java que o consumisse de qualquer forma; (2) o C# comenta a leitura de `duration`/`hideParticles` (campo de baixo valor não lido, mesmo padrão já visto em Spawn Player/movimentação) — aqui os dois campos foram decodificados por completo, pois carregam dado de domínio relevante (duração e visibilidade do efeito) e o formato é confirmado pela especificação (mesmo precedente de "modelar o formato de fio completo" já usado em Update Health).
- **Decisão deliberada de não implementar Entity Metadata (`0x1C`) como pipeline próprio nesta etapa.** Motivo: o C# nunca implementou (nem sequer tentou — a leitura no spawn de jogador está comentada); o formato de fio (bloco de índice+tipo+valor terminado em `0x7F`) é comprovável pela especificação e já é consumido com segurança pelo `MetadataDeEntidadeCodec` existente desde o Incremento 6.1, mas o *significado* de cada índice depende do tipo concreto de entidade (jogador, mob, item, etc.) e não há, nesta etapa, nenhum consumidor concreto que precise dessa semântica. Construir um Receptor sem nenhum mutador real seria uma abstração sem necessidade comprovada (contrariando o CLAUDE.md). Pacotes `0x1C` continuam cobertos pelo mecanismo já existente da DEC-20 (pacote PLAY sem Codec registrado é WARN-logado e descartado com segurança, nunca fatal) — nenhum código novo necessário, nenhuma regra de negócio inventada. Sinalizado explicitamente aqui para o responsável do projeto poder contestar a decisão se desejar.

Entregue

- `domain.bot.EntidadeRemota`: `Map<Integer, ItemStack> equipamento` + `definirEquipamento`/`equipamento`/`equipamentos`; `int ultimaAnimacao` + `registrarAnimacao`/`ultimaAnimacao`; `int ultimoStatus` + `registrarStatus`/`ultimoStatus`; `record EfeitoAtivo(int amplifier, int duration, boolean hideParticles)` + `Map<Integer, EfeitoAtivo> efeitosAtivos` + `aplicarEfeito`/`removerEfeito`/`efeito`/`efeitosAtivos` (extensões aditivas).
- `domain.protocol.v1_8.EntityEquipmentPacket`/`EntityEquipmentCodec`/`EntityEquipmentHandler`/`EventoEntityEquipment`/`ReceptorEntityEquipment` (PLAY, id `0x04`, CLIENTBOUND).
- `domain.protocol.v1_8.AnimationPacket`/`AnimationCodec`/`AnimationHandler`/`EventoAnimation`/`ReceptorAnimation` (PLAY, id `0x0B`, CLIENTBOUND).
- `domain.protocol.v1_8.EntityStatusPacket`/`EntityStatusCodec`/`EntityStatusHandler`/`EventoEntityStatus`/`ReceptorEntityStatus` (PLAY, id `0x1A`, CLIENTBOUND).
- `domain.protocol.v1_8.EntityEffectPacket`/`EntityEffectCodec`/`EntityEffectHandler`/`EventoEntityEffect`/`ReceptorEntityEffect` (PLAY, id `0x1D`, CLIENTBOUND).
- `domain.protocol.v1_8.RemoveEntityEffectPacket`/`RemoveEntityEffectCodec`/`RemoveEntityEffectHandler`/`EventoRemoveEntityEffect`/`ReceptorRemoveEntityEffect` (PLAY, id `0x1E`, CLIENTBOUND).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com os 5 pacotes.
- 47 testes novos (JUnit 5 + AssertJ, sem mocks): 33 nos 5 novos pacotes (2 Codec + 1 Handler + 2-3 Receptor cada, incluindo entidade desconhecida e, no Entity Effect, `entityId` do próprio bot), 8 em `EntidadeRemotaTest` (equipamento, animação, status, efeitos), 5 em `RegistroDePacotesV1_8Test`, 5 em `PipelineDeProtocoloV1_8Test`, 1 em `AdaptadorConexaoBotV1_8Test` (integração real: mob nasce, recebe equipamento/animação/status/efeito, efeito é removido e um segundo efeito é aplicado, tudo refletido em `EntidadesDoMundo`).

Restrições respeitadas

Nenhuma interface pública alterada; nenhum novo agregado/bounded context/Port/Caso de Uso; nenhuma regra de negócio inventada sem validação (C# para o que existe, especificação oficial do protocolo 47 para o que não existe); nenhuma DEC contrariada — Entity Metadata reforça o mecanismo já formalizado na DEC-20 em vez de contorná-lo.

Riscos e observações

Equipamento/animação/status/efeitos são majoritariamente reflexo puro de estado, sem consumidor funcional ainda (mesmo perfil de risco já aceito para velocidade no Incremento 6.3) — mantidos por completude da modelagem de "estado visual da entidade", conforme escopo definido pelo próprio responsável do projeto ao aprovar o Incremento 6.4. Os efeitos de status do próprio bot (usados no C# para cálculo de velocidade de movimento — `Entity.cs`, verificação de Speed/Slowness) não são rastreados em lugar nenhum do lado Java nesta etapa; consistente com a ausência de motor de física/movimento já aceita desde o Incremento 6.3, não uma lacuna nova.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 374 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`). Mudança 100% aditiva. Formato de fio dos 5 pacotes cross-validado contra o schema `minecraft-data` (protocolo 47) antes da implementação.

Próximo passo sugerido

Incremento 6 (com seus 4 sub-incrementos) está concluído. A Milestone 5 ainda tem frentes abertas fora do escopo de entidades — mundo/chunks/blocos, chat enviado pelo bot, Player List/tab list, combate/automação (ver Seção 6, "Não implementado") — nenhuma delas foi detalhada em uma fase de planejamento própria ainda. Recomenda-se abrir uma nova fase de planejamento, análoga à feita para o Incremento 6, antes de iniciar qualquer uma dessas frentes; a escolha de qual delas priorizar cabe ao responsável do projeto.

---

## Milestone 7

Status

Em andamento — Incrementos 7.0 (implementação integral da DEC-20), 7.1 (fundação do agregado Mundo e Chunk Data), 7.2 (Block Change e Multi Block Change) e 7.3 (Map Chunk Bulk) concluídos.

Objetivo

Modelagem do Mundo (Chunks e Blocos) sobre o estado PLAY do protocolo Minecraft 1.8 — a frente identificada como pendente ao encerrar o Incremento 6 da Milestone 5.

### Fase de Planejamento — Modelagem do Mundo (Chunks e Blocos)

Status

Concluída

Objetivo

Produzir o desenho arquitetural do bounded context de Mundo antes de qualquer implementação — análise completa do legado C# (`AdvancedBot.Client.Map.World/Chunk/ChunkSection`, `AdvancedBot.Client.Block/Blocks`, `AdvancedBot.Client.Handler.Handler_v18.cs`, `AdvancedBot.Client.PathFinding.Path/PathFinder/PathGuide/PathPoint`, `AdvancedBot.Client.Commands.CommandBreakBlock/CommandClickBlock/CommandPlaceBlock`, `AdvancedBot.Client.Packets.PacketBlockPlace/PacketPlayerDigging/DiggingStatus`), exclusivamente como fonte de regras de negócio, nunca como referência arquitetural. Nenhum código foi escrito nesta fase.

Resultado

Aprovado pelo responsável do projeto (sem nova DEC)

Entregue

- Documento de arquitetura apresentado como material de discussão (não commitado a `docs-reescrita`, mesmo tratamento das fases de planejamento anteriores). Cobriu: mapeamento dos pacotes PLAY candidatos ao bounded context Mundo (Chunk Data `0x21`, Multi Block Change `0x22`, Block Change `0x23`, Map Chunk Bulk `0x26`, Explosion `0x27`, Time Update `0x03`, Change Game State `0x2B`, World Border `0x44`, Update Sign `0x33`, Update Block Entity `0x35`, Block Action `0x24`, Block Break Animation `0x25`) cruzados com o legado e, para os pacotes sem precedente no C#, com a especificação do protocolo 47; distinção entre os efetivamente usados pelo legado (Chunk Data, Multi Block Change, Block Change, Map Chunk Bulk, Explosion, Update Sign) e os nunca implementados; determinação de que Mundo é um agregado interno de `SessaoDeJogo` (mesmo critério já aplicado a Inventário e Entidades — sem novo bounded context, Port ou Caso de Uso); `Bloco` como Value Object, não Entity (mesmo tratamento de `ItemStack`); roteiro de incrementos (7.0 fechamento de uma lacuna de implementação pré-existente, 7.1 fundação + Chunk Data, 7.2 Block Change/Multi Block Change; Map Chunk Bulk, Explosion, Time Update/Change Game State e os demais pacotes sem consumidor real ficando como candidatos não comprometidos).
- **Achado crítico:** a **DEC-20** (Política de Tolerância a Pacotes PLAY Não Registrados, aprovada em 2026-07-16) nunca havia sido implementada — `TransporteSocket.readLoop` continuava encerrando a conexão para qualquer `RuntimeException` em qualquer `EstadoConexao`, sem a distinção por PLAY que a decisão definiu. Registrado como pré-requisito bloqueante (Incremento 7.0) antes de qualquer pacote de mundo, já que esta frente introduz o maior número de pacotes novos de uma vez desde o início da Milestone 5 — sem a tolerância, o primeiro pacote de mundo ainda não coberto por um servidor real encerraria a conexão inteira.

Observação de escopo

Exclusivamente documental — nenhuma classe, teste ou linha de código de produção foi criada nesta fase.

Validação executada

Não aplicável a código (fase exclusivamente documental).

### Incremento 7.0 — Implementação Integral da DEC-20 (Tolerância a Pacotes PLAY Não Registrados)

Status

Concluído

Objetivo

Implementar integralmente a DEC-20 exatamente como aprovada em 2026-07-16, fechando a lacuna identificada na fase de planejamento, antes de qualquer código relacionado a Mundo.

Determinação de arquitetura (sem nova DEC)

Execução de uma decisão já aprovada, não uma decisão nova. Verificado explicitamente antes de implementar que nenhum outro ponto do pipeline contraria a DEC-20: `ProtocolDispatcher.dispatch`/`RoteadorDeEventos.rotear` também lançam exceção quando não encontram, respectivamente, um `PacketHandler`/`ReceptorDeEvento`, mas isso está fora do escopo da DEC-20 por desenho (que cobre exclusivamente a falha de localização de Codec) — na prática todo pacote registrado sempre chega junto do seu Handler/Receptor no mesmo incremento, então esse caminho seguir fatal é correto (indicaria bug de wiring, não lacuna de cobertura).

Entregue

- `infrastructure.protocol.PacoteNaoRegistradoException` (novo, subtipo de `IllegalArgumentException`).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8.localizarCodec`/`localizarId`: passam a lançar `PacoteNaoRegistradoException` em vez de `IllegalArgumentException` genérico.
- `infrastructure.network.TransporteSocket.readLoop`: isola a busca do Codec num try/catch próprio; `PacoteNaoRegistradoException` em `EstadoConexao.PLAY` → log WARN (SLF4J, estado implícito por só ocorrer em PLAY, id em hexadecimal, tamanho do frame descartado) + descarte do frame + continuação do loop; qualquer outro estado, ou falha durante `codec.decode(...)` (Codec encontrado mas que falha), mantém o comportamento anterior (conexão encerrada).
- 3 testes novos: `PacoteNaoRegistradoExceptionTest` (2 testes), `TransporteSocketTest.deveDescartarPacotePlayNaoRegistradoESeguirRecebendoOsProximos` (1 teste, prova que a conexão não fecha e o próximo pacote válido — Keep Alive — ainda é recebido); 2 testes existentes de `RegistroDePacotesV1_8Test` fortalecidos para exigir o tipo específico da exceção (mantendo compatibilidade por subtipo).

Restrições respeitadas

Nenhum item relacionado a Mundo (Chunk Data, Block Change, `Chunk`, `Bloco`) foi tocado nesta etapa.

Riscos e observações

Nenhum risco novo introduzido; nenhuma regressão nos 374 testes pré-existentes.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, 377 testes executados, 0 falhas, 3 skipped deliberadamente (`HandshakeServidorRealTest`, `LoginServidorRealTest`, `LoginComCompressaoServidorRealTest`).

### Incremento 7.1 — Fundação do Agregado Mundo e Chunk Data (Coluna Completa)

Status

Concluído

Objetivo

Implementar a fundação do agregado Mundo (`Mundo`/`Chunk`/`ChunkSection`/`Bloco`) e o pacote Chunk Data (`0x21`, PLAY/CLIENTBOUND), replicando fielmente `Handler_v18.P21ReadChunkData`/`ReadChunk`/`CalcChunkSize` e `AdvancedBot.Client.Map.World`/`Chunk`/`ChunkSection` do legado C#.

Análise do legado

Chunk Data carrega `x`/`z` (int), `groundUpContinuous` (bool), `primaryBitMask` (ushort — **não** VarInt) e um array de bytes de tamanho `VarInt` cujo conteúdo é, para cada seção presente na bitmask (bit `i` setado, na ordem 0–15), 4096 blocos codificados como **2 bytes little-endian** (`byte baixo | byte alto << 8`), onde os bits 4–15 são o id do bloco (12 bits) e os bits 0–3 são o metadata (4 bits); bioma/skylight/blocklight, quando presentes, ocupam o restante do array já extraído e nunca são lidos pelo `ReadChunk` do C# (a própria matemática de `CalcChunkSize` confirma que o layout agrupa todos os arrays de tipo de bloco de todas as seções presentes primeiro, deixando luz/bioma como um bloco final nunca consultado). `groundUpContinuous=true` sempre descarta o `Chunk` anterior e cria um novo (seções fora da bitmask viram ar); `groundUpContinuous=false` reaproveita o `Chunk` existente (ou cria um novo se nenhum existir), substituindo apenas as seções presentes na bitmask e preservando as demais. `flag && bitmask==0` é o sinal de descarregamento do chunk (`World.SetChunk(x,z,null)`) — não existe pacote dedicado de "unload" no protocolo 1.8.

Determinação de arquitetura (sem nova DEC)

- **Mundo é um agregado interno de `SessaoDeJogo`, não um novo bounded context** — mesmo critério já aplicado a `InventarioDoJogador` (Incremento 5) e `EntidadesDoMundo` (Incremento 6): sem persistência, sem Port/Caso de Uso próprios, ciclo de vida igual ao de `SessaoDeJogo`. Já antecipado nominalmente pela DEC-19.
- **`Chunk` é Entity interna (não Aggregate Root)** dentro de `Mundo`, indexado por `PosicaoDeChunk` (novo record `(int chunkX, int chunkZ)`) num `ConcurrentHashMap` — mesmo padrão de `EntidadesDoMundo`.
- **`Bloco` (`domain.protocol.v1_8`) é Value Object (record `short id, byte metadata`), não Entity** — mesmo tratamento de `ItemStack`: puramente derivado da decodificação do protocolo, sem identidade própria.
- **Divergência deliberada e documentada:** o C# trunca o id de 12 bits para `byte` (0–255) ao armazenar em `World`/`Chunk`/`ChunkSection` (`(byte)((num4 >> 4) & 0xFF)`). Aqui o id de 12 bits (0–4095) é preservado sem truncar — nenhum bloco vanilla do protocolo 47 usa id acima de 255, então a diferença só apareceria com servidores não-vanilla; truncar seria replicar um bug de armazenamento, não uma regra de negócio (mesmo espírito das divergências já registradas no Incremento 6.1 — divisão double correta, coleção única de entidades).
- **Simplificação de implementação registrada (não é regra de negócio):** `SecaoDeChunk` armazena id/metadata em arrays primitivos (`short[4096]`/`byte[4096]`) em vez do empacotamento nibble (`byte[2048]`) usado pelo C# — mesma semântica (0–15 para metadata), sem a otimização de memória de uma era com restrições diferentes. `PosicaoDeChunk` é um record em vez do empacotamento manual em `long` (`ToChunkPos`) do C# — Java não precisa da otimização de chave primitiva.
- **Bioma/skylight/blocklight consumidos sem serem expostos** — mesmo idioma já estabelecido para NBT (`ItemStackCodec`) e Entity Metadata (`MetadataDeEntidadeCodec`): como o array de dados já foi extraído por inteiro (`readByteArray(readVarInt())`), o restante depois do bloco de tipos é simplesmente nunca indexado, sem necessidade de lógica de "pular" byte a byte.
- **Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova** — mesma determinação de todo incremento da Milestone 5/6: Receptor (domínio) chama `SessaoDeJogo`/`Mundo` diretamente.

Entregue

- `domain.bot`: `PosicaoDeChunk` (record), `SecaoDeChunk` (package-private), `Chunk` (`bloco`/`definirBloco`/`definirSecaoCompleta`, cria seção sob demanda), `Mundo` (`registrarChunk`/`descarregarColuna`/`chunkEm`/`blocoEm`/`definirBloco`, `ConcurrentHashMap<PosicaoDeChunk, Chunk>`). `SessaoDeJogo` ganha o campo final `mundo` e o acessor `mundo()` — mesmo padrão de `entidades`/`inventario`.
- `domain.protocol.v1_8`: `Bloco` (record); `ChunkDataPacket`/`ChunkDataCodec`/`ChunkDataHandler`/`EventoChunkData`/`ReceptorChunkData` (PLAY, id `0x21`, CLIENTBOUND). `ReceptorChunkData` replica `ReadChunk`: descarrega a coluna quando `colunaCompleta && mascaraDeBits==0`; senão cria um `Chunk` novo (coluna completa) ou reaproveita o existente/cria um novo se ausente (coluna parcial), substitui apenas as seções presentes na bitmask, e registra o resultado em `Mundo`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` (`handlersV1_8()`/`receptoresV1_8()`) atualizados com `ChunkDataPacket` (id `0x21`, sem colisão com os 32 pacotes já registrados).
- Testes novos (contagem consolidada com o Incremento 7.2 abaixo): `SecaoDeChunkTest`, `ChunkTest`, `MundoTest` (incluindo chunk desconhecido → no-op e não-exceção, mesmo contrato de `EntidadesDoMundo.porId`), `ChunkDataCodecTest` (round-trip, formato little-endian, seções ausentes nulas conforme bitmask, descarregamento), `ChunkDataHandlerTest`, `ReceptorChunkDataTest` (coluna completa, descarregamento, mescla de coluna parcial preservando seções não alteradas, criação de chunk novo quando coluna parcial chega antes do chunk existir).

Restrições respeitadas

Nenhuma automação, mineração, colocação/quebra de bloco por comando, pathfinding ou navegação — a modelagem de `Mundo`/`Bloco` é o pré-requisito de dados que essas frentes futuras consumiriam, mas construí-las agora seria abstração sem necessidade comprovada (nenhum consumidor real ainda). Nenhum outro pacote de mundo (Map Chunk Bulk, Explosion, Time Update, Change Game State, World Border, Update Sign, Update Block Entity, Block Action, Block Break Animation) foi implementado.

Riscos e observações

- Semântica de coluna parcial (`groundUpContinuous=false`) confirmada por leitura direta do C# nesta etapa (não estava clara na fase de planejamento) — ver "Análise do legado" acima.
- Crescimento de memória sem limite: `Mundo` retém todo chunk já visto num mapa concorrente, sem eviction — mesmo padrão de `EntidadesDoMundo` (que também não tem eviction), mas chunks são ordens de magnitude maiores (mínimo 4096+2048 bytes por seção × até 16 seções por coluna); primeiro risco de capacidade real do projeto, a monitorar em sessões longas, não resolvido agora.
- `BufferLeitorDePacote.readByteArray(int)` não valida `position + length <= buffer.length` (risco pré-existente, não introduzido por este incremento) — Chunk Data é o tipo de pacote mais propenso a expor esse gap primeiro; sem consequência observada nos testes.

### Incremento 7.2 — Block Change e Multi Block Change

Status

Concluído

Objetivo

Implementar a mutação incremental de blocos num `Mundo` já carregado: Block Change (`0x23`, mutação única) e Multi Block Change (`0x22`, lote), replicando fielmente os `case`s 34/35 do `Handler_v18.cs`.

Análise do legado

Block Change carrega `location` (tipo "Position" do protocolo — `long` empacotado com x em 26 bits, y em 12 bits e z em 26 bits, decodificado/codificado exatamente como `ReadBuffer.ReadLocation`/`WriteBuffer.WriteLocation` do C#) seguido do mesmo VarInt combinado id(12 bits)+metadata(4 bits) usado pelo Chunk Data. Multi Block Change carrega `chunkX`/`chunkZ` (int, multiplicados por 16 no C#) seguidos de um `VarInt` de contagem e, por registro, um `short` empacotado (nibble X local, nibble Z local, byte Y absoluto) mais o mesmo VarInt de id+metadata. Ambos aplicam a mutação via `World.SetBlockAndData`, que **não faz nada silenciosamente se o chunk de destino não estiver carregado** (`GetChunk` retornando `null`) — apenas cria a `ChunkSection` sob demanda se o chunk já existir mas a seção-alvo ainda for `null`.

Determinação de arquitetura (sem nova DEC)

- **Contrato "chunk desconhecido → no-op"** replicado fielmente de `World.SetBlockAndData` — `Mundo.definirBloco` não lança exceção nem cria um chunk fantasma quando a posição não está carregada, mesmo espírito do contrato "entidade desconhecida → no-op" já estabelecido no Incremento 6.2.
- **Id de 12 bits preservado sem truncar**, pela mesma razão já registrada no Incremento 7.1 (nenhum bloco vanilla usa id acima de 255).
- **Conversão de coordenada local→absoluta feita no `ReceptorMultiBlockChange`**, não no Codec: `MultiBlockChangePacket` mantém `chunkX`/`chunkZ` brutos e `MudancaDeBloco` mantém `xLocal`/`zLocal` locais (fiéis ao formato de fio), evitando qualquer ambiguidade de round-trip com lista vazia; o Receptor multiplica `chunkX`/`chunkZ` por 16 uma única vez e soma o offset local por registro — aritmética trivial no limite entre formato de fio e chamada de domínio, mesmo tipo de ajuste já feito por outros Receptores (ex.: `ReceptorRespawn` estreitando `int`→`byte`).
- **Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova.**

Entregue

- `domain.protocol.v1_8`: `BlockChangePacket`/`BlockChangeCodec`/`BlockChangeHandler`/`EventoBlockChange`/`ReceptorBlockChange` (PLAY, id `0x23`, CLIENTBOUND); `MudancaDeBloco` (record `xLocal`/`y`/`zLocal`/`bloco`), `MultiBlockChangePacket`/`MultiBlockChangeCodec`/`MultiBlockChangeHandler`/`EventoMultiBlockChange`/`ReceptorMultiBlockChange` (PLAY, id `0x22`, CLIENTBOUND). Ambos os Receptores chamam `sessaoDeJogo.mundo().definirBloco(...)`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com os dois pacotes (ids `0x22`/`0x23`, sem colisão).
- Testes novos (consolidado com 7.1): `BlockChangeCodecTest` (round-trip, coordenadas extremas ±30.000.000, formato de id/metadata), `BlockChangeHandlerTest`, `ReceptorBlockChangeTest` (chunk carregado e chunk desconhecido → no-op), `MultiBlockChangeCodecTest` (round-trip, empacotamento de posição local, lista vazia), `MultiBlockChangeHandlerTest`, `ReceptorMultiBlockChangeTest` (conversão local→absoluta correta, chunk desconhecido → no-op); extensões em `RegistroDePacotesV1_8Test` (3 lookups), `PipelineDeProtocoloV1_8Test` (3 cenários de pipeline completa) e `AdaptadorConexaoBotV1_8Test` (`deveRefletirChunkDataEMutacoesDeBlocoNoMundo` — integração real ponta a ponta: Chunk Data carrega uma coluna, Block Change e Multi Block Change mutam células individuais, tudo refletido em `sessaoDeJogo.mundo()`).

Restrições respeitadas

Nenhuma automação, mineração, colocação/quebra de bloco por comando, pathfinding ou navegação — mesma restrição do Incremento 7.1.

Riscos e observações

Nenhum risco novo além dos já registrados no Incremento 7.1 (mesma base de dados/agregado).

Validação executada (7.1 + 7.2, sessão combinada)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **420 testes executados, 0 falhas, 3 skipped deliberadamente**. Nenhuma interface já aprovada foi alterada (`Codec`, `PacketHandler`, `ReceptorDeEvento`, `RegistroDePacotes`, `EventoDeProtocolo`, `ConexaoBotPort`, `ConexaoMinecraft`, `LeitorDePacote`, `EscritorDePacote` permanecem exatamente como antes) — mudança 100% aditiva.

Próximo passo sugerido (histórico)

Map Chunk Bulk (`0x26`) implementado no Incremento 7.3, reaproveitando o decodificador de coluna deste incremento, conforme já previsto aqui. Demais candidatos não comprometidos, a critério do responsável do projeto: Explosion (`0x27`, reaproveitando a mutação do Incremento 7.2, sem aplicar knockback ao próprio bot — sem motor de física), Time Update/Change Game State (estado global pequeno, com a ressalva de que Change Game State tem semântica mista Jogador/Mundo a decidir no momento da implementação), Update Sign, e os pacotes sem precedente no C# (Update Block Entity, Block Action, Block Break Animation, World Border) validados contra a especificação do protocolo 47. Nenhuma DEC pendente bloqueia.

### Incremento 7.3 — Map Chunk Bulk

Status

Concluído

Objetivo

Implementar a carga em lote de múltiplas colunas completas (Map Chunk Bulk, `0x26`, PLAY/CLIENTBOUND), reaproveitando o decodificador de seções do Incremento 7.1, conforme já previsto no "Próximo passo sugerido" do Incremento 7.2.

Análise do legado

`Handler_v18.P26ReadMapChunkBulk` lê `skyLightSent` (bool) **uma única vez, valendo para todas as colunas do pacote** — diferente do Chunk Data, onde a presença de sky light é inferida da dimensão atual (`Client.Dimension == 0`), não lida do próprio pacote. Em seguida lê a contagem de colunas (`VarInt`) e, **primeiro**, os cabeçalhos de todas as colunas em sequência (`x:int`, `z:int`, `primaryBitMask:ushort`); **só depois**, os dados de cada coluna são lidos em sequência, com o tamanho de cada bloco de dados calculado via `CalcChunkSize(HammingWeight32(bitmask), skyLightSent, hasBiomes:true)` — não prefixado por `VarInt` como no Chunk Data. Cada coluna é sempre tratada como coluna completa (`ReadChunk(..., fullChunk:true, ...)`, hardcoded) — não existe conceito de coluna parcial em Map Chunk Bulk.

Determinação de arquitetura (sem nova DEC)

- **`SecoesDeChunkCodec` extraído** (`domain.protocol.v1_8`, package-private, sem estado) a partir da lógica que antes vivia inline em `ChunkDataCodec` (decode/encode de seções little-endian por bitmask) — reaproveitado por ambos os Codecs agora que um segundo consumidor real precisa da mesma lógica; `ChunkDataCodec` refatorado para delegar a ele, sem alteração de comportamento (`ChunkDataCodecTest` permanece verde, sem alterações). Mesmo padrão de extração já usado para `AnguloCodec`/`MetadataDeEntidadeCodec` — construir o compartilhamento quando o segundo consumidor real aparece, não antes.
- **`Mundo.registrarColunaCompleta(chunkX, chunkZ, secoes)` extraído** — a lógica de "criar um `Chunk` novo, popular as seções presentes e registrar" que antes vivia inline em `ReceptorChunkData` (ramo de coluna completa) passa a viver em `Mundo`, reaproveitada por `ReceptorChunkData` e pelo novo `ReceptorMapChunkBulk`. `ReceptorChunkData` refatorado para delegar a ele, sem alteração de comportamento (`ReceptorChunkDataTest` permanece verde).
- **`ColunaDeChunk`** (novo record `chunkX`/`chunkZ`/`mascaraDeBits`/`secoes`, com `equals`/`hashCode` via `Arrays.deepEquals`, mesmo padrão de `ChunkDataPacket`) representa uma coluna dentro do lote — reaproveitável se outro pacote de mundo precisar do mesmo formato de coluna no futuro.
- **`MapChunkBulkCodec.encode` preenche com zero o restante de cada coluna** (block light/sky light/bioma, não modelados) até o tamanho total calculado por `SecoesDeChunkCodec.calcularTamanho` — necessário para que o próprio round-trip (`encode`+`decode`) permaneça alinhado entre colunas, já que `decode` não lê um tamanho prefixado e sim calcula quantos bytes ler; mesmo idioma de `SpawnMobCodec.encode` escrevendo velocidade/metadata não modeladas como zero.
- **Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova.**

Entregue

- `domain.protocol.v1_8.SecoesDeChunkCodec` (novo, package-private): `decode`/`encode`/`calcularTamanho`, extraído de `ChunkDataCodec`.
- `domain.protocol.v1_8.ColunaDeChunk` (novo record); `MapChunkBulkPacket`/`MapChunkBulkCodec`/`MapChunkBulkHandler`/`EventoMapChunkBulk`/`ReceptorMapChunkBulk` (PLAY, id `0x26`, CLIENTBOUND). `ReceptorMapChunkBulk` chama `sessaoDeJogo.mundo().registrarColunaCompleta(...)` por coluna.
- `domain.bot.Mundo.registrarColunaCompleta` (novo método, reaproveitado por `ReceptorChunkData` e `ReceptorMapChunkBulk`).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `MapChunkBulkPacket` (id `0x26`, sem colisão).
- 12 testes novos: `MapChunkBulkCodecTest` (round-trip sem/com sky light, lista vazia, estrutura de cabeçalhos-antes-dos-dados), `MapChunkBulkHandlerTest`, `ReceptorMapChunkBulkTest` (múltiplas colunas, substituição completa de chunk existente); `MundoTest` (2 testes de `registrarColunaCompleta`); extensões em `RegistroDePacotesV1_8Test` (1 lookup), `PipelineDeProtocoloV1_8Test` (1 cenário) e `AdaptadorConexaoBotV1_8Test` (`deveRegistrarMultiplasColunasDoMapChunkBulkNoMundo` — integração real ponta a ponta com 2 colunas).

Restrições respeitadas

Nenhuma automação, mineração, colocação/quebra de bloco por comando, pathfinding ou navegação. Bioma/block light/sky light consumidos (na decodificação) ou preenchidos com zero (na codificação) sem serem expostos como dado de domínio — mesmo idioma do Incremento 7.1.

Riscos e observações

Mesmos riscos já registrados no Incremento 7.1 (mesma base de dados/agregado — crescimento de memória sem eviction, `readByteArray` sem validação de limite). Nenhum risco novo introduzido por este incremento.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **432 testes executados, 0 falhas, 3 skipped deliberadamente**. Nenhuma interface já aprovada foi alterada — mudança 100% aditiva (as duas extrações/refatorações internas — `SecoesDeChunkCodec`, `Mundo.registrarColunaCompleta` — não alteraram nenhum contrato público nem quebraram nenhum teste existente).

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: Explosion (`0x27`), Time Update (`0x03`)/Change Game State (`0x2B`), Update Sign (`0x33`), e os pacotes sem precedente no C# (Update Block Entity, Block Action, Block Break Animation, World Border). Nenhuma DEC pendente bloqueia.

Observação de numeração (não bloqueante)

Este trabalho foi solicitado e conduzido como "Milestone 7". A última seção de milestone deste documento antes desta é "Milestone 5" (cujo Incremento 6 — Entidades do Mundo — é a entrega mais recente registrada); não existe uma seção "Milestone 6" formal neste documento. Esta seção foi adicionada como "Milestone 7" conforme solicitado, sem renomear ou reestruturar retroativamente o conteúdo já existente da Milestone 5 — a reconciliação de numeração, se desejada, fica a critério do responsável do projeto.

---

### Incremento 7.4 — Explosion

Status

Concluído

Objetivo

Implementar Explosion (`0x27`, PLAY/CLIENTBOUND) — o único pacote de mundo, entre os candidatos restantes ao final do Incremento 7.3, que já possui consumo real e não trivial no legado (mutação de blocos + motion do jogador local).

Análise do legado

`case 0x27` inline em `Handler_v18.HandlePacket` (sem método dedicado, ao contrário de `P21`/`P26`), gated por `if (!Client.MapAndPhysics) break;`. Lê `x`/`y`/`z` (`float`), `radius` (`float`, lido e **descartado** — nunca atribuído a variável nem usado), `count` (**`int` de 4 bytes, não VarInt**), e então `count` registros de 3 `sbyte` (offsets relativos `deltaX`/`deltaY`/`deltaZ`); cada bloco afetado é somado a `(int)x`/`(int)y`/`(int)z` (truncamento, não arredondamento) e zerado via `World.SetBlockAndData(bx, by, bz, 0, 0)`. Por fim lê `motionX`/`motionY`/`motionZ` (`float`), somados a `Player.MotionX/Y/Z` (acumulado, não substituído).

Determinação de arquitetura (sem nova DEC)

- **`RegistroDeExplosao`** (novo record `deltaX`/`deltaY`/`deltaZ`, `byte`) representa cada offset relativo — dado bruto de fio, mesmo padrão de `MudancaDeBloco`: a conversão para coordenada absoluta do mundo (`base + delta`) acontece no `ReceptorExplosion`, não no Codec.
- **`radius` consumido no Codec sem ser exposto** em `ExplosionPacket` — mesmo idioma "consumir sem expor" já usado para bioma/sky light/block light em `SecoesDeChunkCodec` e NBT em `ItemStackCodec`; no `encode`, grava-se `0f` como placeholder (nenhum consumidor real depende do valor de fio, assim como no legado).
- **`SessaoDeJogo` ganha `motionX`/`motionY`/`motionZ` (`double`) + `aplicarImpulsoDeExplosao(dx, dy, dz)`** — primeira modelagem de motion do jogador local no Java. Adição aditiva que preserva fielmente `Player.MotionX/Y/Z += ...` do legado; nenhuma física nova foi introduzida, apenas o dado passa a ser retido (antes não havia nenhum consumidor desse valor no Java, então nada se perde por não existir ainda nenhuma simulação de física).
- **Divergência documentada**: a flag `Client.MapAndPhysics` do legado (que pode pular o pacote inteiro) não tem equivalente na modelagem Java atual — nenhum outro Receptor de Mundo já implementado (Chunk Data/Block Change/Multi Block Change/Map Chunk Bulk) possui um gate de feature flag equivalente. Para manter consistência com o padrão já estabelecido nesta milestone, `ReceptorExplosion` aplica a mutação incondicionalmente. Se essa flag for necessária no futuro, cabe avaliação e possivelmente uma DEC própria — não introduzida aqui por ausência de precedente ou necessidade concreta.
- Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova.

Entregue

- `domain.protocol.v1_8.RegistroDeExplosao` (novo record); `ExplosionPacket`/`ExplosionCodec`/`ExplosionHandler`/`EventoExplosion`/`ReceptorExplosion` (PLAY, id `0x27`, CLIENTBOUND).
- `domain.bot.SessaoDeJogo`: campos `motionX`/`motionY`/`motionZ` + método `aplicarImpulsoDeExplosao`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `ExplosionPacket` (id `0x27`, sem colisão).
- 13 testes novos: `ExplosionCodecTest` (round-trip, lista vazia, radius descartado, count como int de 4 bytes), `ExplosionHandlerTest`, `ReceptorExplosionTest` (bloco zerado com chunk carregado, no-op sem chunk, offsets relativos ao centro, acúmulo de motion); extensão em `SessaoDeJogoTest` (1); extensões em `RegistroDePacotesV1_8Test` (1 lookup) e `PipelineDeProtocoloV1_8Test` (1 cenário). O teste de integração real ponta a ponta é compartilhado com o Incremento 7.5 (ver abaixo).

Restrições respeitadas

Nenhuma automação, mineração, colocação/quebra de bloco por comando, pathfinding ou navegação. `radius` consumido sem ser exposto como dado de domínio, mesmo idioma dos incrementos anteriores.

Riscos e observações

Mesmos riscos já registrados nos Incrementos 7.1–7.3 (mesma base de dados/agregado `Mundo`). Nenhum risco novo introduzido por este incremento.

Validação executada

Ver validação consolidada ao final do Incremento 7.5 (implementados na mesma sessão).

---

### Incremento 7.5 — Change Game State (Time Update e Spawn Position avaliados e mantidos não registrados)

Status

Concluído — com uma divergência de escopo deliberada e documentada abaixo (Time Update e Spawn Position).

Objetivo

Avaliar e, quando aplicável, implementar Time Update (`0x03`), Spawn Position (`0x05`) e Change Game State (`0x2B`) para o bounded context Mundo/estado do jogador.

Análise do legado (achado crítico)

Busca exaustiva nos 4 projetos C# disponíveis (`AdvancedBot 2.8 - Projeto`, `Projeto Adv 2.4.4`, `Projeto Adv 2.4.5`, `Projeto Adv 2.4.5 - Anthack`) confirma que **Time Update (`0x03`) e Spawn Position (`0x05`) não possuem nenhum `case` no switch de `Handler_v18.HandlePacket`** em nenhuma das quatro versões — não existe nenhuma variável `worldAge`/`timeOfDay`/`spawnX`/`spawnY`/`spawnZ` em lugar nenhum do código-fonte do bot legado. Os pacotes chegam e são descartados por completo (cada `ReadBuffer` já é framed/isolado por pacote via `MinecraftStream.ReadPacket`, então a ausência de leitura não desalinha os pacotes seguintes). Já Change Game State (`0x2B`) tem `case` inline: lê `reason` (`byte`) sempre, e `value` (`float`) **apenas quando `reason==3`**, atualizando `Client.Gamemode = (int)value`; as demais 9 `reason` possíveis do protocolo vanilla (clima, cama inválida, fade, etc.) são completamente ignoradas, inclusive no wire (o restante do buffer framed é descartado).

Determinação de arquitetura (sem nova DEC)

- **Time Update e Spawn Position permanecem deliberadamente NÃO registrados** em `RegistroDePacotesV1_8` — nenhum Packet/Codec/Handler/Evento/Receptor foi criado para eles nesta etapa. Esta é a fidelidade **exata** ao legado: como nenhuma versão do C# jamais leu ou usou esses pacotes, "não implementar" reproduz "ignorar por completo" byte a byte, e o mecanismo já existente da DEC-20 (Incremento 7.0) garante o comportamento equivalente em Java — um pacote PLAY sem Codec registrado é descartado (log WARN) e a conexão continua normalmente, sem exceção. Criar decodificação e/ou armazenamento de estado (ex.: `worldAge`, posição de spawn) para pacotes que o legado nunca consumiu constituiria introdução de regra de negócio nova sem validação (vedado pelo CLAUDE.md); por isso nenhum artefato foi criado para eles. Caso o responsável do projeto queira tracking de spawn point ou de tempo de mundo no futuro, isso deve ser tratado como uma decisão arquitetural nova (possível DEC), não como continuação de fidelidade ao legado.
- **Change Game State**: o wire é fixo (`reason:ubyte` + `value:float`, sempre os dois presentes — não é um formato variável). O Codec sempre lê os dois campos (fidelidade de fio, necessária para round-trip), mas o `ReceptorChangeGameState` só aplica a mutação (`SessaoDeJogo.atualizarGamemode`) quando `reason==3`, replicando exatamente a única `reason` que o legado já tratava. As demais `reason` são decodificadas (chegam ao Evento) mas não produzem efeito algum, tal como no C#.
- **`SessaoDeJogo.atualizarGamemode(byte)`** — novo método dedicado; `gamemode` já existia como campo, mas só era mutável em conjunto com `entityId`/`dimension` via `registrarEntradaNoJogo`/`registrarRespawn`. Mudança aditiva, sem alterar nenhum contrato existente.
- Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova.

Entregue

- `domain.protocol.v1_8.ChangeGameStatePacket`/`ChangeGameStateCodec`/`ChangeGameStateHandler`/`EventoChangeGameState`/`ReceptorChangeGameState` (PLAY, id `0x2B`, CLIENTBOUND).
- `domain.bot.SessaoDeJogo`: método `atualizarGamemode`.
- Time Update (`0x03`) e Spawn Position (`0x05`): **nenhum artefato criado** — decisão documentada acima; permanecem cobertos pelo descarte seguro da DEC-20.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `ChangeGameStatePacket` (id `0x2B`, sem colisão).
- 8 testes novos: `ChangeGameStateCodecTest` (round-trip, `value` lido mesmo quando `reason != 3`), `ChangeGameStateHandlerTest`, `ReceptorChangeGameStateTest` (atualiza gamemode quando `reason==3`, ignora nas demais); extensão em `SessaoDeJogoTest` (1); extensões em `RegistroDePacotesV1_8Test` (1 lookup) e `PipelineDeProtocoloV1_8Test` (1 cenário).
- 1 teste de integração real ponta a ponta **compartilhado com o Incremento 7.4** — `AdaptadorConexaoBotV1_8Test.deveRefletirExplosionEChangeGameStateNaSessao`: via socket loopback, envia Chunk Data + Explosion + Change Game State em sequência e verifica simultaneamente o bloco zerado, o motion acumulado e o gamemode atualizado na mesma `SessaoDeJogo`.

Restrições respeitadas

Nenhuma automação, mineração, colocação/quebra de bloco por comando, pathfinding ou navegação. Nenhum dado do C# convertido automaticamente. As 9 `reason` não tratadas de Change Game State permanecem fielmente ignoradas, como no legado. Time Update/Spawn Position deliberadamente não implementados, fiel ao legado.

Riscos e observações

Mesmos riscos já registrados nos incrementos anteriores de Mundo (crescimento de memória sem eviction em `Mundo`/`SessaoDeJogo`, `readByteArray` sem validação de limite). Nenhum risco novo. A ausência de Time Update/Spawn Position significa que o bot não terá noção de spawn point nem de ciclo dia/noite até uma decisão arquitetural explícita futura — não bloqueia nenhuma funcionalidade já implementada.

Validação executada (Incrementos 7.4 e 7.5, implementados na mesma sessão)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **453 testes executados, 0 falhas, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva: nenhuma interface já aprovada foi alterada, nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: Update Sign (`0x33`), e os pacotes sem precedente no C# (World Border, Update Block Entity, Block Action, Block Break Animation — validar contra a especificação oficial do protocolo 47 antes de implementar); ou outras frentes da Milestone 5 (chat enviado pelo bot, Player List/tab list, combate/automação). Se o responsável do projeto quiser tracking de spawn point ou de tempo de mundo (Time Update/Spawn Position), isso requer uma decisão arquitetural explícita (possível DEC) antes de qualquer implementação — não é uma continuação natural de fidelidade ao legado, já que o legado nunca implementou esses dois pacotes. Nenhuma DEC pendente bloqueia os demais candidatos.

---

## Milestone 8

Status

Em andamento — Incrementos 8.1 (DEC-21), 8.2 (Player List) e 8.3 (Chat Enviado pelo Bot) concluídos.

Objetivo

Consolidar as frentes remanescentes de Jogador identificadas ao final da Milestone 7 (Player List/tab list) e abrir, pela primeira vez, o sentido bot→servidor do Play State (Chat Enviado pelo Bot), formalizando previamente o papel do Caso de Uso nesse novo sentido (DEC-21).

### Incremento 8.1 — DEC-21: Papel do Caso de Uso em Ações Iniciadas pelo Bot

Status

Concluído

Objetivo

Formalizar, como decisão arquitetural, o fluxo `CasoDeUso → SessaoDeJogo → ConexaoMinecraft → Packet → Servidor` para ações iniciadas pelo bot, complementando o fluxo reativo a protocolo já formalizado pela DEC-19 (`Servidor → Packet → Codec → Handler → Evento → Receptor → SessaoDeJogo`), antes de implementar Chat Enviado pelo Bot.

Resultado

Aprovado (ver [DEC-21](01-Decisoes-Arquiteturais.md))

Entregue

- **DEC-21**: formaliza que um Caso de Uso (`application.usecase`) é criado apenas quando uma ação é iniciada por algo externo ao protocolo (o bot, um usuário, um futuro script), nunca para reação a um `EventoDeProtocolo`; o Caso de Uso recebe um `Bot`, valida que `sessaoDeJogo` não é nula, e invoca um método de intenção em `SessaoDeJogo`, que constrói o `Packet` e chama `conexao.send(...)` diretamente — sem nenhum Port novo, reaproveitando o mesmo mecanismo já usado desde a Milestone 5 por `responderKeepAlive`/`atualizarPosicao`. Nenhuma DEC anterior foi contrariada — a DEC-19 já antecipava textualmente este exato padrão.

Observação de escopo

Exclusivamente documental — nenhuma classe, teste ou linha de código de produção foi criada ou alterada nesta fase.

Validação executada

Não aplicável a código (fase exclusivamente documental).

### Incremento 8.2 — Lista de Jogadores (Player List)

Status

Concluído

Objetivo

Implementar o pacote Player List (`0x38`, PLAY/CLIENTBOUND), rastreando nome e nome de exibição dos jogadores conhecidos pelo bot, replicando fielmente `Handler_v18.ReadPlayerList`.

Análise do legado

`ReadPlayerList` multiplexa 5 sub-ações num único pacote via um campo `action:VarInt` que vale para todas as entradas do pacote (`count:VarInt` entradas idênticas em formato): **0 — adicionar jogador** (UUID, nome, lista de properties de perfil — sempre descartada, nunca usada para textura/skin —, gamemode e ping — ambos lidos e descartados, gap pré-existente do legado —, e nome de exibição opcional); **1 — atualizar gamemode** (UUID + gamemode, ambos lidos e descartados por completo); **2 — atualizar ping** (UUID + ping, idem); **3 — atualizar nome de exibição** (UUID + nome de exibição opcional); **4 — remover jogador** (UUID). Apenas nome e nome de exibição são efetivamente retidos (`PlayerNick.Name`/`DisplayName`).

Divergência real entre legado e protocolo (documentada, não é regra de negócio replicável)

O ramo do legado para a ação 3 (`case 3`) contém um bug de alinhamento de leitura: quando o UUID ainda não é conhecido, o código lê a string do nome de exibição **sem antes ler o byte booleano "tem nome de exibição"** que o protocolo sempre envia para essa ação — o byte booleano é incorretamente consumido como parte do VarInt de comprimento da string seguinte, o que desalinhraria a leitura de qualquer entrada seguinte no mesmo pacote caso o C# fosse usado como fonte literal de fio. Por fidelidade ao **formato de fio** (não ao bug), `PlayerListItemCodec` sempre lê o booleano antes do texto opcional, para todas as entradas da ação 3, no mesmo espírito de "Codec nunca pula bytes obrigatórios do protocolo com base em estado de domínio" já estabelecido pelos precedentes de Change Game State (Milestone 7, Incremento 7.5) e Update Health/Respawn. A intenção de negócio do legado (criar uma entrada quando o UUID é desconhecido) foi preservada: `ListaDeJogadores.atualizarNomeDeExibicao` cria uma entrada com `nome=null` quando o UUID não é conhecido.

Determinação de arquitetura (sem nova DEC)

- **Player List não é um novo bounded context** — mesmo critério já aplicado a Inventário/Entidades/Mundo: `ListaDeJogadores` é um agregado interno de `SessaoDeJogo`, sem persistência, Port ou Caso de Uso próprios.
- **`ItemDeListaDeJogadores`** (novo `sealed interface`, `domain.protocol.v1_8`) com 5 variantes (`ItemAdicionarJogador`, `ItemAtualizarGamemode`, `ItemAtualizarPing`, `ItemAtualizarNomeDeExibicao`, `ItemRemoverJogador`) — mesmo padrão de tipos selados já usado por `EntidadeRemota`/`EntidadeJogadorRemoto`/`EntidadeMob`, aproveitando `switch` exaustivo do Java 21 no Codec e no Receptor.
- **`JogadorConhecido`** (`domain.bot`, record `nome`/`nomeDeExibicao`) e **`ListaDeJogadores`** (`domain.bot`, `Map<UUID, JogadorConhecido>` concorrente) — mesmo padrão de `EntidadesDoMundo`.
- **Properties de perfil (textura/skin) consumidas sem serem expostas** — mesmo idioma já usado para NBT (`ItemStackCodec`) e bioma/luz (`SecoesDeChunkCodec`).
- **Gamemode/ping (ações 1 e 2) decodificados no Codec mas descartados no Receptor** — fidelidade ao gap pré-existente do legado, mesmo padrão de `RespawnPacket.difficulty`/`levelType`.
- Nenhum Caso de Uso novo, nenhum Port novo, nenhuma DEC nova — é reação a protocolo (Receptor→`SessaoDeJogo` direto), não ação iniciada pelo bot.

Entregue

- `domain.protocol.v1_8`: `ItemDeListaDeJogadores` (sealed) + 5 variantes; `PlayerListItemPacket`/`PlayerListItemCodec`/`PlayerListItemHandler`/`EventoPlayerListItem`/`ReceptorPlayerListItem` (PLAY, id `0x38`, CLIENTBOUND).
- `domain.bot`: `JogadorConhecido` (record), `ListaDeJogadores` (novo agregado). `SessaoDeJogo` ganha o campo final `listaDeJogadores` e o acessor `listaDeJogadores()`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `PlayerListItemPacket` (id `0x38`, sem colisão).
- 24 testes novos: `ListaDeJogadoresTest` (7), `PlayerListItemCodecTest` (9, incluindo teste dedicado provando que as properties de uma entrada não corrompem a leitura da entrada seguinte no mesmo pacote), `PlayerListItemHandlerTest` (1), `ReceptorPlayerListItemTest` (5); extensões em `RegistroDePacotesV1_8Test` (1 lookup por Codec) e `PipelineDeProtocoloV1_8Test` (1 cenário completo); 2 testes de integração real ponta a ponta em `AdaptadorConexaoBotV1_8Test` (adicionar+atualizar nome refletidos na lista; remoção refletida na lista).

Restrições respeitadas

Nenhuma UI, nenhum consumo de textura/skin, nenhuma automação. Nenhum outro pacote de Mundo/Jogador foi tocado nesta etapa.

Riscos e observações

- `ListaDeJogadores` não tem eviction além da remoção explícita (ação 4) — diferente de `Mundo`/`EntidadesDoMundo`, que não têm nenhum mecanismo de remoção automática por desconexão de entidade; risco de memória menor que os agregados anteriores.
- Divergência do bug de alinhamento da ação 3 documentada acima — não afeta nenhum teste existente, apenas registrada para rastreabilidade futura caso o legado seja consultado novamente para esse pacote.

Validação executada

Ver validação consolidada ao final do Incremento 8.3 (implementados na mesma sessão).

### Incremento 8.3 — Chat Enviado pelo Bot

Status

Concluído

Objetivo

Implementar o primeiro fluxo de ação iniciada pelo bot (DEC-21): envio bruto de uma mensagem de chat (Chat Message serverbound, `0x01`), sem parser de comandos, automações ou plugins.

Análise do legado

`MinecraftClient.SendMessage` (`AdvancedBot.Client.MinecraftClient.cs`) trunca a mensagem para 99 caracteres quando o comprimento excede 100 (`msg.Length > 100 ? msg.Substring(0, 99) : msg` — um off-by-one do próprio legado, não corrigido, por fidelidade), ignora silenciosamente mensagens vazias (`msg.Length <= 0`), e delega a hooks de plugin (`onSendChat`) e roteamento de comandos locais prefixados por `$` (`CmdManager.RunCommand`) — ambos automação/scripting, fora do escopo autorizado. O wire do Chat Message serverbound (`PacketChatMessage.cs`, id `0x01`) é apenas `VarInt id + String mensagem` — sem o byte de posição do Chat Message clientbound (`0x02`).

Determinação de arquitetura (sem nova DEC)

- **`EnvioDeChatPacket`** (nome distinto de `ChatMessagePacket`, mesmo critério já usado para `RespostaKeepAlivePacket`/`ConfirmacaoDePosicaoPacket`) — Record com apenas `mensagem`, sem campo de posição.
- **Regra de truncamento (99 caracteres) e no-op em mensagem vazia/nula aplicadas em `SessaoDeJogo.enviarMensagem`**, não no Codec — mesma separação já usada por `atualizarPosicao` (regra de negócio em `SessaoDeJogo`, Codec apenas fiel ao formato de fio).
- **`EnvioDeChatHandler`/`EventoEnvioDeChat` criados e registrados em `handlersV1_8()`, sem `ReceptorDeEvento` correspondente** — mesmo precedente exato de `RespostaKeepAlivePacket`/`ConfirmacaoDePosicaoPacket` (pacotes puramente SERVERBOUND nunca são de fato decodificados pelo `readLoop`, que só resolve Codecs CLIENTBOUND; o Handler existe por uniformidade e testabilidade, não por necessidade de despacho em produção).
- **`CasoDeUsoEnviarMensagemDeChat`** (novo, `application.usecase`, primeiro Caso de Uso do Play State) — aplica exatamente o padrão formalizado pela DEC-21: recebe `Bot`, valida `getSessaoDeJogo() != null` (lança `IllegalStateException` caso contrário), chama `sessaoDeJogo.enviarMensagem(mensagem)`. Nenhum import de tipo de protocolo/pacote.
- Nenhum Port novo, nenhuma DEC nova (a DEC-21 já cobre este incremento).

Entregue

- `domain.protocol.v1_8.EnvioDeChatPacket`/`EnvioDeChatCodec`/`EnvioDeChatHandler`/`EventoEnvioDeChat` (PLAY, id `0x01`, SERVERBOUND).
- `domain.bot.SessaoDeJogo.enviarMensagem(String)` — trunca para 99 caracteres acima de 100 (fiel ao legado), no-op em mensagem nula/vazia, chama `conexao.send(new EnvioDeChatPacket(...))`.
- `application.usecase.CasoDeUsoEnviarMensagemDeChat` (novo).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `EnvioDeChatPacket` (id `0x01` SERVERBOUND, sem colisão com `ChatMessagePacket` id `0x02` CLIENTBOUND).
- 15 testes novos: `EnvioDeChatCodecTest` (3), `EnvioDeChatHandlerTest` (1), `CasoDeUsoEnviarMensagemDeChatTest` (2, incluindo o caso de sessão de jogo ausente), 4 em `SessaoDeJogoTest` (envio normal, truncamento, no-op vazio, no-op nulo); extensões em `RegistroDePacotesV1_8Test` (1 lookup, incluindo verificação de não colisão com `ChatMessagePacket`) e `PipelineDeProtocoloV1_8Test` (1 cenário); 1 teste de integração real ponta a ponta em `AdaptadorConexaoBotV1_8Test` (`sessaoDeJogo.enviarMensagem(...)` → bytes corretos recebidos pelo servidor fake via socket loopback).

Restrições respeitadas

Não implementados: comandos, parser de comandos (`$`), automações, hooks de plugin — apenas envio bruto da mensagem.

Riscos e observações

- Nenhuma proteção contra spam/rate-limit no domínio — aceitável, mesma determinação já registrada na fase de planejamento (futuras automações, se autorizadas, cuidam disso).
- Truncamento em 99 (não 100) caracteres é fidelidade deliberada a um off-by-one do legado, não uma escolha de design — documentado aqui para não ser "corrigido" inadvertidamente em um incremento futuro sem decisão explícita.

Validação executada (Incrementos 8.2 e 8.3, implementados na mesma sessão)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **492 testes executados, 0 falhas, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva: nenhuma interface já aprovada foi alterada, nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation) ou restante da Milestone 5 (movimentação livre do bot — bloqueada por uma decisão pendente sobre motor de física, já que o legado acopla o envio de posição/rotação a `Player.Tick()` —, combate/automação — fora de escopo por política do projeto). Nenhuma DEC pendente bloqueia os candidatos que não envolvem física.

---

## Milestone 9

Status

CONCLUÍDA — Incrementos 9.1 (DEC-22), 9.2 (Movimentação) e 9.3 (Rotação) concluídos.

Objetivo

Implementar as capacidades fundamentais de envio de ações do jogador (movimento e rotação) do bot para o servidor — sem automação, física ou Tick loop — para reutilização futura por comandos, macros, scripts e inteligência do bot.

### Incremento 9.1 — DEC-22: Ações Fundamentais do Jogador (Movimentação e Rotação)

Status

Concluído

Objetivo

Analisar a arquitetura existente (`SessaoDeJogo`, `ConexaoMinecraft`, DEC-19/20/21, `RegistroDePacotesV1_8`) e o legado C# (`MinecraftClient.Tick()`, `PacketPlayerPos`/`PacketPlayerLook`/`PacketPosAndLook`/`PacketUpdate`) para decidir como as ações de movimento/rotação do jogador devem ser modeladas, antes de implementar qualquer Packet novo.

Resultado

Aprovado (ver [DEC-22](01-Decisoes-Arquiteturais.md))

Entregue

- **DEC-22**: formaliza que a Milestone 9 implementa exclusivamente a capacidade de envio explícito e sob demanda de ações de movimento/rotação (via Caso de Uso, seguindo o fluxo da DEC-21), nunca o Tick loop automático do legado (motor de física, permanece fora de escopo e não bloqueia mais o candidato "movimentação livre do bot" registrado na Milestone 5/8, já que o bloqueio sempre foi sobre a automação, não sobre a capacidade de envio). Decide reaproveitar `ConfirmacaoDePosicaoPacket`/`ConfirmacaoDePosicaoCodec` (id `0x06` SERVERBOUND, já registrado) para qualquer futuro caso combinado de posição+rotação em vez de introduzir uma segunda classe na mesma chave — `RegistroDePacotesV1_8.registrar` indexa um único `Codec` por `(EstadoConexao, id, SentidoDoPacote)`, e `TransporteSocket.send` resolve o `Codec` de envio só por essa chave, nunca por `Class`; uma segunda classe no id `0x06` sobrescreveria silenciosamente o `Codec` do eco de confirmação já validado. Decide mutação otimista de estado (`SessaoDeJogo.x/y/z/yaw/pitch` atualizados no envio, mesmo padrão já usado por `atualizarPosicao`). `Player` (id `0x03`, somente `OnGround`) permanece deliberadamente fora do escopo — ligado ao Tick automático do legado, sem precedente de uso isolado.

Observação de escopo

Exclusivamente documental — nenhuma classe, teste ou linha de código de produção foi criada ou alterada nesta fase.

Validação executada

Não aplicável a código (fase exclusivamente documental).

### Incremento 9.2 — Movimentação do Jogador

Status

Concluído

Objetivo

Implementar o pacote Player Position (`0x04`, PLAY/SERVERBOUND), permitindo que `SessaoDeJogo` envie a posição do bot ao servidor sob demanda, conforme DEC-22.

Análise do legado

`MinecraftClient.Tick()` envia `PacketPlayerPos` (`X`, `FeetY`, `Z`, `OnGround`) quando só a posição mudou no tick. `SessaoDeJogo.y` já armazena a coordenada de pés diretamente (populada por `PlayerPositionAndLookPacket` clientbound, sem offset de olho), logo `x`/`y`/`z` mapeiam 1:1 para o formato de fio, sem a conversão `FeetY = Y - 1.62` que o C# aplica sobre seu próprio `Player.PosY` (rastreado como olho internamente no legado).

Entregue

- `domain.protocol.v1_8.PlayerPositionPacket`/`PlayerPositionCodec`/`PlayerPositionHandler`/`EventoPlayerPosition` (PLAY, id `0x04`, SERVERBOUND).
- `domain.bot.SessaoDeJogo.mover(double x, double y, double z, boolean onGround)` — muta `x`/`y`/`z` (mutação otimista, DEC-22) e envia `PlayerPositionPacket`.
- `application.usecase.CasoDeUsoMoverJogador` (novo) — segue o padrão da DEC-21: recebe `Bot`, valida `getSessaoDeJogo() != null`, chama `sessaoDeJogo.mover(...)`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `PlayerPositionPacket` (id `0x04` SERVERBOUND — sem colisão com `EntityEquipmentPacket`, que usa o mesmo id `0x04` mas CLIENTBOUND, validado por teste dedicado).

Restrições respeitadas

Nenhuma física, nenhum Tick automático, nenhuma validação de limites de movimento (velocidade, colisão) — envio explícito e sob demanda apenas, conforme DEC-22.

### Incremento 9.3 — Rotação do Jogador (Look)

Status

Concluído

Objetivo

Implementar o pacote Player Look (`0x05`, PLAY/SERVERBOUND), seguindo exatamente o mesmo padrão arquitetural do Incremento 9.2.

Análise do legado

`MinecraftClient.Tick()` envia `PacketPlayerLook` (`Yaw`, `Pitch`, `OnGround`) quando só a rotação mudou no tick — sem nenhuma conversão de campo (diferente de posição, `Yaw`/`Pitch` não têm equivalente ao offset olho/pés).

Entregue

- `domain.protocol.v1_8.PlayerLookPacket`/`PlayerLookCodec`/`PlayerLookHandler`/`EventoPlayerLook` (PLAY, id `0x05`, SERVERBOUND).
- `domain.bot.SessaoDeJogo.olhar(float yaw, float pitch, boolean onGround)` — muta `yaw`/`pitch` (mutação otimista, DEC-22) e envia `PlayerLookPacket`.
- `application.usecase.CasoDeUsoRotacionarJogador` (novo) — mesmo padrão de `CasoDeUsoMoverJogador`.
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` atualizados com `PlayerLookPacket` (id `0x05` SERVERBOUND, sem colisão).

Restrições respeitadas

Idênticas ao Incremento 9.2.

Riscos e observações (Incrementos 9.2 e 9.3)

- Mutação otimista de estado assume que o servidor aceita o movimento/rotação enviados; se o servidor corrigir (anti-cheat), a próxima `PlayerPositionAndLookPacket` clientbound já sobrescreve `x`/`y`/`z`/`yaw`/`pitch` corretamente via `atualizarPosicao` (comportamento pré-existente, nenhuma reconciliação nova foi necessária).
- Nenhum método combinado "mover e olhar" iniciado pelo bot foi criado — registrado na DEC-22 que, se necessário no futuro, deve reaproveitar `ConfirmacaoDePosicaoPacket`/`ConfirmacaoDePosicaoCodec` (id `0x06`), nunca uma nova classe.
- `Player` (id `0x03`, bare `OnGround`) permanece deliberadamente não implementado — ligado ao Tick automático do legado, sem pedido nesta milestone.

Validação executada (Incrementos 9.2 e 9.3, implementados na mesma sessão)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **511 testes executados, 0 falhas, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva: nenhuma interface já aprovada foi alterada, nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation); combinação posição+rotação reaproveitando `ConfirmacaoDePosicaoPacket` (ver DEC-22), se houver necessidade real; `Player` bare (id `0x03`). Combate/automação/física continuam fora de escopo por política do projeto — Milestone 9 não implementou Tick loop nem motor de física, só a capacidade de envio sob demanda. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 10

Status

CONCLUÍDA — Incrementos 10.1 (planejamento), 10.2 (Swing Arm), 10.3 (Player Digging) e 10.4 (Player Block Placement) concluídos.

Objetivo

Implementar as primeiras interações do jogador com o mundo (balançar braço, escavar bloco, colocar bloco) seguindo exatamente o fluxo de ação iniciada pelo bot já formalizado pela DEC-21/DEC-22 — sem mineração automática, sem física, sem lógica de inventário automático.

### Incremento 10.1 — Planejamento Arquitetural

Status

Concluído

Objetivo

Analisar arquitetura atual (DEC-19/20/21/22, `SessaoDeJogo`, `RegistroDePacotesV1_8`), legado oficial (`Projeto Adv 2.4.5`) e protocolo 47 antes de implementar Swing Arm, Player Digging e Player Block Placement.

Análise

1. **Integração com a arquitetura atual:** as três ações são 100% instâncias do fluxo já formalizado pela DEC-21 (`CasoDeUso → SessaoDeJogo → ConexaoMinecraft → Packet → Servidor`), sem nenhuma reação a protocolo envolvida (nenhuma delas tem contrapartida CLIENTBOUND relevante ao domínio do bot nesta milestone).
2. **Responsabilidades de `SessaoDeJogo`:** único tradutor entre intenção de domínio e protocolo (mesmo papel já fixado pela DEC-19/DEC-21) — ganha métodos de intenção pontuais (`balancarBraco`, `iniciarQuebraDeBloco`/`cancelarQuebraDeBloco`/`finalizarQuebraDeBloco`, `colocarBloco`), nenhum estado observável novo (diferente da Milestone 9, nenhuma das três ações tem um campo cliente-autoritativo análogo a `x`/`y`/`z`/`yaw`/`pitch` para mutar).
3. **Casos de Uso exigidos:** um por ação externa ao protocolo, mesmo critério da DEC-21. Cinco no total: `CasoDeUsoBalancarBraco`, `CasoDeUsoIniciarQuebraDeBloco`, `CasoDeUsoCancelarQuebraDeBloco`, `CasoDeUsoFinalizarQuebraDeBloco`, `CasoDeUsoColocarBloco` — um Caso de Uso por método de intenção (mesmo padrão 1:1 já usado por `CasoDeUsoMoverJogador`/`CasoDeUsoRotacionarJogador`/`CasoDeUsoEnviarMensagemDeChat`, preferido a um único Caso de Uso com 3 métodos para Player Digging, mantendo a consistência já estabelecida em vez de abrir uma exceção pontual).
4. **Novos agregados:** nenhum. Nenhuma das três ações produz estado observável que justifique um novo agregado (diferente de Mundo/Inventário/Entidades/ListaDeJogadores, que existem porque há estado real a rastrear).
5. **Novos Ports:** nenhum. As três ações usam a `ConexaoMinecraft` já retida desde o login (DEC-19), mesmo critério explícito da DEC-21 ("Critérios para NÃO Criar Novos Ports").
6. **Novas DEC:** nenhuma necessária. DEC-21 já cobre integralmente o padrão de "ação iniciada pelo bot"; DEC-22 já cobre o precedente de reaproveitar/estender `RegistroDePacotesV1_8` sem colisão. Nenhuma das três ações introduz uma questão estrutural nova (nenhum reaproveitamento forçado de `Packet` por colisão de chave, nenhuma mutação de estado otimista) que exigisse uma decisão nova — ver item 8 para a única sutileza tratada (sentinela de Block Placement), resolvida sem alterar nenhum contrato.
7. **Padrões reutilizados:** fluxo `CasoDeUso → SessaoDeJogo → Packet` (DEC-21); empacotamento de `Position` (long com x 26 bits/y 12 bits/z 26 bits) já usado por `BlockChangeCodec`/`MultiBlockChangeCodec`; formato `Slot`/`ItemStack` (`ItemStackCodec`) já usado por `WindowItems`/`SetSlot`/`EntityEquipment`; registro de Handler mesmo sem Receptor para pacotes puramente SERVERBOUND, mesmo precedente de `EnvioDeChatPacket`/`PlayerPositionPacket`/`PlayerLookPacket` ("uniformidade/testabilidade").
8. **Divergências entre legado e protocolo:** nenhuma divergência real de campos/ordem/tipos — o legado (`PacketSwingArm.cs`, `PacketPlayerDigging.cs`/`DiggingStatus.cs`, `PacketBlockPlace.cs`) bate exatamente com a especificação oficial do protocolo 47 para os três pacotes. Uma sutileza de fidelidade foi identificada e resolvida (não é divergência entre legado e protocolo, é uma nuance de encoding): o valor sentinela `x=y=z=-1`/`direction=-1` do Player Block Placement ("usar item na mão sem bloco-alvo") exige extensão de sinal do campo `y` (12 bits) na decodificação — diferente de `BlockChangeCodec`, que nunca precisou disso porque `y` ali nunca é negativo. Documentado no Codec, sem necessidade de DEC (não altera nenhum contrato público, é uma correção de fidelidade interna ao novo Codec).
9. **Divisão em incrementos:** 10.1 (este, planejamento) → 10.2 (Swing Arm) → 10.3 (Player Digging: início/cancelamento/término, sem mineração automática) → 10.4 (Player Block Placement, sem lógica de inventário automático).

Resultado

Aprovado. Nenhuma DEC nova, nenhum Port novo, nenhum agregado novo — escopo 100% coberto pelos padrões já formalizados nas DEC-19/20/21/22.

Observação de escopo

Exclusivamente documental — nenhuma classe, teste ou linha de código de produção foi criada ou alterada nesta fase.

### Incremento 10.2 — Swing Arm

Status

Concluído

Objetivo

Implementar o pacote Animation (`0x0A`, PLAY/SERVERBOUND) — ação de balançar o braço, sem nenhum campo no protocolo 47.

Análise do legado

`PacketSwingArm.cs`: para `client.Version >= v1_8`, o pacote é serializado **sem nenhum campo** — apenas o VarInt do id. O campo `EntityID` só existe no fallback para protocolo pré-1.8 (não aplicável aqui). Nome de classe em português (`BalancarBracoPacket`, não `AnimationPacket`) porque o protocolo reutiliza "Animation" nas duas direções — `AnimationPacket` já está registrado para o Entity Animation CLIENTBOUND (`0x0B`, Milestone 5 Incremento 6.4); usar o mesmo nome colidiria como classe Java no pacote `domain.protocol.v1_8` (mesmo critério já usado por `EnvioDeChatPacket`/`ConfirmacaoDePosicaoPacket`/`RespostaKeepAlivePacket`).

Entregue

- `domain.protocol.v1_8.BalancarBracoPacket`/`BalancarBracoCodec`/`BalancarBracoHandler`/`EventoBalancarBraco` (PLAY, id `0x0A`, SERVERBOUND, sem campos).
- `domain.bot.SessaoDeJogo.balancarBraco()` — sem mutação de estado (nenhum campo observável associado), apenas `conexao.send(new BalancarBracoPacket())`.
- `application.usecase.CasoDeUsoBalancarBraco` (conforme DEC-21).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8.handlersV1_8()` atualizados com `BalancarBracoPacket` (id `0x0A` SERVERBOUND — sem colisão, nenhum pacote CLIENTBOUND ocupa `0x0A` nesta milestone).

Testes

`BalancarBracoCodecTest` (round-trip vazio), `BalancarBracoHandlerTest`, `CasoDeUsoBalancarBracoTest` (sucesso + `IllegalStateException` sem sessão), `SessaoDeJogoTest.deveBalancarBracoEnviandoOPacketSemCampos`, cenário de pipeline completa (`PipelineDeProtocoloV1_8Test`), teste de integração ponta a ponta sobre socket loopback real (`AdaptadorConexaoBotV1_8Test.deveEnviarBalancarBracoAoServidorAoChamarBalancarBracoNaSessaoDeJogo`), e localização por id/registro (`RegistroDePacotesV1_8Test`).

### Incremento 10.3 — Player Digging

Status

Concluído

Objetivo

Implementar o pacote Player Digging (`0x07`, PLAY/SERVERBOUND) — início, cancelamento e término de escavação, sem mineração automática.

Análise do legado

`PacketPlayerDigging.cs`/`DiggingStatus.cs`: `status` (byte) + `location` (Position, long empacotado 26/12/26) + `face` (byte), nessa ordem — idêntico à especificação oficial do protocolo 47. O enum `DiggingStatus` do legado tem 6 valores (`StartedDigging=0`, `CancelledDigging=1`, `FinishedDigging=2`, `DropItemStack=3`, `DropItem=4`, `FinishUse=5`); toda a lógica de automação de mineração (`AutoMiner.cs`, `DiggingHelper.cs` — cálculo de força de quebra vs. dureza do bloco, ferramenta, encantamentos, efeitos de poção) vive fora da classe de pacote e **não foi portada**, conforme instrução explícita desta milestone.

Entregue

- `domain.protocol.v1_8.PlayerDiggingPacket`/`PlayerDiggingCodec`/`PlayerDiggingHandler`/`EventoPlayerDigging` (PLAY, id `0x07`, SERVERBOUND; campo `status` cru, sem enum — mesmo critério já usado por `ChangeGameStatePacket.razao`; Codec fiel a todos os 6 valores possíveis de status, mesmo que só 3 tenham método de intenção).
- `domain.bot.SessaoDeJogo`: `iniciarQuebraDeBloco(x,y,z,face)` (status 0), `cancelarQuebraDeBloco(x,y,z,face)` (status 1), `finalizarQuebraDeBloco(x,y,z,face)` (status 2) — três métodos de intenção, sem mutação de estado (nenhum campo observável de "bloco sendo escavado" foi criado; automação/timing de quebra é exatamente o que fica de fora, conforme instrução).
- `application.usecase.CasoDeUsoIniciarQuebraDeBloco`, `CasoDeUsoCancelarQuebraDeBloco`, `CasoDeUsoFinalizarQuebraDeBloco` (um por método de intenção, mesmo padrão 1:1 já usado pelos demais Casos de Uso de PLAY).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `AdaptadorConexaoBotV1_8.handlersV1_8()` atualizados com `PlayerDiggingPacket` (id `0x07` SERVERBOUND — sem colisão com `RespawnPacket`, que usa o mesmo id `0x07` mas CLIENTBOUND, validado por teste dedicado, mesmo padrão já usado para `0x04`/`0x06`).

Restrições respeitadas

Nenhuma mineração automática: nenhum Tick loop, nenhum cálculo de força de quebra vs. dureza de bloco, nenhuma seleção automática de ferramenta, nenhum timer — cada chamada envia exatamente um pacote, sob demanda.

Testes

`PlayerDiggingCodecTest` (round-trip para os 3 status usados, incluindo coordenadas negativas), `PlayerDiggingHandlerTest`, 3 testes de Caso de Uso (`CasoDeUsoIniciarQuebraDeBlocoTest`/`CancelarQuebraDeBlocoTest`/`FinalizarQuebraDeBlocoTest`, sucesso + exceção sem sessão), 3 testes em `SessaoDeJogoTest` (um por status), cenário de pipeline completa, teste de integração ponta a ponta (`AdaptadorConexaoBotV1_8Test.deveEnviarPlayerDiggingAoServidorAoChamarIniciarQuebraDeBlocoNaSessaoDeJogo`), e localização por id/registro (incluindo teste dedicado de não colisão com `RespawnPacket` clientbound).

### Incremento 10.4 — Player Block Placement

Status

Concluído

Objetivo

Implementar o pacote Player Block Placement (`0x08`, PLAY/SERVERBOUND) — colocação de bloco, sem lógica de inventário automático.

Análise do legado

`PacketBlockPlace.cs`: `location` (Position) + `direction` (byte) + `item` (Slot) + `cursorX`/`cursorY`/`cursorZ` (byte cada), nessa ordem — idêntico à especificação oficial do protocolo 47 para `client.Version == v1_8`. O legado usa o valor sentinela `X=Y=Z=-1`/`Direction=byte.MaxValue` (0xFF) para "usar item na mão sem bloco-alvo" (chamado em `LeftClickItem()`/`PlaceCurrentBlock()` para itens especiais como baldes/isqueiro/sementes) — suportado pelo Codec por fidelidade de fio (ver Incremento 10.1, item 8), mas **nenhum método de intenção dedicado a esse caso foi criado** (fora de escopo: "usar item"/inventário automático). O campo `item` reaproveita o mesmo formato "Slot" (`ItemStackCodec`) já usado por `WindowItems`/`SetSlot`/`EntityEquipment`, incluindo suporte a item nulo (slot vazio).

Entregue

- `domain.protocol.v1_8.PlayerBlockPlacementPacket`/`PlayerBlockPlacementCodec`/`PlayerBlockPlacementHandler`/`EventoPlayerBlockPlacement` (PLAY, id `0x08`, SERVERBOUND). Codec com extensão de sinal explícita do campo `y` na decodificação (única diferença técnica em relação a `BlockChangeCodec`/`PlayerDiggingCodec`, documentada no próprio Codec — necessária para representar corretamente o sentinela `-1`).
- `domain.bot.SessaoDeJogo.colocarBloco(x,y,z,direction,item,cursorX,cursorY,cursorZ)` — sem mutação de estado (mundo/blocos permanecem server-authoritative via `ChunkData`/`BlockChange`, nenhuma predição local), sem valores padrão de cursor (caller decide, sem "mágica" oculta).
- `application.usecase.CasoDeUsoColocarBloco` (conforme DEC-21).
- `infrastructure.protocol.v1_8.RegistroDePacotesV1_8` e `AdaptadorConexaoBotV1_8.handlersV1_8()` atualizados com `PlayerBlockPlacementPacket` (id `0x08` SERVERBOUND — sem colisão com `PlayerPositionAndLookPacket`, que usa o mesmo id `0x08` mas CLIENTBOUND, validado por teste dedicado).

Restrições respeitadas

Nenhuma lógica de inventário automático: nenhuma seleção automática de item, nenhum reenvio automático do "segundo Player Block Placement" que o legado faz para itens especiais (baldes/sementes/isqueiro — ver `MinecraftClient.PlaceCurrentBlock`), nenhuma validação de slot ativo. O Caso de Uso recebe o `ItemStack` já pronto do chamador.

Testes

`PlayerBlockPlacementCodecTest` (round-trip com item presente, item nulo, e o sentinela `-1`/`-1`), `PlayerBlockPlacementHandlerTest`, `CasoDeUsoColocarBlocoTest` (sucesso + exceção sem sessão), `SessaoDeJogoTest.deveColocarBlocoEnviandoOPlayerBlockPlacementPacket`, cenário de pipeline completa, teste de integração ponta a ponta (`AdaptadorConexaoBotV1_8Test.deveEnviarPlayerBlockPlacementAoServidorAoChamarColocarBlocoNaSessaoDeJogo`), e localização por id/registro (incluindo teste dedicado de não colisão com `PlayerPositionAndLookPacket` clientbound).

Validação executada (Incrementos 10.2 a 10.4, implementados na mesma sessão)

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **548 testes executados, 0 falhas, 0 erros, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva: nenhuma interface já aprovada foi alterada (`SessaoDeJogo`, `RegistroDePacotesV1_8`, `AdaptadorConexaoBotV1_8` só receberam adições), nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: combinação de posição+rotação reaproveitando `ConfirmacaoDePosicaoPacket` (DEC-22, ainda não implementada); "usar item na mão" (sentinela `-1`/`-1`/`-1` de Player Block Placement, Codec já suporta, falta método de intenção dedicado); `Player` bare (id `0x03`); Entity Action (`0x0B` serverbound — sneak/sprint/leave bed); restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation). Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por instrução explícita desta milestone. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 11

Status

CONCLUÍDA

Objetivo

Ação combinada de posição+rotação iniciada pelo bot (mover e olhar em um único pacote), reaproveitando `ConfirmacaoDePosicaoPacket`/Codec (id `0x06`, PLAY/SERVERBOUND) conforme já decidido pela DEC-22 — candidato escolhido pelo responsável do projeto entre os listados ao encerrar a Milestone 10 (ver Seção 10).

### Incremento 11.1 — Movimentação e Rotação Combinadas (Move And Look)

Status

Concluído

Análise do legado

`AdvancedBot.Client.MinecraftClient.cs` (loop principal de tick, ~linhas 803-820): a cada tick, se `Player.IsPositionChanged` e `Player.IsRotationChanged` forem ambos verdadeiros, o bot envia um único `PacketPosAndLook(PosX, PosY, PosZ, Yaw, Pitch, OnGround)` em vez de `PacketPlayerPos`+`PacketPlayerLook` separados (usados nos ramos `else if` quando só um dos dois mudou) — a mesma decisão de design que a DEC-22 já havia antecipado para o lado Java. `AdvancedBot.Client.Packets.PacketPosAndLook.cs` confirma o mesmo formato de fio já implementado por `ConfirmacaoDePosicaoPacket`/`ConfirmacaoDePosicaoCodec` desde a Milestone 5 Incremento 2 (x, y, z, yaw, pitch, onGround — o campo `FeetY` do C# é um artifício de serialização interno, não exposto ao domínio Java, decisão já tomada naquele incremento e não revisada nesta milestone). Nenhuma divergência nova encontrada entre legado e protocolo; nenhum Packet/Codec novo necessário, confirmando a decisão prévia da DEC-22.

Entregue

- `domain.bot.SessaoDeJogo.moverEOlhar(double x, double y, double z, float yaw, float pitch, boolean onGround)` — mutação otimista de x/y/z/yaw/pitch (mesmo padrão de `mover`/`olhar`), seguida de `conexao.send(new ConfirmacaoDePosicaoPacket(...))`. Nenhum Packet/Codec/Handler/Evento novo — reaproveita o registro já existente em `RegistroDePacotesV1_8` (id `0x06` SERVERBOUND) e o `ConfirmacaoDePosicaoHandler` já ligado em `AdaptadorConexaoBotV1_8.handlersV1_8()` desde a Milestone 5, exatamente como a DEC-22 previu.
- `application.usecase.CasoDeUsoMoverEOlharJogador` (conforme DEC-21) — um Caso de Uso por método de intenção, mesmo padrão 1:1 já usado por `CasoDeUsoMoverJogador`/`CasoDeUsoRotacionarJogador`.
- Nenhuma DEC nova, nenhum Port novo, nenhum agregado novo — escopo 100% coberto pela DEC-21 (fluxo de ação iniciada pelo bot) e pela DEC-22 (reaproveitamento explícito de `ConfirmacaoDePosicaoPacket` para este exato caso, evitando colisão de chave em `RegistroDePacotesV1_8`).
- Mudança 100% aditiva — nenhuma interface já aprovada foi alterada (`SessaoDeJogo`, `RegistroDePacotesV1_8`, `AdaptadorConexaoBotV1_8` só receberam adições).

Testes

`SessaoDeJogoTest.deveMoverEOlharAtualizandoPosicaoERotacaoEEnviandoConfirmacaoDePosicaoPacket`; `CasoDeUsoMoverEOlharJogadorTest` (sucesso + `IllegalStateException` sem sessão); teste de integração ponta a ponta sobre socket loopback real (`AdaptadorConexaoBotV1_8Test.deveEnviarConfirmacaoDePosicaoAoServidorAoChamarMoverEOlharNaSessaoDeJogo`). Nenhum teste de round-trip de Codec novo — `ConfirmacaoDePosicaoCodec` já validado desde a Milestone 5 Incremento 2, sem alteração nesta milestone.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **552 testes executados, 0 falhas, 0 erros, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores).

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: "usar item na mão" (sentinela `-1`/`-1`/`-1` de Player Block Placement, Codec já suporta, falta método de intenção dedicado); `Player` bare (id `0x03`); Entity Action (`0x0B` serverbound — sneak/sprint/leave bed); restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation); criptografia AES-CFB8/modo online; integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()`. Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 12

Status

CONCLUÍDA

Objetivo

Deslocar o foco de novos pacotes de protocolo para a arquitetura de execução de comandos do bot — permitir que uma instrução externa em texto acione as ações já implementadas pelas Milestones 8–11, reutilizando obrigatoriamente `SessaoDeJogo` e `InventarioDoJogador` sempre que possível, sem introduzir nenhum pacote de protocolo, motor de física ou pathfinding novos.

### Incremento 12.1 — Arquitetura de Execução de Comandos (DEC-23)

Status

Concluído

Análise do legado

`AdvancedBot.Client.Commands.ICommand`/`CommandResult` e `AdvancedBot.Client.CommandManagerNew` (`AdvancedBot.Client.CommandManagerNew.cs`): comando modelado como classe abstrata com acesso direto a `Client`/`Player`/`World`, metadados (`DisplayName`/`Description`/`Aliases`/`Parameters`) e três mecanismos exclusivos de automação contínua (`isMacro`, `Toggle()`/`IsToggled`, `Tick()`, chamado a cada tick para todos os comandos registrados). `CommandManagerNew` mantém ~29 comandos, localiza por alias (case-insensitive), despacha com `try/catch` genérico que vira `CommandResult.Error`, e traduz apenas `Error`/`MissingArgs` em mensagem padrão — `Success`/`ErrorSilent` não geram mensagem própria (cada comando imprime seu próprio feedback via `Client.PrintToChat`, saída local do console do operador, não chat do Minecraft).

Decisão (DEC-23, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): `Comando`/`ResultadoComando`/`GerenciadorDeComandos` vivem em `interfaces.comando` (camada já aprovada pela DEC-12, populada pela primeira vez nesta milestone). Contrato mínimo, single-shot (`ResultadoComando executar(Bot, String alias, String[] argumentos)`), sem `Tick`/`Toggle`/`isMacro` — mecanismo de macro fica adiado até o primeiro macro real ser construído, mesma disciplina da "regra de três" já registrada na DEC-18. `ResultadoComando`: `SUCESSO`/`ARGUMENTOS_FALTANDO`/`ERRO`/`NAO_ENCONTRADO` (substitui `Success`/`MissingArgs`/`Error`/`ErrorSilent` — `ErrorSilent` não reproduzido, pois o bot Java não tem canal de saída de texto para o operador ainda; `NAO_ENCONTRADO` nomeia explicitamente o que no legado é tratado fora do próprio enum). Regra de acesso: um `Comando` de ação nunca chama `SessaoDeJogo` diretamente — sempre delega a um `CasoDeUso` já aprovado (reforça a DEC-21, que já previa textualmente "um futuro... comando" como origem legítima de uma ação); pode ler estado público de `Bot`/`SessaoDeJogo` para resolver parâmetros (ex.: item ativo da hotbar), mesmo raciocínio já usado pelo legado (`CommandPlaceBlock` resolve `Client.ItemInHand` diretamente no próprio comando).

Entregue

- `interfaces.comando.Comando` (interface), `ResultadoComando` (enum), `GerenciadorDeComandos` (registro por alias, parsing de `"alias arg1 arg2..."`, despacho com captura de `RuntimeException`→`ERRO`, mesmo padrão do `RunCommand` do legado).
- 8 comandos concretos, todos delegando a um Caso de Uso já existente e aprovado (Milestones 9–10), sem nenhum pacote de protocolo, Port ou agregado novo: `ComandoMover`/`ComandoOlhar`/`ComandoMoverEOlhar` (→ `CasoDeUsoMoverJogador`/`CasoDeUsoRotacionarJogador`/`CasoDeUsoMoverEOlharJogador`), `ComandoBalancarBraco` (→ `CasoDeUsoBalancarBraco`), `ComandoIniciarQuebraDeBloco`/`ComandoCancelarQuebraDeBloco`/`ComandoFinalizarQuebraDeBloco` (→ os três Casos de Uso de Player Digging), `ComandoColocarBloco` (→ `CasoDeUsoColocarBloco`, resolvendo o item da hotbar ativa via `InventarioDoJogador.slotAtivo()`, cursor default `(8,8,8)` na ausência de ray casting).
- Nenhuma interface já aprovada foi alterada (`CasoDeUso`, `SessaoDeJogo`, `ConexaoMinecraft`, `Bot` permanecem exatamente como são) — mudança 100% aditiva.

Comandos do legado avaliados e explicitamente excluídos (candidatos remanescentes, cada um documentado na DEC-23 com o motivo específico): `CommandMove`/`CommandGoto` (dependem de fila de movimento por tick e pathfinding A*, inexistentes no Java); `CommandSneak` (depende do pacote Entity Action `0x0B`, não implementado); `CommandHotbarClick`/`CommandInvClick`/`CommandInvCaptcha`/`CommandDropAll`/`CommandGive`/`CommandUseEntity`/`CommandUseBow` (pacotes de inventário/entidade/uso de item não implementados); `CommandKillAura`/`CommandMiner`/`CommandHerbalism`/`CommandAntiAFK`/`Solk.CommandPesca`/`Solk.CommandPescaV2`/`Solk.CommandMob`/`Solk.CommandMobPlus`/`Solk.CommandMobTeleport` (automação/combate, excluídos por política do projeto desde a Milestone 5); `CommandHelp`/`CommandPlayerList` (seu valor é o texto impresso; sem canal de saída para o operador ainda, DEC-02 não decidida); `CommandClearChat`/`CommandProxy` (sem equivalente de domínio); `CommandScript`/`CommandPortal`/`CommandRetard`/`CommandTwerk`/`CommandReco` (dependem de infraestrutura ainda não pronta — orquestração de comandos, geometria de mundo, movimentação por tick, `ConexaoBotPort.disconnect()`).

Testes

`GerenciadorDeComandosTest` (localização por alias case-insensitive, parsing de alias/argumentos incluindo ausência de argumentos, `NAO_ENCONTRADO`, captura de exceção→`ERRO`, propagação de resultado do comando, cenário ponta a ponta real com `ComandoBalancarBraco`+`SessaoDeJogo`+socket loopback); um teste por comando concreto cobrindo sucesso (packet enviado corretamente verificado byte a byte via record equality), `ARGUMENTOS_FALTANDO` e exceção quando o bot não está em sessão de jogo ativa; `ComandoColocarBlocoTest` cobre também a resolução do item ativo da hotbar (slot 0 e slot diferente de 0) e o cursor default.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **583 testes executados, 0 falhas, 0 erros, 3 skipped deliberadamente** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi alterado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: canal de saída de texto para o operador (CLI ou API, DEC-02 ainda não decidida) — desbloquearia `ComandoAjuda`/`ComandoListarJogadores`; ray casting contra `Mundo` (desbloquearia reconstrução fiel de `CommandBreakBlock`/`CommandClickBlock`/`CommandPlaceBlock`, com auto-look/auto-tool); Entity Action `0x0B` serverbound (desbloquearia um `ComandoSneak`); "usar item na mão" (sentinela `-1`/`-1`/`-1` de Player Block Placement); restante da Milestone 7; criptografia AES-CFB8/modo online; integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()`. Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 13

Status

CONCLUÍDA

Objetivo

Iniciar a construção de capacidades reutilizáveis que sirvam de base para futuras automações (mineração, combate, pesca, macros), reaproveitando obrigatoriamente `Mundo`, `SessaoDeJogo` e `Bloco` já existentes, sem introduzir nenhum Packet, Port ou Use Case novo.

### Incremento 13.1 — Raycast Fiel ao Legado sobre Mundo (DEC-24)

Status

Concluído

Análise do legado

`AdvancedBot.Client.Map.World.RayCast(Vec3d start, Vec3d end, bool stopOnNonAir, bool allowWater)` (`World.cs:266`): algoritmo de travessia de voxels por eixo dominante (mesma família do `clipBlock`/`rayTraceBlocks` do Minecraft vanilla), usado por `AutoMiner`, `CommandBreakBlock`, `CommandPlaceBlock`, `CommandClickBlock`, `CommandHerbalism` e `Solk/MacroUtils` como primeiro passo antes de qualquer ação sobre um bloco. `Entity.RayCastBlocks(double radius)`/`GetLookVector()`/`CalculateLookVector(float yaw, float pitch)` (`Entity.cs:505-535`): conveniência que dispara o raycast a partir da posição/rotação do próprio jogador (6 blocos de alcance em todos os usos encontrados). `Blocks.IsSolid(int id)` (`Blocks.cs:485`): tabela estática de ids não-sólidos, consultada apenas quando `stopOnNonAir=false`.

Entre as capacidades candidatas apresentadas para esta milestone (raycast contra Mundo/Entidades; canal de saída de texto para o operador; pacotes PLAY restantes; criptografia/modo online), o raycast foi escolhido por ser a de maior prioridade arquitetural e menor risco de integração: nenhum Packet/Port/Use Case novo, reaproveita `Mundo`/`SessaoDeJogo` já existentes, evidência de legado direta e concentrada em poucos arquivos.

Decisão (DEC-24, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): porte literal do algoritmo para `Mundo.tracarRaio(...)`, incluindo o quirk de que o voxel de destino nunca é testado por solidez (retorna `null` se o raio chegar até ele sem bater em nada antes) e a semântica contraintuitiva de `permitirAgua` (quando `true`, líquido conta como acerto e PARA o raio; quando `false`, líquido é sempre atravessado, independentemente de `pararEmNaoAr`). `Bloco` ganha `solido()` (porte de `Blocks.IsSolid`). `SessaoDeJogo.tracarRaioParaBlocos(alcance)` porta `RayCastBlocks`/`GetLookVector`/`CalculateLookVector`, preservando o offset `yaw - 180` do legado antes de converter para radianos. `CanSeePlayer`/`CanSeeEntity` não foram portados como métodos dedicados — no legado ambos já são apenas uma chamada a `World.RayCast` mirando a posição da entidade-alvo, o que `Mundo.tracarRaio` já cobre por composição; a tabela `EntityProperty`/altura-por-tipo-de-mob que `CanSeeEntity` usaria para mobs genéricos (jogadores usam a constante fixa `1.62`) fica de fora, por ser um levantamento de dados maior sem caso de uso real ainda.

Entregue

- `domain.protocol.v1_8.Bloco.solido()` (novo método): porte de `Blocks.IsSolid(int)`.
- `domain.bot.Mundo.tracarRaio(origemX,Y,Z, destinoX,Y,Z, pararEmNaoAr, permitirAgua): ResultadoDoRaio` (novo): porte de `World.RayCast`.
- `domain.bot.ResultadoDoRaio` (novo record: `x`, `y`, `z`, `face`): porte de `HitResult`, limitado aos campos que o algoritmo popula (`HitVector`/`PointedEntity` nunca são atribuídos por `World.RayCast`).
- `domain.bot.SessaoDeJogo.tracarRaioParaBlocos(double alcance): ResultadoDoRaio` (novo): porte de `RayCastBlocks`.
- Correção em `domain.bot.Mundo.blocoEm`: bounds-check fiel a `World.GetBlock` (`y` em `[0,256)`, `x`/`z` dentro de ±30.000.000) — sem isso, `tracarRaio` lançaria `ArrayIndexOutOfBoundsException` ao percorrer `y<0`/`y>=256` (cenário real ao minerar perto do bedrock ou do limite de altura, não um caso extremo hipotético). Não altera assinatura nem comportamento para nenhuma entrada já válida/testada.
- Nenhuma interface já aprovada foi alterada — mudança 100% aditiva.

Testes

`BlocoTest` (novo): os 49 ids não-sólidos do legado classificados corretamente (retranscritos de forma independente da implementação), amostra de blocos sólidos comuns e vizinhos dos não-sólidos (checagem de off-by-one), metadata não afeta solidez. `MundoTest`: bounds-check de `blocoEm` (y negativo, y no limite superior, coordenada horizontal além do limite mundial — nenhum lança exceção); `tracarRaio` — miss (nada bloqueia até o destino), acerto em bloco sólido com verificação de face, líquido atravessado quando `pararEmNaoAr=true`/`permitirAgua=false`, líquido como acerto quando `permitirAgua=true`, o quirk do bloco de destino nunca testado, origem `NaN`. `SessaoDeJogoTest`: `tracarRaioParaBlocos` acerta bloco abaixo ao olhar reto para baixo (`pitch=90`), retorna `null` quando nada está no alcance.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **597 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi alterado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: primeiro Caso de Uso/Comando que consuma `tracarRaioParaBlocos` (ex.: mineração com auto-mira, reconstrução mais fiel de `CommandBreakBlock`/`CommandPlaceBlock`/`CommandClickBlock`); checagem de linha de visão para combate via `Mundo.tracarRaio` direto contra a posição de uma entidade (`EntidadesDoMundo`); altura de olho por tipo de mob (desbloquearia `CanSeeEntity` genérico, hoje só `CanSeePlayer`-equivalente é trivial via constante `1.62`); canal de saída de texto para o operador (CLI ou API, DEC-02 ainda não decidida); Entity Action `0x0B` serverbound; "usar item na mão" (sentinela `-1`/`-1`/`-1` de Player Block Placement); restante da Milestone 7; criptografia AES-CFB8/modo online; integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()`. Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 14

Status

CONCLUÍDA

Objetivo

Reconstrução fiel de `CommandBreakBlock`/`CommandClickBlock`/`CommandPlaceBlock` do legado com auto-look/auto-mira, consumindo `SessaoDeJogo.tracarRaioParaBlocos`/`Mundo.tracarRaio` (DEC-24, Milestone 13) — candidato de maior prioridade arquitetural já previsto nas "Consequências Positivas" daquela DEC, reaproveitando obrigatoriamente `SessaoDeJogo`/`Mundo`/`InventarioDoJogador`/Casos de Uso já existentes, sem introduzir nenhum Packet, Port ou agregado novo.

### Incremento 14.1 — Ações de Bloco com Auto-Mira: Break/Click/Place (DEC-25)

Status

Concluído

Análise do legado

`AdvancedBot.Client.Commands.CommandBreakBlock.cs`/`CommandClickBlock.cs`/`CommandPlaceBlock.cs`: os 3 comandos olham para o bloco-alvo (`Entity.LookToBlock`/`LookTo`, `Entity.cs:471-495`) e confirmam via raycast antes de agir. `CommandClickBlock` usa `World.RayCast` direto até o alvo; `CommandBreakBlock`/`CommandPlaceBlock` usam `Player.RayCastBlocks(6.0)` (a mesma capacidade portada na Milestone 13). `MinecraftClient.BreakBlock`/`PlaceCurrentBlock` (`MinecraftClient.cs:870-892`) concentram os efeitos colaterais de rede (SwingArm + Player Digging start/finish; SwingArm + Player Block Placement, com um segundo envio sem bloco-alvo para itens especiais). `PacketBlockPlace.cs` confirma os valores exatos do sentinela "usar item na mão" (`x=y=z=-1`/`direction=255`/`cursor=(0,0,0)`).

Decisão (DEC-25, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): novos `SessaoDeJogo.olharParaBloco`/`usarItemNaMao` (porte de `LookTo`/`LookToBlock` e do construtor sentinela de `PacketBlockPlace`, sem o jitter aleatório do legado — variação cosmética, omitida para preservar o determinismo dos testes de igualdade exata já usados em todo o projeto); 3 novos Comandos (`ComandoClicarBloco`/`ComandoQuebrarBloco`/`ComandoColocarBlocoAutoMira`) delegando exclusivamente a Casos de Uso já aprovados (Milestones 9-10) mais os 2 novos; `ComandoQuebrarBloco` porta apenas o caminho base de `CommandBreakBlock` (opções `rp`/`rt`), excluindo `ncp`/`at` (Tick loop/física e seleção automática de ferramenta, fora de escopo por política do projeto); `ComandoColocarBlocoAutoMira` não porta o branch de item especial de `PlaceCurrentBlock` por ser comprovadamente inalcançável a partir de `CommandPlaceBlock` (que exige `ItemInHand.ID < 256`, excluindo todos os ids especiais).

Entregue

- `domain.bot.SessaoDeJogo.olharParaBloco(int,int,int)` e `usarItemNaMao(ItemStack)` (novos métodos de intenção, ambos delegando a `olhar`/`colocarBloco` já aprovados — zero Packet/Codec novo).
- `application.usecase.CasoDeUsoOlharParaBloco`/`CasoDeUsoUsarItemNaMao` (novos, mesmo padrão 1:1 já usado por todos os demais Casos de Uso).
- `interfaces.comando.ComandoClicarBloco` (alias `clicarbloco`/`clickblock`), `ComandoQuebrarBloco` (alias `quebrarbloco`/`breakblock`), `ComandoColocarBlocoAutoMira` (alias `colocarblocoautomira` — "placeblock" já pertence a `ComandoColocarBloco` desde a Milestone 12, colisão evitada com alias próprio, mesmo critério do Incremento 10.2).
- Nenhuma interface já aprovada foi alterada (`SessaoDeJogo`, `Comando`, `ResultadoComando`, `GerenciadorDeComandos`, `Mundo` só receberam adições ou novos chamadores) — incremento 100% aditivo.

Testes

`SessaoDeJogoTest` (+2: `olharParaBloco` calculando yaw/pitch e enviando `PlayerLookPacket`; `usarItemNaMao` enviando o sentinela), `CasoDeUsoOlharParaBlocoTest`/`CasoDeUsoUsarItemNaMaoTest` (2 cada: sucesso + `IllegalStateException` sem sessão), `ComandoClicarBlocoTest` (6: acerto+clique esquerdo nas coordenadas pedidas, acerto+clique direito nas coordenadas do raio, sem acerto usando face padrão, distância excedida, argumentos faltando, sem sessão), `ComandoQuebrarBlocoTest` (7: sucesso, divergência sem `rt`, divergência ignorada com `rt`, sem acerto, posição relativa, argumentos faltando, sem sessão), `ComandoColocarBlocoAutoMiraTest` (7: sucesso, posição relativa, sem item selecionado, item não é bloco, sem acerto, argumentos faltando, sem sessão).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **623 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi alterado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: canal de saída de texto para o operador (CLI ou API, DEC-02 ainda não decidida) — desbloquearia `ComandoAjuda`/`ComandoListarJogadores`; checagem de linha de visão para combate via `Mundo.tracarRaio` direto contra a posição de uma entidade; altura de olho por tipo de mob (desbloquearia `CanSeeEntity` genérico); `Player` bare (`0x03`) e Entity Action (`0x0B` serverbound — desbloquearia `ComandoSneak`); restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation); criptografia AES-CFB8/modo online; integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()`; estratégia de wiring/DI para a fábrica de conexão em produção (`infrastructure.config` permanece vazio). Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 15

Status

CONCLUÍDA

Objetivo

Reconstruir a infraestrutura de saída de mensagens para o operador utilizada pelos comandos do bot (Help, PlayerList, mensagens de erro/sucesso, feedback operacional), fechando a lacuna documentada desde a DEC-23 e repetida como candidato pendente nas Milestones 12, 13 e 14 — sem criar nenhuma interface gráfica.

### Incremento 15.1 — Infraestrutura de Saída de Mensagens, Ajuda e PlayerList (DEC-26)

Status

Concluído

Análise do legado

`AdvancedBot.Client.MinecraftClient.PrintToChat(string msg)` (`MinecraftClient.cs:830-842`): acumula a mensagem em `ChatMessages` (`List<string>`), sob `lock`, truncando por trás quando excede `MaximumChatLines` (`=150`, `MinecraftClient.cs:973`) **antes** de adicionar — a ordem trim-antes-do-add faz o buffer oscilar em 151 elementos em regime permanente, não 150; marca `ChatChanged=true` (flag de repintura da UI WinForms). `CommandManagerNew.RunCommand` (`CommandManagerNew.cs:50-88`): mensagens de fallback (`Comando não encontrado`, `Error`, `MissingArgs`) via o mesmo `PrintToChat`; `Success`/`ErrorSilent` não geram mensagem do gerenciador. `CommandHelp.cs`/`CommandPlayerList.cs`: portados por completo; `PlayerNick.ToString()` (`PlayerNick.cs:28-31`) confirmado como retornando `RealNick`, não `DisplayName` — a listagem de jogadores usa nome, não nome de exibição.

Decisão (DEC-26, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): novo `domain.bot.SaidaDoOperador` (porte literal de `ChatMessages`/`MaximumChatLines`/`lock`, incluindo o regime permanente de 151 elementos; `ChatChanged` não portado por falta de consumidor real — DEC-02 ainda não decidida). `Bot` ganha o campo `saidaDoOperador` (getter via Lombok, sem mudança de construtor — não em `SessaoDeJogo`, já que `PrintToChat` no legado é chamado inclusive antes do login). `GerenciadorDeComandos.executar` ganha as 3 mensagens de fallback do `CommandManagerNew` (mesma assinatura pública, mudança interna ao corpo do método). Novos `ComandoAjuda` (alias `ajuda` + `help`/`?` do legado) e `ComandoListarJogadores` (alias `listarjogadores` + `playerlist`/`players` do legado), ambos escrevendo em `bot.getSaidaDoOperador()` sem Caso de Uso — leitura de estado público/escrita local, não ação de protocolo (nenhum `Packet` enviado), mesma exceção já usada por `ComandoColocarBloco` ao ler `InventarioDoJogador`. Nenhum Port novo, nenhuma interface pública alterada.

Entregue

- `domain.bot.SaidaDoOperador` (novo): buffer de mensagens limitado e thread-safe, porte de `ChatMessages`/`MaximumChatLines`.
- `domain.bot.Bot` ganha o campo final `saidaDoOperador` (sem mudança de construtor).
- `interfaces.comando.GerenciadorDeComandos.executar` ganha mensagens de fallback (`NAO_ENCONTRADO`/`ARGUMENTOS_FALTANDO`/`ERRO`), mesma assinatura pública.
- `interfaces.comando.ComandoAjuda` (novo, porta `CommandHelp` — recebe `GerenciadorDeComandos` via construtor para listar o catálogo já registrado) e `ComandoListarJogadores` (novo, porta `CommandPlayerList`).
- Nenhuma interface já aprovada foi alterada (`Comando`, `ResultadoComando`, construtor de `Bot` permanecem exatamente como são) — incremento 100% aditivo.

Testes

`SaidaDoOperadorTest` (4: buffer vazio, acumulação em ordem, imutabilidade da cópia retornada, truncamento preservando o regime permanente de 151); `BotTest` (+1: `saidaDoOperador` não nula e vazia desde a criação); `GerenciadorDeComandosTest` (+4: mensagem de fallback para alias não encontrado/erro/argumentos faltando, e ausência de mensagem em sucesso); `ComandoAjudaTest` (4: listagem completa ordenada por nome, filtro por termo de busca em nome/alias, formatação de descrição/alias/parâmetros sem prefixo de cifrão, sucesso mesmo sem nenhum outro comando registrado); `ComandoListarJogadoresTest` (4: contagem e nomes corretos, nome nulo de `JogadorConhecido` tratado sem exceção, lista vazia, `IllegalStateException` sem sessão de jogo ativa).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **640 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado (apenas estendido com novos métodos de teste em arquivos já existentes).

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: checagem de linha de visão para combate via `Mundo.tracarRaio` direto contra a posição de uma entidade; altura de olho por tipo de mob (desbloquearia `CanSeeEntity` genérico); `Player` bare (`0x03`) e Entity Action (`0x0B` serverbound — desbloquearia `ComandoSneak`); restante da Milestone 7 (Update Sign, World Border, Update Block Entity, Block Action, Block Break Animation); criptografia AES-CFB8/modo online; integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()`; estratégia de wiring/DI para a fábrica de conexão em produção e para o catálogo de comandos (`infrastructure.config` permanece vazio — nenhuma composition root monta `GerenciadorDeComandos`+comandos em produção ainda); decisão de transporte (CLI ou API, DEC-02) para efetivamente consumir `SaidaDoOperador`. Mineração automática, física, Tick loop e lógica de inventário automático continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 16

Status

CONCLUÍDA

Objetivo

Reconstruir a infraestrutura de navegação (PathFinding) do bot como mecanismo reutilizável — não uma macro — a ser consumido futuramente por mineração, combate, coleta e exploração.

### Incremento 16.1 — Algoritmo de Busca de Caminho sobre Mundo (DEC-27)

Status

Concluído

Análise do legado

`AdvancedBot.Client.PathFinding.PathFinder`/`Path`/`PathPoint` (`AdvancedBot.Client.PathFinding\PathFinder.cs`, `Path.cs`, `PathPoint.cs`): busca best-first sobre o grid de blocos (fila de prioridade binária por `DistanceToTarget`, heurística Manhattan com peso 2× no eixo Y, expansão nas 4 direções horizontais, cap de 512 iterações, cap de 3 blocos de queda por "ponto seguro"; classificação de nó via `GetNodeType`: Aberto/Bloqueado/Água/Lava/Cerca/Alçapão). `AdvancedBot.Client.PathFinding.PathGuide` (`PathGuide.cs`): consumidor que executa o caminho tick a tick, lendo/escrevendo `MotionX/Y/Z`/`OnGround`/`IsCollidedHorizontally`/`ActivePotions`/`GetMoveSpeed()` — depende inteiramente de um motor de física que não existe no domínio Java. `World.CreatePathTo` (`World.cs:195-202`): pré-checagem de distância + delegação a um `PathFinder` novo por chamada. Único ponto de chamada de `PathGuide.Create` em todo o projeto: `MinecraftClient.RequestPathTo`/`FindPath` (`MinecraftClient.cs:646-671`), sempre com os mesmos 5 valores fixos (`radius=80f, allowWoodenDoor=true, movementBlockAllowed=false, pathInWater=true, canDrown=false`). Dois achados de código morto confirmados por rastreamento manual: `canEntityDrown` nunca é `true` em nenhuma chamada de todo o projeto (não só fora do escopo desta milestone); `NodeType._2` é inalcançável em `GetNodeType` (as duas únicas escritas da flag correspondente são sempre seguidas, na mesma iteração, por um retorno antecipado de outro tipo).

Decisão (DEC-27, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): o algoritmo puro (`PathFinder`/`Path`/`PathPoint`) entra em escopo, portado por completo sobre `Mundo`/`Bloco` já existentes; `PathGuide` e todos os seus consumidores (`CommandGoto`, `CommandFollow`, `CommandPortal`, `AutoMiner`) permanecem fora de escopo, sem nenhuma reavaliação de status — a DEC-22 (motor de física/Tick automático fora de escopo) não é reaberta nem alterada, porque a peça que dependeria dela simplesmente não é construída. Novo `domain.bot.Mundo.criarCaminhoPara(...)` (porta `World.CreatePathTo`, incluindo a pré-checagem de distância com a mesma inconsistência de offset do legado entre a checagem e a busca interna); novo `domain.bot.BuscadorDeCaminho` (package-private, mesmo precedente de `SecaoDeChunk` — porta `PathFinder` + a fila de prioridade `Path` como classe aninhada privada `FilaDeNos`); novo `domain.bot.PontoDeCaminho` (público, porta `PathPoint`, com `equals`/`hashCode` por X/Y/Z). `SessaoDeJogo.criarCaminhoPara(destX,destY,destZ)` usa a própria posição e os 4 valores fixos observados no único call site real do legado. `canDrown` não é portado como parâmetro (código morto comprovado); `NodeType._2` não tem equivalente em `TipoDeNo` (6 valores, branch inalcançável removido); `entity.IsOnLava()` portado como verificação pontual inline (sem introduzir uma classe `AABB` genérica sem consumidor); chave de memoização por coordenada portada como `record ChavePonto(x,y,z)` em vez do hash bit a bit do legado (otimização de performance específica do C#, sem efeito observável).

Entregue

- `domain.bot.PontoDeCaminho` (novo, público): X/Y/Z + as duas distâncias (euclidiana, manhattan com peso 2× em Y); porte de `PathPoint`.
- `domain.bot.BuscadorDeCaminho` (novo, package-private): porte de `PathFinder`, incluindo a fila de prioridade binária (`FilaDeNos`, classe aninhada privada, porte de `Path`) e a classificação de nó (`tipoDoNo`, porte de `GetNodeType`, sem o branch inalcançável `NodeType._2`).
- `domain.bot.Mundo.criarCaminhoPara(origemX,origemY,origemZ,destinoX,destinoY,destinoZ,raio,permitirPortaDeMadeira,bloqueioDeMovimentoPermitido,podeNadar)` (novo): porte de `World.CreatePathTo`.
- `domain.bot.SessaoDeJogo.criarCaminhoPara(destX,destY,destZ)` (novo): conveniência usando a própria posição e os valores fixos do legado.
- `Mundo.piso` passa de `private` para visibilidade de pacote (reaproveitado por `BuscadorDeCaminho`) — não é mudança de contrato público.
- Nenhuma interface já aprovada foi alterada, nenhum Packet/Port/Caso de Uso/`Comando`/bounded context novo — incremento 100% aditivo.

Testes

`MundoTest` (+10: caminho reto em faixa plana com sequência exata de pontos, origem igual a destino, corte por raio+8, origem isolada sem opção segura, desvio de parede sólida de 2 blocos de altura, lava sempre bloqueia mesmo com nó acima classificado como aberto, água permite pousar quando `podeNadar=false` mas é recusada quando `podeNadar=true` — validando a diferença exata de comportamento entre os dois modos —, porta de madeira exige as duas flags simultaneamente para ser transponível, confirmando que `permitirPortaDeMadeira` isolada não basta); `SessaoDeJogoTest` (+2: `criarCaminhoPara` usa a própria posição como origem, retorna nulo sem opção segura).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **652 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto (lista inalterada em relação à Milestone 15, com a ressalva de que "pathfinding A*" deixou de faltar — apenas "fila de movimento por tick" continua bloqueando `CommandMove`/`CommandGoto`): checagem de linha de visão para combate via `Mundo.tracarRaio`; altura de olho por tipo de mob (`CanSeeEntity` genérico); `Player` bare/Entity Action (`ComandoSneak`); restante da Milestone 7; criptografia/modo online; integração de disconnect (`ComandoReco`); estratégia de wiring/DI para produção; decisão de transporte (CLI ou API, DEC-02). Tick loop/motor de física automático — e, por consequência, `PathGuide`/execução de caminho tick a tick — continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 17

Status

CONCLUÍDA

Objetivo

Reconstruir a infraestrutura de mineração do bot como primitivas de domínio reutilizáveis — não uma macro — a ser consumida futuramente por uma macro de mineração completa.

### Incremento 17.1 — Registro de Blocos e Calculadora de Força de Quebra (DEC-28)

Status

Concluído

Análise do legado

`AdvancedBot.Client.DiggingHelper` (`DiggingHelper.cs`): `StrengthVsBlock`/`CanHarvestBlock`/`ToolStrengthVsBlock`/`PlayerStrengthVsBlock` — fórmula de força de quebra por tick (hardness do bloco, ferramenta certa/errada, bônus de Efficiency, multiplicadores de Haste/Mining Fatigue, penalidade debaixo d'água e fora do chão), consumida por `AutoMiner.Tick`/`CommandBreakBlock.Tick` (ambos Tick loop, fora de escopo). `AdvancedBot.Client.Blocks`/`Block` (`Blocks.cs`/`Block.cs`): registro de `Hardness`/`Material`/`HarvestTools`/`Diggable`/`Transparent`/`StackSize`/`DisplayName`/`Variations` por id, construtor estático lendo um `blocks.json` embutido em `AdvancedBot.Properties.Resources.resx` (`Resources.blocks`, mesmo formato do `minecraft-data` já usado como referência cruzada neste projeto); só `Hardness`/`Material`/`HarvestTools` têm consumidor comprovado (a fórmula de força) — os demais campos não são usados por nenhum caminho em escopo. `Resources.materials` (mesmo `.resx`): tabela plana material→ferramenta→força (236 blocos e 43 entradas de ferramenta extraídos byte a byte via decodificação base64 do `.resx`, não recriados de memória). `AdvancedBot.Client.Entity` (`Entity.cs`): `ActivePotions` (Dictionary por effect id), `OnGround`, `IsUnderWater()` (bloco na posição de pés contra ids de água 8/9 — totalmente derivável hoje via `Mundo.blocoEm`, sem lacuna). `AdvancedBot.Client.ItemStack` (`ItemStack.cs`): `GetEnchantmentLevel(id)` lendo NBT `"ench"` — sem equivalente Java (`ItemStackCodec` descarta NBT por decisão deliberada desde a Milestone 5.5). `ReceptorEntityEffect` (Java, já existente desde a Milestone 5.6.4) documenta que o entityId do próprio bot nunca está em `EntidadesDoMundo` — efeitos do próprio bot (Haste/Mining Fatigue inclusos) continuam não modelados. `AutoMiner.cs`/`CommandMiner.cs`/`CommandBreakBlock.cs` (opção `ncp`, já excluída na Milestone 14): consumidores Tick, permanecem fora de escopo por DEC-22/23.

Decisão (DEC-28, ver [01-Decisoes-Arquiteturais.md](01-Decisoes-Arquiteturais.md)): a fórmula (`DiggingHelper`) e o registro de dados (`Blocks`/`Block`, só os 3 campos com consumidor comprovado) entram em escopo como calculadora pura sobre `Bloco`/`ItemStack` já existentes. Os três insumos sem estado equivalente no domínio Java (nível de Efficiency, amplifier de Haste, amplifier de Mining Fatiga) tornam-se parâmetros explícitos da calculadora — mesmo padrão já usado por `onGround` em `SessaoDeJogo.mover`/`olhar` desde a DEC-22 —, com `-1` como sentinela de "efeito não ativo" (mesma convenção de `EntidadeRemota.ultimaAnimacao`/`ultimoStatus`); nenhuma das três lacunas é resolvida ou reaberta por esta DEC. "Submerso", ao contrário, é fielmente derivável hoje e ganha método real e wired, não um parâmetro cego. `RegistroDeBlocos` package-private (mesmo precedente de `SecaoDeChunk`/`BuscadorDeCaminho`); `CalculadoraDeQuebraDeBloco` pública, mesmo padrão de capacidade pura sem consumidor de produção obrigatório já usado por `Mundo.tracarRaio` (DEC-24) e `Mundo.criarCaminhoPara` (DEC-27). Nenhum gatilho de parada da instrução da milestone se aplica: não é novo bounded context, não é novo Port, não é alteração de contrato público, não é alteração de DEC existente, e há evidência suficiente no legado para os 236 blocos/43 ferramentas portados.

Entregue

- `domain.protocol.v1_8.RegistroDeBlocos` (novo, package-private): `dureza(id)`/`material(id)`/`ferramentasDeColeta(id)` — porte de `Blocks.GetHardness`/`.Material`/`.HarvestTools`, dados extraídos byte a byte do `blocks.json` embutido no `.resx` legado (236 blocos; array dimensionado em 256, maior id legado é 255/`structure_block`).
- `domain.protocol.v1_8.CalculadoraDeQuebraDeBloco` (novo, público): `podeColher`/`forcaDaFerramenta`/`forcaDoJogador`/`forcaDeQuebra` — porte completo de `DiggingHelper`, incluindo a tabela de ferramentas (`materials.json`, 43 entradas) como lista privada. `nivelEficiencia`/`amplifierCeleridade`/`amplifierFadiga`/`noChao` são parâmetros explícitos pelos motivos documentados na DEC-28.
- `domain.bot.SessaoDeJogo.estaSubmerso()` (novo): porte de `Entity.IsUnderWater` — bloco na posição atual (via `Mundo.blocoEm`) contra ids de água (8/9).
- Nenhuma interface já aprovada foi alterada, nenhum Packet/Port/Caso de Uso/`Comando`/bounded context novo — incremento 100% aditivo.

Testes

`RegistroDeBlocosTest` (6: dureza/material/ferramentas de pedra, dureza negativa de bedrock, tronco sem ferramenta exigida, restrição de minério de diamante, dados padrão para id desconhecido e para id de lacuna dentro do array); `CalculadoraDeQuebraDeBlocoTest` (21: `podeColher` com/sem ferramenta e com/sem restrição, `forcaDaFerramenta` sem correspondência e com correspondência exata da tabela, `forcaDoJogador` com bônus de Efficiency condicionado a força base >1, multiplicadores de Haste/Mining Fatigue em cada branch — incluindo o branch padrão ≥3 —, divisores de submerso/fora-do-chão isolados e combinados, `forcaDeQuebra` com dureza negativa (zero) e dureza zero (Infinity, fiel à divisão por zero do legado), divisor 30 vs. 100 conforme `podeColher`); `SessaoDeJogoTest` (+4: `estaSubmerso` falso sem chunk carregado, verdadeiro para água parada e água corrente, falso para bloco sólido).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **683 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

Próximo passo sugerido

Candidatos não comprometidos, a critério do responsável do projeto: um `Comando`/macro de mineração completa consumindo `CalculadoraDeQuebraDeBloco`+`Mundo.criarCaminhoPara` (ainda bloqueado na parte de execução por Tick — automação de mineração continua fora de escopo por política do projeto); suporte a NBT real em `ItemStack` (desbloquearia `nivelEficiencia` real em vez do parâmetro `0`); rastreamento de efeitos do próprio bot em `SessaoDeJogo` (desbloquearia `amplifierCeleridade`/`amplifierFadiga` reais em vez do sentinela `-1` — decisão arquitetural própria, não resolvida por esta milestone); checagem de linha de visão para combate via `Mundo.tracarRaio` direto contra uma entidade; altura de olho por tipo de mob (`CanSeeEntity` genérico); `Player` bare/Entity Action (`ComandoSneak`); restante da Milestone 7; criptografia/modo online; integração de disconnect (`ComandoReco`); estratégia de wiring/DI para produção; decisão de transporte (CLI ou API, DEC-02). Tick loop/motor de física automático — e, por consequência, mineração automática — continuam fora de escopo por política do projeto. Nenhuma DEC pendente bloqueia os candidatos remanescentes.

---

## Milestone 18

Status

CONCLUÍDA

Objetivo

Encerrar formalmente o bounded context de Mundo (Milestone 7), implementando o único pacote remanescente com precedente real no legado (Update Sign) e registrando a exclusão fundamentada dos 4 pacotes restantes que nunca tiveram implementação real no legado.

### Incremento 18.1 — Update Sign e Encerramento Formal da Milestone 7

Status

Concluído

Análise do legado

`Handler_v18.cs` (`AdvancedBot.Client.Handler/Handler_v18.cs`, `case 51:`, id `0x33`): lê um `Location` empacotado (mesmo formato de `PacketBlockChange` — x 26 bits/y 12 bits/z 26 bits em um `long`) seguido de 4 strings (`ChatParser.ParseText` — mesmo tratamento já decidido para Chat Message na Milestone 5.3, texto mantido cru) e grava em `World.Signs` (`Dictionary<Vec3i, string[]>`, `AdvancedBot.Client.Map/World.cs`), dicionário independente das seções de chunk. Dois comportamentos do legado avaliados e não portados: a leitura é protegida por `Client.MapAndPhysics` (mesmo tratamento já dado a Explosion na Milestone 7.4 — sem equivalente no domínio Java, ignorado); e uma expurgação de placas a mais de 300 blocos do jogador a cada atualização (otimização de memória específica do C#, mesma categoria já registrada para a chave de memoização de `BuscadorDeCaminho` na DEC-27 — sem efeito observável, não portada, mesmo padrão de `Chunk`s que também nunca são expurgados por distância neste domínio). Busca exaustiva confirmada por rastreamento manual em `Handler_v18.cs` (todos os 35 `case` do switch principal enumerados) e por busca de classes `Packet*` dedicadas em todo o `Projeto Adv 2.4.5`: **nenhum precedente existe** para World Border (`0x44`), Update Block Entity (`0x35`) e Block Action (`0x24`) — nenhuma classe de pacote, nenhum `case` no Handler. Block Break Animation (`0x25`) idem. Mesma categoria de achado já registrada para Time Update/Spawn Position na Milestone 7.5 (ausência de implementação em todas as versões do Handler auditadas), agora estendida a mais 4 pacotes — cobertos com fidelidade pelo descarte seguro da DEC-20.

Decisão: puramente aditiva, sem DEC nova (mesmo critério das Milestones 7.1-7.5, que também nunca precisaram de DEC própria além da DEC-20 já existente — Update Sign é só mais um pacote do mesmo bounded context Mundo já aprovado). `UpdateSignPacket`/`UpdateSignCodec`/`UpdateSignHandler`/`EventoUpdateSign`/`ReceptorUpdateSign` seguem exatamente o padrão de Block Change; `Mundo.registrarPlaca`/`placaEm` usam um novo dicionário interno (`Map<PosicaoDeBloco, String[]>`, `PosicaoDeBloco` como record privado aninhado) independente do mapa de chunks — **ao contrário de `definirBloco`, não faz no-op quando o chunk correspondente não está carregado**, fiel ao dicionário próprio do legado. World Border/Update Block Entity/Block Action/Block Break Animation permanecem deliberadamente não registrados — mesmo tratamento de Time Update/Spawn Position (DEC-20).

Entregue

- `domain.protocol.v1_8.UpdateSignPacket`/`UpdateSignCodec`/`UpdateSignHandler`/`EventoUpdateSign` (novos; PLAY, id `0x33`, CLIENTBOUND); sem `Receptor` próprio — `ReceptorUpdateSign` (novo) delega a `Mundo.registrarPlaca`.
- `domain.bot.Mundo.registrarPlaca(x,y,z,linhas)`/`placaEm(x,y,z)` (novos) — porte de `World.Signs`, sem a expurgação por distância e sem o gate `Client.MapAndPhysics` (divergências documentadas acima).
- Milestone 7 (Modelagem do Mundo) encerrada oficialmente: dos 11 pacotes candidatos originais, 7 implementados (Chunk Data, Block Change, Multi Block Change, Map Chunk Bulk, Explosion, Change Game State, Update Sign) e 4 avaliados e excluídos por ausência comprovada de precedente no legado (Time Update, Spawn Position — já excluídos na Milestone 7.5 —, e agora World Border, Update Block Entity, Block Action, Block Break Animation).
- Nenhuma interface já aprovada foi alterada, nenhum Packet/Port/Caso de Uso/`Comando`/bounded context novo além do próprio pacote — incremento 100% aditivo.

Testes

`UpdateSignCodecTest` (3: round-trip preservando valores, coordenadas negativas nos limites do mundo, validação de `linhas` nulo ou com tamanho diferente de 4); `UpdateSignHandlerTest` (1: tradução para evento); `ReceptorUpdateSignTest` (1: registro no `Mundo` independente de chunk carregado); `MundoTest` (+3: registrar/recuperar placa, registrar placa sem chunk carregado, retornar nulo para placa desconhecida); `RegistroDePacotesV1_8Test` (+2: localizar Codec por id, localizar id por tipo); `PipelineDeProtocoloV1_8Test` (+1: pipeline completa).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **693 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

---

## Milestone 19

Status

CONCLUÍDA

Objetivo

Adicionar um segundo consumidor ao raycast fiel ao legado entregue na DEC-24 (Milestone 13): checagem de linha de visão do bot contra a posição de um jogador remoto, primitiva de combate sem nenhuma automação associada.

### Incremento 19.1 — Linha de Visão contra Jogador Remoto

Status

Concluído

Análise do legado

`Entity.CanSeePlayer`/`Entity.CanSeeEntity` (`AdvancedBot.Client/Entity.cs:442-469`): `CanSeePlayer(MPPlayer p)` traça um raio dos pés do bot (`PosX/PosY/PosZ`, sem offset de altura) até o olho do jogador-alvo (`p.X, p.Y + 1.62, p.Z` — mesma constante de altura de olho já avaliada e aceita como trivial na Milestone 13) via `World.RayCast(start, end, stopOnNonAir: false, allowWater: true)`, retornando `true` quando o raio não bate em nada (`== null`). `CanSeeEntity(IEntity)` despacha para `CanSeePlayer` quando o alvo é `MPPlayer`; para mobs, depende de `EntityProperty.Height` por tipo de mob (dado não levantado no domínio Java — mesma lacuna já registrada e deixada de fora na Milestone 13) e retorna `false` sem essa propriedade.

Decisão: sem DEC nova (mesmo critério de `Mundo.tracarRaio`/DEC-24 — outro consumidor da mesma capacidade já aprovada, sem alterar contrato nenhum). Portado apenas o caminho `CanSeePlayer`, que não depende de nenhum dado ausente; `CanSeeEntity` genérico para mobs permanece fora de escopo, mesma lacuna de dado (altura por tipo de mob) já documentada na Milestone 13 e na Seção 10.

Entregue

- `domain.bot.SessaoDeJogo.podeVerJogador(EntidadeJogadorRemoto jogador)` (novo) — porte de `Entity.CanSeePlayer`, delega a `Mundo.tracarRaio` já existente (`pararEmNaoAr=false`/`permitirAgua=true`, olho do alvo em `y + 1.62`).
- Nenhuma interface já aprovada foi alterada, nenhum Packet/Port/Caso de Uso/`Comando`/agregado/bounded context novo — incremento 100% aditivo, puro consumidor de capacidade já existente.

Testes

`SessaoDeJogoTest` (+2: linha de visão livre retorna verdadeiro sem nenhum chunk carregado, parede sólida de 3 blocos de altura entre bot e jogador retorna falso).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **695 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

---

## Milestone 20

Status

CONCLUÍDA

Objetivo

Reconstruir a ação de agachar do jogador (Entity Action, subconjunto sneak) como primitiva single-shot, seguindo o mesmo padrão de ação iniciada pelo bot já estabelecido desde a DEC-21/DEC-23.

### Incremento 20.1 — Entity Action (Sneak) e ComandoAgachar/ComandoPararDeAgachar

Status

Concluído

Análise do legado

`PacketEntityAction` (`AdvancedBot.Client.Packets/PacketEntityAction.cs`, id `0x0B` SERVERBOUND): `EntityID`/`ActionID`/`JumpBoost`, todos `VarInt` no fio para protocolo >= 1.8. Rastreamento completo de todo `new PacketEntityAction(...)` em `Projeto Adv 2.4.5`: `ActionID=0`/`1` (iniciar/parar agachamento) têm call site real fora de qualquer loop — `CommandSneak.cs` (comando dedicado, toggle) e `CommandTwerk.cs` (reaproveita os mesmos 2 ids). `ActionID=2` (leave bed) não tem nenhum call site em todo o projeto — código morto comprovado. `ActionID=3`/`4` (sprint) só são enviados dentro do Tick loop principal (`MinecraftClient.cs:786-793`, guardado por `if (beingTicked)`/`Player.Tick()`, comparando `Player.WasSprinting != Player.IsSprinting`) — fora de escopo, mesmo motivo da DEC-22 (motor de física/Tick automático). `ActionID=5` (jump boost/montaria) também não tem call site algum. Adicionalmente, `Player` bare (`PacketUpdate`, id `0x03`, só campo `OnGround`) foi investigado como candidato irmão (par "Player bare + Entity Action" da Seção 10): seu único call site em todo o legado (`MinecraftClient.cs:819`) está dentro do mesmo bloco de Tick acima, no `else` final da cadeia posição/rotação/sprint — enviado a cada tick quando nada mudou, puro ping de `OnGround` para o servidor. Sem nenhum call site fora do Tick loop, é excluído pelo mesmo motivo do sprint (DEC-22), não por ausência de precedente.

Decisão: sem DEC nova (mesmo critério de DEC-21/DEC-23 já aplicado a Balancar Braço/Player Digging/Player Block Placement — mais uma ação single-shot iniciada pelo bot sobre o padrão já aprovado). Como o contrato `Comando` não tem `Toggle()` (excluído desde a DEC-23), o toggle único do legado (`CommandSneak`) é modelado como 2 comandos single-shot simétricos em vez de um comando com estado, mesmo padrão já usado para separar `PlayerDigging` em `iniciarQuebraDeBloco`/`cancelarQuebraDeBloco`/`finalizarQuebraDeBloco` na Milestone 10 (nenhum boolean de estado "agachado" adicionado a `SessaoDeJogo` — ação disparada e esquecida, mesmo tratamento de digging, não de posição/rotação). `Player` bare (`0x03`) e os `ActionID` 2/3/4/5 permanecem deliberadamente não portados — achados documentados acima, cobertos pelo descarte seguro da DEC-20 quando aplicável.

Entregue

- `domain.protocol.v1_8.EntityActionPacket`/`EntityActionCodec`/`EntityActionHandler`/`EventoEntityAction` (novos; PLAY, id `0x0B`, SERVERBOUND, sem colisão com `AnimationPacket` clientbound no mesmo id); sem `Receptor` (mesmo precedente de `EnvioDeChatPacket`/`PlayerDiggingPacket` — ação enviada pelo bot, não reagida).
- `domain.bot.SessaoDeJogo.agachar()`/`pararDeAgachar()` (novos) — enviam `EntityActionPacket` com `actionId` 0/1 e `jumpBoost=0`, fiéis a todos os call sites legados em escopo.
- `application.usecase.CasoDeUsoAgachar`/`CasoDeUsoPararDeAgachar` (novos, conforme DEC-21).
- `interfaces.comando.ComandoAgachar` (aliases `agachar`/`sneak`) e `ComandoPararDeAgachar` (aliases `pararagachar`/`unsneak`), novos, delegando aos Casos de Uso acima.
- Nenhuma interface já aprovada foi alterada, nenhum Port/agregado/bounded context novo — incremento 100% aditivo.

Testes

`EntityActionCodecTest` (2: round-trip preservando valores, distinção entre iniciar/parar agachamento); `EntityActionHandlerTest` (1: tradução para evento); `SessaoDeJogoTest` (+2: `agachar` envia `actionId=0`, `pararDeAgachar` envia `actionId=1`); `CasoDeUsoAgacharTest`/`CasoDeUsoPararDeAgacharTest` (2 cada: sucesso com sessão ativa, exceção sem sessão); `ComandoAgacharTest`/`ComandoPararDeAgacharTest` (2 cada, mesmo padrão); `RegistroDePacotesV1_8Test` (+2: sem colisão com Animation clientbound, localizar id por tipo); `PipelineDeProtocoloV1_8Test` (+1: pipeline completa).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **710 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

---

## Milestone 21

Status

CONCLUÍDA

Objetivo

Construir a fundação da Engine de Execução Contínua do bot — ciclo de vida (iniciar/pausar/retomar/parar), scheduler genérico de tarefas periódicas e tick engine —, sem nenhuma macro/automação física. Pivô explícito de escopo: da Milestone 4 até a Milestone 20 cada milestone entregou uma capacidade isolada de protocolo/domínio; a Milestone 21 entrega, pela primeira vez, infraestrutura de execução que sustentará automações futuras, não uma automação em si.

### Incremento 21.1 — Ciclo de Vida do Bot, AgendadorDeTarefasPort e MotorDeTick (DEC-29)

Status

Concluído

Análise do legado

`AdvancedBot.Client\Main.cs` (`Main_Load`, ~299-327): `Thread` dedicada (`tickThread`) percorre `Clients` a cada ~50ms (20 Hz, `Stopwatch`-timed, sem catch-up), chamando `Tick()` de cada cliente dentro de um `try{}catch{}` que isola falha de UM bot sem derrubar o ciclo inteiro. `MinecraftClient.cs.Tick()` (~755-828) só processa corpo relevante `if (beingTicked)` (flag ligada após login, desligada ao desconectar) e, quando ativo, percorre `CurrentPath.Tick()` (PathGuide — lê/escreve Motion/OnGround/física, fora de escopo) e `CmdManager.Tick()` (chama `Tick()` de TODOS os ~29 comandos incondicionalmente; `isMacro`/`currentMacro` confirmados código morto em todo o projeto). Fora do tick de 20Hz, dois timers independentes e desacoplados de física já existiam: `autReconnectTimer` (`Main.cs:178`, `Interval=15000`, reconecta bots travados) e `Statistics.timer1` (poll de 1s). Busca exaustiva por Pause/Resume/Suspend/Idle não encontrou nenhum precedente de execução no legado (só `SuspendLayout`/`ResumeLayout` de WinForms, irrelevante).

Decisão: DEC-29 — resolve explicitamente que um scheduler/tick engine genérico, sem nenhuma leitura/escrita de física e sem nenhuma tarefa real registrada, não reabre o bloqueio de "Tick loop/motor de física automático" já firmado pela DEC-22/DEC-23/DEC-27 (mesma distinção "mecanismo vs. conteúdo" que a DEC-27 já usou para separar `PathFinder` de `PathGuide`). Ver DEC-29 em 01-Decisoes-Arquiteturais.md para as alternativas e a análise completa dos 4 gatilhos de parada (nenhum se aplica).

Entregue

- `domain.bot.EstadoExecucao` (novo enum: `PARADO`/`EXECUTANDO`/`PAUSADO` — sem precedente no legado, decisão de infraestrutura, não de fidelidade).
- `domain.bot.Bot` ganha `estadoExecucao` (padrão `PARADO`), `iniciar()`/`pausar()`/`retomar()`/`parar()` (transições mínimas, `pausar`/`retomar` rejeitam estado inválido com `IllegalStateException`), e `tarefasContinuas`/`registrarTarefa(TarefaContinua)`/`removerTarefa(TarefaContinua)` (lista thread-safe, `CopyOnWriteArrayList`).
- `domain.bot.TarefaContinua` (nova interface funcional, `void executar(Bot bot)`) — contrato de trabalho periódico, deliberadamente cego a física; zero implementações reais nesta milestone.
- `application.port.AgendadorDeTarefasPort` (novo, segundo Port do projeto depois de `ConexaoBotPort`): `ScheduledFuture<?> agendar(Runnable, Duration)`, `void encerrar()`.
- `infrastructure.execucao.AgendadorDeTarefasVirtualThread` (novo, implementa o Port): relógio de thread de plataforma única (`ScheduledExecutorService` single-thread) disparando cada execução em uma Virtual Thread nova (`Thread.ofVirtual()`, conforme DEC-03) — tarefas independentes nunca bloqueiam a cadência umas das outras.
- `infrastructure.execucao.MotorDeTick` (novo, o "tick engine" da milestone): percorre bots registrados, pula quem não está `EXECUTANDO` (mesmo espírito de `beingTicked`), invoca cada `TarefaContinua` isolando falha por tarefa (log WARN, nunca derruba o ciclo nem as demais tarefas/bots), protegido contra reentrância (`AtomicBoolean` — um ciclo concorrente é descartado com log, não enfileirado).
- Nenhuma interface existente alterada; nenhum Packet/Codec/Comando/bounded context novo; zero leitura/escrita de Motion/OnGround/velocidade em qualquer classe nova.

Testes

`BotTest` (+8: estado inicial `PARADO`, `iniciar`→`EXECUTANDO`, `pausar`→`PAUSADO`, rejeição de `pausar` a partir de `PARADO`, `retomar`→`EXECUTANDO`, rejeição de `retomar` fora de `PAUSADO`, `parar` de qualquer estado, registrar/remover `TarefaContinua`); `AgendadorDeTarefasVirtualThreadTest` (3, novo: execução periódica via `CountDownLatch`, execução de fato em Virtual Thread, `cancel(true)` interrompe disparos futuros); `MotorDeTickTest` (5, novo: executa tarefas de bot `EXECUTANDO`, ignora bot não-`EXECUTANDO`, isola falha de uma tarefa sem afetar as demais, ignora bot removido, descarta ciclo concorrente enquanto o anterior ainda executa — verificado com `CountDownLatch`, sem sleep).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **726 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 já registrados desde incrementos anteriores). Mudança 100% aditiva — nenhum teste pré-existente foi modificado.

---

## Milestone 22

Status

CONCLUÍDA — Incrementos 22.1 a 22.7 concluídos (DEC-30/DEC-31; 22.7 sem DEC nova).

Objetivo

Projetar a arquitetura responsável pelo ciclo de vida contínuo dos bots sobre o MotorDeTick (Milestone 21): Auto Reconnect, login automático, delay/tentativas/backoff, proxy por bot e rotação, detecção de disconnect, eventos de conexão/perda de conexão, scheduler/Tick do legado, início/pausa/fim/reinício de tarefas contínuas após reconexão, preservação de estado, registro de tarefas por plugins e dependências entre automações. Análise levantada EXCLUSIVAMENTE contra `Projeto Adv 2.4.5` — nenhum outro projeto/backup/fork AdvancedBot consultado.

### Fase de Planejamento — Análise Arquitetural da Engine de Execução Contínua

**Levantamento do legado** (todos os itens abaixo verificados diretamente no código C#, com file:line; detalhamento completo na sessão que produziu este incremento):

- **Auto Reconnect:** dois mecanismos independentes e sobrepostos. (A) `Main.cs:178`, `autReconnectTimer` (`Interval=15000`) — varredura "AntStop" a cada 5 min reais reconectando todo client com `!IsBeingTicked()` incondicionalmente, mais um modo `"Bot por Bot"` (`ReconnectType.type_1`) que acelera o próprio `Interval` (15000→75→1ms) para escalonar. (B) `MinecraftClient.Tick()` (`:823-827`) — contador interno `kickTicks++ > 40` (~2s a 50ms/tick) só em `ReconnectType.type_0`. Um 5º gatilho existe dentro dos 3 macros Solk vivos (auto-heal próprio, 10s de espera). **Nenhum cap de tentativas em nenhum dos 5; nenhum backoff** — o único ajuste de intervalo (timer acelerando 15000→1ms) é o oposto de backoff.
- **Login automático:** sempre o mesmo `StartClient()` (`MinecraftClient.cs:209-279`), initial ou reconexão — sem caminho separado. `LoginResp` (sessão Mojang) é cacheado e reaproveitado em reconexões seguintes para contas premium (irrelevante hoje — Java só suporta modo offline). Timeout de handshake: 20000ms (`:768-772`).
- **Delay/Tentativas/Backoff:** busca exaustiva por `retry|backoff|tentativa|attempt` não encontra nenhum precedente de rede — os únicos hits são lógica de gameplay dos macros Solk (retry de clique de inventário), sem relação com reconexão.
- **Proxy por bot:** `ConProxy` (`MinecraftClient.cs:47`), campo por instância, atribuído no construtor a partir de `ProxyList.NextProxy()` (`Start.cs:170-171`). `Proxy.cs` implementa SOCKS4/SOCKS5/HTTP CONNECT de fato (não é só um dado — é handshake de protocolo de rede).
- **Rotação de proxy:** `ProxyList.NextProxy()` (`ProxyList.cs:36-45`) — round-robin sobre **um único pool global compartilhado** (`Main.Proxies`), índice único (corrida entre bots concorrentes, não corrigida no legado). Disparada em toda exceção de conexão e no motivo de kick específico "muitas contas conectadas com esse IP" (`HandlePacketDisconnect`, `:732-743`).
- **Detecção de Disconnect:** pacote Kick/Disconnect (id varia por versão de protocolo; Java já implementa o de PLAY via `DisconnectPlayPacket`/M5); falha de rede via `PacketStream.OnError` (evento C# real, mas de um único consumidor); timeout de keep-alive client-side, **750 ticks (~37,5s)** sem resposta do servidor (`:773-777`) — capacidade que o Java **não tem** hoje (só responde ao keep-alive do servidor, nunca detecta ausência).
- **Eventos de conexão/perda de conexão:** busca exaustiva confirma **zero** evento/delegate real de conexão (`OnConnected`/`OnDisconnected` não existem). Todo estado de conexão no legado é poll direto de `beingTicked`/`IsBeingTicked()` (~15 pontos de chamada). `IPlugin.onClientConnect` existe mas é iteração manual de dicionário, não `event`, e não tem par de desconexão.
- **Scheduler das macros / Tick do legado:** `CommandManagerNew.Tick()` chama `Tick()` de **todos** os ~29 comandos incondicionalmente a cada ciclo; cada um se auto-ignora (`if (!IsToggled) return;` ou equivalente). `ICommand` não tem `Pause()`/`Stop()` — só `Toggle()`/`IsToggled`. Busca exaustiva por Pause/Suspend/Idle confirma (2ª vez, independente da DEC-29): **sem precedente de pausa** no legado.
- **AutoMiner/AutoAttack/AutoFish/AntiAFK/Follow/Herbalism/Repair/Solk (inventário de lifecycle, sem conteúdo de física/algoritmo):** `AutoMiner` (2 camadas: `CommandMiner` fino + `AutoMiner` helper com `IsMining` próprio, coordenação cross-bot via `HashSet` **estático** de blocos-alvo). `CommandKillAura` (toggle padrão, exclui os próprios outros bots do operador via lista global `Program.FrmMain.Clients`). `CommandPesca`/`CommandMobTeleport`/`CommandMob` (Solk, vivos — `CommandPescaV2`/`CommandMobPlus` existem no código mas **nunca são registrados**, mortos). `CommandAntiAFK` (toggle simples, só enfileira `Movement.Jump`). `CommandFollow` (não usa `Toggle()`/`IsToggled` — estado é só um campo `following`; desliga `Retard` à força ao iniciar). `CommandHerbalism` (auto-desliga se raycast não acha alvo). **"Repair" não existe como classe própria** — é só um estado (`REPARAR`) dentro das 2 máquinas de pesca; troca de ferramenta no Mob é um mecanismo diferente (baú/trade), não "reparo". Dependência cross-macro documentada: Portal/Follow desligam Retard à força; `CurrentPath` é um campo único compartilhado (last-writer-wins, sem checagem); `FileLock` (mutex de arquivo no SO) serializa acesso a baú compartilhado entre processos/contas.
- **Início/pausa/fim/reinício após reconexão:** `CmdManager`/`Commands`/todo campo de cada `ICommand` **nunca são recriados** num reconnect (mesmo objeto `MinecraftClient` do início ao fim) — sobrevivem por construção, não por lógica explícita. Toggles simples (KillAura/AntiAFK/Herbalism/Retard) retomam sozinhos assim que `beingTicked` volta a `true`. **As 3 máquinas de estado Solk vivas têm um bug/gap real:** forçam `estado=FINALIZANDO` a cada tick desconectado e **não saem sozinhas** desse estado ao reconectar — travam até o operador re-togglar manualmente (desligar e ligar de novo).
- **Preservação de estado após reconexão:** resetados explicitamente — `TheWorld`, `PlayerManager`, `Player` (entidade nova a cada conexão), `HotbarSlot`. Preservados por omissão (sem lógica de reset) — `CmdManager`/comandos, `Inventory`, `ChatMessages`, credenciais/proxy.
- **Plugins registrando tarefas:** `IPlugin` real existe (`Tick()`, `onClientConnect`, `OnReceivePacket*`, `onReceiveChat`, `onSendChat`, `Unload`), carregado via reflection de uma pasta `Plugins\`. **Zero plugins reais no projeto.** `IPlugin.Tick()` roda **sem** o gate `beingTicked` (diferente de `ICommand`) — ticka mesmo desconectado. 2 dos 8 hooks (`OnReceivePacket*`, `DoCommand` do `PluginManager`) confirmados mortos/sem call site.

**Riscos arquiteturais identificados:**

1. **`MotorDeTick.tick()` é sequencial, não paralelo, por ciclo.** Cada disparo do `AgendadorDeTarefasPort` roda numa Virtual Thread própria, mas dentro dela o `for` percorre **todos** os bots e **todas** as `TarefaContinua` de cada um, um de cada vez, na mesma thread. O guard `AtomicBoolean` **descarta** (não enfileira, não paraleliza) um ciclo inteiro se o anterior ainda está rodando. Uma `TarefaContinua` lenta (ex.: um futuro macro de pesca com espera de I/O, como o padrão Solk do legado) atrasa/derruba ciclos de **todos os outros bots** registrados no mesmo `MotorDeTick`. Achado crítico para "centenas de bots" — não vira milestone agora (nenhuma tarefa real existe para validar contra, mesma disciplina de "regra de três" já usada pelo projeto), mas fica registrado para quando a 1ª tarefa de longa duração for registrada.
2. **Tensão fidelidade vs. escala em Auto Reconnect:** o legado não tem backoff nem cap de tentativas — fidelidade pura replicaria isso, mas centenas de bots reconectando ao mesmo tempo sem jitter/backoff pode sobrecarregar o servidor de destino. Decisão explícita necessária no Incremento 22.3 (não é automática só por fidelidade).
3. **Legado tem 3+ mecanismos de reconexão redundantes e não totalmente coerentes entre si** (timer global 15s, contador interno ~2s, auto-heal de macro específica 10s) — Incremento 22.3 precisa decidir explicitamente qual portar, não copiar os três.
4. **Máquinas de estado complexas travam em reconexão automática** (achado do Solk) — qualquer `TarefaContinua` futura que use um padrão de máquina de estado (em vez de toggle simples) precisa lidar com isso explicitamente; toggles simples resolvem sozinhos.
5. **Pool de proxy do legado usa um único índice global compartilhado**, sem isolamento por bot — corrida entre bots concorrentes já existe no legado; réplica fiel herda essa corrida.

**Limitações da arquitetura Java atual (antes deste incremento):**

- Nenhuma ligação entre `SessaoDeJogo` (que já sabe quando a conexão cai) e `Bot`/`MotorDeTick` (que decidem se tarefas rodam) — resolvido pelo Incremento 22.1/DEC-30 abaixo.
- `TransporteSocket.readLoop` engolia `IOException` (falha bruta de socket) sem nenhum log/sinal — resolvido pelo Incremento 22.1.
- `SessaoBot.autoReconnect` existe desde a Milestone 3, sempre `false`, sem nenhum setter e sem nenhum leitor — continua inerte; ativá-lo é escopo do Incremento 22.3.
- Nenhum campo de proxy em `EnderecoServidor`/`CredenciaisBot` — zero modelagem ainda.
- `infrastructure.config` permanece vazio — nada desta frente roda em produção ainda, só em teste.
- Transporte do operador (DEC-02) ainda não decidido — bloqueia consumo real de `SaidaDoOperador`/comandos de ciclo de vida fora de teste.

**Onde o MotorDeTick (Milestone 21) já resolve problemas do legado:**

- Isolamento de falha por **tarefa individual** (o legado só isola por ciclo inteiro via `try/catch` genérico em `Main.cs`, e `CommandManagerNew.RunCommand` engole exceção por comando só na execução síncrona de `Run`, não no `Tick()`).
- Registro explícito de tarefas por bot (`registrarTarefa`/`removerTarefa`) substitui a lista fixa de ~29 comandos sempre ticked do legado — mesmo padrão que a DEC-23 já preferiu para `Comando` (sem `Tick`/`Toggle`/`isMacro` no contrato mínimo).
- Virtual Thread por disparo evita o problema implícito do legado de uma única `Thread` de plataforma (`tickThread`) compartilhada por todos os bots.
- Pausa (`EstadoExecucao.PAUSADO`) sem precedente no legado já resolvida como decisão de infraestrutura (DEC-29), não uma pendência desta milestone.

**O que falta para suportar centenas de bots executando tarefas continuamente (achados novos desta análise):**

- Paralelização real por bot/tarefa dentro do `MotorDeTick` (Risco 1) — candidata a milestone futura, condicionada a uma tarefa real de longa duração para validar contra.
- Backoff/jitter de reconexão em massa (Risco 2) — decisão do Incremento 22.3.
- Wiring de produção (DI) do `MotorDeTick`/`AgendadorDeTarefasPort`/`ConexaoBotPort` — sem isso, nada desta frente roda fora de teste (Incremento 22.7).
- Proxy por bot + rotação — zero modelagem hoje (Incrementos 22.5/22.6).
- Detecção de keep-alive de saída (client-side) — Java só responde ao keep-alive do servidor, nunca detecta ausência (candidato dentro do Incremento 22.2).

**Divisão em milestones pequenas e incrementais (M22.1–M22.7):**

- **22.1 — Propagação de Perda de Conexão para o Ciclo de Vida do Bot (DEC-30).** Objetivo: dar a `Bot`/`MotorDeTick` visibilidade de quando uma conexão cai (voluntária, servidor, ou falha de rede), fechando os gaps "`CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` não integrados" (desde M4/M21). Escopo: `ConexaoMinecraft.estaAberta()`, `SessaoDeJogo.estaEncerrada()`/`encerrarVoluntariamente()`, `Bot.registrarDesconexao()`, checagem em `MotorDeTick.tick()`, `AdaptadorConexaoBotV1_8.disconnect()` implementado, `CasoDeUsoDesconectarBot` integrado à Port. Dependências: nenhuma. Riscos: fork arquitetural genuíno (poll vs. push vs. EventBus) — resolvido via DEC-30. Critérios de aceite: desconexão de servidor e desconexão voluntária resultam em `Bot.estadoExecucao=PARADO`/`SessaoBot.state=DISCONNECTED` sem intervenção do operador. **STATUS: CONCLUÍDO nesta sessão — ver Incremento 22.1 abaixo.**
- **22.2 — Detecção de Timeout de Keep Alive (client-side).** Objetivo: portar o timeout de 750 ticks (~37,5s) do legado (`MinecraftClient.cs:773-777`) — hoje Java só responde ao keep-alive do servidor, nunca detecta ausência. Escopo: `SessaoDeJogo` contabiliza tempo desde o último keep-alive recebido; ao ultrapassar o limite, aciona o mesmo caminho do 22.1. Dependências: 22.1. Riscos: baixo, fidelidade direta (conversão de ticks de 50ms para `Duration`). Critérios de aceite: teste simulando ausência de keep-alive além do limite dispara desconexão.
- **22.3 — Auto Reconnect (mecanismo + política).** Objetivo: portar a reconexão automática sobre `AgendadorDeTarefasPort` (M21) e o campo `autoReconnect` já existente (hoje inerte) em `SessaoBot`. Escopo: decidir explicitamente qual(is) dos mecanismos redundantes do legado portar (Risco 3), decidir política de backoff/jitter para escala (Risco 2, tensão fidelidade vs. engenharia — decisão explícita do responsável, não herdada automaticamente), ativar `autoReconnect` via novo método em `Bot`/Caso de Uso. Dependências: 22.1. Riscos: Riscos 2 e 3 acima — requer decisão explícita, não só port automático. Critérios de aceite: bot desconectado com `autoReconnect=true` reconecta sozinho dentro do intervalo escolhido; `autoReconnect=false` não reconecta.
- **22.4 — Retomada de Tarefas Contínuas Após Reconexão.** Objetivo: decidir e implementar o que acontece com `tarefasContinuas` de um bot ao reconectar — `Bot` nunca é recriado (mesma forma do legado), então a lista sobrevive por construção; falta decidir se o bot volta a `EXECUTANDO` sozinho após reconexão bem-sucedida. Dependências: 22.1, 22.3. Riscos: nenhuma tarefa real existe ainda para validar o caso de uma tarefa presa em estado intermediário (Risco 4) — risco documentado, não resolvido, para quando a 1ª macro real for construída. Critérios de aceite: uma `TarefaContinua` fake não executa desconectada e retoma sozinha após reconexão, sem chamada manual a `iniciar()`/`registrarTarefa`.
- **22.5 — Proxy por Bot (modelagem de dados + conexão).** Objetivo: portar o campo de proxy por bot (`ConProxy`) do legado. Escopo: nova Value Object de configuração de proxy (host/port/tipo SOCKS4/SOCKS5/HTTP, fiel a `ProxyType`), `FabricaDeConexaoMinecraftV1_8` roteando a conexão através do proxy quando presente. Dependências: nenhuma (independente da frente de reconnect/tick). Riscos: handshake real de SOCKS4/5/HTTP é lógica de protocolo de rede, não só dado. Critérios de aceite: bot com proxy configurado conecta através dele (validável com proxy local de teste).
- **22.6 — Rotação de Proxy.** Objetivo: portar `ProxyList.NextProxy()` (pool round-robin) e os gatilhos de troca (falha de conexão, motivo de kick específico). Dependências: 22.5, 22.1, 22.3. Riscos: pool global compartilhado do legado (Risco 5) — replicar fielmente (com a mesma corrida) ou corrigir para pool por bot é uma decisão de arquitetura a ser explicitamente colocada para o responsável, não decidida unilateralmente. Critérios de aceite: troca de proxy após falha de conexão simulada.
- **22.7 — Wiring de Produção (DI) do MotorDeTick/AgendadorDeTarefasPort/ConexaoBotPort.** Objetivo: sair do estado "`infrastructure.config` vazio". Escopo: `@Configuration` Spring ligando `MotorDeTick` a um agendamento real, registrando/removendo bots no `MotorDeTick` nos momentos certos. Dependências: idealmente após 22.1–22.4. Riscos: decisão de transporte CLI/API (DEC-02) permanece bloqueando o consumo de fora de testes — DI sozinho não resolve isso. Critérios de aceite: aplicação Spring Boot sobe, um bot criado é registrado no `MotorDeTick`, e um ciclo real de tick acontece fora de teste JUnit.

Nenhuma DEC pendente bloqueia 22.2–22.7; cada uma pode ser reavaliada/reordenada independentemente pelo responsável do projeto.

### Incremento 22.1 — Propagação de Perda de Conexão para o Ciclo de Vida do Bot (DEC-30)

Status

Concluído

Análise do legado

Ver "Levantamento do legado" acima (Detecção de Disconnect, Eventos de conexão/perda de conexão). Achado central: o legado nunca resolveu propagação de desconexão com um mecanismo de notificação — resolveu com polling puro de `beingTicked`/`IsBeingTicked()` (`MinecraftClient.cs:89,750-753`), sem nenhum `event` real de conexão em todo o projeto.

Decisão: DEC-30 — poll de `SessaoDeJogo.estaEncerrada()` dentro de `MotorDeTick.tick()`, não um callback/listener nem um EventBus. Ver DEC-30 em 01-Decisoes-Arquiteturais.md para as 3 alternativas e a análise completa.

Entregue

- `domain.network.ConexaoMinecraft.estaAberta()` (novo método `default` retornando `true` — preserva os 65 fakes de teste existentes que implementam esta interface, sem exigir alteração em nenhum).
- `infrastructure.network.TransporteSocket.estaAberta()` (sobrescreve, retorna o campo `active` já existente).
- `domain.bot.SessaoDeJogo.estaEncerrada()` (novo — `motivoDesconexao != null || !conexao.estaAberta()`) e `encerrarVoluntariamente()` (novo, simétrico a `encerrarPorDesconexaoDoServidor`, para o caminho de desconexão pelo operador).
- `domain.bot.Bot.registrarDesconexao()` (novo — `estadoExecucao=PARADO` + `session=session.disconnect()`, chamado quando uma desconexão é detectada).
- `infrastructure.execucao.MotorDeTick.tick()` ganha a checagem: para bot `EXECUTANDO` com `SessaoDeJogo` presente e `estaEncerrada()==true`, chama `bot.registrarDesconexao()` antes do filtro de `EstadoExecucao` já existente — na mesma passada, o bot já para de executar tarefas neste ciclo.
- `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8.disconnect(Bot)` implementado de fato (antes lançava `UnsupportedOperationException` desde a Milestone 4) — delega a `SessaoDeJogo.encerrarVoluntariamente()`.
- `application.usecase.CasoDeUsoDesconectarBot` ganha construtor com `ConexaoBotPort` (antes não tinha nenhuma dependência e nenhum call site de produção/teste) — `disconnect(bot)` agora chama `porta.disconnect(bot)` + `bot.registrarDesconexao()`.
- Nenhuma interface já aprovada teve assinatura alterada (`ConexaoMinecraft.estaAberta()` é `default`); nenhum Packet/Codec/Comando/bounded context novo.

Testes

`SessaoDeJogoTest` (+4: não encerrada logo após conectar, encerrada após desconexão do servidor, encerrada após encerramento voluntário, encerrada quando a conexão fecha sem motivo explícito — cobrindo o caminho de falha bruta de rede via `estaAberta()`); `BotTest` (+1: `registrarDesconexao` para o bot e desconecta a sessão); `MotorDeTickTest` (+2: registra desconexão e ignora tarefas quando a sessão de jogo está encerrada; não mexe em bot sem sessão de jogo); `TransporteSocketTest` (+1: `estaAberta()` antes/depois de `close()`); `AdaptadorConexaoBotV1_8Test` (+1: `disconnect(bot)` encerra a conexão de um bot conectado de verdade, via socket loopback real); `CasoDeUsoDesconectarBotTest` (novo arquivo, +2: chama a porta com o bot recebido; para o bot e desconecta a sessão após chamar a porta).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **737 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 726→737 (+11), mudança 100% aditiva — nenhum teste pré-existente foi modificado.

### Incremento 22.2 — Detecção de Timeout de Keep-Alive (client-side)

Status

Concluído

Análise do legado

`MinecraftClient.cs:773-777` — `if (keepAliveTicks++ > 750) { Disconnect("Conexão perdida: timeout"); return; }`, dentro de `Tick()`. **750 ticks a ~50ms ≈ 37500ms.** `keepAliveTicks` é reiniciado por `ResetKeepAlive()` (`:636-639`) sempre que um pacote Keep Alive do servidor é processado (`Handler_v17.cs:24-27`, `Handler_v18.cs` equivalente). Java só respondia ao Keep Alive do servidor (`SessaoDeJogo.responderKeepAlive`, Milestone 5) — nunca detectava a ausência dele.

Decisão: fidelidade direta, sem DEC nova — é conversão de constante (750 ticks × 50ms), não uma decisão arquitetural nova; reaproveita o mecanismo de propagação de desconexão já decidido na DEC-30.

Entregue

- `domain.bot.SessaoDeJogo.LIMITE_KEEP_ALIVE` (novo, `public static final Duration`, 37500ms, fiel a `MinecraftClient.cs:773-777`).
- `domain.bot.SessaoDeJogo.responderKeepAlive(int id)` passa a atualizar um novo campo `ultimoKeepAliveRecebidoEm` (porta de `ResetKeepAlive()`) além de enviar a resposta ao servidor — comportamento existente preservado, só ganhou um efeito colateral adicional.
- `domain.bot.SessaoDeJogo.keepAliveExpirou(Duration limite)` (novo, query pura).
- `infrastructure.execucao.MotorDeTick.tick()` — para bot `EXECUTANDO` com `SessaoDeJogo`, se `keepAliveExpirou(LIMITE_KEEP_ALIVE)` e a sessão ainda não está encerrada, chama `encerrarPorDesconexaoDoServidor("Conexão perdida: timeout")` (mesma mensagem do legado) — reaproveita 100% do caminho de detecção de desconexão já entregue na DEC-30/Incremento 22.1.
- Nenhuma interface já aprovada alterada; nenhum Packet/Port/agregado/bounded context novo.

Testes

`SessaoDeJogoTest` (+3: não expira com limite ainda não atingido; expira quando o limite já foi ultrapassado — via `Duration.ofMillis(-1)`, determinístico, sem sleep real; `responderKeepAlive` reinicia a contagem).

Validação executada

Incluída na rodada consolidada ao final do Incremento 22.6 (ver abaixo) — 766 testes, 0 falhas.

### Incremento 22.3 — Auto Reconnect: Mecanismo e Política (DEC-31)

Status

Concluído

Análise do legado

Ver DEC-31 (01-Decisoes-Arquiteturais.md) para o levantamento completo dos dois mecanismos redundantes do legado (`kickTicks>40` ~2000ms por bot; `autReconnectTimer` 15000ms compartilhado) e a ausência total de backoff/cap de tentativas.

Decisão: DEC-31 — regra funcional idêntica ao legado (retry incondicional até sucesso, sem cap), intervalo entre tentativas extraído para uma política injetável (`PoliticaDeReconexao`), com jitter configurável para evitar tempestade de reconexão.

Entregue

- `domain.bot.PoliticaDeReconexao` (novo, interface funcional `Duration proximoIntervalo(int tentativa)`).
- `domain.bot.PoliticaDeReconexaoComJitter` (novo, record — `intervaloBase` + `jitterMaximo`; `jitterMaximo=ZERO` reproduz fidelidade literal ao legado).
- `infrastructure.execucao.GerenciadorDeReconexao` (novo, mesma categoria arquitetural de `MotorDeTick`): `registrar(Bot)`/`remover(Bot)` (mesmo formato de `MotorDeTick`), `verificar()` percorrendo bots registrados; para bot com `SessaoBot.state()==DISCONNECTED` e `autoReconnect()==true`, aplica a política para decidir quando tentar de novo e chama `CasoDeUsoConectarBot.connect(bot)` (reaproveitado sem alterações, DEC-21); falha loga (`WARN`) e reagenda conforme a política — nunca desiste.
- Ativação do campo `SessaoBot.autoReconnect` (existente desde a Milestone 3, sempre `false`, sem nenhum leitor até agora): passa a ser lido de fato pela primeira vez, via `bot.getSession().autoReconnect()`.
- Nenhum novo Port — reaproveita `ConexaoBotPort` (via `CasoDeUsoConectarBot`) e não precisa de `AgendadorDeTarefasPort` diretamente nesta forma de teste (wiring de produção fica para o Incremento 22.7); nenhum agregado/bounded context novo.

Testes

`PoliticaDeReconexaoComJitterTest` (novo, 4 testes: rejeita intervalo/jitter negativos, intervalo fixo quando jitter=zero, resultado sempre entre base e base+jitter). `GerenciadorDeReconexaoTest` (novo, 10 testes: ignora bot conectado; ignora bot sem autoReconnect; reconecta e retoma execução — ver Incremento 22.4; não tenta de novo antes do intervalo da política elapsar; tenta de novo depois do intervalo elapsar — com Duration curta real, sem flakiness de jitter; ignora bot removido).

Validação executada

Incluída na rodada consolidada ao final do Incremento 22.6 (ver abaixo) — 766 testes, 0 falhas.

### Incremento 22.4 — Retomada Automática das Tarefas Contínuas Após Reconnect

Status

Concluído

Análise do legado

`CmdManager`/`Commands`/todo campo de cada `ICommand` nunca são recriados num reconnect (`MinecraftClient.cs`, mesma instância do início ao fim) — comandos de toggle simples (KillAura/AntiAFK/Herbalism) retomam sozinhos assim que `beingTicked` volta a `true`, sem qualquer código dedicado a "retomar". As 3 máquinas de estado Solk vivas são a exceção (travam em `FINALIZANDO`, ver levantamento da Milestone 22) — mas isso é um problema de *conteúdo* de macro com estado complexo, fora de escopo desta frente.

Decisão: sem DEC nova — `Bot` (agregado) nunca é recriado no fluxo Java (mesma forma do legado), então `tarefasContinuas` já sobrevive à desconexão por construção desde a Milestone 21; a única peça que faltava era decidir se o bot volta a `EstadoExecucao.EXECUTANDO` sozinho após uma reconexão bem-sucedida.

Entregue

- `infrastructure.execucao.GerenciadorDeReconexao.verificarBot`, no caminho de sucesso, chama `bot.iniciar()` após `CasoDeUsoConectarBot.connect(bot)` retornar sem exceção — reaproveita `Bot.iniciar()` (Milestone 21) sem nenhuma mudança nele. Como `tarefasContinuas` nunca foi tocado por `registrarDesconexao()` (Incremento 22.1) nem por `CasoDeUsoConectarBot`, as tarefas já registradas retomam automaticamente no próximo ciclo do `MotorDeTick`, sem qualquer chamada manual a `registrarTarefa`.
- Nenhum código novo dedicado além dessa única chamada — o mecanismo já existia (M21 + 22.1); faltava só decidir e ligar o "retomar" ao sucesso da reconexão.

Testes

Coberto por `GerenciadorDeReconexaoTest.deveReconectarBotDesconectadoComAutoReconnectAtivadoERetomarExecucao` (Incremento 22.3, acima) — verifica `EstadoExecucao.EXECUTANDO` após reconexão bem-sucedida.

Risco documentado (não resolvido, sem consumidor real ainda): se uma futura `TarefaContinua` real for implementada como máquina de estado (em vez de um toggle simples), ela pode reproduzir o bug do Solk do legado (travar num estado intermediário após reconectar) — a retomada aqui só reativa o `MotorDeTick` para a tarefa, não garante que o *conteúdo* da tarefa lide bem com uma interrupção no meio da execução. Revisitar quando a 1ª `TarefaContinua` com estado complexo for construída.

Validação executada

Incluída na rodada consolidada ao final do Incremento 22.6 (ver abaixo) — 766 testes, 0 falhas.

### Incremento 22.5 — Proxy por Bot

Status

Concluído

Análise do legado

`MinecraftClient.cs:47` (`public Proxy ConProxy`) — campo por instância, atribuído no construtor. `Proxy.cs` (546 linhas) implementa SOCKS4/SOCKS5/HTTP CONNECT manualmente (C# não tem suporte nativo). `ProxyType.cs`: `Socks4=4, Socks5=5, HTTP=2`. Todo call site de conexão: `(ConProxy == null) ? new TcpClient(...) : ConProxy.Connect(...)`.

Decisão: sem DEC nova — extensão puramente aditiva de `EnderecoServidor` (novo campo `ConfiguracaoProxy proxy`, construtor de 2 argumentos preservado via sobrecarga que delega com `proxy=null`), sem tocar `ConexaoBotPort`/`Bot`/`CredenciaisBot`. Única divergência de implementação (documentada, não de comportamento observável): SOCKS4/SOCKS5/HTTP CONNECT são implementados nativamente via `java.net.Proxy`/`Socket(Proxy)` do próprio JDK, em vez de reconstruir o handshake manual de `Proxy.cs` — SOCKS4 e SOCKS5 mapeiam para o mesmo `Proxy.Type.SOCKS` (o JDK negocia a versão internamente); a distinção de 3 tipos é preservada só na configuração (`TipoDeProxy`), fiel ao `ProxyType` do legado.

Entregue

- `domain.bot.TipoDeProxy` (novo enum: `SOCKS4`, `SOCKS5`, `HTTP`).
- `domain.bot.ConfiguracaoProxy` (novo record: `host`/`port`/`tipo`).
- `domain.bot.EnderecoServidor` ganha um 3º campo `proxy` (nullable) via novo construtor canônico de 3 argumentos; o construtor de 2 argumentos existente vira uma sobrecarga que delega com `proxy=null` — **zero call site existente quebrado** (dezenas de `new EnderecoServidor(host,port)` em todo o projeto continuam compilando sem alteração).
- `domain.bot.Bot.enderecoServidor` deixou de ser `final` (era usado só internamente via getter Lombok, nenhuma API pública mudou) e ganhou `trocarProxy(ConfiguracaoProxy)` (novo — reconstrói o `EnderecoServidor` com o mesmo host/port e o novo proxy).
- `infrastructure.network.v1_8.FabricaDeConexaoMinecraftV1_8.apply` passa a abrir o `Socket` através de `enderecoServidor.proxy()` quando presente (`new Socket(new java.net.Proxy(tipo, enderecoDoProxy))`), preservando exatamente o comportamento anterior quando `proxy()==null`.

Testes

`ConfiguracaoProxyTest` (novo, 3), `EnderecoServidorTest` (novo, 4 — incluindo os 2 casos de validação que já existiam implicitamente), `BotTest` (+1: `trocarProxy` preserva host/porta), `FabricaDeConexaoMinecraftV1_8Test` (+1: conecta de fato através de um proxy HTTP CONNECT fake via socket loopback real, provando que `java.net.Proxy` funciona ponta a ponta para este caso de uso — não foi escrito um teste equivalente para SOCKS4/SOCKS5 isoladamente, já que ambos compartilham o mesmo `Proxy.Type.SOCKS` do JDK; a cobertura dessa rota é por revisão de código, não por um 2º servidor fake).

Validação executada

Incluída na rodada consolidada ao final do Incremento 22.6 (ver abaixo) — 766 testes, 0 falhas.

### Incremento 22.6 — Rotação de Proxy

Status

Concluído

Análise do legado

`ProxyList.cs:36-45` (`NextProxy()`) — round-robin sobre **um único pool global compartilhado** (`Main.Proxies`), índice único não sincronizado. Disparada em toda exceção de conexão (6 call sites em `MinecraftClient.cs`) e no motivo de kick específico `"Já existem muitas contas conectadas com esse IP."` (`HandlePacketDisconnect`, `:732-743`).

Decisão: sem DEC nova — replica o round-robin fielmente, com uma pequena melhoria de qualidade documentada: o índice usa `AtomicInteger` (sem perda de incremento sob concorrência real de várias tentativas de reconexão simultâneas), e a rotação só troca o proxy do bot quando o pool tem pelo menos 1 entrada — o legado, com o pool global vazio, atribuiria `null` incondicionalmente mesmo a um bot que já tinha um proxy próprio configurado; tratado como um acidente de implementação do legado, não uma regra de negócio, e não replicado.

Entregue

- `domain.bot.PoolDeProxies` (novo — round-robin sobre `List<ConfiguracaoProxy>`, porta `ProxyList.NextProxy()`).
- `infrastructure.execucao.GerenciadorDeReconexao` ganha rotação de proxy: (a) toda falha de tentativa de reconexão chama `bot.trocarProxy(proxies.proximo())` quando o pool não está vazio (fiel aos 6 call sites de exceção do legado); (b) na primeira observação de um bot desconectado, se `SessaoDeJogo.motivoDesconexao()` igual a `"Já existem muitas contas conectadas com esse IP."`, rotaciona imediatamente antes mesmo da primeira tentativa (fiel ao 7º call site do legado, reaproveitando `motivoDesconexao()` entregue pela DEC-30).

Testes

`PoolDeProxiesTest` (novo, 3: rejeita lista nula, retorna nulo com pool vazio, gira em round-robin). `GerenciadorDeReconexaoTest` (+3: rotaciona após falha; não rotaciona com pool vazio; rotaciona imediatamente por motivo de "muitas contas"; não rotaciona por outro motivo de desconexão).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **766 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 737→766 (+29 testes, cobrindo os Incrementos 22.2 a 22.6). Mudança 100% aditiva em todos os 5 incrementos — nenhum teste pré-existente foi modificado, nenhuma assinatura de interface já aprovada foi alterada.

### Incremento 22.7 — Wiring de Produção (DI) do MotorDeTick/AgendadorDeTarefasPort/GerenciadorDeReconexao/ConexaoBotPort

Status

Concluído

Análise do legado

`Main.cs` (`Main_Load`, ~299-327): composição de dependências acontece inline no bootstrap da UI WinForms — `tickThread` iniciada diretamente, `autReconnectTimer`/`Statistics.timer1` instanciados e ligados no mesmo método, sem nenhum container de DI ou fase de composição separada (sem equivalente formal de "Composition Root" no legado). `ConnectAndHandshakeAsync`/`ConnectCallback` (`MinecraftClient.cs:421-502`): a conexão TCP em si (`TcpClient.BeginConnect`) não tem nenhum timeout de conexão explícito no legado — só `ReceiveTimeout`/`SendTimeout` (30000ms) depois de já conectado; o watchdog de 20000ms (`Tick()`, `:768-772`) cobre handshake+login, já portado com fidelidade desde a Milestone 22.2/`AdaptadorConexaoBotV1_8`.

Decisão: sem DEC nova — escopo é puramente wiring de produção (`@Configuration`/`SmartLifecycle` do Spring) sobre componentes já aprovados nas DEC-29/30/31; nenhuma regra de negócio nova, nenhuma interface pública alterada. Um pequeno conjunto de valores de infraestrutura sem precedente direto no legado foi necessário para ligar os componentes — mesmo critério que a própria DEC-31 já previu ao deixar `intervaloBase`/`jitterMaximo` como "decisão pendente de wiring, não uma lacuna da DEC":
- Cadência do tick de produção: 50ms/20Hz — fiel ao `tickThread` do legado (`Main.cs:299-327`) e ao texto da própria DEC-29 ("uma tarefa agendada a 50ms cujo corpo é `MotorDeTick.tick()`").
- Cadência de varredura do `GerenciadorDeReconexao.verificar()`: 1000ms — sem equivalente direto no legado (que conflava cadência de varredura e intervalo de retry num único timer, `autReconnectTimer`); o intervalo de retry em si continua 100% decidido por `PoliticaDeReconexao` (DEC-31), não por esta cadência de varredura.
- `PoliticaDeReconexaoComJitter` de produção: `intervaloBase=2000ms` (fiel ao mecanismo interno do legado, `kickTicks>40` a ~50ms/tick), `jitterMaximo=1000ms` (novo, sem precedente — distribui o instante de cada retry para evitar tempestade de reconexão quando muitos bots caem juntos, objetivo de escala que a própria DEC-31 já registrou).
- Timeout de conexão TCP (`FabricaDeConexaoMinecraftV1_8`): 10000ms — sem equivalente no legado; orçamento novo para não travar uma Virtual Thread indefinidamente contra um servidor de destino inalcançável.
- `PoolDeProxies` de produção: vazio (`List.of()`) — fiel ao `Main.Proxies` do legado, que só ganha entradas quando o operador de fato configura proxies (fora de escopo, sem CLI/API nesta milestone).

Entregue

- `infrastructure.config.ConfiguracaoDeConexao` (novo, `@Configuration`): beans `Function<EnderecoServidor,ConexaoMinecraft>` (`FabricaDeConexaoMinecraftV1_8`), `ConexaoBotPort` (`AdaptadorConexaoBotV1_8`), `CasoDeUsoCriarBot`, `CasoDeUsoConectarBot`, `CasoDeUsoDesconectarBot`.
- `infrastructure.config.ConfiguracaoDeExecucao` (novo, `@Configuration`): beans `AgendadorDeTarefasPort` (`AgendadorDeTarefasVirtualThread`), `MotorDeTick`, `PoliticaDeReconexao` (`PoliticaDeReconexaoComJitter`), `PoolDeProxies`, `GerenciadorDeReconexao`, e registra `CicloDeVidaDoMotorDeExecucao`.
- `infrastructure.config.CicloDeVidaDoMotorDeExecucao` (novo, `implements SmartLifecycle` do Spring): `start()` agenda `MotorDeTick.tick()` (50ms) e `GerenciadorDeReconexao.verificar()` (1000ms) via `AgendadorDeTarefasPort.agendar(...)`; `stop()` chama `AgendadorDeTarefasPort.encerrar()`, liberando a thread do relógio de forma limpa. `spring.lifecycle.timeout-per-shutdown-phase` (já configurado em `application.yml` desde o scaffold inicial do projeto) governa o orçamento de tempo desta fase no shutdown.
- `application.bootstrap.AdvancedBotApplication.onExit()` ajustado — comentário TODO obsoleto removido (o encerramento de fato agora acontece em `CicloDeVidaDoMotorDeExecucao.stop()`, que roda antes do `@PreDestroy` no shutdown do Spring).
- Registro/remoção de bots individuais no `MotorDeTick`/`GerenciadorDeReconexao` **não** foi ligado a nenhum call site de produção nesta milestone — hoje não existe nenhum ponto de entrada (CLI/API, DEC-02 ainda não decidida) que crie/conecte um bot fora de teste; `MotorDeTick.registrar`/`GerenciadorDeReconexao.registrar` continuam prontos e testados, aguardando esse consumidor futuro — mesmo padrão de "capacidade sem chamador de produção obrigatório" já aceito desde a DEC-24/27/28/29.
- Nenhuma interface já aprovada teve assinatura alterada; nenhum Packet/Port/agregado/bounded context novo.

Testes

`BootstrapDeProducaoTest` (novo, 3, `@SpringBootTest(classes = AdvancedBotApplication.class)`): todos os beans de produção sobem corretamente e com o tipo concreto esperado; um ciclo real de `MotorDeTick.tick()` executa contra um bot registrado, disparado sozinho pelo agendamento de produção (50ms), sem nenhuma chamada manual a `tick()` — via `CountDownLatch`, sem sleep; `CicloDeVidaDoMotorDeExecucao.stop()` encerra o `AgendadorDeTarefasPort` de forma limpa (`RejectedExecutionException` em agendamento posterior), teste isolado com `@DirtiesContext`. `AdvancedBotApplicationTests` (já existente desde o scaffold) continua validando o boot do contexto completo.

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **794 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 791→794 (+3), mudança 100% aditiva. Validação adicional fora do JUnit: aplicação subida de fato via `mvn spring-boot:run` (processo real, não `SpringBootTest`) — log confirma `CicloDeVidaDoMotorDeExecucao` iniciando o agendamento na subida e o `SpringApplicationShutdownHook` do próprio Spring Boot encerrando-o de forma limpa (`AgendadorDeTarefasPort.encerrar()`) antes do `@PreDestroy` de `AdvancedBotApplication` — ciclo de vida completo (start→shutdown gracioso) confirmado em processo de produção real, não só em teste de integração.

---

## Milestone 23

Status

CONCLUÍDO

Objetivo

Implementar a primeira Macro real do projeto sobre a engine de execução contínua da Milestone 21/22 (`MotorDeTick`/`TarefaContinua`), analisando exclusivamente `CommandAntiAFK` do legado. Análise levantada EXCLUSIVAMENTE contra `Projeto Adv 2.4.5` — nenhum outro projeto/backup/fork AdvancedBot consultado.

### Análise do legado

`CommandAntiAFK` (`AdvancedBot.Client.Commands/CommandAntiAFK.cs`): `ICommand` simples, toggle único (sem máquina de estado), 2 campos (`delay` int ms, default 5000; `lastJump` long). Timer wall-clock via `Utils.GetTickCount64()` (não contagem de tick de jogo). `Tick()`: se ligado e `tickCount - lastJump > delay`, enfileira `Movement.Jump` e reseta `lastJump`. Consumo da fila em `Entity.Tick()`/`Move()` (`Entity.cs:96-200,238-328`) — `Jump` só vira `MotionY=0.42` se `OnGround==true`; o arco inteiro (gravidade `-0.08`/tick, drag `*0.98`, colisão AABB vertical via `AABB.ClipYCollide`) é física completa, disparada a cada tick via o gate `MapAndPhysics` em `MinecraftClient.Tick()` (`:787-821`), nunca sob demanda. Reconexão: nenhuma integração dedicada — o campo vive no próprio `Player`/`Entity`, nunca recriado; o toggle simples retoma sozinho quando `beingTicked` volta a `true` (já coberto genericamente pelo `GerenciadorDeReconexao`/`bot.iniciar()` da M22.4, zero trabalho novo). Dependência de Mundo: só indireta, via física (atrito lê bloco sob os pés). Dependência de SessaoDeJogo equivalente: nenhuma direta no legado (tudo em `Entity`/`Player`). Dependência de Inventário: nenhuma. Dependência de Movimento: total — a própria física decide quando/quanto pular e quando mandar pacote.

Achado central: o único mecanismo de `CommandAntiAFK` depende integralmente de um motor de física local (gravidade, colisão vertical, `OnGround`) nunca portado para Java — bloqueio explícito da **DEC-22** (Milestone 9), reafirmado sem reabertura por DEC-23/27/28/29/30/31, com o próprio `CommandAntiAFK` citado nominalmente na lista "fora de escopo por política" (Seção 10 abaixo, versão anterior deste documento). Análise interrompida neste ponto e apresentada ao responsável do projeto, conforme instrução explícita da sessão.

Decisão: **DEC-32** — reabertura parcial da DEC-22, restrita a um motor de física **vertical mínimo** (gravidade + colisão vertical + `OnGround` + impulso de pulo, sem movimento horizontal/água/lava/escada, que permanecem bloqueados), com fidelidade de forma de bloco no nível **alturas parciais simples** (slab/snow layer/soul sand/carpet — caixa única de altura variável; stairs/fences/trapdoors/beds/doors/cake permanecem cubo cheio, fora de escopo). Ambas as escolhas decididas explicitamente pelo responsável do projeto após apresentação das alternativas. Ver DEC-32 em 01-Decisoes-Arquiteturais.md para a análise completa.

Entregue

- `domain.protocol.v1_8.Bloco.alturaSuperficie()` (novo) — altura do topo de colisão (0.0-1.0), fiel ao subconjunto aprovado de `BlockUtils.AddAABBsToList`: slabs 44/126/182/205 (0.5 ou 1.0 conforme metadata), snow layer 78 (`(metadata&7)*0.125`), soul sand 88 (0.875), carpet 171 (0.0625), default 1.0 (cubo cheio) para qualquer outro sólido.
- `domain.bot.SessaoDeJogo.onGround()` (novo getter, campo `onGround` default `false`), `pular()` (novo — arma `motionY=0.42` se `onGround`, fiel ao gate de `Entity.cs:137-140`), `aplicarFisicaVertical()` (novo — resolve colisão vertical contra blocos sólidos do Mundo via porte direto de `AABB.ClipYCollide`, varrendo as até 4 colunas XZ tocadas pela caixa da entidade; atualiza `y`/`onGround`/`motionY`; só chama `mover(x,y,z,onGround)` — já existente desde a DEC-22 — quando `y` de fato muda; aplica gravidade/drag no fim, mesma ordem exata de `Entity.Tick()`).
- `domain.bot.TarefaAntiAFK` (novo, `implements TarefaContinua`) — timer wall-clock (`System.currentTimeMillis()`, mesmo padrão de `SessaoDeJogo.keepAliveExpirou`), chama `pular()` antes de `aplicarFisicaVertical()` quando o delay elapsa (mesma ordem do `Tick()` legado); guarda defensiva contra `sessaoDeJogo==null` (lacuna documentada e aceita desde a DEC-29, "até a 1ª TarefaContinua real existir" — esta é essa tarefa).
- `interfaces.comando.ComandoAntiAFK` (novo) — toggle sem CasoDeUso dedicado (registrar/remover `TarefaContinua` não envia Packet), busca uma `TarefaAntiAFK` já registrada via `instanceof` em `bot.getTarefasContinuas()`; delay opcional (`[delay]`, default 5000ms); mensagens via `SaidaDoOperador` (`"§6AntiAFK §aON"`/`"§6AntiAFK §cOFF"`, fiel ao formato do legado). Divergência deliberada: delay inválido (`<=0`) retorna `ERRO` **sem** ligar nada — o legado toggla incondicionalmente antes de validar, deixando um bug de jump-spam com delay quebrado armazenado; não portado.
- Nenhuma interface já aprovada teve assinatura alterada; nenhum Packet/Port/Caso de Uso novo; nenhum bounded context novo.

Testes

`BlocoTest` (+6: `alturaSuperficie()` para cubo cheio, slab inferior/superior, snow layer, soul sand, carpet). `SessaoDeJogoTest` (+12: `onGround` inicial falso, `pular()` no-op fora do chão, `pular()` aplica impulso no chão, queda por gravidade e pouso em bloco normal, sem reenvio de pacote em repouso, pouso correto em slab/snow/soul sand/carpet, colisão de teto ao subir). `TarefaAntiAFKTest` (novo, 4: pula com delay elapsado e no chão, não pula antes do delay, não pula sem estar no chão, no-op sem sessão de jogo). `ComandoAntiAFKTest` (novo, 5: liga com delay padrão/parseado, desliga na segunda chamada, rejeita delay zero/negativo e não-numérico sem registrar nada).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **791 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 766→791 (+25), mudança 100% aditiva — nenhum teste pré-existente foi modificado.

---

## Milestone 24

Status

CONCLUÍDA — Incremento 24.1 (DEC-33). Abre a Fase 2 do projeto (reconstrução de macros/automações, Fase 1 — Infraestrutura e Protocolo — encerrada na Milestone 23). Demais frentes da Fase 2 ficam como candidatos na Seção 10, pendentes de escolha do responsável do projeto.

Objetivo

Encerrada a Fase 1, iniciar a Fase 2: reconstruir os comportamentos (macros/automações) do AdvancedBot legado. Primeiro passo, antes de qualquer macro nova: análise exaustiva do framework de macros do legado (`Macro`/`ICommand`/`ICommand.Tick`, AutoMiner, AutoFish, AutoAttack, Herbalism, Solk, Follow, Portal e demais automações) e inventário completo da arquitetura Java atual, para confirmar quais peças já existem, quais faltam, e se há lacuna arquitetural real. Análise levantada EXCLUSIVAMENTE contra `Projeto Adv 2.4.5` — nenhum outro projeto/backup/fork AdvancedBot consultado.

### Fase de Planejamento — Framework de Macros do Legado e Inventário da Arquitetura Java

**Levantamento do legado (framework central):**

- `ICommand` (`AdvancedBot.Client.Commands/ICommand.cs`) é, apesar do nome, uma **classe abstrata**, não uma interface — base comum tanto para comandos single-shot quanto para macros contínuas. Campos: `IsToggled`/`ToggleText`/`isMacro` (este último **morto** — setado no construtor, nunca lido em lugar nenhum do projeto, confirmado por grep). `Run(alias, args)` (entrada via chat, retorna `CommandResult`) e `Tick()` (`virtual`, no-op por padrão, sem parâmetro nem retorno — conclusão/continuação é 100% implícita via estado interno de cada subclasse, sem nenhuma sinalização "pronto").
- `CommandManagerNew` é o registro: lista fixa de ~29 comandos (`new CommandXxx(c)`, wiring em código, sem factory/reflection/DI); `Tick()` chama `Tick()` de **todos incondicionalmente**, cada um se autogoverna via `IsToggled`.
- Motor central: `Thread` dedicada em `Main.cs` (~50ms/ciclo via `Stopwatch`, sem catch-up), itera `List<MinecraftClient> Clients` chamando `Tick()` de cada um; o `try/catch` envolve o `for` inteiro, não cada cliente (uma exceção pula o resto do ciclo para os demais bots — achado não replicado em Java, `MotorDeTick` já isola por tarefa individual desde a DEC-29). Dentro de `MinecraftClient.Tick()`, a flag `beingTicked` (true após handshake, false na desconexão) é o único gate real de todo trabalho por tick, incluindo macros — não existe lista de "ativos/inativos", só essa flag. Confirmado o paralelo direto com o par já existente em Java: `Clients.Add` (uma vez, na criação) ≈ `MotorDeTick.registrar` e `beingTicked` ≈ `Bot.estadoExecucao`.
- Confirmados **3 subsistemas distintos de "automação"** coexistindo no legado, tratados como conceitos separados neste levantamento: (1) toggles simples sobre `ICommand.Tick()` (maioria — AutoMiner, KillAura, Herbalism, Follow, AntiAFK); (2) motor de script do usuário (`CommandScript`/`ScriptContext`/`JsScriptContext` via Jint, arquivos `.txt`/`.js` em `macros/`) — não analisado em profundidade nesta sessão, fora do escopo pedido; (3) máquinas de estado assíncronas só nos 5 macros "Solk" (`CommandMob`/`CommandMobPlus`/`CommandMobTeleport`/`CommandPesca`/`CommandPescaV2`, `async void Tick()` com guarda de reentrância `isProcessing` — necessária porque `await Task.Delay` devolve controle ao chamador no meio do ciclo).

**Levantamento das 7 automações pedidas (arquivo/classe, dependências, bloqueio real):**

- **AutoMiner** (`AutoMiner.cs` + `CommandMiner.cs`): mineração automática com pathfinding + seleção de ferramenta + quebra temporizada. Depende de movimento **horizontal** (`RequestPathTo`/`CurrentPath`/`PathGuide`) — bloqueada pela DEC-22 (não reaberta para horizontal).
- **AutoFish** (não existe classe "AutoFish" — é `CommandPesca`, Solk): 100% baseada em teleporte por comando de chat (`/home ...`), **sem pathfinding, sem movimento horizontal físico** — não esbarra na DEC-22. Depende de interação com baú/janela (container/window), capacidade que a Java atual **não tem ainda** (`InventarioDoJogador` só cobre a janela do próprio jogador, 45 slots).
- **AutoAttack** (não existe "AutoAttack" — é `CommandKillAura`): **permanentemente fora de escopo por política do projeto** (combate/automação, já registrado nesta seção antes desta sessão) — não é candidato, independente de dependência técnica.
- **Herbalism** (`CommandHerbalism.cs`): fazenda estacionária de cana-de-açúcar — **sem pathfinding, sem movimento horizontal**, só olha para baixo, quebra instantaneamente (`BreakBlock`, não a quebra temporizada de `DiggingHelper`) e replanta (`PlaceCurrentBlock`). Todas as capacidades de que depende já existem em Java hoje (raycast, `CasoDeUsoColocarBloco`; seleção de hotbar é a única peça pequena a confirmar). **Não bloqueada pela DEC-22 nem por nenhuma lacuna nova de framework.**
- **Solk**: o alias literal `"solk"` mapeia para `CommandMobTeleport`, que está **incompleto no próprio legado** (loop de ataque comentado, só tenta um exploit de corner-glitch). A macro "Solk" completa de fato é `CommandMob` (alias `"solkmob"`) — 100% teleporte por chat, sem pathfinding, mas com um loop de **ataque sustentado contra mobs** (850 hits a 100ms). Ambiguidade real, não resolvida nesta sessão: esse loop de combate conta como "combate/automação (KillAura e afins)", já permanentemente fora de escopo? Fica registrada como pergunta em aberto na Seção 10.
- **Follow** (`CommandFollow.cs`): anda até um jogador nomeado — depende de execução de caminho (`RequestPathTo`/`CurrentPath`), ou seja, movimento horizontal. Bloqueada pela DEC-22.
- **Portal** (`CommandPortal.cs`): sem `Tick()` (é single-shot) — localiza um portal (dicionário estático ou busca por força bruta) e dispara `RequestPathTo` uma vez. Mesma dependência de execução de caminho de Follow/AutoMiner — bloqueada pela DEC-22.
- Outras automações descobertas além das 7 pedidas (inventariadas, não analisadas em profundidade): `CommandUseBow`, `CommandInvCaptcha` (bypass de captcha), `CommandBreakBlock`, `CommandDropAll`, `CommandRetard`, `CommandTwerk`, `CommandUseEntity`. `CommandMobPlus`/`CommandPescaV2` existem no legado mas **nunca são registrados** em `CommandManagerNew` — código morto confirmado, mesmo padrão já visto em milestones anteriores.

**Inventário da arquitetura Java atual (o que já cobre o quê):**

- `MotorDeTick`/`TarefaContinua` (DEC-29) e `TarefaAntiAFK`/`ComandoAntiAFK` (DEC-32) já existem — mas nenhum bot criado pelos Casos de Uso reais chegava a ser registrado no motor (ver achado central abaixo).
- `BuscadorDeCaminho`/`Mundo.criarCaminhoPara` (DEC-27) portam só o **algoritmo** de busca de caminho — a **execução** tick-a-tick (equivalente a `PathGuide` do legado) nunca foi portada, e só seria necessária para AutoMiner/Follow/Portal, todos já bloqueados pela DEC-22 de qualquer forma.
- `Mundo.tracarRaio` (DEC-24) e `CalculadoraDeQuebraDeBloco`/`RegistroDeBlocos` (DEC-28) existem prontos, mas sem nenhum consumidor de produção ainda — Herbalism seria o primeiro consumidor realista de raycast fora de combate/mineração completa.
- `interfaces.comando`/`Comando` (DEC-23) e `application.usecase` (16 Casos de Uso) seguem os padrões já estabelecidos; nenhuma peça nova de framework é necessária para uma macro simples do tipo Herbalism/AntiAFK — só compor `TarefaContinua` + `Comando` de toggle, mesmo par já provado por `TarefaAntiAFK`/`ComandoAntiAFK`.
- **Nenhuma capacidade de interação com janelas/baús (containers)** existe hoje — necessária para AutoFish/Solk, não necessária para Herbalism/AntiAFK.

**Achado central (a lacuna real, anterior a qualquer macro nova):** `MotorDeTick.registrar(Bot)` e `Bot.iniciar()` não tinham **nenhum call site de produção** — grep de todo `src/main/java` só retorna a própria declaração e os testes. `CasoDeUsoCriarBot`/`CasoDeUsoConectarBot` nunca chamavam nenhum dos dois. O próprio `BootstrapDeProducaoTest` (Incremento 22.7) evidencia isso: para provar que o agendamento de produção dispara `MotorDeTick.tick()`, o teste constrói um `Bot` manualmente e chama `registrar()`/`iniciar()` direto, contornando os Casos de Uso reais — critério de aceite do 22.7 (Seção 5, Milestone 22) foi satisfeito à risca, mas não exigia fechar esta camada. Na prática, mesmo `TarefaAntiAFK` (única macro real existente) nunca executava fora de teste — não por falta de CLI/API (DEC-02, problema diferente, de transporte para o operador), mas porque o próprio caminho `criar→conectar` não fechava o ciclo. Ver **DEC-33** para a decisão e a correção.

**Conclusão da análise de suficiência arquitetural:** a arquitetura atual (`MotorDeTick`/`TarefaContinua`/Scheduler, `SessaoDeJogo`, Casos de Uso, `Mundo`, algoritmo de PathFinder, Raycast, `CalculadoraDeQuebraDeBloco`, `Comando`) é **suficiente, sem lacuna nova de framework**, para macros estacionárias/sem movimento horizontal e sem interação de container — Herbalism é o caso concreto. Duas lacunas reais e específicas de **conteúdo** (não de framework) bloqueiam o restante: (1) execução de movimento horizontal — bloqueada pela DEC-22, exige decisão explícita de reabertura com escopo próprio antes de AutoMiner/Follow/Portal; (2) interação com containers/janelas (baús) — lacuna nova, não coberta por nenhuma DEC existente, bloqueia AutoFish e o `CommandMob` do Solk.

**Incremento 24.1 — Wiring de Produção: Bot Criado/Conectado Passa a Ser Ticado (DEC-33).** Implementado nesta sessão, sem aguardar novo prompt, por atender às 3 condições da instrução recebida: nenhum gatilho de parada da governança acionado, nenhuma DEC existente precisou ser redefinida, nenhum bounded context novo necessário. Ver DEC-33 para a análise completa (alternativas, justificativa, consequências).

Entregue

- `application.port.MotorDeExecucaoPort` (novo — `registrar(Bot)`/`remover(Bot)`, mesma categoria de `ConexaoBotPort`/`AgendadorDeTarefasPort`).
- `infrastructure.execucao.MotorDeTick` passa a `implements MotorDeExecucaoPort` (métodos já existentes, só ganham contrato explícito).
- `application.usecase.CasoDeUsoCriarBot` ganha construtor com `MotorDeExecucaoPort`, registra o bot criado.
- `application.usecase.CasoDeUsoConectarBot.connect()` chama `bot.iniciar()` no caminho de sucesso (efeito colateral correto: reconexão automática via `GerenciadorDeReconexao`/DEC-31 também volta a marcar `EXECUTANDO`).
- `infrastructure.config.ConfiguracaoDeConexao` atualizado para injetar o novo Port em `casoDeUsoCriarBot`.
- Nenhuma interface já aprovada teve assinatura alterada, exceto o construtor de `CasoDeUsoCriarBot` — sem call site de produção fixo antes desta própria DEC.

Testes

`CasoDeUsoCriarBotTest` (+1: registra o bot criado no `MotorDeExecucaoPort`). `CasoDeUsoConectarBotTest` (+2: marca `EXECUTANDO` no sucesso, permanece `PARADO` na exceção). `FluxoCriacaoConexaoETickDoBotTest` (novo, 1 teste) — prova end-to-end que um bot criado e conectado pelos Casos de Uso reais é de fato ticado pelo `MotorDeTick` e executa sua `TarefaContinua`, sem nenhuma chamada manual de teste, pela primeira vez. `CasoDeUsoCriarBotTest`/`FluxoConexaoBotV1_8Test` existentes atualizados só para o novo construtor, sem mudança de asserção pré-existente.

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **798 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 791→798 (+7: 4 desta milestone; os +3 restantes já estavam presentes no working tree no início da sessão, parte do Incremento 22.7/Milestone 23 ainda não commitado no git — fora do escopo desta milestone). Mudança 100% aditiva.

---

## Milestone 25

Status

CONCLUÍDA — Incremento 25.1 (Herbalismo) e correção de fidelidade DEC-34 (altura de olho).

Objetivo

Construir a primeira macro concreta da Fase 2 sobre a arquitetura já validada suficiente pela Milestone 24: Herbalismo (fazenda estacionária de cana-de-açúcar), escolhida pelo responsável do projeto entre os candidatos por não depender de movimento horizontal (DEC-22) nem de nenhuma lacuna nova de framework. Análise levantada EXCLUSIVAMENTE contra `Projeto Adv 2.4.5`.

### Análise do legado

`CommandHerbalism` (`AdvancedBot.Client.Commands/CommandHerbalism.cs`, 69 linhas) — `ICommand` simples (sem máquina de estado), toggle único. `Tick()`: olha para o próprio bloco dos pés (`Player.LookToBlock(x, Floor(AABB.MinY), z)`), traça raio de 6 blocos (`RayCastBlocks(6.0)`); sem acerto, imprime erro e desliga (`IsToggled=false`); acerto 1 bloco abaixo dos pés → troca hotbar pra cana (338, via `HotbarSlot` setter, só se não já estiver com ela) e replanta (`PlaceCurrentBlock`); acerto em cana (bloco 83) → quebra instantânea (`Client.BreakBlock` — Swing+Start+Finish sem tempo real, distinto do `DiggingHelper`/quebra temporizada) e, se o bloco abaixo do que quebrou também for cana, quebra os dois no mesmo tick (client-side predict-clear via `SetBlockAndData` antes do 2º raycast).

Achado de fidelidade não superficial, encontrado ao validar campo a campo: `Entity.PosY` do legado (usado como origem por `LookTo`/`RayCastBlocks`/`CanSeePlayer`) é **altura do olho** (`Entity.cs:326`: `PosY = AABB.MinY + 1.62`), não dos pés. `SessaoDeJogo.olharParaBloco`/`tracarRaioParaBlocos` (DEC-24/DEC-25) e `podeVerJogador` (Milestone 19) usavam `x/y/z` (pés, exigência do protocolo) como origem direta, sem essa soma — divergência invisível para os 3 comandos já em produção (miram blocos distantes, onde o erro de ângulo não muda o resultado), mas categórica para Herbalismo (mira o próprio bloco dos pés: sem a correção, o raio sai para cima, não para baixo — confirmado empiricamente). Apresentado ao responsável do projeto com as alternativas antes de qualquer implementação; decisão: **DEC-34**, corrigir globalmente os 3 métodos.

Entregue

- **DEC-34**: `SessaoDeJogo.olharParaBloco`/`tracarRaioParaBlocos`/`podeVerJogador` passam a somar `ALTURA_DO_OLHO` (1.62, mesma constante já usada para entidades remotas) à origem do próprio bot. Assinaturas inalteradas.
- `domain.protocol.v1_8.SelecionarSlotDaHotbarPacket`/`SelecionarSlotDaHotbarCodec` (novos) — porta `PacketHeldItemChange` do legado no sentido **SERVERBOUND** (PLAY 0x09, campo `short` de 2 bytes) — coexiste com o `HeldItemChangePacket` já registrado CLIENTBOUND no mesmo id (mesma disciplina da DEC-16: direção faz parte da chave do registro, não a classe).
- `SessaoDeJogo.selecionarSlotDaHotbar(int slot)` (novo) — porta o setter `MinecraftClient.HotbarSlot` (valida `[0,8]`, envia o pacote, atualiza `InventarioDoJogador.slotAtivo`). O guard `IsBeingTicked()` do legado não tem equivalente — `SessaoDeJogo` só existe já conectado, por construção.
- `domain.bot.TarefaHerbalismo` (novo, `implements TarefaContinua`) e `interfaces.comando.ComandoHerbalismo` (novo) — mesmo par toggle de `TarefaAntiAFK`/`ComandoAntiAFK` (DEC-32): sem CasoDeUso dedicado para registrar/remover a tarefa. Replantar reproduz o branch "double-send" de `MinecraftClient.PlaceCurrentBlock` para o item 338 (`colocarBloco` + `usarItemNaMao` sequenciais) — branch que a Milestone 14/DEC-25 tinha comprovado mas deliberadamente não portado por ser inalcançável a partir de `ComandoColocarBlocoAutoMira` (exige item < 256; cana é 338); aqui, com cana, ele É alcançado.
- Nenhum bounded context novo; nenhuma interface pública pré-existente com assinatura alterada (exceto os 3 métodos corrigidos pela DEC-34, mesma assinatura, fórmula interna diferente).

Testes

`SelecionarSlotDaHotbarCodecTest` (novo, 3: round-trip). `RegistroDePacotesV1_8Test` (+2: localiza codec/id do novo pacote). `SessaoDeJogoTest` (+2: seleciona slot enviando pacote e atualizando estado; rejeita slot fora de `[0,8]` sem enviar nada; +2 valores recalculados por causa da DEC-34: `deveOlharParaBlocoCalculandoYawEPitchEEnviandoPlayerLookPacket` e `tracarRaioParaBlocosDeveAtingirBlocoAbaixoAoOlharRetoParaBaixo`). `CasoDeUsoOlharParaBlocoTest` (1 valor recalculado). `ComandoQuebrarBlocoTest` (1 teste com bloco/coordenadas ajustados — mesmo comportamento funcional, geometria corrigida). `TarefaHerbalismoTest` (novo, 8: replanta sem trocar hotbar já com cana na mão; troca hotbar antes de replantar; não replanta sem cana em nenhum slot; quebra um segmento no próprio nível dos pés; não quebra segmento adicional quando o de baixo não é cana; quebra dois segmentos no mesmo tick com predict-clear; imprime mensagem e remove a si mesma sem acerto de raio; no-op sem sessão de jogo). `ComandoHerbalismoTest` (novo, 2: liga/desliga).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **814 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 798→814 (+16), mudança funcionalmente aditiva (a correção DEC-34 muda ângulos/coordenadas exatas de 3 comandos já aprovados, não o resultado observável — todos continuam acertando o bloco pedido).

---

## Milestone 26

Status

CONCLUÍDA — Incremento 26.1 (DEC-35, motor de física horizontal + `GuiaDeCaminho`).

Objetivo

Determinar o menor subconjunto do motor de movimento horizontal necessário para reconstruir `CommandGoto`/`AutoMiner`/`CommandFollow`/`CommandPortal`, sem reabrir a física completa do Minecraft, e implementar o primeiro incremento resultante caso nenhum gatilho de parada se aplique. Análise levantada EXCLUSIVAMENTE contra `Projeto Adv 2.4.5`.

### Análise do legado

`Entity.cs` — `Tick()` (:96-200, consome `Player.MoveQueue`/`Movement` via `MoveRelative`, delega a `Move()`), `Move(xa,ya,za)` (:238-328, colisão 3-eixos com assistência de "step" `stepHeight=0.5` e atualização de `IsCollidedHorizontally`/`OnGround`), `MoveRelative` (converte input relativo ao yaw em `MotionX/MotionZ`), `GetMoveSpeed()` (base 0.1 + sprint + potions do próprio bot). `AABB.cs` — `ClipXCollide`/`ClipYCollide`/`ClipZCollide` estruturalmente idênticos entre si. `PathFinding/PathGuide.cs` — `Create` (delega a `World.CreatePathTo`, já portado pela DEC-27), `Tick()` (pulo automático quando `IsCollidedHorizontally && OnGround`, mira o próximo `PathPoint` por ângulo absoluto via `move(x,z)`, **não** usa `MoveRelative`/yaw), `getClosestIndex` (recuperação quando a distância ao ponto atual excede 2.5). `Commands/CommandGoto.cs` (single-shot), `CommandFollow.cs` (Tick, repath a cada 2.5 blocos), `CommandPortal.cs` (single-shot), `AutoMiner.cs`+`CommandMiner.cs` (Tick — combina `RequestPathTo`/`CurrentPath` **e** `Player.MoveQueue.Enqueue(Movement.Forward)` direto, um segundo mecanismo de movimento distinto de `PathGuide`).

Achado central: `CommandGoto`/`CommandFollow`/`CommandPortal` dependem exclusivamente de `PathGuide` (execução de caminho, nunca portada) — nenhum dos três usa `MoveQueue`/`Movement`/`MoveRelative` por yaw. `AutoMiner` é o único consumidor que usa os dois mecanismos (`PathGuide` **e** `MoveQueue` para uma caminhada exploratória própria). A DEC-22 bloqueia "motor de física"/movimento automático; a DEC-32 (Milestone 23) já abriu uma exceção pontual só para o eixo vertical (`TarefaAntiAFK`). Diferenciação feita (ver DEC-35 para o texto completo): mecanismo de movimento (`Entity.Move`/`Clip*Collide`) vs. física completa do Minecraft (água/lava/escada/sprint/`MoveQueue`, permanece bloqueada) vs. execução de caminho (`PathGuide`, consumidor do mecanismo) vs. consumidores/macros (`CommandGoto`/`Follow`/`Portal`/`AutoMiner`, nenhum implementado nesta sessão).

Decisão: **DEC-35** — reabertura parcial da DEC-22, generalizando o motor vertical da DEC-32 para também resolver colisão horizontal (X/Z), sem a assistência de "step" do legado (qualquer resistência horizontal é tratada como colisão cheia, resolvida pelo mesmo pulo automático que `GuiaDeCaminho` porta de `PathGuide.Tick()`). `MoveQueue`/`Movement`/`MoveRelative` por yaw (mecanismo exclusivo de `AutoMiner`) permanecem fora de escopo. Nenhum gatilho de parada acionado: mesmo precedente direto da DEC-32, nenhum Port/bounded context/agregado novo — primeiro incremento implementado na mesma sessão, conforme instrução.

Entregue

- `domain.bot.SessaoDeJogo.aplicarFisicaVertical()` renomeado para `aplicarFisica()` e generalizado — resolve colisão nos 3 eixos (Y, X, Z, mesma ordem do legado `Entity.Move()`), reutilizando `Bloco.alturaSuperficie()` (DEC-32) também para X/Z. Comportamento idêntico para o único consumidor pré-existente (`TarefaAntiAFK`, nunca seta motion horizontal) — provado pelos testes existentes mantidos verdes sem alteração de asserção.
- `SessaoDeJogo.colididoHorizontalmente()`, `velocidadeHorizontal()`, `somarMotionHorizontal(double,double)` (novos, pacote) — a fórmula de velocidade onGround→efetiva (duplicada 3× no legado) unificada num único método, reaproveitado por `aplicarFisica()` (fricção) e por `GuiaDeCaminho` (mira do próximo ponto).
- `domain.bot.GuiaDeCaminho` (novo, público) — porte de `PathGuide`: `criar(SessaoDeJogo,x,y,z)` (delega a `SessaoDeJogo.criarCaminhoPara`, já existente desde a DEC-27, retorna `null` sob as mesmas condições), `finalizado()`, `tick(SessaoDeJogo)` (pulo automático ao colidir horizontalmente no chão, mira o próximo `PontoDeCaminho` por ângulo absoluto, avança índice a ≤0.8 de distância ou realinha ao ponto mais próximo a ≥2.5). Bônus de altura de pulo por poção de Speed do legado (`PathGuide.cs:58-61`) não portado — efeitos do próprio bot não são modelados (mesma lacuna aceita pela DEC-28).
- Nenhum `Comando`/`CasoDeUso`/`TarefaContinua`/Port/bounded context/agregado novo — `GuiaDeCaminho` fica sem consumidor de produção nesta sessão, mesmo precedente de `BuscadorDeCaminho` após a DEC-27 (capacidade pura entregue, consumidor real fica para os incrementos seguintes: `CommandGoto`/`Follow`/`Portal`, e depois `AutoMiner` — este exigindo ainda `MoveQueue`/`Movement`/`MoveRelative` por yaw, DEC própria futura).

Testes

`SessaoDeJogoTest` (+2: colisão horizontal contra parede sólida clampando o deslocamento e sinalizando `colididoHorizontalmente()`; atrito aplicado ao motion horizontal sem parede, valor exato conferido). `GuiaDeCaminhoTest` (novo, 3: `criar` retorna `null` sem nenhum caminho possível; percorre corredor reto no chão até finalizar dentro do limiar de 0.8 do destino; sobe um degrau de 1 bloco via pulo automático ao colidir horizontalmente, sem nenhuma assistência de "step"). Testes pré-existentes de física vertical (`TarefaAntiAFKTest`, e os de `SessaoDeJogoTest` sobre pouso em slab/snow/soul sand/carpet/colisão de teto) mantidos com as mesmas asserções, só renomeados para `aplicarFisica()`.

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **819 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 814→819 (+5), mudança 100% aditiva exceto o rename interno `aplicarFisicaVertical`→`aplicarFisica` (3 call sites atualizados, comportamento idêntico para o único consumidor pré-existente, provado pelos testes).

---

## Milestone 27

Status

CONCLUÍDA — Incremento 27.1 (`ComandoGoto`, primeiro consumidor real de `GuiaDeCaminho`/DEC-35).

Objetivo

Implementar `CommandGoto` do legado — o menor consumidor real de execução de caminho identificado na análise da Milestone 26 — provando o ciclo completo `Comando → GuiaDeCaminho → SessaoDeJogo` sem exigir `TarefaContinua`.

### Análise do legado

`CommandGoto.cs` (single-shot, delega a `MinecraftClient.RequestPathTo(x,y,z,errMsg)`). `RequestPathTo` (`MinecraftClient.cs:646-666`) despacha `FindPath` numa `Task.Factory.StartNew` separada, que chama `PathGuide.Create(Player,x,y,z)` e atribui o resultado a `CurrentPath` (campo público mutável, único caminho ativo por client), imprimindo `errMsg` só se `Create` retornar `null`. `MinecraftClient.Tick()` (`:778-785`) tica `CurrentPath` incondicionalmente a cada ciclo de rede (antes de `CmdManager.Tick()`), limpando-o quando `Finished()`.

Achado central: não existe, em todo o domínio Java, nenhum ponto que tique um `GuiaDeCaminho` de produção — a Milestone 26 entregou a capacidade pura, sem consumidor. `ComandoGoto` precisa de um lugar para o caminho viver entre chamadas e de um mecanismo que o tique a cada ciclo, independente de qualquer `TarefaContinua` (já que um comando single-shot só executa uma vez).

Decisão: sem DEC nova — mesmo precedente de `ComandoMover`/DEC-23 (comando single-shot delegando a uma capacidade de domínio já aprovada). O caminho ativo (`SessaoDeJogo.caminhoAtual()`, novo) é ticado por `MotorDeTick.tick()` de forma incondicional, no mesmo ponto onde o legado tica `CurrentPath` no laço principal — mecanismo de infraestrutura compartilhado, não uma `TarefaContinua`, fiel à natureza do campo `CurrentPath` do legado (nunca foi modelado como um "comando"/toggle no C#). Nenhum Port/agregado/bounded context novo.

Entregue

- `domain.bot.SessaoDeJogo.caminhoAtual()`/`definirCaminho(GuiaDeCaminho)`/`limparCaminho()` (novos) — porta o campo `MinecraftClient.CurrentPath` do legado para dentro da sessão de jogo do bot (um caminho ativo por bot, não por client).
- `infrastructure.execucao.MotorDeTick.tick()` — passa a tickar `sessaoDeJogo.caminhoAtual()` a cada ciclo (bots `EXECUTANDO`), limpando-o quando `finalizado()`; inserido antes do laço de `TarefaContinua`, mesma posição relativa de `CurrentPath.Tick()`/`CmdManager.Tick()` no legado.
- `interfaces.comando.ComandoGoto` (novo, porte de `CommandGoto`) — calcula `GuiaDeCaminho.criar(sessaoDeJogo,x,y,z)` de forma síncrona e o define via `definirCaminho`; em caso de falha, imprime a mensagem de erro do legado preservando fielmente o typo ("possivel" sem acento) e a cor incorreta (`§a`, verde, não `§c`) do `CommandGoto.cs` original.
- Divergência de execução registrada (não de resultado observável): o legado despacha o cálculo do caminho numa `Task.Factory.StartNew` separada; `BuscadorDeCaminho`/`GuiaDeCaminho.criar` nunca tiveram um wrapper assíncrono no domínio Java (nenhum precedente desde a DEC-27), então `ComandoGoto` calcula de forma síncrona. Sem efeito no resultado observável (mesmo caminho é calculado); risco de latência perceptível do comando fica registrado em Pendências Conhecidas.

Testes

`SessaoDeJogoTest` (+1: `caminhoAtual`/`definirCaminho`/`limparCaminho`). `MotorDeTickTest` (+3: tica o caminho ativo antes das tarefas contínuas movendo o bot; limpa o caminho quando finalizado, mesmo após vários ciclos; não tica caminho de bot que não está `EXECUTANDO`). `ComandoGotoTest` (novo, 4: define o caminho quando encontrado; imprime a mensagem fiel ao legado quando nenhum caminho é encontrado; argumentos faltando; exceção sem sessão de jogo ativa).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **827 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 819→827 (+8), mudança 100% aditiva.

---

## Milestone 28

Status

CONCLUÍDA — Incremento 28.1 (`TarefaFollow`/`ComandoFollow`, terceira Macro real).

Objetivo

Implementar `CommandFollow` do legado — segue um jogador remoto olhando para ele e recalculando `GuiaDeCaminho` sempre que o alvo se desloca o suficiente — reaproveitando o mesmo par toggle `Comando`/`TarefaContinua` já estabelecido por `TarefaAntiAFK`/`TarefaHerbalismo` (DEC-32/DEC-34).

### Análise do legado

`CommandFollow.cs` — `Run` resolve o alvo via `PlayerManager.GetPlayerByNick` (case-insensitive contra `RealNick`) e reseta `lastFollowPos` para o sentinela `(0,-555,0)`; sem argumentos, alterna OFF (`CurrentPath = null; following = null`). `Tick()` — a cada ciclo: se o alvo não existe mais (`PlayerManager.PlayerExists`) ou está a mais de 80 blocos (`Utils.DistTo` usando `AABB.MinY`, isto é, pés), imprime aviso e para; senão olha para o alvo (`Entity.LookTo(X, Y+1.62, Z)`, olho do alvo) e, se o alvo andou `>= 2.5` blocos desde o último recálculo, chama `RequestPathTo` (sem `errMsg`) e atualiza `lastFollowPos`. Diferente de `CommandGoto`/`CommandPortal`, `CommandFollow` mantém uma única instância sempre viva no `CmdManager`, usando o campo nullable `following` como toggle interno (não é registrado/removido dinamicamente).

Decisão: sem DEC nova — mesmo padrão de toggle via registro/remoção de `TarefaContinua` já usado por `ComandoAntiAFK`/`ComandoHerbalismo` (divergência de implementação já aceita desde a DEC-32/DEC-34: o legado usa um campo nullable interno persistente, o Java usa registrar/remover uma instância nova a cada troca de alvo — mesmo resultado observável). Resolução de nick reaproveita `ListaDeJogadores`/`EntidadesDoMundo`, ambos já existentes desde a Milestone 5/8, com dois métodos novos e aditivos (`ListaDeJogadores.uuidPorNome`, `EntidadesDoMundo.porUuid`). Nenhum Port/agregado/bounded context novo.

Entregue

- `domain.bot.SessaoDeJogo.olharParaEntidade(EntidadeRemota)` (novo) — porte de `Entity.LookTo`, mira a posição contínua de OLHO de uma entidade remota (sem o offset de centro de bloco de `olharParaBloco`/`LookToBlock`).
- `domain.bot.ListaDeJogadores.uuidPorNome(String)` (novo, porte de `PlayerManager.GetPlayerByNick`, comparação case-insensitive contra `nome`) e `domain.bot.EntidadesDoMundo.porUuid(UUID)` (novo) — resolvem nick → `EntidadeJogadorRemoto` em dois passos, mesma composição do legado (`UUID2Nick` + `Players`).
- `domain.bot.TarefaFollow` (novo, `TarefaContinua`) — porte de `CommandFollow.Tick()`: olha para o alvo todo tick; recalcula `GuiaDeCaminho` (síncrono, mesma divergência de execução da Milestone 27) quando o alvo anda `>= 2.5` blocos desde o último recálculo (sentinela `Y=-555.0` reproduzido fielmente, garantindo o primeiro recálculo imediato); para de seguir, limpa o caminho e remove a si mesma quando o alvo some ou fica a mais de 80 blocos.
- `interfaces.comando.ComandoFollow` (novo, porte de `CommandFollow.Run`) — sem argumentos, alterna OFF (remove a tarefa ativa e limpa o caminho); com um nick, resolve o jogador e troca (ou inicia) o alvo seguido, removendo a tarefa anterior se houver.
- Omissão registrada: o toggle do comando `"retard"` do legado (`Client.CmdManager.GetCommand("retard").IsToggled = false`, presente em `CommandFollow.Run`/`CommandPortal.Run`) não tem equivalente — `CommandRetard` ("Faz o bot se mover aleatoriamente") nunca foi portado, nenhuma milestone o cobre.

Testes

`SessaoDeJogoTest` (+1: `olharParaEntidade`). `ListaDeJogadoresTest` (+2: `uuidPorNome` case-insensitive; retorna `null` para nome desconhecido). `EntidadesDoMundoTest` (+2: encontra jogador remoto por uuid; retorna `null` sem jogadores/uuid desconhecido). `TarefaFollowTest` (novo, 6: olha e força o primeiro recálculo de caminho; não recalcula sem deslocamento suficiente; recalcula ao deslocar `>= 2.5`; para de seguir e limpa o caminho quando o alvo sai do alcance; para de seguir quando o alvo deixa de existir; no-op sem sessão de jogo). `ComandoFollowTest` (novo, 6: argumentos faltando sem alvo ativo; começa a seguir; mensagem de erro para jogador desconhecido; troca de alvo removendo a tarefa anterior; para de seguir limpando o caminho; exceção sem sessão de jogo ativa com argumentos).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **844 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 827→844 (+17), mudança 100% aditiva.

---

## Milestone 29

Status

CONCLUÍDA — Incremento 29.1 (`ComandoPortal`, encerramento do Épico Fase 2 "execução de caminho").

Objetivo

Implementar `CommandPortal` do legado — localiza o portal de um minijogo do servidor CraftLandia (por nome conhecido ou por varredura de blocos) e usa `GuiaDeCaminho` para chegar até ele, mesmo mecanismo de `ComandoGoto` (Milestone 27).

### Análise do legado

`CommandPortal.cs` — `FindPortal(player,nome)`: se `nome=="**"`, delega a `BruteForcePortalFinder` (varre blocos id `90` num raio de 8 chunks ao redor do jogador, laço `chunkX → chunkZ → x local → z local → y`, y é o mais interno, chunk não carregado é pulado); senão busca no dicionário estático `portals` (populado uma única vez a partir do resource embutido `AdvancedBot.Properties.Resources.portals` — JSON com 25 minijogos do servidor CraftLandia, decodificado desta sessão) e retorna a posição conhecida mais próxima (`Utils.DistToSq` contra `PosX/PosY/PosZ`, ou seja, origem de OLHO do jogador); a chave `"*"` do dicionário é a união de todas as posições conhecidas. `Run` chama `RequestPathTo` com `errMsg = "Não foi possível encontrar o caminho até o portal."` (cor `§c`, correta) quando o portal é encontrado, ou imprime `"Não foi possível encontrar o portal."` quando `FindPortal` retorna `null`.

Decisão: sem DEC nova — mesmo mecanismo de execução de caminho de `ComandoGoto`/`ComandoFollow`, sem mecanismo adicional. O dado estático de portais (`Resources.portals`) foi decodificado (base64 → JSON, 25 entradas) e hardcoded diretamente em `domain.bot.RegistroDePortais` — sem introduzir parser JSON/dependência nova no projeto (Jackson não está no classpath, `spring-boot-starter` puro), mesmo racional já aceito para `domain.protocol.v1_8.RegistroDeBlocos` (DEC-28, dado estático fixo do legado hardcoded em vez de lido de um arquivo em runtime). Divergência de implementação, não de dado: os 25 grupos/posições são reproduzidos integralmente.

Entregue

- `domain.bot.PosicaoDePortal` (novo, record — porte de `Vec3i`) e `domain.bot.RegistroDePortais` (novo, package-private) — dado estático dos 25 minijogos do servidor CraftLandia, decodificado do resource `AdvancedBot.Properties.Resources.portals` do legado; chave `"*"` reconstruída como a união de todas as posições.
- `domain.bot.SessaoDeJogo.localizarPortal(String)` (novo) — porte de `CommandPortal.FindPortal`/`BruteForcePortalFinder`: `"**"` varre os chunks carregados (mesma ordem de laços do legado, chunk não carregado pulado sem consultar bloco); qualquer outro nome busca em `RegistroDePortais` e retorna a posição mais próxima da origem do bot (origem de OLHO, `y + ALTURA_DO_OLHO`, fiel a `PosX/PosY/PosZ` do legado).
- `interfaces.comando.ComandoPortal` (novo, porte de `CommandPortal.Run`) — localiza o portal e usa `GuiaDeCaminho.criar`/`SessaoDeJogo.definirCaminho`, mesmo padrão de `ComandoGoto`; mensagens de erro fiéis ao legado (`"Não foi possível encontrar o portal."`/`"Não foi possível encontrar o caminho até o portal."`, ambas `§c`).
- Mesma omissão do toggle `"retard"` registrada na Milestone 28 (`CommandPortal.Run` também o desliga no legado; `CommandRetard` nunca foi portado).

Testes

`SessaoDeJogoTest` (+6: posição conhecida mais próxima por nome; escolhe a mais próxima entre várias posições do mesmo nome; `null` para nome desconhecido; `"*"` busca em todos os minijogos conhecidos; `"**"` varre chunks carregados procurando bloco de portal; `"**"` retorna `null` sem bloco de portal no raio). `ComandoPortalTest` (novo, 6: mensagem de erro para nome desconhecido; nome em qualquer capitalização; mensagem de erro de caminho quando portal conhecido mas sem caminho possível; define o caminho quando o portal é encontrado e há caminho até ele; argumentos faltando; exceção sem sessão de jogo ativa).

Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **856 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 844→856 (+12), mudança 100% aditiva. Épico Fase 2 "execução de caminho" (Milestones 26-29) encerrado: `GuiaDeCaminho` (capacidade pura, M26) agora tem três consumidores de produção (`ComandoGoto`/`TarefaFollow`/`ComandoPortal`); `AutoMiner` continua bloqueado por um mecanismo adicional (`MoveQueue`/`Movement`/`MoveRelative` por yaw, DEC própria futura).

---

## Milestone 30

Status

CONCLUÍDA — Incremento 30.1 (`TarefaMineracao`/`ComandoMinerar`, Épico "Mineração", encerramento em milestone única).

Objetivo

Reconstruir a infraestrutura restante necessária para a MacroMineração (`AutoMiner`/`CommandMiner` do legado) — o único bloqueio explicitamente registrado desde a DEC-35 (Milestone 26): o item 4 de "Próximos incrementos" daquela DEC, que exigia uma DEC própria antes de qualquer implementação.

### Análise do legado

`AdvancedBot.Client/AutoMiner.cs` — `Tick()`: enquanto uma quebra está em andamento (`status==Breaking`), mira o bloco, balança o braço e acumula `DiggingHelper.StrengthVsBlock` a cada tick até `digSum>=1f` (envia `FinishedDigging`) ou 15s sem terminar (`EndDestroyBlock`, sem enviar `CancelledDigging` — comportamento de fábrica preservado, não corrigido); caso contrário, varre um cubo de raio 8 (config `MinerRadius`) ao redor dos pés buscando o bloco de menor custo (`distância + prioridade*2.3`, penalidade `x16` para blocos 1 nível abaixo dos pés) entre os blocos configurados em `MinerBlocks` mais, quando `MinerForceListBlocks=false` (default), dois sentinelas hardcoded decodificados nesta sessão (`-131071`→pedra/prioridade 65534, `-65533`→terra/prioridade 65535 — "cavar através" como fallback de preenchimento); ao encontrar candidato, mira/pede caminho via `RequestPathTo` (`PathGuide`, já coberto por `GuiaDeCaminho`/DEC-27/DEC-35) **e** enfileira 6x `Movement.Forward` em `Player.MoveQueue` como "caminhada exploratória" enquanto o path assíncrono (`Task.Factory.StartNew`) é calculado; então tenta quebrar imediatamente qualquer bloco já dentro do alcance do raio de 6 (`IsBlockListedAndForced` — sempre `true` quando `MinerForceListBlocks=false`, sem filtrar por lista nem excluir vinhas nesse ramo, ao contrário do laço de varredura). `CommandMiner.Run` ignora os `args` recebidos — toda configuração vem de `Program.Config` (diálogo "Opções->Minerador...", sem equivalente de infraestrutura no domínio Java).

Decisão: **DEC-36** — não portar `MoveQueue`/`Movement`/`MoveRelative` por yaw. O único call site do legado existe para compensar a natureza assíncrona de `Task.Factory.StartNew`/`RequestPathTo`; `GuiaDeCaminho.criar` já é síncrono no domínio Java (mesma divergência de execução já aceita por `ComandoGoto`/`TarefaFollow`/`ComandoPortal`), então a latência que o nudge compensava não existe aqui — sem consumidor real, mesma "regra de três" já usada pela DEC-24/25/27/28/35. A DEC-22 permanece vigente para esse mecanismo específico. `Program.Config` também não tem equivalente — `TarefaMineracao` reproduz fielmente o comportamento de fábrica (nunca configurado pelo operador): `MinerBlocks` vazio (só os dois sentinelas fixos), `MinerRadius=8`, `MinerForceListBlocks=false`, `MinerSelectBestTool` sempre ligado (sem toggle exposto, mesma categoria de omissão do toggle `"retard"`); `MinerStopInvFull`/`MinerCmdsInvFull` (parada e fila de mensagens temporizadas ao encher o inventário) deliberadamente não portados — 100% dependentes de config sem contrapartida no Java, e o comportamento de fábrica já é "nunca para por inventário cheio".

Entregue

- `domain.bot.TarefaMineracao` (novo, `TarefaContinua`) — porte de `AutoMiner.Tick`/`SearchNearestBlock`/`BreakBlock`/`SelectBestTool`: busca no cubo de raio 8, custo fiel (distância + prioridade*2.3, penalidade x16 abaixo dos pés), `GuiaDeCaminho.criar`/`SessaoDeJogo.definirCaminho` para se deslocar (sem o nudge `MoveQueue`, DEC-36), quebra qualquer bloco no alcance do raio de 6 via `CalculadoraDeQuebraDeBloco.forcaDeQuebra` (DEC-28, sentinelas `nivelEficiencia=0`/`amplifierCeleridade=-1`/`amplifierFadiga=-1`) acumulado por tick até `1f`, timeout de 15s sem `CancelledDigging` (fiel). Reservas de blocos-alvo (`digBlocks` do legado) portadas como campo de instância por bot, não estático global entre bots (divergência de implementação documentada — isolamento por bot é estritamente mais correto e consistente com toda `TarefaContinua` já existente).
- `interfaces.comando.ComandoMinerar` (novo, porte de `CommandMiner.Run`) — toggle simples, mesmo padrão de `ComandoHerbalismo`/`ComandoAntiAFK`/`ComandoFollow`; args ignorados (fiel).
- **DEC-36** — formaliza a decisão de não reabrir a DEC-22 para `MoveQueue`/`Movement`/`MoveRelative` por yaw, fechando o item 4 pendente da DEC-35.
- Zero novo Port/bounded context/agregado/interface pré-existente alterada — `TarefaMineracao` consome só API pública já existente (`SessaoDeJogo`/`GuiaDeCaminho`/`CalculadoraDeQuebraDeBloco`).

Testes

`TarefaMineracaoTest` (novo, 6: nenhuma ação sem sessão de jogo; nenhum pacote enviado quando não há pedra/terra ao redor; inicia quebra do bloco de pedra mais próximo dentro do alcance do raio; prefere pedra a terra quando equidistantes — prioridade 65534 < 65535; não reinicia quebra em andamento; finaliza quebra após acumular força suficiente — `FinishedDigging`). `ComandoMinerarTest` (novo, 3: liga registrando a tarefa; desliga removendo na segunda chamada; ignora argumentos recebidos).

Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **865 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 856→865 (+9), mudança 100% aditiva. Épico "Mineração" encerrado nesta milestone única: `AutoMiner`/`CommandMiner` do legado portados (`TarefaMineracao`/`ComandoMinerar`), fechando o último item pendente da DEC-35 sem reabrir a DEC-22 além do já concedido pela DEC-32/DEC-35.

---

## Milestone 31

Status

BLOQUEADA — Fase de Planejamento concluída, nenhuma DEC nova criada, nenhum código implementado nesta sessão. Aguarda decisão do responsável do projeto sobre a arquitetura mínima de Container/Janela (achado desta sessão) antes de qualquer implementação.

Objetivo

Reconstruir a infraestrutura necessária para a macro de pesca do legado (`AutoFish`/`CommandPesca`, Solk) — candidato já identificado na Milestone 24 como dependente de "interação com baú/janela (container/window), capacidade que a Java atual não tem ainda". Esta sessão aprofunda a análise, exclusivamente contra `CommandPesca.cs` (alias `"solkpesca"`, único registrado em `CommandManagerNew` — `CommandPescaV2.cs` confirmado código morto, mesmo tratamento já dado a `CommandMobPlus` na Milestone 24) e `MacroUtils.cs` do `Projeto Adv 2.4.5`, para determinar o menor conjunto de mecanismos necessários e confirmar se algum incremento é implementável de forma independente.

### Análise do legado

- `CommandPesca` é uma máquina de estados assíncrona (padrão "Solk", `async void Tick()` com guarda de reentrância `isProcessing`) com 10 estados: `RECEM_LOGOU`→`INICIANDO`→`LIMPAR_INVENTARIO`→`VERIFICAR_INVENTARIO`→{`BUSCAR_VARA`|`COMPRAR_LINHA`|`REPARAR`|`GUARDAR_ITENS`|`PESCAR`}→`FINALIZANDO`. 100% baseada em teleporte por comando de chat (`/home ...`) — sem `RequestPathTo`/`PathGuide`/movimento horizontal físico, não esbarra na DEC-22.
- Núcleo da pesca (`pescando()`/`jogarVara()`): loop de até 1000 iterações enquanto `ItemInHand.Metadata < 45` (durabilidade da vara), cada iteração envia `PacketBlockPlace(ItemInHand)` sem alvo (sentinela -1/-1/-1) — **já portado integralmente por `SessaoDeJogo.usarItemNaMao`/`CasoDeUsoUsarItemNaMao` (Milestone 14, DEC-25)**. Sem entidade "fishing hook"/bobber própria no legado e sem detecção de captura — a única "detecção" é indireta, via incremento de durabilidade da vara a cada uso (dado já modelado, `ItemStack.damage`). Sem raycast/look novo além de `Pitch=-90` fixo.
- `VERIFICAR_INVENTARIO`/`LIMPAR_INVENTARIO`: leitura de `ItemInHand.ID`/`Metadata`, contagem de slots vagos, descarte de item (`inv.DropItem`, equivalente a `PacketPlayerDigging` status 4 — não portado ainda, lacuna aditiva pequena e não bloqueante, distinta do achado central), seleção de hotbar slot (já portado, `SessaoDeJogo.selecionarSlotDaHotbar`, Milestone 25).
- `COMPRAR_LINHA` (`MacroUtils.comprar`): **não usa container** — right-click numa placa via raycast, alternando sneak (`PacketEntityAction` 0/1) ao redor do clique, mesma composição de `SessaoDeJogo.agachar`/`pararDeAgachar` (Milestone 20) + `ComandoClicarBloco`/raycast (Milestone 13/14), já existentes em Java.
- `BUSCAR_VARA`/`GUARDAR_ITENS`/parte de `REPARAR` dependem de **abrir um baú** (`MacroUtils.abrirBau` — right-click seguido de espera por `Client.OpenWindow != null`), **ler seus slots** (`Client.OpenWindow.Slots`), **clicar em slots do baú e do inventário para transferir item** (`Inventory.Click`/`OpenWindow.Click`, protocolo `Click Window` + `Confirm Transaction`) e **fechar o baú** (`PacketCloseWindow`). Os 3 estados restantes do fluxo completo passam por essa capacidade — não há como reconstruir a macro completa sem ela.

### Achado central (motivo do bloqueio)

Nenhum Packet de container existe hoje em `domain.protocol.v1_8`/`infrastructure.protocol.v1_8.RegistroDePacotesV1_8` (confirmado por busca: `OpenWindow`/`CloseWindow`/`ClickWindow`/`ConfirmTransaction` — 0 ocorrências). `ReceptorSetSlot` (Milestone 5, Incremento 5) já trata `windowId != 0` como no-op deliberado ("baús/janelas não-jogador tratado como no-op por construção") — o próprio contrato reconhece a lacuna sem tentar cobri-la. `WindowItemsPacket`/`SetSlotPacket` decodificam o formato de fio genérico (qualquer `windowId`), mas nenhum agregado de domínio guarda o conteúdo de uma janela não-jogador — `InventarioDoJogador` é fixo em 45 slots, especificamente a janela do próprio jogador (`windowId=0`).

Faltam, no mínimo: `OpenWindowPacket` (`0x2D` CLIENTBOUND), `CloseWindowPacket` (`0x2E` CLIENTBOUND / `0x0D` SERVERBOUND), `ClickWindowPacket` (`0x0E` SERVERBOUND) e `ConfirmTransactionPacket` (`0x32` CLIENTBOUND / `0x0F` SERVERBOUND) — e, mais relevante, um **agregado novo** (`Janela`, categoria análoga a `InventarioDoJogador`, mas com contagem de slots variável por tipo de janela e ciclo de vida próprio de abrir/fechar) para representar o estado de uma janela não-jogador aberta. Isso é exatamente um dos gatilhos de parada explícitos recebidos nesta sessão ("novo agregado") — implementar sem decisão prévia violaria a instrução.

### Conclusão de suficiência arquitetural

Nenhum incremento do épico é implementável de forma independente e útil sem essa capacidade: o núcleo de pesca (estado `PESCAR`) já está 100% coberto pela infraestrutura existente (`usarItemNaMao`), mas uma macro que só executasse esse estado — sem nunca conseguir buscar a vara/comprar linha/guardar itens, todos dependentes de baú — divergiria do comportamento do legado de forma que constituiria regressão funcional, não porte fiel; não implementado por esse motivo, não por dificuldade técnica do núcleo em si. `COMPRAR_LINHA` isoladamente não depende de container e é tecnicamente implementável hoje, mas não tem valor como incremento isolado — é um sub-passo interno de uma máquina de estados, não um comando/macro utilizável por si.

Não implementado nesta sessão. Sem DEC nova, sem código, sem Packet/agregado/Comando/Caso de Uso novo.

Próximo passo sugerido

Decisão do responsável do projeto sobre uma DEC de arquitetura mínima de Container/Janela (candidata a DEC-37) antes de qualquer implementação de AutoFish/Solk — escopo sugerido: `Janela` (novo agregado, `domain.bot` — slots, `windowId`, tipo, mesma categoria de `InventarioDoJogador`), os 4 Packets listados acima (`domain.protocol.v1_8`), e decisão explícita sobre a semântica de `Confirm Transaction` (o legado assume sucesso otimista, sem aguardar confirmação real do servidor antes do próximo clique — mesmo padrão de mutação otimista já usado em `SessaoDeJogo.mover`, DEC-22).

---

## Pivô de Estratégia da Fase 2 (2026-07-23, mesma sessão do bloqueio da Milestone 31)

O responsável do projeto decidiu mudar a estratégia da Fase 2: em vez de continuar portando uma macro por vez e descobrir dependências faltando no meio da implementação (exatamente o que bloqueou a Milestone 31), a partir deste ponto a Fase 2 constrói primeiro uma **plataforma de infraestrutura reutilizável** ("Container Framework" e épicos análogos) — macros futuras passam a ser apenas orquestradoras de componentes já existentes, sem precisar de novo Packet/agregado/DEC estrutural. Produzido um mapeamento exaustivo do legado (`Projeto Adv 2.4.5`) agrupando toda primitiva reutilizável usada por `AutoMiner`/`AutoFish`/`CommandPesca`/Herbalismo/`Follow`/`Portal`/`AntiAFK`/`Solk`/`Repair`/`Mob`/`MobPlus` em capacidades arquiteturais (inventário, containers, movimentação de itens, seleção de slot, uso de item, clique em janela, confirmação de transação, chat, comandos, entidades, etc.), com prioridade/risco/dependências/necessidade de DEC por capacidade e uma sequência de épicos de infraestrutura (EPIC-I1 a EPIC-I13) desenhada para maximizar reaproveitamento. Épico raiz identificado: **EPIC-I1 — Fundação de Janela/Container**, exatamente o gatilho de bloqueio da Milestone 31 (candidata a DEC-37). Ver Milestone 32 abaixo para a implementação.

---

## Milestone 32 — EPIC-I1: Container Framework (DEC-37)

Status

CONCLUÍDA — épico implementado em sessão única, conforme instrução do responsável.

Objetivo

Reconstruir de forma genérica toda a infraestrutura de interação com janelas do protocolo 1.8 (baú, baú-duplo, ender chest, mesa de trabalho, fornalha, suporte de fermentação, farol, funil, dispenser, dropper, mesa de encantamento, trade de NPC, inventário de cavalo, bigorna — qualquer Window, não só baú), desbloqueando a Milestone 31 (AutoFish) e qualquer macro futura que dependa de container, sem nenhuma regra específica de macro nesta camada.

### Decisão (DEC-37)

Formaliza o agregado `Janela` (novo, `domain.bot`) e os 6 Packets/Codecs necessários (`OpenWindow` 0x2D CB, `CloseWindow` 0x2E CB, `FecharJanela` 0x0D SB, `ClickWindow` 0x0E SB, `ConfirmTransaction` 0x32 CB, `ConfirmarTransacao` 0x0F SB — pares CB/SB em classes distintas por direção, mesmo precedente de `HeldItemChangePacket`/`SelecionarSlotDaHotbarPacket` da DEC-16, já que o registro só permite um id de envio por `Class`). Decisões adicionais tomadas nesta sessão, sem stop-trigger por serem aditivas/bem definidas:

- **Indexação unificada de slot**: em vez de replicar o parâmetro `isChestOpen` do legado (fonte de um bug real documentado no mapeamento de primitivas — `MacroUtils.moverItemDoInvParaOBau` passava os argumentos errados por posição), todo método de clique em `SessaoDeJogo` usa a MESMA numeração unificada que o próprio protocolo já usa no campo `Slot` de `Click Window`: `[0, janela.numeroDeSlots())` é a janela, o restante é o inventário do bot (offset `+9` para pular crafting/armadura, que não aparecem na janela combinada). Elimina a classe inteira de bug do legado por construção.
- **Cursor de clique por sessão, não estático**: `SessaoDeJogo.itemNoCursor` é campo de instância — o legado (`Inventory.ClickedItem`) era `static`, compartilhado entre todos os bots do processo (bug identificado no mapeamento de primitivas, Bloco A1). Não replicado.
- **`JanelaDeSlots`** (nova interface, `domain.bot`) unifica `localizarItem`/`localizarEspacoLivre`/`contarEspacosLivres` via default method, implementada tanto por `Janela` quanto por `InventarioDoJogador` (extensão aditiva, zero mudança de comportamento) — substitui os 2 pares de funções quase-duplicadas do legado (`MacroUtils.localizarSlotDoItemNoBau`/`findItem`) por uma única implementação.
- **`moverItem`** consolida numa única primitiva as 4 variantes quase-idênticas e mutuamente inconsistentes de `MacroUtils.moverItemDoBauParaOInventario`/`moverItemDoInventarioParaOBau`/`moverItemDoBauParaInv`/`moverItemDoInvParaOBau` do legado — 2 cliques (`pegarItem` + `pegarItem`) sobre a indexação unificada, sem parâmetro `isChestOpen`.
- **`trocarSlots`** (mode 2, troca com hotbar) — sem precedente no legado (`Inventory.Click` só simula esquerdo/direito/shift), adicionado por completude do framework: mesmo pacote `ClickWindow`, semântica padrão e bem documentada do protocolo 1.8, decisão aditiva de infraestrutura (mesma categoria da DEC-29/DEC-35).
- **`esperarJanela`/`esperarFechamento`** do pedido original **não foram implementados como espera bloqueante** — um `Thread.sleep`/polling síncrono dentro de um método violaria a restrição já documentada do `MotorDeTick` (sem paralelismo por bot/tarefa, uma tarefa bloqueante trava o ciclo inteiro para todos os bots). Em vez disso, `janelaAtual()` já é a primitiva de consulta não-bloqueante que um `Comando`/`TarefaContinua` futuro, ticado, pode usar para "esperar" através de polling entre ciclos — mesmo padrão já usado por `GuiaDeCaminho.finalizado()`.
- **Reconciliação de `Confirm Transaction`**: fiel ao legado (`Handler_v152` case 106) — quando o servidor rejeita, o bot reenvia `ConfirmarTransacaoPacket(accepted:true)` automaticamente (`ReceptorConfirmTransaction`), sem corrigir o estado local dos slots nem reenviar o clique original (a correção real vem do próximo `Set Slot`/`Window Items`, mesma limitação do legado, preservada por fidelidade).
- **`Window Property`** (`0x31`, usada por progresso de fornalha/mesa de encantamento/bigorna) e **`Creative Inventory Action`** (`0x10`) deliberadamente **não implementados** — o legado nunca implementa `Window Property` (`Handler_v152` case 105: `SkipBytes` puro) e `Creative Inventory Action` só serve `CommandGive`, já excluído do port desde a DEC-23. Mesma "regra de três"/proveniência de necessidade real já aplicada em todas as DECs anteriores.

### Entregue

- `domain.bot.Janela` (novo agregado — `windowId`/`tipo`/`titulo`/`numeroDeSlots`/`entityId` opcional/slots/contador de transação próprio) e `domain.bot.JanelaDeSlots` (nova interface, implementada também por `InventarioDoJogador`).
- 6 famílias de Packet/Codec/Handler/Evento (+Receptor onde há consumo real): `OpenWindowPacket` (CB 0x2D, com campo condicional `entityId` só para tipo `"EntityHorse"`), `CloseWindowPacket` (CB 0x2E)/`FecharJanelaPacket` (SB 0x0D), `ClickWindowPacket` (SB 0x0E), `ConfirmTransactionPacket` (CB 0x32)/`ConfirmarTransacaoPacket` (SB 0x0F).
- `ReceptorSetSlot`/`ReceptorWindowItems` estendidos: `windowId != 0` deixa de ser no-op (lacuna documentada desde a Milestone 5.5/31) e passa a rotear para `SessaoDeJogo.atualizarSlotDaJanela`/`definirConteudoDaJanela` quando o id bate com a janela aberta, com o mesmo offset `+9` da indexação unificada.
- `SessaoDeJogo` ganha a API completa do Container Framework: `janelaAtual`/`abrirJanela`/`fecharJanela`/`fecharJanelaRecebidaDoServidor`/`clicarSlot`/`shiftClique`/`trocarSlots`/`pegarItem`/`largarItem`/`moverItem`/`confirmarTransacao`/`inventarioDaJanela`/`localizarItemNaJanela`/`localizarEspacoLivreNaJanela`, mais a simulação local de clique (pickup/split/mescla/troca, fiel a `Inventory.Click` — inclusive a mesma simplificação do legado de tratar shift-clique como clique normal na predição local) e a correção do bug `ItemStack.IsSameItem` do legado (comparava NBT do item consigo mesmo; aqui compara `itemId`+`damage` de verdade).
- 8 Casos de Uso (`CasoDeUsoFecharJanela`/`ClicarSlot`/`ShiftClique`/`TrocarSlots`/`PegarItem`/`LargarItem`/`MoverItem`/`ConfirmarTransacao`), mesmo padrão de wrapper fino já usado em todo o projeto.
- `interfaces.comando.ComandoClicarSlot` (porte de `CommandInvClick`, superset estritamente mais capaz — alcança a indexação unificada, não só a janela) e `ComandoFecharJanela` (novo, sem precedente de nome direto no legado, mas comportamento idêntico a `MacroUtils.fecharBau`).
- Zero interface pública pré-existente com assinatura alterada; zero bounded context novo.

### Testes

98 testes novos (Codec round-trip, Handler, Receptor — incluindo os casos de extensão de `ReceptorSetSlot`/`ReceptorWindowItems` —, `Janela`/`JanelaDeSlots` via `InventarioDoJogador`, ~25 casos novos em `SessaoDeJogoTest` cobrindo cada primitiva e os 4 ramos da simulação de clique, `RegistroDePacotesV1_8Test`, 8 testes de Caso de Uso, testes de `ComandoClicarSlot`/`ComandoFecharJanela`).

### Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **963 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 865→963 (+98), mudança 100% aditiva. Desbloqueia formalmente a Milestone 31 (AutoFish/`CommandPesca`) e qualquer épico futuro do roadmap de infraestrutura (EPIC-I2 em diante) que dependa de container.

---

## Pivô de Estratégia: Unidade de Trabalho passa a ser o Domínio (2026-07-23)

O responsável decide que a unidade de trabalho não é mais uma milestone/épico por sessão — passa a ser um **domínio da plataforma**, que pode conter vários épicos. Instrução: esgotar completamente o domínio solicitado enquanto existir reaproveitamento arquitetural, sem parar após o primeiro épico, só interrompendo nos gatilhos já conhecidos (novo bounded context/agregado/Port, conflito entre DECs, redefinição de arquitetura, dependência de decisão do responsável, mudança de política). Nunca implementar macro diretamente — sempre extrair capacidade reutilizável; macro só conecta componentes já prontos. Documentação e testes só são atualizados ao final do domínio, não a cada incremento. Primeiro domínio escolhido: **Inventory/Container** (cadeia Container → Inventory → Item Transfer → Equipment → Item Search → Inventory Utils), por já ter o épico raiz — EPIC-I1/DEC-37 — concluído na sessão anterior.

## Milestone 33 — Domínio Inventory/Container: Auditoria e Fechamento

Status

CONCLUÍDA — domínio esgotado nesta sessão (auditoria completa + lacunas reais implementadas).

Objetivo

Auditar exaustivamente o domínio Inventory/Container (legado C# vs. árvore Java atual, não só o mapeamento de primitivas produzido antes da Milestone 32, que só tinha lido o C#) e implementar toda capacidade reutilizável ausente, sem implementar nenhuma macro.

### Auditoria (achados, não repetir busca)

- **Container + Item Transfer**: 100% cobertos desde a Milestone 32/DEC-37 — `Janela`/`JanelaDeSlots`, `SessaoDeJogo.clicarSlot`/`shiftClique`/`trocarSlots`/`pegarItem`/`largarItem`/`moverItem`/`confirmarTransacao`, todos com Caso de Uso correspondente. Confirma a suspeita já registrada no pivô da Fase 2 ("Bloco A" cross-check pendente) — nada a reimplementar.
- **Camada `interfaces.comando`**: lacuna real encontrada — 6 dos 8 Casos de Uso do Container Framework (`ShiftClique`/`TrocarSlots`/`PegarItem`/`LargarItem`/`MoverItem`/`ConfirmarTransacao`) não tinham nenhum `Comando` conectando-os, só `ClicarSlot`/`FecharJanela`. Puramente aditivo (nenhuma capacidade nova, só conexão), implementado.
- **Item Search**: `JanelaDeSlots.localizarItem`/`localizarEspacoLivre`/`contarEspacosLivres` já cobriam a maior parte. Lacuna real: o legado tem `CommandMob.buscarSlotsCom`/`CommandMobPlus.buscarSlotsCom`/`CommandPescaV2.verificarSlotsCom` (retornam TODOS os slots com um item, não só o primeiro) sem equivalente Java — 3 ocorrências no legado (regra de três da DEC atendida), implementado como `JanelaDeSlots.localizarTodosOsItens(short)`.
- **Inventory Utils**: lacuna real encontrada — `CommandMob.limparCursor()` (recupera um cursor "preso" descartando o item no primeiro slot livre) não tinha porte. Implementado como `SessaoDeJogo.limparCursor()`, generalizado para também funcionar sem nenhuma janela aberta (mesmo "superset mais capaz" da DEC-37). `sort`/`contar quantidade total de um item`/`limpar inventário por whitelist` **não têm nenhum precedente no legado** (grep exaustivo, zero ocorrências de sort; `limparInventario` do legado é uma whitelist hardcoded específica da máquina de estados do `CommandPesca`, não uma capacidade genérica) — não implementados, evitando introduzir tecnologia sem necessidade real (regra do CLAUDE.md).
- **Equipment/Armadura**: analisado exaustivamente — **o legado não tem nenhum sistema de vestir armadura**. Toda referência a armor no C# é (a) constantes de item ID sem lógica (`Items.cs`), (b) armazenamento passivo em código morto/nunca registrado (`CommandPescaV2.guardarFerramentaEArmadura`, que só guarda em baú, nunca veste), ou (c) o já portado `EntityEquipmentPacket`/`EntidadeRemota.equipamento` (Milestone 5.6.4) — que é leitura de equipamento de **outras** entidades, não do próprio bot. Existe precedente de "auto-seleção do melhor item" só para ferramentas (`MacroUtils.selecionarMelhorItem`, comparação de força vs. bloco), sem equivalente algum para armadura (nenhuma tabela de "melhor proteção" no legado). Construir uma lógica de auto-equipar-melhor-armadura agora seria **inventar uma regra de negócio sem validação no legado** — proibido pelo CLAUDE.md. **Decisão: Equipment não é uma lacuna funcional** — o legado nunca veste armadura em nenhuma macro, então não há comportamento a preservar nem consumidor a atender. Vestir/trocar armadura manualmente já funciona hoje via `clicarSlot`/`moverItem`/`trocarSlots` genéricos sobre os slots 5-8 (armadura aparece na indexação unificada quando não há janela aberta), sem necessidade de nenhuma API semântica nova. Fica registrado como candidato dependente de decisão do responsável (não um bloqueio), caso um dia se queira definir uma regra de auto-equipe inédita.

### Entregue

- `domain.bot.JanelaDeSlots.localizarTodosOsItens(short itemId): List<Integer>` (novo default method — porte de `CommandMob.buscarSlotsCom`).
- `domain.bot.SessaoDeJogo.limparCursor()` (novo — porte generalizado de `CommandMob.limparCursor`, prefere slot livre da janela aberta, cai para o inventário do bot pulando crafting/armadura, lança `IllegalStateException` se não houver espaço livre em lugar nenhum).
- `application.usecase.CasoDeUsoLimparCursor` (novo, mesmo padrão de wrapper fino de todo o projeto).
- `interfaces.comando.ComandoShiftClique`/`ComandoTrocarSlots`/`ComandoPegarItem`/`ComandoLargarItem`/`ComandoMoverItem`/`ComandoConfirmarTransacao`/`ComandoLimparCursor` (7 novos — conectam os Casos de Uso do Container Framework, já aprovados desde a Milestone 32, à camada de comandos, mesmo padrão de `ComandoClicarSlot`).
- Zero interface pública pré-existente alterada, zero Packet/Port/agregado/bounded context novo, **zero DEC nova** (crescimento puramente aditivo, mesmo padrão já estabelecido desde a DEC-19).

### Testes

31 testes novos (`JanelaTest`/`InventarioDoJogadorTest` para `localizarTodosOsItens`, 5 casos novos em `SessaoDeJogoTest` para `limparCursor`, `CasoDeUsoLimparCursorTest`, e 3 testes por `Comando` novo × 7 comandos, com 1 caso extra em `ComandoConfirmarTransacaoTest` para o branch `aceito=false`).

### Validação executada

`mvn test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **994 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 963→994 (+31), mudança 100% aditiva. **Domínio Inventory/Container esgotado**: qualquer macro futura que precise de container/transferência de item/busca/limpeza de cursor (AutoFish/M31, Solk, uma futura macro de reparo) passa a ser pura composição sobre primitivas já prontas, sem infraestrutura nova.

---

## Milestone 34 — Domínio Interação com o Mundo (Blocos/Entidades/Estruturas): Auditoria e Fechamento

Status

CONCLUÍDA — domínio esgotado nesta sessão (auditoria completa + única lacuna real implementada).

Objetivo

Auditar exaustivamente toda a interação genérica do bot com o mundo (blocos, portas/alavancas/botões, placas, NPCs, entidades, uso de item, uso de item em bloco, estruturas) — legado C# (`Projeto Adv 2.4.5`) vs. árvore Java atual, classificar em já implementado/parcialmente implementado/ausente, e implementar toda primitiva reutilizável ausente com 3+ consumidores no legado, sem implementar nenhuma macro (automações do legado tratadas exclusivamente como consumidoras futuras).

### Auditoria (achados, não repetir busca)

- **Interação com blocos (clicar/quebrar/colocar com auto-mira)**: 100% coberta desde a Milestone 14/DEC-25 (`ComandoClicarBloco`/`ComandoQuebrarBloco`/`ComandoColocarBlocoAutoMira`) — nada a reimplementar.
- **Portas/alavancas/botões/placas/NPCs (trade)/estruturas genéricas**: **zero precedente próprio no legado** — busca exaustiva em `AdvancedBot.Client.Commands/*.cs` não encontra nenhuma classe/lógica específica de porta, alavanca, botão ou trade com NPC (Villager). Toda interação física desse tipo no legado (`MacroUtils.RightclickBlock`/`openChest`, usadas por `CommandPesca`/Solk para placas de compra/venda e baús) é só um right-click genérico de bloco (raycast + `PacketBlockPlace` sem item ou com item da mão) — já coberto desde a Milestone 14 (`ComandoClicarBloco`, botão direito) e desde a Milestone 32/DEC-37 (Container). Nenhuma regra de negócio nova a portar; construir uma API semântica dedicada a "porta"/"alavanca" seria inventar comportamento sem precedente (proibido pelo CLAUDE.md).
- **Placas (leitura/escrita)**: leitura já coberta desde a Milestone 18 (`UpdateSignPacket`, CLIENTBOUND). Escrita de placa (`Update Sign` SERVERBOUND) **sem nenhum call site em todo o legado** — nenhuma automação cria/edita placas. Não implementada.
- **Estruturas (portais)**: 100% coberta desde a Milestone 29 (`ComandoPortal`/`RegistroDePortais`). Nenhuma outra "estrutura" com precedente no legado.
- **Uso de item / uso de item em bloco**: 100% coberto desde a Milestone 14/DEC-25 (`SessaoDeJogo.usarItemNaMao`, sentinela `-1/-1/-1` para item sem alvo — baldes, isqueiro etc.) e Milestone 14 (`ComandoColocarBlocoAutoMira`/`ComandoClicarBloco` para item usado sobre um bloco-alvo). Confirmado contra `MacroUtils`/`CommandUseBow`/`CommandHotbarClick` — todos compõem sobre esses dois primitivos já existentes, sem lacuna.
- **Interação com entidades/NPCs (`UseEntity`)**: **lacuna real encontrada** — `PacketUseEntity`/`CommandUseEntity.cs` (interagir ou atacar uma entidade uma única vez, botão 0=atacar/1=interagir) **nunca foi portado** (zero ocorrência em `src/main/java`, confirmado por grep). Consumidores no legado: `CommandUseEntity` (comando direto), `CommandUseBow` (mira e dispara, usa o próprio player como "entidade" via aim, não via `UseEntity`) e `CommandKillAura` (permanentemente fora de escopo por política). Implementada como primitiva genérica de protocolo — único gap real do domínio.
- **Ferramenta ideal por bloco (`MacroUtils.selecionarMelhorItem`)**: pertence ao domínio Inventory/Mineração, não a este — já coberta pela composição existente em `TarefaMineracao` (Milestone 30); fora do escopo desta auditoria.
- **Macros avaliadas só como consumidoras (não implementadas)**: `CommandTwerk` (toggle de agachar repetido — primitiva `agachar`/`pararDeAgachar` já existe desde a Milestone 20), `CommandRetard` (movimento aleatório — depende de `Player.MoveQueue`/`Movement` por yaw, mesmo mecanismo não portado por decisão da DEC-36/Milestone 30), `CommandInvCaptcha` (bypass de anti-cheat via cliques de inventário — já composto sobre `clicarSlot`/Milestone 32, fora do escopo de "interação com mundo"), `CommandReco` (reconexão manual — infraestrutura de rede, não interação com mundo).

### Entregue

- `domain.protocol.v1_8.UseEntityPacket`/`UseEntityCodec` (novo, PLAY `0x02` SERVERBOUND — porte de `PacketUseEntity`; só os 2 campos que o legado de fato envia, `EntityID`+`tipoAcao` 0/1; o 3º tipo do protocolo, `INTERACT_AT` com vetor de acerto, não tem nenhum call site no legado e não é suportado pelo Codec).
- `domain.bot.SessaoDeJogo.interagirComEntidade(entityId, tipoAcao)` (novo, com constantes públicas `ACAO_INTERAGIR_ENTIDADE`/`ACAO_ATACAR_ENTIDADE`).
- `application.usecase.CasoDeUsoOlharParaEntidade` (novo — faltava o wrapper de `interfaces.comando` para `SessaoDeJogo.olharParaEntidade`, que já existia desde a Milestone 28 mas só tinha consumidor direto em `TarefaFollow`) e `CasoDeUsoInteragirComEntidade` (novo).
- `interfaces.comando.ComandoUseEntity` (novo — porte de `CommandUseEntity`: nick explícito ou `@any`/sem argumento para o jogador visível mais próximo em raio 4 blocos via `podeVerJogador`/Milestone 19; nick explícito exige visibilidade + distância ao quadrado ≤36; modo 2º argumento, ausente ou diferente de `"1"` = atacar, `"1"` = interagir, fiel à inversão de índice do próprio legado). Divergência documentada e deliberadamente não portada: o filtro `IsBot` do legado (exclui da busca `@any` outros bots controlados pelo mesmo processo) não tem equivalente — não existe registro de múltiplas instâncias de `Bot` no domínio Java ainda, mesma categoria de omissão do toggle `"retard"` em `ComandoFollow` (Milestone 28).
- Zero interface pública pré-existente alterada, zero Packet/Port/agregado/bounded context novo, **zero DEC nova** (crescimento puramente aditivo, mesmo padrão já estabelecido desde a DEC-19 — novo Packet SERVERBOUND + novo caminho `Comando → Caso de Uso → SessaoDeJogo → Packet`, mesma forma de toda milestone desde a DEC-21).

### Testes

13 testes novos (`UseEntityCodecTest` round-trip + distinção interagir/atacar, `CasoDeUsoOlharParaEntidadeTest`, `CasoDeUsoInteragirComEntidadeTest`, `ComandoUseEntityTest` com 7 casos — ataque padrão, modo interagir, `@any` mais próximo, `@any` sem alvo, nick não encontrado, nick muito longe, sem sessão ativa).

### Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **1007 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 994→1007 (+13), mudança 100% aditiva. **Domínio Interação com o Mundo esgotado**: blocos/portas/alavancas/botões/placas/estruturas/uso-de-item já estavam 100% cobertos por milestones anteriores (14, 18, 25, 29, 32); a única lacuna real (interação/ataque manual a entidades) foi fechada. Qualquer macro futura que precise interagir com uma entidade (trade com NPC, `CommandUseBow`, um eventual porte pontual de partes não-combativas de `CommandKillAura`) é pura composição sobre `interagirComEntidade`, sem infraestrutura nova.

---

## Milestone 35 — Backlog "Composição Pura": implementação em lote de todo item sem DEC/decisão/mudança arquitetural pendente

Status

CONCLUÍDA — lote inteiro implementado como uma única unidade de trabalho, sem pausa entre itens (instrução explícita do responsável).

Objetivo

Nova mudança de estratégia (2026-07-23, mesma sessão do artifact "Mapa completo da migração AdvancedBot" — ver `project_backlog_definitivo` na memória, não commitado em `docs-reescrita`): em vez de auditar domínio por domínio, implementar de uma vez todo item do backlog global já classificado como pronto por composição pura (Épico 1) ou retoque trivial (Épico 2) — zero DEC nova, zero decisão do responsável, zero mudança arquitetural, infraestrutura 100% já existente. Épicos 3 (dado de altura por mob, ainda não levantado), 4 (decisão de política de combate) e 5 (motor de scripting, greenfield) explicitamente fora desta sessão por não atenderem esses critérios.

### Entregue

- **`ComandoLimparChat`** (porte de `CommandClearChat`) + `SaidaDoOperador.limpar()` (novo, 1 método).
- **`SessaoDeJogo.soltarItemEmUso()`** (novo — Player Digging status 5/`FinishUse`, porte do construtor de 1 argumento de `PacketPlayerDigging`). Sem consumidor de produção ainda (mira em jogador depende da decisão de política do Épico 4/`CommandUseBow`) — mesmo padrão "ships without a consumer" já aceito desde `tracarRaio`/DEC-24.
- **`TarefaLargarTudo`/`ComandoLargarTudo`** (porte de `CommandDropAll`) — `largarItem` (Milestone 32) já era exatamente `Inventory.DropItem`; só o laço pautado (1 slot a cada 3 ticks, slots 5-44) era novo.
- **`CasoDeUsoSelecionarSlotDaHotbar`** (novo, faltava o wrapper de `interfaces.comando` — `SessaoDeJogo.selecionarSlotDaHotbar` já existia desde a Milestone 25) + **`ComandoClicarItemDaHotbar`** (porte de `CommandHotbarClick`, compõe com `CasoDeUsoUsarItemNaMao`/M14).
- **`TarefaTwerk`/`ComandoTwerk`** (porte de `CommandTwerk`) — compõe só sobre `agachar`/`pararDeAgachar` (M20); o limiar de 3 ticks sem reset (alterna todo tick daí em diante) é comportamento literal do legado, preservado.
- **`CasoDeUsoReconectarBot`** (novo — desconecta se já havia `SessaoDeJogo`, depois chama `CasoDeUsoConectarBot`, fiel a `Client.StartClient()`) + **`ComandoReconectar`** (porte de `CommandReco`). Divergência documentada: sem suporte a `[IP:porta]` alternativo (exigiria um setter genérico de host/porta em `EnderecoServidor`, fora do escopo de composição pura).
- **`TarefaAutoFish`/`ComandoAutoFish`** (porte de `Solk/CommandPesca`, "solkpesca") — a maior peça do lote, 14 sub-estados internos portando fielmente a máquina de 9 estados do legado (`RECEM_LOGOU`/`LIMPAR_INVENTARIO`/`VERIFICAR_INVENTARIO`/`BUSCAR_VARA`/`COMPRAR_LINHA`/`GUARDAR_ITENS`/`REPARAR`/`PESCAR`). Achado crítico de fidelidade durante a implementação: `MinecraftClient.SendMessage` do legado intercepta qualquer mensagem começada com `"$"` e a roteia para `CmdManager.RunCommand` (auto-invocação LOCAL de outro comando) em vez de enviá-la como chat real — só mensagens sem `"$"` viram `PacketChatMessage`. Isso muda a leitura de `cmdPular = "$move j 40"` (usado em quase toda transição de estado do legado): não é chat, é uma chamada local a `CommandMove`, que por sua vez depende do mecanismo `Player.MoveQueue`/`Movement` por yaw (DEC-36, deliberadamente não portado, sem consumidor real até então). Substituído por `pular()+aplicarFisica()` (mesma composição de `TarefaAntiAFK`/DEC-32 — mesmo propósito observável de assentar a colisão após um teleporte, sem precisar do mecanismo bloqueado). Duas outras divergências documentadas no próprio código da classe: (a) `deveGuardarItem` do legado decide preservar equipamento encantado via NBT (`ItemStack.GetEnchantmentLevel`) — NBT de item é lacuna já registrada desde a Milestone 17/DEC-28, não nova; portado como sentinela `false` (a fase de guardar itens roda por completo mas nunca move nada, degradação aceita); (b) o fallback "nenhum baú com espaço" do legado usa `Player.MoveQueue` (mesmo mecanismo do item anterior) — só o primeiro baú candidato é tentado aqui, sem esse fallback de movimento lateral. `abrir baú` usa poll de `sessaoDeJogo.janelaAtual()` (mais preciso que o `await Task.Delay` cego do legado) e reaproveita um único helper privado de clique-com-botão-direito (elimina a duplicação que existia entre `RightclickBlock`/`abrirBau`/`$clickblock` no legado). Esperas encadeadas (`await Task.Delay`) viram um portão único de tempo real por fase (`System.currentTimeMillis()`, mesmo idioma de `TarefaMineracao.digInicioEm`).
- Zero interface pública pré-existente alterada, zero Packet/Port/agregado/bounded context novo, **zero DEC nova** em todo o lote (crescimento puramente aditivo, mesmo padrão desde a DEC-19/DEC-21 — cada item é só um novo `Comando`/`TarefaContinua` composto sobre primitivas já aprovadas).

### Testes

29 testes novos, distribuídos por todo o lote (`SaidaDoOperadorTest`/`ComandoLimparChatTest`; `SessaoDeJogoTest` para `soltarItemEmUso`; `TarefaLargarTudoTest`/`ComandoLargarTudoTest`; `CasoDeUsoSelecionarSlotDaHotbarTest`/`ComandoClicarItemDaHotbarTest`; `TarefaTwerkTest`/`ComandoTwerkTest`; `CasoDeUsoReconectarBotTest`/`ComandoReconectarTest`; `TarefaAutoFishTest`/`ComandoAutoFishTest`).

### Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **1036 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 1007→1036 (+29), mudança 100% aditiva. **Backlog de composição pura esgotado** — todo item classificado como Épico 1/2 no artifact "Mapa completo da migração AdvancedBot" está implementado. Restam só os Épicos 3 (dado de altura por mob), 4 (decisão de política de combate — Solk/CommandMob e o restante de UseBow) e 5 (motor de scripting, Fase 3), nenhum dos quais atende aos critérios desta sessão (infra 100% pronta sem DEC/decisão/mudança arquitetural).

---

## Milestone 36 — DEC-38: Reabertura Parcial da DEC-23, Política de Combate (Épico 4)

Status

CONCLUÍDA — decisão de política registrada e único gap de infraestrutura genérica identificado foi fechado. Nenhuma macro (Solk/CommandMob/CommandUseBow) foi implementada nesta sessão.

Objetivo

Resolver o bloqueio do Épico 4, registrado desde a Milestone 35: a DEC-23 excluía "automação/combate" (`CommandKillAura`/`CommandMiner`/`CommandHerbalism`/`CommandAntiAFK`/`Solk.*`) como um único item indiferenciado. Instrução explícita do responsável nesta sessão: separar (a) combate como mecanismo genérico — ataque automático indiscriminado, PvP automático, ataque através de parede, qualquer forma de KillAura/Aura — de (b) macros do legado cujo propósito não é combate, mas cuja máquina de estados inclui atacar uma entidade específica já identificada como uma etapa entre várias (ex.: `CommandMob.hitarMob()` ataca só o `EntityMob` já resolvido por `buscarMobProximo()`).

### Entregue

- **DEC-38** (`01-Decisoes-Arquiteturais.md`): reabre parcialmente a DEC-23 (mesmo padrão da DEC-32/DEC-35 sobre a DEC-22 — nenhum texto da DEC-23 é alterado). Formaliza Categoria 1 (automação de combate genérica — permanece fora de escopo, sem exceção, sem prazo) separada da Categoria 2 (ataque a entidade específica já resolvida, como etapa de uma automação de negócio maior — passa a candidata, aprovação individual macro por macro). Fundamenta a distinção no precedente já em produção desde a Milestone 34: `UseEntityPacket`/`SessaoDeJogo.interagirComEntidade`/`ComandoUseEntity` já tratam ataque pontual a alvo específico como interação com o mundo, não como combate, com zero DEC própria à época.
- **`EntidadesDoMundo.mobMaisProximo(x, y, z, raio): EntidadeMob`** (novo — `domain.bot`) — única lacuna de infraestrutura genérica identificada: não existia nenhuma forma de localizar a `EntidadeMob` mais próxima por proximidade (equivalente genérico de `buscarMobProximo()` do legado; `porId`/`porUuid` já existiam, mas nenhuma busca por tipo+proximidade). Primitiva pura, mesma categoria de `Mundo.tracarRaio` (DEC-24) — sem Packet, sem Port, sem Use Case, sem checagem de linha de visão (fiel ao `CanSeeEntity` desativado no próprio legado). Raio é parâmetro do chamador, não constante travada (regra de três, DEC-18) — primeiro consumidor genérico.
- Todo o restante da infraestrutura necessária para as macros da Categoria 2 (`UseEntityPacket`, `interagirComEntidade`, `Mundo.tracarRaio`, `GuiaDeCaminho`, `MotorDeTick`/`TarefaContinua`) já existia e foi apenas confirmada como reutilizável — nenhuma mudança adicional.
- **`Solk`/`CommandMob`/`CommandMobTeleport`/`CommandPesca` e `CommandUseBow` continuam não implementados** — desbloqueados como candidatos pela DEC-38, mas cada um exige aprovação individual do responsável ("caso aprovado") antes de qualquer código de macro, exatamente como qualquer outra macro da Fase 2.

### Testes

5 testes novos em `EntidadesDoMundoTest` (`mobMaisProximo`: nenhum mob registrado, único mob fora do raio, mais próximo dentre vários dentro do raio, ignora `EntidadeJogadorRemoto`, mob exatamente na borda do raio).

### Validação executada

`mvn test` (JDK 21.0.9, Maven 3) — BUILD SUCCESS, **1041 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 1036→1041 (+5), mudança 100% aditiva. Zero interface pública pré-existente alterada. Zero Packet/Port/agregado/bounded context novo. **Épico 4 do backlog global desbloqueado quanto à política** — a implementação de cada macro específica (Solk/CommandMob, CommandUseBow) permanece pendente de aprovação individual do responsável.

---

## Milestone 37 — Solk/CommandMob (Categoria 2/DEC-38): Macro de Farm de Mob

Status

CONCLUÍDA — `TarefaMob`/`ComandoMob` (porte de `Solk/CommandMob.cs`, "solkmob") implementados por composição pura sobre capacidades já aprovadas. Zero DEC nova (mesmo padrão "crescimento aditivo" da Milestone 35 — nenhuma decisão arquitetural nova, só liga primitivas já existentes e as duas capacidades genéricas extraídas nesta sessão).

Objetivo

Aprovação individual do responsável para reconstruir integralmente `CommandMob`/Solk (Categoria 2 da DEC-38 — ataque à entidade já resolvida por `buscarMob`, uma entre várias etapas de uma automação de farm/venda/troca de ferramenta, não combate genérico). Decompor a macro em capacidades reutilizáveis antes de codificar, reaproveitar tudo que já existe, implementar só o realmente ausente, e generalizar qualquer capacidade que sirva outras macros.

### Decomposição (antes de codificar)

`CommandMob` tem 8 fases reais (RECEM_LOGOU, BUSCAR_MOB, VERIFICAR_INVENTARIO, TROCAR_FERRAMENTA, VENDER, MATAR_MOB, MOB_NAO_ENCONTRADO, e VERIFICAR_POCAO — inalcançável com a config default, ver divergências). Nenhuma delas exigiu Packet/Port/agregado novo:

- **Buscar mob por proximidade**: `EntidadesDoMundo.mobMaisProximo` (Milestone 36/DEC-38) — já pronta, primeiro consumidor de produção.
- **Atacar a entidade já resolvida**: `SessaoDeJogo.interagirComEntidade`/`UseEntityPacket`/`ACAO_ATACAR_ENTIDADE` (Milestone 34) — já pronta.
- **Mirar a entidade / pular após teleporte**: `olharParaEntidade` (Milestone 28), `pular()`+`aplicarFisica()` (mesma substituição de `$move j N` já usada em `TarefaAntiAFK`/`TarefaAutoFish`/`TarefaTwerk`) — já prontas.
- **Abrir baú por coordenada / trocar ferramenta / limpar espadas extras**: Container/Janela (DEC-37, `moverItem`/`localizarEspacoLivreNaJanela`/`fecharJanela`) — já pronto. **Abrir com retentativa** (`MacroUtils.abrirBau` do legado) só existia como método **privado** dentro de `TarefaAutoFish` (`tentarAbrirBau`) — extraído nesta sessão para `AbridorDeBau` (novo, `domain.bot`, package-private), reaproveitado pelas duas macros (regra de três aplicada uma única vez).
- **Clicar bloco com botão direito (raycast + `colocarBloco`)**: idem — também só existia privado em `TarefaAutoFish`, extraído para `MacroUtils.clicarBlocoComBotaoDireito` (novo, `domain.bot`, package-private).
- **Clicar placa de compra/venda N vezes agachando** (`MacroUtils.comprar`/`vender` do legado — mesma forma, N diferente): generalizado como `MacroUtils.clicarPlacaAgachando(sessaoDeJogo, x, y, z, vezes)` (novo) — `TarefaAutoFish.comprarLinhaComprando` (vezes=4) e `TarefaMob.venderVendendo` (vezes=6) reaproveitam a mesma função.
- **Contar slots vagos do inventário** (`MacroUtils.contarQntDeSlotsVagos` do legado): generalizado como `MacroUtils.contarSlotsVagos` (novo) — mesmo caso, extraído de `TarefaAutoFish` e reaproveitado por `TarefaMob`.
- **Resolver slot unificado / item na mão**: idem, extraídos para `MacroUtils.slotUnificadoDoInventario`/`MacroUtils.itemNaMao` (novos) — mesma lógica que já existia privada em `TarefaAutoFish`.
- **Busca de baú por cubo ao redor dos pés** (`CommandMob.buscarBauProximo`): algoritmo próprio de `CommandMob`, **não** unificado com `TarefaAutoFish.detectarBaus` (ordem de varredura e seleção diferentes já no próprio legado, entre os dois comandos) — implementado como método privado em `TarefaMob`, fiel ao original.
- **Máquina de estados / execução contínua**: `TarefaContinua`/`MotorDeTick`/`Bot.registrarTarefa` (Milestone 21/24) — já prontos, mesmo padrão de toggle de `ComandoAutoFish`/`ComandoAntiAFK`.

Nenhuma capacidade nova de protocolo, pathfinding (`CommandMob` usa exclusivamente teleporte por comando de chat, fiel ao legado) ou física horizontal foi necessária.

### Entregue

- **`domain.bot.MacroUtils`** (novo, package-private) — `clicarBlocoComBotaoDireito`, `clicarPlacaAgachando`, `contarSlotsVagos`, `slotUnificadoDoInventario`, `itemNaMao`. Extraído de `TarefaAutoFish` (que tinha essas 5 capacidades duplicadas como métodos privados) no momento em que `TarefaMob` precisou de 4 delas — `TarefaAutoFish` refatorada para reaproveitar em vez de manter a duplicata.
- **`domain.bot.AbridorDeBau`** (novo, package-private) — abrir baú com até 11 retentativas/~2s (fiel a `MacroUtils.abrirBau`), extraído de `TarefaAutoFish.tentarAbrirBau` pelo mesmo motivo, reaproveitado por `TarefaMob`.
- **`domain.bot.TarefaMob`** (novo, porte de `Solk/CommandMob.cs`) — 12 sub-estados internos (RECEM_LOGOU, BUSCAR_MOB, MOB_NAO_ENCONTRADO, VERIFICAR_INVENTARIO, TROCAR_FERRAMENTA_TELEPORTANDO/ABRINDO_BAU/TROCANDO/VOLTANDO, VENDER_TELEPORTANDO/VENDENDO/VOLTANDO, MATAR_MOB), 100% composição sobre as capacidades acima.
- **`interfaces.comando.ComandoMob`** (novo, aliases `solkmob`/`mob`) — toggle simples sobre `TarefaMob`, mesmo padrão de `ComandoAutoFish`.
- Divergências documentadas na própria classe (não capacidades ausentes, mesma disciplina das macros anteriores): (1) `FileLock` multi-conta/multi-processo do legado sem equivalente (arquitetura single-tenant, DEC-09); (2) `VERIFICAR_POCAO`/`BUSCAR_POCAO` inalcançável com a config default (`homesVenda` sem "blaze"), config por conta não portada (mesma decisão já tomada por `TarefaAutoFish`); (3) filtro de encantamento Flame ao trocar espada depende de NBT, lacuna já registrada desde a Milestone 17/DEC-28; (4) limpeza de espadas extras em slots indevidos portada sem os laços de retentativa com poll real do legado (predição otimista, mesma simplificação do Container Framework/DEC-37); (5) `$move j N` sempre vira `pular()+aplicarFisica()`; (6) esperas encadeadas viram um único portão de tempo real; (7) interrupções reativas a chat de outra conta/servidor não portadas (dependem do parser de `ChatComponent`, lacuna já registrada desde a Milestone 5 Incremento 3); (8) os 850 hits de `MATAR_MOB` são distribuídos em ataques individuais gated a cada 100ms (não um laço bloqueante de ~85s, que travaria o `MotorDeTick`), mesmo idioma não-bloqueante de `TarefaAutoFish.pescar()`.
- Zero DEC nova (a política já foi decidida pela DEC-38 na Milestone 36; esta milestone só implementa dentro dela). Zero Packet/Port/agregado/bounded context novo. Zero interface pública pré-existente alterada (a refatoração de `TarefaAutoFish` moveu métodos privados para `MacroUtils`/`AbridorDeBau`, sem mudar nenhum comportamento observável nem assinatura pública).

### Macros agora desbloqueadas

`Solk/CommandMob` portado e funcional. **Candidatos remanescentes do Épico 4** (não implementados nesta sessão, mesma categoria "aprovação individual" da DEC-38): `CommandMobTeleport` (Solk, vivo no legado) e `CommandUseBow` quando limitado a mirar/disparar contra um alvo já identificado. `CommandMobPlus`/`CommandPescaV2` continuam código morto confirmado (nunca registrados em `CommandManagerNew` no legado) — não candidatos.

### Testes

10 testes novos: `TarefaMobTest` (8 — teleporte inicial envia `/spawn`+`/home mob`; não repete ação aguardando; no-op sem sessão; encontra mob dentro do raio e inicia combate; não encontra mob fora do raio e reenvia `/home mob`; ataca a entidade já resolvida em `MATAR_MOB` via `UseEntityPacket`/`ACAO_ATACAR_ENTIDADE`; volta para `VERIFICAR_INVENTARIO` quando o mob desaparece; vai para `TROCAR_FERRAMENTA` quando a espada está ausente/gasta), `ComandoMobTest` (2 — liga/desliga o toggle). `TarefaAutoFishTest`/`ComandoAutoFishTest` existentes continuam 100% verdes após a extração de `MacroUtils`/`AbridorDeBau` (nenhum comportamento mudou, só a localização do código).

### Validação executada

`mvn test` (JDK 21.0.9, Maven 3) — BUILD SUCCESS, **1051 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 1041→1051 (+10), mudança 100% aditiva (mais a refatoração interna, sem impacto observável, de `TarefaAutoFish`).

---

## Milestone 38 — Pivô de Estratégia: Unidade de Trabalho passa a ser Capacidade Fundamental (Foundation APIs)

Status

CONCLUÍDA — matriz de domínios funcionais levantada a partir do conhecimento acumulado (sem nova auditoria macro por macro); Foundation APIs faltantes identificadas e implementadas por extração de duplicação já existente. Nenhuma macro nova implementada ou conectada nesta sessão.

Objetivo

Instrução explícita do responsável: a partir desta sessão a unidade de trabalho deixa de ser Macro/Épico e passa a ser Capacidade Fundamental reutilizável — o objetivo é que qualquer macro futura seja pura composição sobre uma API completa de interação do bot com o mundo, sem exigir infraestrutura nova. Sessão restrita a: agrupar ações por domínio funcional, marcar o que já existe/falta, implementar só as Foundation APIs realmente ausentes (reutilizáveis, sem decisão arquitetural nova, sem duplicação) — nenhuma macro implementada ou conectada.

### Matriz de Domínios Funcionais

Levantada a partir das ~29 macros do legado (`AdvancedBot.Client.Commands`, inventário já fechado desde a DEC-23) cruzadas com o estado Java acumulado das Milestones 1-37 — sem reabrir nenhuma análise de macro individual já feita.

| Domínio | Capacidades | Estado |
|---|---|---|
| Blocos | quebrar (3 status)/colocar/olhar/raycast/dureza-material-ferramentas | Fechado (M10/13/14/17, DEC-24/25/28) |
| Entidades | interagir-atacar/olhar/buscar-mob-por-proximidade/buscar-jogador-visível-por-proximidade/ciclo-de-vida (spawn/move/teleport/effects) | Fechado (M5/28/34, DEC-38/M36; LOS genérica p/ mob deliberadamente fora de escopo, decisão própria da DEC-24) |
| Inventário | ler/mover/largar/pegar/selecionar-hotbar/localizar-item/localizar-espaço-livre/**selecionar-item-na-hotbar** | Fechado (M5.5/10, agora com busca genérica — ver Foundation APIs) |
| Containers (baú/fornalha/brewing/beacon/anvil/hopper/dispenser/trade-NPC/cavalo) | abrir/clicar/shift-clique/trocar/confirmar-transação/fechar/**abrir-com-retentativa** | Fechado a nível de primitiva (DEC-37, `tipo` genérico por string — qualquer Window do protocolo); nenhuma macro ainda usa fornalha/brewing/anvil/beacon/hopper/dispenser, mas a primitiva já serve todos sem código novo |
| NPCs (trade real de Villager) | — | Sem gap real: nenhuma macro do legado usa trade de Villager (só placas de loja via clique de bloco genérico) |
| Placas | clicar (raycast+botão direito)/**clicar N vezes agachando** | Fechado p/ interação; leitura de texto (Update Sign CB) sem consumidor no legado |
| Portais | localizar/detectar | Fechado (RegistroDePortais, M29) |
| Chat | enviar/receber-bruto | Fechado p/ envio/recebimento; parser de `ChatComponent`/JSON permanece lacuna registrada desde M5 Incremento 3 — exige decisão própria, fora desta sessão |
| Movimento | mover/olhar/pular/física vertical+horizontal/pathfinding (`GuiaDeCaminho`/`criarCaminhoPara`) | Fechado dentro do escopo já decidido; MoveQueue/Movement-por-yaw permanece excluído por decisão própria (DEC-36) |
| Combate (Categoria 2) | atacar entidade já identificada | Fechado (DEC-38); Categoria 1 (KillAura) permanentemente excluída por política |
| Itens | usar-na-mão/soltar-em-uso/**item-na-mão (unificado)** | Fechado; NBT permanece lacuna registrada desde M17/DEC-28 |
| Equipamentos | ler equipamento de entidades remotas; **seleção automática de melhor ferramenta** | Fechado — bot nunca veste armadura própria no legado (confirmado M33, não é lacuna) |
| Efeitos | ler efeitos de entidades remotas (mirror) | Parcial — efeitos do PRÓPRIO bot (auto-buff) é lacuna já registrada na DEC-28, exige decisão própria |
| Mundo | ler bloco/chunk/raycast/dados de bloco | Fechado (M5/13, `Mundo`/`RegistroDeBlocos`) |
| Crafting | — | Sem gap real: nenhuma macro do legado craftea automaticamente (clique de slot genérico já cobre o grid quando necessário) |
| Furnace/Brewing/Beacon/Anvil/Dispenser/Hopper | abrir/clicar slots (genérico) | Fechado a nível de primitiva (Container Framework, DEC-37) — específicos de receita/propriedade (`Window Property`) deliberadamente não implementados, sem precedente no legado |

### Foundation APIs Criadas Nesta Sessão

Todas por extração de duplicação real já encontrada (regra de três já cumprida ou capacidade nomeada explicitamente pelo responsável) — zero decisão arquitetural nova, zero Packet/Port/agregado novo:

- **`SessaoDeJogo.jogadorVisivelMaisProximo(raio)`** (novo) — generalizado do método privado `ComandoUseEntity.jogadorVisivelMaisProximo`, simétrico a `EntidadesDoMundo.mobMaisProximo` (Milestone 36). `ComandoUseEntity` refatorado para reaproveitar.
- **`MacroUtils.selecionarMelhorFerramenta(sessaoDeJogo, bloco)`** (novo) — extraído de `TarefaMineracao` (fiel a `AutoMiner.SelectBestTool`), capacidade "Seleção automática de ferramenta" nomeada explicitamente pelo responsável. `TarefaMineracao` refatorada para reaproveitar.
- **`MacroUtils.selecionarSlotComItem(sessaoDeJogo, itemId)`** (novo) — generalizado de um trecho inline de `TarefaHerbalismo.replantar` (buscar item na hotbar antes de agir). `TarefaHerbalismo` refatorada para reaproveitar.
- **`MacroUtils.itemNaMao`** — já existia (Milestone 37); nesta sessão ganhou o 3º e 4º consumidores (`TarefaMineracao`/`TarefaHerbalismo` tinham cada uma sua própria cópia idêntica, agora eliminadas).

### Foundation APIs Já Existentes (confirmadas, reutilizadas sem alteração)

`EntidadesDoMundo.mobMaisProximo` (M36), `interagirComEntidade`/`UseEntityPacket` (M34), `olharParaEntidade`/`olharParaBloco` (M13/28), `Mundo.tracarRaio` (DEC-24), `CalculadoraDeQuebraDeBloco` (DEC-28), `GuiaDeCaminho`/`criarCaminhoPara`/`BuscadorDeCaminho` (DEC-27/35), `MotorDeTick`/`TarefaContinua`/`Bot.registrarTarefa` (DEC-29/33), Container/`Janela`/`JanelaDeSlots` (DEC-37), `AbridorDeBau`/`clicarBlocoComBotaoDireito`/`clicarPlacaAgachando`/`contarSlotsVagos`/`slotUnificadoDoInventario` (Milestone 37), `usarItemNaMao`/`soltarItemEmUso` (M14/35), `RegistroDeBlocos`/`RegistroDePortais` (DEC-24/M29).

### Domínios Completamente Fechados

Blocos, Entidades (Categoria 2), Inventário, Containers (nível de primitiva — inclui Furnace/Brewing/Beacon/Anvil/Dispenser/Hopper), Placas (interação), Portais, Movimento (dentro do escopo decidido), Combate (Categoria 2), Itens (exceto NBT), Equipamentos (sem gap real), Mundo, Crafting (sem gap real), NPCs (sem gap real — sem trade de Villager no legado).

### Domínios Parcialmente Fechados (exigem decisão arquitetural própria, fora desta sessão)

Chat (parser de `ChatComponent`/JSON — lacuna desde M5 Incremento 3), Efeitos (auto-buff do próprio bot — lacuna desde DEC-28), Itens (NBT — lacuna desde M17/DEC-28), Movimento (MoveQueue/Movement-por-yaw — excluído por DEC-36, reabertura exigiria nova DEC).

### Macros que passam a depender apenas de composição

Com a matriz fechada, os candidatos remanescentes do Épico 4 e além tornam-se composição pura sobre Foundation APIs já prontas, sem infraestrutura nova: `Solk/CommandMobTeleport`, `CommandUseBow` (mira/carrega/solta contra alvo já identificado — `usarItemNaMao`+`soltarItemEmUso` já bastam), `CommandScript` (Fase 3, orquestra comandos já existentes). Nenhum foi implementado nesta sessão (fora de escopo — "não conectar macro").

### Testes

Nenhum teste novo dedicado (as 4 extrações são refatorações comportamentalmente idênticas, já cobertas pelas suítes existentes — `ComandoUseEntityTest`, `TarefaMineracaoTest`, `TarefaHerbalismoTest`, `TarefaAutoFishTest`/`TarefaMobTest`). Suíte completa validada sem nenhuma falha após as 4 refatorações.

### Validação executada

`mvn test` (JDK 21.0.9, Maven 3) — BUILD SUCCESS, **1051 testes executados, 0 falhas, 0 erros, 3 skipped** (mesma contagem da Milestone 37 — sessão 100% refatoração/extração, nenhum teste novo, nenhuma regressão).

---

## Milestone 39 — DEC-39: Fechamento dos Domínios Chat/NBT/Efeitos do Próprio Bot (Foundation APIs)

Status

CONCLUÍDA — 3 dos 4 domínios "parcialmente fechados" da Milestone 38 fechados como Foundation APIs. Movimento/MoveQueue permanece deliberadamente fora de escopo (DEC-36, já resolvida). Nenhuma macro implementada ou conectada.

Objetivo

Instrução explícita do responsável: continuar o pivô da Milestone 38, fechando os últimos domínios fundamentais restantes — Chat, NBT, Efeitos do próprio jogador (nessa prioridade) — como infraestrutura reutilizável, sem nenhum comportamento de negócio/wiring/macro.

### Entregue

- **NBT** (`domain.protocol.v1_8.TagNBT`, novo — sealed interface, 11 records aninhados, um por tipo de tag do protocolo 1.8) + **`NbtCodec`** (novo, package-private — leitor/escritor genérico, extraído 1:1 do switch que `ItemStackCodec` já usava só para *descartar* NBT desde a Milestone 10). `ItemStack` ganha 4º componente (`nbt`), **aditivo**: construtor secundário de 3 argumentos preserva 100% dos call sites existentes (`nbt=null`, comportamento idêntico ao anterior). `ItemStackCodec` decodifica/codifica a árvore de verdade.
- **Efeitos do próprio bot**: `SessaoDeJogo` ganha `aplicarEfeito`/`removerEfeito`/`efeito`/`efeitosAtivos`, reaproveitando `EntidadeRemota.EfeitoAtivo` (sem duplicar o tipo). `ReceptorEntityEffect`/`ReceptorRemoveEntityEffect` roteiam para `SessaoDeJogo` quando `entityId` é o do próprio bot (antes: no-op silencioso, já que `EntidadesDoMundo` nunca contém o próprio bot) — mantém o roteamento já existente para entidades remotas intocado.
- **Chat**: `domain.protocol.v1_8.ParserDeChatComponent` (novo) — extrai texto plano de um `ChatComponent` JSON (`text`+`extra` recursivo), parser JSON próprio e minimalista (RFC 8259) em vez de dependência externa nova (`jackson-databind` não é dependência real deste projeto). `translate`/`with` não resolvem a chave de tradução (fora de escopo, exigiria `en_us.lang` inteiro) — concatena só os argumentos. `SessaoDeJogo` ganha `textoPlanoDaUltimaMensagemDeChat()`.
- **DEC-39** registrada em `01-Decisoes-Arquiteturais.md` — todas as 3 extensões são aditivas (nenhum contrato público quebrado; o único ponto de risco, `ItemStack`, é neutralizado pelo construtor de 3 args compatível), zero gatilho de parada acionado.

### Domínios Fechados

Chat (extração de texto plano — parser de tradução completo permanece fora, sem consumidor real), NBT (árvore genérica — leitura de regra de negócio como "tem encantamento X" é composição futura, não modelada aqui), Efeitos (próprio bot — resolução de bônus como `amplifierCeleridade`/DEC-28 é composição futura sobre `efeito(id)`, não modelada aqui).

### Testes

21 testes novos: `NbtCodecTest` (5 — raiz nula, escrita de TAG_END, round-trip dos 11 tipos aninhados, lista vazia, erro quando raiz não é compound), `ItemStackCodecTest` (+2, 1 teste renomeado para refletir NBT exposto em vez de descartado), `ParserDeChatComponentTest` (11 — vazio/nulo, componente simples, string JSON pura, `extra` recursivo, `translate`/`with`, escapes, campos de formatação ignorados, JSON inválido/truncado devolvido cru, array vazio/não-vazio no nível raiz), `SessaoDeJogoTest` (+3 — texto plano de chat, aplicar/consultar efeito próprio, remover efeito próprio), `ReceptorEntityEffectTest`/`ReceptorRemoveEntityEffectTest` (2 testes reescritos para refletir o novo roteamento para o próprio bot, antes documentavam o no-op).

### Validação executada

`mvn test` (JDK 21.0.9, Maven 3) — BUILD SUCCESS, **1072 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 1051→1072 (+21), mudança 100% aditiva (nenhuma assinatura pública pré-existente removida ou alterada).

---

## Milestone 40 — EPIC-APP1: Primeira API Pública (REST + WebSocket) da Plataforma (DEC-40)

Status

CONCLUÍDA — camada `interfaces.rest`/`interfaces.websocket` completa sob `/api/v1`, construída inteiramente sobre Casos de Uso e `interfaces.comando` já aprovados. Nenhuma regra de negócio nova. Nenhum épico do roadmap de fechamento de domínios (seção abaixo) foi tocado ou reordenado — EPIC-APP1 é uma mudança de fase ortogonal, instruída explicitamente pelo responsável, que passa a rodar em paralelo/antes da continuidade desse roadmap.

Objetivo

Instrução explícita do responsável: mudar o foco de "portar funcionalidades" para "executar bots reais através da aplicação Java". Construir a primeira API pública da plataforma (REST + WebSocket, não React ainda) para que um frontend React futuro consiga controlar centenas de bots. Toda a camada construída sobre os Casos de Uso já existentes, sem lógica de negócio em Controllers. Legado C# deliberadamente não consultado nesta etapa (instrução explícita — não há comportamento de legado a preservar, o C# nunca teve API).

### Entregue

- **`application.registry.GerenciadorDeBots`** (novo, registry por `IdentificadorBot`) + **`CasoDeUsoRemoverBot`** (novo). `CasoDeUsoCriarBot` ganha `GerenciadorDeBots` no construtor (único ajuste de assinatura pré-existente desta milestone).
- **4 Casos de Uso finos de transição de estado**: `CasoDeUsoIniciarBot`/`CasoDeUsoPararBot`/`CasoDeUsoPausarBot`/`CasoDeUsoRetomarBot`.
- **`infrastructure.config.ConfiguracaoDeComandos`** (novo) — conecta pela primeira vez em produção o `GerenciadorDeComandos`/~38 `Comando*` aprovados desde a Milestone 12/DEC-23. **`infrastructure.config.ConfiguracaoDeCasosDeUso`** (novo) — expõe como bean os ~24 Casos de Uso de ação/inventário que já existiam mas nunca tinham sido instanciados fora de teste.
- **`domain.bot.Conta`/`PerfilDeServidor`** (novos, records com identidade) + **`application.port.RepositorioDeContas`/`RepositorioDeServidores`** (portas novas) + adapters in-memory (`infrastructure.persistence`, antes vazio). `PoolDeProxies` ganha `adicionar`/`remover`/`listar`.
- **`application.registry.NotificadorDeEventos`** (novo, pub/sub por bot + global) + dois hooks opcionais aditivos (`SaidaDoOperador.definirOuvinte`, `Bot.definirOuvinteDeEstado`) + **`interfaces.websocket.BotEventsWebSocketHandler`**/`WebSocketConfig` (WebSocket cru, sem STOMP — `/ws/bots/{id}/events`, `/ws/events`).
- **`infrastructure.config.ApiKeyFilter`** (novo, header `X-API-Key`) — decisão deliberada de não introduzir Spring Security/OAuth nesta etapa (YAGNI).
- **11 Controllers REST** (`interfaces.rest.v1`): `BotController` (criar/remover/iniciar/parar/pausar/retomar/conectar/desconectar/reconectar/listar/detalhar), `AcaoController` (mover/olhar/quebrar/colocar/interagir/usar-item/chat), `InventarioController`, `MundoController` (estado/bloco/entidades/jogadores), `MacroController`, `ComandoController`, `ContaController`, `ServidorController`, `ProxyController`, `LogController`, `MetricasController`. DTOs em `interfaces.rest.dto`, `GlobalExceptionHandler` (400/404/409/500), `PaginacaoSupport` (offset/limit), `BotLookup`, `RecursoNaoEncontradoException`.
- **`pom.xml`**: `spring-boot-starter-web`, `spring-boot-starter-websocket`, `spring-boot-starter-validation`, `springdoc-openapi-starter-webmvc-ui` (Swagger UI em `/swagger-ui.html`). `application.yml` ganha `advancedbot.api.key`/`advancedbot.cors.allowed-origins`/`springdoc.*`.
- **DEC-40** registrada em `01-Decisoes-Arquiteturais.md`.

### Testes

17 testes novos: `CasoDeUsoRemoverBotTest`, `CasoDeUsoIniciarBotTest`/`CasoDeUsoPararBotTest`/`CasoDeUsoPausarBotTest`/`CasoDeUsoRetomarBotTest`, `NotificadorDeEventosTest`, `GerenciadorDeBotsTest`, `PoolDeProxiesTest`(+1), `BotTest`(+1), `CasoDeUsoCriarBotTest`(+1). Nenhum teste de Controller HTTP automatizado nesta sessão (verificação end-to-end feita manualmente via `mvn spring-boot:run` real, ver abaixo) — candidato a próxima sessão se o projeto adotar `@WebMvcTest`/`MockMvc`.

### Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7) — BUILD SUCCESS, **1089 testes executados, 0 falhas, 0 erros, 3 skipped** (mesmos 3 de sempre). 1072→1089 (+17). Validação manual end-to-end com `mvn spring-boot:run` real e `curl`/WebSocket (Node.js `WebSocket` nativo): `POST /api/v1/servidores` → `POST /api/v1/bots` (via `servidorId`) → `GET /api/v1/bots`/`GET /api/v1/bots/{id}` → `POST .../start` (204, estado muda para EXECUTANDO) → `GET /api/v1/metricas` reflete a contagem → `GET /api/v1/bots/{id}/logs` (vazio) → `GET /api/v1/bots/{id}/macros` (vazio) → `POST /api/v1/bots/{id}/macros/twerk` (200, `SUCESSO`, `TarefaTwerk` ativa) → `POST .../pause` (204) → `POST .../pause` de novo (409, `IllegalStateException` mapeada corretamente) → `DELETE /api/v1/bots/{id}` (204) → `GET /api/v1/bots/{id}` (404) → requisição sem `X-API-Key` (401). WebSocket: conectado em `/ws/events` antes de criar o bot, evento `{"tipo":"estado","payload":"EXECUTANDO",...}` recebido em tempo real ao chamar `/start`, confirmando o fluxo `Bot.iniciar()` → `NotificadorDeEventos` → `BotEventsWebSocketHandler` → client.

### Limitações Conhecidas (documentadas na DEC-40, não bloqueiam a milestone)

- Repositórios de Conta/Servidor/Proxy em memória (perdem estado a cada restart) — PostgreSQL (stack oficial) fica para milestone própria de persistência.
- Autenticação por chave única, sem múltiplos usuários/papéis.
- `EventoDeBot`/`IdentificadorBot` serializados por reflexão Jackson padrão (`{"value":"uuid"}`), sem customização de formato ainda.
- CORS/`allowedOriginPatterns` liberados amplamente por padrão de desenvolvimento — revisar antes de qualquer deploy exposto.
- Sem testes automatizados de Controller HTTP (`@WebMvcTest`/`MockMvc`/`@SpringBootTest` com `TestRestTemplate`) — só verificação manual nesta sessão.

---

**Nota de continuidade:** entre o encerramento desta seção (Milestone 40) e a Milestone 42 abaixo, a
Milestone 41 (EPIC-APP2 — cobertura completa da API para as 12 telas do React) foi implementada e
está registrada integralmente na **DEC-41** (`01-Decisoes-Arquiteturais.md`), mas esta seção não
recebeu a subseção correspondente na sessão em que foi feita — lacuna do próprio processo de
governança, não reaberta/reconstruída aqui (ver DEC-41 para o detalhamento completo: `BotResponse`
enriquecido, `CatalogoController`/`ConfiguracaoController` novos, equipamento/janela expostos,
auto-reconnect por bot, métricas de processo). Consulte a DEC-41 como fonte de verdade para o que
foi entregue naquela sessão.

## Milestone 42 — EPIC-FRONT-01: Persistência PostgreSQL, Testes de Integração REST, CORS (DEC-42)

Status

CONCLUÍDA — persistência in-memory de Conta/Servidor/Proxy substituída por PostgreSQL via JPA,
primeiros testes de integração da camada REST do projeto, bug de CORS em `ApiKeyFilter` encontrado
e corrigido. Ver **DEC-42** para o detalhamento completo (decisões, consequências, impacto por
camada).

Objetivo

Instrução explícita do responsável: migração funcional C#→Java considerada concluída dentro do
escopo aprovado; preparar o backend para consumo pelo React em produção — substituir persistência
in-memory por PostgreSQL usando as portas já existentes, adapters JPA sem alterar o domínio,
migrations, testes de integração REST, validar serialização/CORS/WebSocket, manter compatibilidade
total da API, sem endpoints novos salvo indispensáveis à persistência. Legado C# deliberadamente
não consultado (mesma justificativa da DEC-40/DEC-41 — persistência/infraestrutura de API não tem
equivalente no C#).

### Entregue

- **Adapters JPA**: `RepositorioDeContasJpa`/`RepositorioDeServidoresJpa`/`RepositorioDeProxiesJpa`
  (`infrastructure.persistence`) substituem os adapters in-memory (removidos) atrás das portas já
  existentes `RepositorioDeContas`/`RepositorioDeServidores` e da porta nova `RepositorioDeProxies`
  (`application.port`, mesmo padrão das irmãs — fecha a lacuna de `PoolDeProxies` nunca ter tido
  persistência própria). 3 `@Entity`/`JpaRepository` novos em `infrastructure.persistence.jpa`; os
  records de domínio continuam sem nenhuma anotação de framework.
- **Schema Flyway**: `db/migration/V1__contas_servidores_proxies.sql` (tabelas `contas`/
  `servidores`/`proxies`, `hibernate.ddl-auto=validate`). Tabela `proxies` sem `UNIQUE` — fiel ao
  `PoolDeProxies` in-memory, que sempre aceitou duplicatas.
- **`PoolDeProxies` com write-through**: carregado do `RepositorioDeProxies` no startup
  (`ConfiguracaoDeExecucao`); `ProxyController` grava no repositório e no cache em memória a cada
  `adicionar`/`remover`, nessa ordem.
- **Credenciais via variável de ambiente** (`ADVANCEDBOT_DB_URL`/`_USER`/`_PASSWORD`, mesmo padrão
  de `advancedbot.api.key`), default apontando para role/banco dedicados (`advancedbot`/
  `advancedbot`@`localhost:5432/advancedbot`) criados nesta sessão num PostgreSQL 18 já instalado na
  máquina.
- **`EntityScan`/`EnableJpaRepositories` explícitos** em `AdvancedBotApplication` — achado técnico:
  `AutoConfigurationPackages` (scanner de `@Entity`/`JpaRepository`) deriva do pacote da própria
  classe `@SpringBootApplication`, não do `scanBasePackages` customizado; sem as anotações
  explícitas o contexto Spring falhava ao subir.
- **6 testes de integração REST novos** (`ContaControllerTest`/`ServidorControllerTest`/
  `ProxyControllerTest`, primeiros testes de Controller HTTP do projeto — lacuna aberta desde a
  DEC-40): `@SpringBootTest`+`MockMvc` contra PostgreSQL real dedicado a testes
  (`advancedbot_test`), isolamento via `@Transactional` (rollback automático).
- **Bug de CORS corrigido em `ApiKeyFilter`**: o filtro barrava toda requisição `OPTIONS`
  (preflight) com 401 antes do CORS do Spring MVC rodar — preflight nunca carrega `X-API-Key` por
  especificação do navegador, então isso quebrava CORS para qualquer requisição não-simples (quase
  toda a API a partir de um browser). `shouldNotFilter` passa a excluir `OPTIONS`.

### Testes

1098→1104 testes automatizados (+6: `ContaControllerTest` 3, `ServidorControllerTest` 2,
`ProxyControllerTest` 1), 0 falhas, 0 erros, 3 skipped deliberadamente.

### Validação executada

`mvn clean test` (JDK 21.0.9, Maven 3.9.7, contra `advancedbot_test`) — BUILD SUCCESS. Validação
manual adicional com `mvn spring-boot:run` real contra o banco de desenvolvimento (`advancedbot`):
Flyway migra o schema do zero no primeiro boot e confirma "up to date" (idempotência) num segundo
boot; CRUD completo de conta/servidor/proxy via `curl` (criar/listar/remover, dados limpos ao
final); preflight CORS testado antes e depois da correção do `ApiKeyFilter` (origem permitida → 200
com `Access-Control-Allow-Origin`; origem não permitida → 403); WebSocket `/ws/events` conectado via
cliente Node.js nativo durante a criação de um bot real.

### Limitações Conhecidas

- `advancedbot_test` precisa existir na mesma instância PostgreSQL local para os testes rodarem —
  não é provisionado automaticamente (Docker/Testcontainers indisponíveis neste ambiente).
- Flyway 9.22.3 (gerenciado pelo Spring Boot 3.2.5) loga aviso de versão não testada contra
  PostgreSQL 18 (funcionou sem erro nesta sessão).

---

## Milestone Frontend 01 — EPIC-FRONT-01 (Frontend): Fundação do projeto React

> **Nota de nomenclatura:** o rótulo `EPIC-FRONT-01` já havia sido usado na Milestone 42 (acima)
> para o trabalho de persistência/CORS do **backend**, feito em preparação para esta fase. Esta
> milestone usa o mesmo rótulo de épico (definido assim pelo responsável do projeto na sessão de
> arquitetura de frontend, ver `docs-reescrita/docs/12-Interface/06-Plano-Construcao-Frontend.md`),
> mas se refere a um trabalho distinto — a fundação do **frontend React**. A partir daqui,
> "EPIC-FRONT-01" sem qualificação se refere a esta milestone (frontend); a Milestone 42 continua
> sendo a referência para o EPIC-FRONT-01 de backend/persistência.

Status

CONCLUÍDA — projeto React criado em `advancedbot-frontend/` (raiz do repositório, irmão de
`advancedbot-java/`), com toda a infraestrutura definida em
`docs-reescrita/docs/12-Interface/06-Plano-Construcao-Frontend.md` (arquitetura congelada em sessão
anterior) e `07-Matriz-Frontend-Backend.md`. Nenhuma tela de negócio implementada — escopo
explicitamente restrito à fundação.

Objetivo

Instrução explícita do responsável: com a arquitetura do frontend já congelada (nenhuma decisão em
aberto), implementar integralmente o EPIC-FRONT-01 — toda a fundação necessária para que os épicos
seguintes implementem apenas funcionalidades (Dashboard, Bots, Proxy, etc., nenhuma delas nesta
sessão). Considerar exclusivamente as 3 fontes já congeladas (backend Java, Governança, Documentação
de Interface); legado C# não consultado.

### Entregue

- **Projeto Vite + React 19 + TypeScript**, em `advancedbot-frontend/`, com aliases de path
  (`@`, `@app`, `@shared`, `@features`) espelhados entre `vite.config.ts` e `tsconfig.app.json`.
- **Tailwind CSS v4** (`@tailwindcss/vite`) com tema claro/escuro por classe `.dark` no `<html>`
  (`@custom-variant dark`), tokens de cor como CSS variables (`--color-*`) com override completo em
  `.dark`, aplicados via `ThemeProvider` a partir de um slice Zustand persistido (`light`/`dark`/
  `system`, com listener de `prefers-color-scheme` apenas no modo `system`).
- **React Router v7** (data router, `createBrowserRouter`), rota raiz `AppShell` com `<Outlet />`;
  rota de fundação (`FoundationPage`) e rota 404 (`NotFoundPage`). Rotas de feature entram por
  EPIC-FRONT futuro.
- **TanStack Query** (`QueryClient` único em `shared/lib/queryClient.ts`) com `QueryCache`/
  `MutationCache.onError` centralizados — toda falha de query/mutation vira `AppError` tipado e
  vira Toast automaticamente, sem boilerplate por feature.
- **Zustand**: `uiStore` (tema, colapso da sidebar, bot selecionado — persistido parcialmente em
  `localStorage`) e `toastStore` (fila de notificações).
- **Axios** (`shared/api/httpClient.ts`): instância única com interceptor de request injetando
  `X-API-Key` e interceptor de response normalizando o `ErrorResponse` do `GlobalExceptionHandler`
  do backend em `AppError{status, errorCode, message, path, isNetworkError}`.
- **mitt** (`shared/api/wsClient.ts`): Event Bus (`wsBus`) com eventos `message`/`status`; classe
  `ManagedSocket` com reconexão automática por backoff exponencial (1s→30s); `connectGlobalEventsSocket`
  (`/ws/events`, singleton, conectado durante toda a sessão via `RealtimeProvider`) e
  `connectBotEventsSocket(botId)` (`/ws/bots/{id}/events`, sob demanda, para hooks de domínio
  futuros). Parser único do envelope `EventoDeBot`, incluindo o formato `{"value": uuid}` de
  `IdentificadorBot` (DEC-40) — nenhum outro ponto do código faz esse parsing.
- **Orval integrado ao OpenAPI real do backend**: `orval.config.ts` aponta para `/v3/api-docs`
  (springdoc, já presente no `pom.xml` do backend) e gera clientes Axios + hooks TanStack Query em
  `shared/api/generated/{endpoints,models}` usando `httpClient` como mutator. Executado com sucesso
  contra o backend real rodando localmente: **13 controllers, 57 paths, 71 arquivos gerados**
  (endpoints + models). Script `npm run generate:api`. Nenhum DTO de REST escrito à mão a partir de
  agora — só tipos de WebSocket (`shared/types/ws.ts`, fora do OpenAPI) e view-models de UI continuam
  manuais.
- **Design System base** (`shared/components`): atômicos `Button` (variantes primary/secondary/
  danger/ghost/icon, estado loading com `Spinner`), `Input`, `NumberInput` (com clamp de min/max),
  `SearchBox` (com debounce), `Card`, `Badge`, `Tooltip`, `Spinner`/`OverlaySpinner`, `Skeleton`/
  `SkeletonLines`, `ThemeToggle`; moleculares `Modal` (portal, Escape para fechar, foco em
  `role="dialog"`), `ConfirmDialog` (variante Danger obrigatória para ações destrutivas),
  `DataTable` genérico (paginação, ordenação, virtualização própria sem dependência externa acima de
  200 linhas, estados loading/error/empty via `Skeleton`/`ErrorState`/`EmptyState`); feedback
  `Toast`/`ToastContainer`, `EmptyState`, `ErrorState`, `ErrorBoundary` (classe React, captura falhas
  de renderização fora do ciclo do TanStack Query); layout `Sidebar` (genérico, recebe `items` por
  propriedade — sem regra de negócio), `TopBar`, `Workspace`, `AppShell`, `PageContainer`,
  `PageHeader`.
- **Hooks genéricos não-domínio**: `useDebouncedValue`, `usePagination`.
- **ESLint** (flat config, `typescript-eslint` + `eslint-plugin-react-hooks` +
  `eslint-plugin-react-refresh` + `eslint-config-prettier`) e **Prettier** configurados; scripts
  `lint`, `format`, `format:check`, `typecheck`, `test`, `test:watch`, `test:ui`, `generate:api`.
- **Vitest + React Testing Library + MSW**: `src/test/setup.ts` (jest-dom, cleanup, ciclo de vida do
  servidor MSW) e `src/test/server.ts` (`setupServer()` sem handlers por padrão — cada feature futura
  registra os seus via `server.use(...)`).

### Testes

19 testes automatizados (Vitest), 0 falhas: `cn` (3), `parseBotId`/`normalizeEventoDeBot` (2),
`Button` (3), `uiStore` (3), `toastStore` (2), `describeAppError` (5), suíte de setup MSW validada
via `beforeAll/afterEach/afterAll`.

### Validação executada

`npx tsc -b --noEmit` sem erros; `npx eslint .` sem erros/avisos; `npx prettier --check .` sem
divergências; `npx vitest run` — 6 arquivos de teste, 19 testes, 0 falhas; `npm run build`
(`tsc -b && vite build`) — sucesso, bundle único (~373 KB / ~120 KB gzip, sem code-splitting de
feature ainda pois não há rotas além da fundação). Validação manual adicional: backend Java subido
localmente (`mvn spring-boot:run`, JDK 21, PostgreSQL 18 já em execução na máquina) para gerar o
OpenAPI real consumido pelo Orval; frontend rodado via `npm run dev` (porta 5173, conforme CORS
default do backend) e inspecionado no navegador — `AppShell` renderiza, tema dark aplicado por
padrão (preferência do sistema), toggle de tema e colapso de sidebar funcionam, canal WebSocket
global conectado sem erros no console, nenhum erro de runtime.

### Limitações Conhecidas

- `npm audit` reporta uma vulnerabilidade alta em `react-router`/`react-router-dom`
  (GHSA-qwww-vcr4-c8h2) presente em toda a série publicada 7.12.0-8.2.0, incluindo a versão instalada
  (7.18.1) — o advisory cobre exclusivamente o modo RSC/framework (server actions), não exercido por
  esta SPA (modo biblioteca puro, sem SSR). Documentado em `06-Plano-Construcao-Frontend.md`,
  decisão travada #5.
- Nenhuma feature de produto implementada (fora de escopo desta milestone, por instrução explícita).
- `features/` ainda não populada — cada EPIC-FRONT seguinte cria sua própria subpasta conforme
  `07-Matriz-Frontend-Backend.md`.

---

## Milestone Frontend 02 — EPIC-FRONT-02: Feature Dashboard

Status

CONCLUÍDA — primeira feature funcional do frontend. `features/dashboard/` completa
(components/hooks/services/pages/tests), consumindo exclusivamente hooks gerados pelo Orval,
validada contra o backend Java real (métricas reais, criação/start de bot refletido na tela sem
F5). Dashboard virou a rota raiz (`/`), substituindo a `FoundationPage` (removida — deixaria de ser
usada e de ser código morto).

Objetivo

Instrução explícita do responsável: com a fundação (EPIC-FRONT-01) já validada, implementar
integralmente o EPIC-FRONT-02 — a feature Dashboard completa, sem interrupção por etapas,
consumindo exclusivamente hooks/DTOs gerados pelo Orval, nunca chamada Axios manual dentro da
feature. Fontes oficiais: os dois documentos de arquitetura do frontend, backend Java, OpenAPI do
SpringDoc, hooks/DTOs do Orval — legado C# fora de escopo.

### Achado técnico antes da implementação (bloqueio real, resolvido nesta sessão)

Ao consumir o primeiro hook gerado de verdade (`useMetricas`), a tela não teria funcionado: o
`orval.config.ts` da Milestone Frontend 01 tinha `override.query: { useQuery: true, useMutation:
true, signal: true }`. Com os dois booleanos simultâneos, o Orval 8.23 inverte a geração — todo
endpoint `GET` (`metricas`, `listar`/`listar1`/`listar2` de contas/servidores/proxies) foi gerado
como `useMutation`, e todo `POST`/`PUT`/`DELETE` como `useQuery`. Esse defeito não tinha aparecido
na Milestone Frontend 01 porque nenhuma tela de negócio consumia os hooks gerados ainda. Corrigido
para `{ signal: true }` (deixa o Orval decidir pelo verbo HTTP, comportamento padrão e correto);
`shared/api/generated` regenerado contra o backend real após a correção — confirmado que `GET`
virou `useQuery` e `POST`/`PUT`/`DELETE` viraram `useMutation` em todos os 13 controllers. Corrigido
o gerador, nunca os arquivos gerados (continuam nunca editados à mão). Nenhuma decisão arquitetural
mudou — Orval continua sendo a ferramenta decidida; era um bug de configuração, não uma escolha.

### Entregue

- **`features/dashboard/services/formatters.ts`**: funções puras de formatação (CPU, memória,
  uptime, TPS, duração de tick, contagem) — nenhuma chamada HTTP.
- **`features/dashboard/services/deriveMetrics.ts`**: derivação pura de breakdown por estado de
  execução/sessão a partir de `MetricasResponse`, com rótulos em português para os estados
  conhecidos e fallback para a chave crua em estados desconhecidos (não assume que só existem os 3
  valores hoje observados).
- **`features/dashboard/hooks/useDashboardMetrics`**: wrap de `useMetricas` (gerado) com
  `refetchInterval` de 5s (fallback de polling, decisão travada #16/#17 — CPU/memória/tick não têm
  evento WS dedicado) e invalidação imediata da query ao receber `tipo: "estado"` no canal
  `/ws/events` (via `wsBus`).
- **`features/dashboard/hooks/useDashboardTotals`**: wrap de `useListar2`/`useListar`/`useListar1`
  (contas/servidores/proxies, todos gerados) com `staleTime` de 5 minutos (grupo "estático" da
  decisão travada #16 — nenhuma das três tem evento WS).
- **`features/dashboard/hooks/useMetricsHistory`**: acumula em memória (máx. 30 amostras) os
  valores reais de CPU/TPS a cada atualização da query de métricas, para alimentar os gráficos de
  tendência — não existe endpoint de série histórica no backend, então o gráfico é a tendência
  recente observada pelo próprio cliente, nunca dado inventado ou interpolado.
- **Componentes**: `MetricCard` (12 instâncias na tela), `StateBreakdownCard` (barra de distribuição
  por estado, CSS puro), `MetricTrendChart` (sparkline SVG de CPU/TPS, sem lib de gráfico nova),
  `DashboardGapNotice` (registro visível na própria tela dos 3 GAPs abaixo).
- **`DashboardPage`**: loading (skeleton de página inteira), error (`ErrorState` com retry),
  conteúdo com 12 `MetricCard`, 2 `StateBreakdownCard`, 2 `MetricTrendChart`, `DashboardGapNotice`.
  Grid responsivo (1/2/4 colunas conforme breakpoint), tema claro/escuro herdado do `ThemeProvider`
  já existente.
- **GAPs registrados (não simulados)**: Threads da JVM (não exposto por `MetricasController`);
  Proxy em uso (sem campo de associação a bot em `ProxyResponse`, só total cadastrado);
  Heap vs. memória não-heap (backend só expõe heap via `Runtime` — "Memória JVM" e "Heap" pedidos
  como métricas separadas colapsam num único card real).
- **Router/Sidebar**: `DashboardPage` é agora a rota `index` (`/`); `FoundationPage` removida
  (código morto após a substituição); `AppShell` ganha item de navegação "Dashboard".

### Testes

44 testes automatizados (19→44, +25 desta milestone), 0 falhas: `formatters` (13), `deriveMetrics`
(6), `MetricCard` (2), `DashboardPage` (3 — caminho feliz com dados reais mockados via MSW,
`ErrorState` em falha de rede, `EmptyState` na distribuição quando não há bots).

### Validação executada

`npx tsc -b --noEmit` sem erros; `npx eslint .` sem erros/avisos; `npx prettier --check .` sem
divergências; `npx vitest run` — 10 arquivos de teste, 44 testes, 0 falhas; `npm run build` sucesso.
Validação manual contra o backend Java real (`mvn spring-boot:run`, JDK 21, PostgreSQL 18 local): (1)
Dashboard vazio — cards zerados, `EmptyState` na distribuição, gráficos em "coletando amostras",
CPU/memória/uptime reais da JVM; (2) criado servidor/conta/bot reais via `curl` e chamado
`POST /bots/{id}/start` — a tela atualizou sozinha (sem F5): "Bots totais" 0→1, "Bots executando"
→1, distribuição por estado passou a mostrar "Executando: 1"/"Desconectado: 1", confirmando a
invalidação por evento WS `"estado"`; (3) dados de teste removidos do banco de desenvolvimento ao
final; (4) tema escuro alternado manualmente (classe `.dark` aplicada corretamente); (5) viewport
375px (mobile) sem overflow horizontal (`scrollWidth === clientWidth`); (6) console do navegador
sem erros em nenhum dos passos.

### Limitações Conhecidas

- Totais de Contas/Servidores/Proxies só atualizam por `staleTime` (5 min) ou remontagem da página
  — decisão consciente (nenhum dos três endpoints tem evento WS), não bug.
- `"Bots conectados"/"executando"/"pausados"/"desconectados"` mostram "—" (não "0") quando o backend
  retorna o mapa de estado sem aquela chave presente (nenhum bot naquele estado) — distinção
  deliberada entre "valor zero" e "chave ausente", não simula um zero que o backend não afirmou.

---

## Milestone Frontend 03 — EPIC-FRONT-03: Feature Proxy

Status

CONCLUÍDA — segunda feature funcional do frontend. `features/proxy/` completa
(components/hooks/services/pages/tests): listagem, busca e paginação client-side, criação, edição
e exclusão com confirmação. Pronta para uso em produção. Validada contra o backend Java já em
execução (IntelliJ, não iniciado por esta sessão).

Objetivo

Instrução explícita do responsável: com a arquitetura já congelada e a fundação/Dashboard já
validados, implementar integralmente o EPIC-FRONT-03 — a feature Proxy completa, usando
exclusivamente hooks gerados pelo Orval, sem Axios manual, sem DTO manual, sem endpoint novo,
reaproveitando o Design System da Fase 0. Backend já rodava localmente (IntelliJ) durante todo o
desenvolvimento.

### Entregue

- **`shared/components/atoms/Select.tsx`** (novo): único componente novo do Design System nesta
  sessão — já especificado em `03-Design-System.md` §2 desde a Fase 0, implementado agora porque o
  campo `tipo` do formulário de Proxy (enum fechado `HTTP`/`SOCKS4`/`SOCKS5`, `TipoDeProxy.java`)
  precisava de um select de verdade. Completa o catálogo já congelado, não é uma decisão nova.
- **`features/proxy/services/proxyValidation.ts`**: validação pura (host obrigatório, porta
  1-65535, tipo dentre os 3 valores reais do enum do backend) — sem chamada HTTP.
- **`features/proxy/services/filterProxies.ts`**: busca e paginação client-side puras — o backend
  não tem parâmetro de busca nem paginação em `GET /api/v1/proxies` (retorna array cru).
- **`features/proxy/hooks/useProxyList`**: wrap de `useListar1` (gerado), `staleTime` 5min (grupo
  estático, sem WS).
- **`features/proxy/hooks/useProxyTableState`**: estado de UI local (busca + página), não pertence
  ao servidor.
- **`features/proxy/hooks/useProxyMutations`**: `useCreateProxy`/`useDeleteProxy` (wrap de
  `useAdicionar`/`useRemover1` gerados + invalidação de `getListar1QueryKey()` + toast de sucesso);
  `useUpdateProxy` (mutation composta client-side chamando as funções geradas `remover1` e
  `adicionar` em sequência — ver limitação de backend abaixo).
- **Componentes**: `ProxyTable` (especialização de `DataTable`, sem coluna de latência — GAP já
  conhecido), `ProxyFormModal` (único modal para criar/editar, reaproveita `Modal`/`ConfirmDialog`
  do Design System).
- **`ProxyPage`**: `PageHeader` com ação "Nova proxy", `SearchBox`, `ProxyTable` com paginação,
  `ProxyFormModal`, `ConfirmDialog` de exclusão. Loading/error/empty tratados pelo `DataTable`
  (reuso, sem duplicar lógica).
- **Rota `/proxy`** lazy-loaded (`React.lazy`/`Suspense`, decisão travada #5 — primeira rota da
  aplicação a usar code-splitting de fato); item "Proxy" adicionado à `Sidebar`.
- **Correção incidental no tratamento global de erros** (`shared/lib/queryClient.ts`): o
  `MutationCache.onError` global só sabia formatar `AppError` (vindo do `httpClient`); qualquer
  outro `Error` (como o lançado por `useUpdateProxy` quando a remoção funciona mas a criação falha)
  virava um toast genérico "Erro inesperado", perdendo a mensagem específica. Corrigido para
  preservar `error.message` quando não for `AppError` — melhoria genérica que beneficia qualquer
  mutation composta futura, não só Proxy.

### Achado técnico (limitação real do backend, documentada ao final conforme instrução)

`ProxyController.java` não tem `PUT` — comentário no próprio código confirma que é deliberado:
proxy não tem identidade própria por entrada, host+port+tipo já formam chave natural suficiente
para o CRUD original (criar/listar/remover). "Editar" no frontend é composto client-side (remover a
entrada original + adicionar a nova), usando as mesmas funções geradas pelo Orval — nenhum endpoint
novo foi criado. Duas consequências documentadas na UI/matriz: (1) se a remoção for bem-sucedida e a
criação falhar, a proxy original já foi perdida — `useUpdateProxy` lança um erro com mensagem
específica avisando o operador a recadastrar manualmente, em vez de um erro genérico; (2) o backend
permite duplicatas exatas (sem `UNIQUE`) e `remover()` remove só a primeira ocorrência
(`CopyOnWriteArrayList.remove`/`findFirst()`) — com proxies duplicadas na lista, editar/excluir uma
linha específica pode afetar a outra ocorrência idêntica, não necessariamente a linha clicada. Não é
um bug do frontend: é uma limitação de identidade do modelo de dados do backend, herdada
integralmente pela UI.

### Testes

64 testes automatizados (44→64, +20 desta milestone), 0 falhas: `proxyValidation` (4),
`filterProxies` (6), `ProxyFormModal` (3), `ProxyTable` (2), `ProxyPage` (5 — listagem, empty state,
error state com retry, criação end-to-end via MSW, exclusão com confirmação end-to-end via MSW).

### Validação executada

`npx tsc -b --noEmit` sem erros; `npx eslint .` sem erros (1 aviso pré-existente de
`react-refresh/only-export-components` em `router.tsx`, não bloqueia build); `npx prettier --check .`
sem divergências; `npx vitest run` — 15 arquivos de teste, 64 testes, 0 falhas; `npm run build`
sucesso, `ProxyPage` confirmado como chunk separado (`ProxyPage-*.js`, ~15 KB/~5 KB gzip),
confirmando o code-splitting por rota. Validação manual contra o backend Java já em execução
(IntelliJ, JDK 21, PostgreSQL 18 local, não iniciado por esta sessão): (1) lista vazia — `EmptyState`
correto; (2) criada proxy real via UI (`198.51.100.10:1080 SOCKS5`) — apareceu na tabela e
confirmada via `curl` direto no backend; (3) editada a mesma proxy (porta 1080→1081) via UI — `curl`
confirmou uma única entrada atualizada, sem duplicata (remover+adicionar funcionou corretamente);
(4) excluída via `ConfirmDialog` — tabela voltou ao `EmptyState`, `curl` confirmou lista vazia no
backend; (5) viewport mobile (375px) sem overflow horizontal; (6) console do navegador sem erros em
nenhum passo (um erro de HMR "`RouteFallback is not defined`" observado durante a sessão era
resíduo de um refactor local de nomes ainda em andamento — não presente no código final; reiniciar o
servidor de desenvolvimento e recarregar a página confirmou console limpo).

### Limitações Conhecidas

- Sem `PUT /api/v1/proxies` no backend — "editar" é uma composição client-side de dois endpoints
  reais (ver achado técnico acima), não uma limitação de frontend.
- Duplicatas exatas de proxy (mesmo host+port+tipo) são indistinguíveis na tabela e editar/excluir
  uma delas pode afetar a outra ocorrência — herdado do modelo de dados do backend (sem `UNIQUE`),
  não introduzido pelo frontend.
- Sem coluna de latência/ping na `ProxyTable` — `POST /api/v1/proxies/check` não existe (GAP já
  registrado desde a análise inicial de arquitetura).

---

## Milestone Frontend 05 — EPIC-FRONT-05: Feature Bots

Status

CONCLUÍDA — núcleo da Fase 2 do roadmap (`06-Plano-Construcao-Frontend.md`). `features/bots/`
completa (components/hooks/services/pages/tests): listagem com busca+paginação client-side, criação,
exclusão, todas as ações de execução/sessão, troca de proxy, auto-reconnect, ações em lote e
atualização em tempo real via WebSocket. Sem "editar" — GAP real de backend, documentado abaixo, não
contornado com hack. Validada contra o backend Java já em execução (IntelliJ, não iniciado por esta
sessão).

Objetivo

Instrução explícita do responsável: seguir exatamente o roadmap já congelado, sem reabrir decisões de
arquitetura nem reconsultar governança — implementar integralmente o EPIC-FRONT-05 (Feature Bots)
usando exclusivamente hooks gerados pelo Orval, sem Axios manual, reaproveitando ao máximo o Design
System e os padrões já validados em Proxy/Contas-Servidores/Catálogo/Configurações, promovendo para
`shared` qualquer lógica repetida assim que identificada (sem esperar uma 3ª ocorrência).

### Entregue

- **Promovido para `shared` (2º consumidor real, promoção imediata)**: `shared/types/proxy.ts`
  (`TIPOS_DE_PROXY`/`TipoDeProxy`, antes só em `features/proxy/services/proxyValidation.ts`) e
  `shared/lib/proxyFormValidation.ts` (`validateProxyForm`/`isProxyFormValid`/`ProxyFormValues`,
  antes só em `features/proxy/services/proxyValidation.ts`) — a troca de proxy de um bot
  (`ProxyBotRequest`) usa exatamente o mesmo formato host+porta+tipo de `ProxyRequest`. `features/
  proxy/services/proxyValidation.ts` mantido como reexport, nenhum import existente quebrou.
- **`features/bots/services/botState.ts`**: `EstadoDeExecucao`/`EstadoDeSessao` tipados manualmente
  como view-model de UI (valores reais confirmados em `EstadoExecucao.java`/`EstadoSessao.java` do
  backend — `PARADO`/`EXECUTANDO`/`PAUSADO` e `DISCONNECTED`/`CONNECTING`/`CONNECTED`; o OpenAPI
  expõe `estadoExecucao`/`estadoSessao` como `string` cru, sem enum gerado).
- **`features/bots/services/botValidation.ts`**: validação pura do formulário de criação —
  `POST /api/v1/bots` aceita conta por `contaId` OU credenciais inline, e servidor por `servidorId`
  OU host+porta inline (confirmado em `BotController.resolverCredenciais`/`resolverEndereco`); a
  validação reflete essa regra real (uma das duas formas obrigatória por campo), sem inventar
  restrição adicional (backend não tem bean validation em `CriarBotRequest`).
- **`features/bots/services/filterBots.ts`**: busca client-side (username/host/porta/estado) — sem
  parâmetro de busca em `GET /api/v1/bots`, mesmo padrão de Proxy/Contas/Servidores.
- **`features/bots/hooks/useBotList`**: wrap de `useListar3` (`offset=0&limit=500`, paginação
  client-side sobre o lote completo, mesma decisão de Contas/Servidores), `staleTime` 10s (mais baixo
  que o grupo "estático" porque bots têm evento WS `"estado"` real) + assinatura de `wsBus` que
  invalida `getListar3QueryKey()` a cada evento `"estado"` do canal global.
- **`features/bots/hooks/useBotTableState`**: estado de UI local (busca + página).
- **`features/bots/hooks/useBotMutations`**: `useCreateBot`/`useDeleteBot` (wrap de
  `useCriar2`/`useRemover3` + invalidação + toast) — **sem `useUpdateBot`**, GAP documentado abaixo.
- **`features/bots/hooks/useBotActions`**: uma mutation por ação
  (iniciar/parar/pausar/retomar/conectar/desconectar/reconectar/trocarProxy/definirAutoReconnect),
  todas via hooks gerados, cada uma invalidando a lista + toast de sucesso; `useBatchBotAction`
  compõe ações em lote com `Promise.allSettled` sobre as funções cruas geradas (`iniciar`, `parar`
  etc.) — GAP de backend (sem endpoint de lote), decisão travada #12 do doc 06.
- **Componentes**: `BotStatusBadge` (dois badges independentes — execução e sessão são eixos
  ortogonais no backend, não um estado combinado inventado), `BotTable` (especialização de
  `DataTable`, checkbox de seleção + botões de ação contextuais por estado — evita oferecer uma ação
  inválida como "Pausar" num bot já `PARADO` em vez de deixar o backend rejeitar com
  `IllegalStateException`), `BotFormModal` (criação com `Tabs` — reuso do componente promovido em
  EPIC-FRONT-04 — para alternar conta/servidor existente vs. inline), `BotProxyModal` (troca de
  proxy, reaproveita a validação promovida para `shared`).
- **`BotsPage`**: `PageHeader` com ação "Novo bot", `SearchBox`, barra de ações em lote (aparece só
  com seleção ativa), `BotTable` com paginação, `BotFormModal`, `BotProxyModal`, `ConfirmDialog` de
  exclusão. Loading/error/empty tratados pelo `DataTable` (reuso, sem duplicar lógica).
- **Rota `/bots`** lazy-loaded, item "Bots" adicionado à `Sidebar` (logo após Dashboard, refletindo a
  ordem do roadmap); badge de versão do `TopBar` atualizado para `EPIC-FRONT-05`.

### GAP de backend confirmado (documentado, não implementado como solução de contorno)

Sem "editar" bot — e, diferente de Proxy/Conta/Servidor, a composição client-side remover+criar não
é uma opção segura aqui: `BotResponse` nunca devolve a senha usada na criação (`CriarBotRequest.
password` é write-only), então recriar o bot exigiria pedir a senha de novo ao operador de qualquer
forma — não há ganho real em fingir uma "edição" que na prática seria "excluir e criar de novo com
os mesmos dados que o operador já teria que redigitar". A Feature Bots oferece só criar e excluir,
documentado em `07-Matriz-Frontend-Backend.md`. Ações em lote continuam sendo N chamadas paralelas ao
endpoint individual (`connect-batch` e equivalentes não existem no backend, GAP §8 do doc 06) — se
uma falhar, o toast reporta quantas tiveram sucesso e quantas falharam, sem interromper as demais.

### Testes

Suite completa do frontend: 25 arquivos de teste, 108 testes, 0 falhas (adicionado `features/bots/
tests/BotsPage.test.tsx` com 6 casos via MSW — listagem, empty state, criação com credenciais e
host/porta manuais, ação individual de iniciar com atualização de estado, exclusão com confirmação,
ação em lote sobre bots selecionados).

### Validação executada

`npx tsc -b --noEmit` sem erros; `npx eslint .` sem erros (5 avisos pré-existentes de
`react-refresh/only-export-components` em `router.tsx`, mesma categoria já registrada, não
bloqueantes); `npx prettier --check .` sem divergências após `--write` (17 arquivos reformatados,
a maioria pré-existente de sessões anteriores que nunca haviam rodado o formatter mais recente,
nenhuma mudança de conteúdo/lógica); `npx vitest run` — 25 arquivos, 108 testes, 0 falhas; `npm run
build` sucesso, `BotsPage` confirmado como chunk separado (`BotsPage-*.js`, ~19 KB/~5 KB gzip).
Validação manual contra o backend Java já em execução (IntelliJ, JDK 21, PostgreSQL local, não
iniciado por esta sessão), via Browser pane: (1) lista vazia — `EmptyState` correto; (2) criado bot
real via UI com credenciais manuais + host/porta manual (`manualbot@127.0.0.1:25565`) — apareceu na
tabela com estado `Parado`/`Desconectado`; (3) ação "Iniciar" real — estado mudou para `Executando`
(badge verde), botões contextuais trocaram para Pausar/Parar; (4) troca de proxy real via
`BotProxyModal` (`10.0.0.5:1080 SOCKS5`) — badge de proxy atualizada na tabela; (5) exclusão via
`ConfirmDialog` — tabela voltou ao `EmptyState`; console do navegador sem erros em nenhum passo.

### Limitações Conhecidas

- Sem "editar" bot — `BotResponse` não devolve a senha usada na criação, recompor via remover+criar
  não é seguro (ver GAP acima). Diferente das limitações de Proxy/Conta/Servidor, não é uma questão
  de identidade da entidade, é a ausência estrutural do dado necessário para recriar com segurança.
- Sem endpoint de lote nativo (`connect-batch` e equivalentes) — ações em lote são N chamadas
  paralelas client-side via `Promise.allSettled`, GAP já registrado desde a análise inicial.
- Sem evento WS dedicado para mudança de proxy/auto-reconnect — cobertos pela invalidação explícita
  de cada mutation (`onSuccess`), não pelo evento `"estado"` (que cobre só execução/sessão).

---

## Milestone Frontend 06 — EPIC-FRONT-06: Bot Details

Status

CONCLUÍDA — núcleo restante da Fase 2 do roadmap (`06-Plano-Construcao-Frontend.md`). `features/bots/
details/` completa: 5 sub-abas roteadas sob `/bots/:id` (Console, Ações, Inventário, Mundo, Macros),
todas via hooks gerados pelo Orval, canal WS por bot conectado pela primeira vez nesta sessão.
Validada contra o backend Java já em execução (IntelliJ, não iniciado por esta sessão) — 3 GAPs reais
confirmados manualmente e documentados abaixo, nenhum contornado com hack.

Objetivo

Instrução explícita do responsável: continuar exatamente do ponto anterior, arquitetura congelada,
sem reabrir decisões nem reconsultar o legado — implementar integralmente o EPIC-FRONT-06 (Bot
Details) em uma única sessão, cobrindo layout, navegação por abas, estado compartilhado do bot
selecionado, Console/logs em tempo real/eventos WS, Inventário/Equipamento/Containers, Mundo/Chunk/
Jogadores/Entidades, Macros do bot, painel de ações (movimento/interação), com Loading/Skeleton/
Empty/Error em todas as telas, promovendo para `shared` qualquer componente/hook/utilitário que
ganhasse um 2º consumidor real durante a implementação (sem esperar um 3º).

### Entregue

- **Promovido para `shared` (2º consumidor real)**: nenhuma nova promoção nesta sessão além das já
  feitas no EPIC-FRONT-05 — `ComandoCatalogoResponse`/`useCatalogoMacros` (Catálogo, EPIC-FRONT-04)
  ganhou seu 2º consumidor real (Macros do bot cruza a lista de macros ativas com o catálogo), mas já
  vivia em `shared`/`features/catalogo` desde antes, nenhuma movimentação de código foi necessária.
- **`features/bots/details/services/`**: `botState` reaproveitado de `features/bots/` (execução/
  sessão); `itemStack.ts` (formatação de `ItemStackDto`, `count` é `string` no OpenAPI gerado, não
  numérico); `botDetailsNav.ts` (lista de abas + helper `botDetailsPath`, único ponto de verdade da
  navegação entre sub-abas).
- **`features/bots/details/hooks/`**: `useBotDetail` (estado compartilhado do bot selecionado, wrap
  de `useDetalhar2` + invalidação no evento WS global `"estado"` filtrado pelo `botId`),
  `useBotEventsSocket` (abre `connectBotEventsSocket(id)` uma única vez no layout, mantido vivo entre
  troca de abas — 1º consumidor real do canal per-bot, que só existia como infraestrutura desde a
  Fundação), `useRealtimeLogs` (busca inicial + buffer WS `"log"` concatenados na leitura, sem
  sincronizar `query.data` para `useState` a cada render — evita o anti-padrão de estado derivado),
  `useBotConsole` (chat + comando, histórico duplo REST+WS), `useInventario`/`useInventarioActions`
  (refetch no foco, 10 mutations de slot), `useEstadoMundo`/`useMundoEntidades` (polling leve, sem
  push), `useBotActionsPanel` (14 mutations de movimento/interação), `useBotMacros` (macros ativas +
  ativar/desativar, invalida também `getDetalhar2QueryKey(id)` porque `BotResponse.macrosAtivas`
  espelha a mesma lista usada por `BotTable`).
- **Componentes novos** (`features/bots/details/components/`, só 1 consumidor até agora — não
  promovidos): `BotDetailsHeader`, `ConsoleLogViewer` (auto-scroll, Loading/Empty/Error próprios),
  `ItemSlot`/`InventorySlotGrid` (sem componente de grade no Design System ainda — `DataTable` é
  orientado a linhas).
- **Páginas** (`features/bots/details/pages/`): `BotDetailsPage` (layout — cabeçalho + `Tabs` +
  `<Outlet/>`, conecta o WS por bot), `ConsolePage`, `AcoesPage`, `InventarioPage`, `MundoPage`,
  `MacrosPage`.
- **Rotas aninhadas `/bots/:id/{console,acoes,inventario,mundo,macros}`** (decisão travada #5 do doc
  06), índice redireciona para `console`; `BotTable` da listagem ganhou link de navegação
  (`username` → `botDetailsPath(id)`).

### GAPs de backend confirmados (validados manualmente contra o backend real, não contornados)

1. **Sessão PLAY exigida**: `GET /bots/{id}/estado`, todo `/mundo/*` e todo `/inventario/*` respondem
   `409 Conflito de estado — "Bot não está em uma sessão de jogo ativa (PLAY)"` quando o bot não está
   conectado a um servidor Minecraft real. Confirmado criando um bot e nunca conectando-o — as 3 telas
   (Mundo, Inventário, e o `estado` do cabeçalho) mostraram `ErrorState`+toast corretamente, sem dado
   simulado. Comportamento correto do backend, não uma falha do frontend.
2. **`MacroResponse.tipo` não corresponde a `nome`/`aliases` do Catálogo**: ativar a macro `antiafk`
   (via alias do catálogo) e reconsultar `GET /bots/{id}/macros` devolveu `{"tipo":"TarefaAntiAFK"}` —
   um valor derivado internamente pelo backend (provável nome de classe Java), não o alias original
   usado para ativar. A tela usa `tipo` para descrever a macro (caindo para o valor cru quando não há
   correspondência no catálogo) e para decidir quais macros do catálogo já estão ativas (comparação
   que também falha nesse caso). Sem alternativa no frontend — o backend não devolve o alias original
   em `MacroResponse`.
3. **`DELETE /bots/{id}/macros/{alias}` sem efeito observável com bot desconectado**: usando o
   próprio `tipo` (`TarefaAntiAFK`) como `alias` — única opção disponível dado o GAP #2 — a chamada
   respondeu `200 OK` duas vezes em teste manual, mas `GET /bots/{id}/macros` continuou devolvendo a
   mesma macro ativa depois. Confirmado via `read_network_requests` na Browser pane (corpo da resposta
   do `GET` idêntico antes/depois do `DELETE` bem-sucedido). Não reproduzido com bot conectado — pode
   ser comportamento esperado do backend (macro só é efetivamente removida com sessão ativa), mas o
   frontend não tem como diferenciar isso de uma falha silenciosa sem mais contexto do backend.

Nenhum dos 3 GAPs foi contornado com heurística cliente-side — documentados aqui e em
`07-Matriz-Frontend-Backend.md` para o responsável do backend avaliar.

### Testes

Suite completa do frontend: 26 arquivos de teste, 116 testes, 0 falhas (adicionado
`features/bots/details/tests/BotDetailsPage.test.tsx` com 8 casos via MSW cobrindo as 5 sub-abas —
cabeçalho, navegação por `Tabs`, Console (chat+comando+limpar logs), Ações (mover), Inventário
(grid+janela vazia), Mundo (estado+consulta de bloco+listas vazias), Macros (ativar)). Corrigido
durante a sessão: `BotsPage.test.tsx` (EPIC-FRONT-05) não tinha `MemoryRouter` — quebrou quando
`BotTable` ganhou o link de navegação (`react-router-dom`'s `<Link>` exige contexto de rota); ajustado
para `MemoryRouter`, sem mudar nenhuma asserção de negócio.

### Validação executada

`npx tsc -b --noEmit` sem erros; `npx eslint .` sem erros (11 avisos pré-existentes de
`react-refresh/only-export-components` em `router.tsx`, mesma categoria já registrada — cresceu de 5
para 11 só porque o arquivo ganhou mais `const` de rota lazy, não é regressão de qualidade); `npx
prettier --check .` sem divergências; `npx vitest run` — 26 arquivos, 116 testes, 0 falhas; `npm run
build` sucesso, `BotDetailsPage`/`ConsolePage`/`AcoesPage`/`InventarioPage`/`MundoPage`/`MacrosPage`
confirmados como chunks separados. Validação manual contra o backend Java já em execução (IntelliJ,
JDK 21, PostgreSQL local, não iniciado por esta sessão), via Browser pane: (1) criado bot real
(`detailsbot@127.0.0.1:25565`) e navegado até `/bots/{id}/console` pelo link da listagem; (2)
executado o comando real `help` — resposta `SUCESSO` + saída completa da lista de comandos chegou via
evento WS `"log"` em tempo real, confirmando o canal `/ws/bots/{id}/events` funcional de ponta a
ponta; (3) aba Mundo/Inventário — `409` real do backend corretamente exibido como `ErrorState`+toast
(bot nunca conectado a um servidor); (4) aba Macros — catálogo real carregado (`Follow`, `AntiAFK`,
`Herbalismo`, `Miner`, `Mob`, `Pesca`, `DropAll`, `Twerk`), macro `antiafk` ativada com sucesso
(GAPs #2/#3 acima descobertos exatamente nesta etapa); (5) aba Ações renderizou todos os controles
(Movimento/Câmera/Postura/Bloco alvo/Entidade alvo/Usar item) sem erro; (6) bot de teste excluído ao
final via `ConfirmDialog`; console do navegador sem erros em nenhum passo.

### Limitações Conhecidas

- Ver os 3 GAPs de backend confirmados acima — nenhum é uma limitação de frontend, todos exigem
  decisão/ajuste do responsável pelo backend.
- Duplicidade rara possível no histórico de comandos do Console (`useExecutar` via REST + eco do
  mesmo comando via WS `"comando"`) — aceita como limitação conhecida, documentada no código.
- `ConsoleLogViewer`/`ItemSlot`/`InventorySlotGrid` nascem dentro da feature (só 1 consumidor até
  agora) — candidatos a promoção para `shared/components` no dia em que outra feature (ex. Viewer 3D
  REST-only, Fase 4) precisar de grade de itens ou visualizador de log.

---

## Roadmap Definitivo Pós-Milestone 39 (Pivô de Estratégia: Épicos de Fechamento)

Instrução explícita do responsável (sessão de 2026-07-25, após a Milestone 39): parar de escolher trabalho macro por macro ou domínio por domínio; usar todo o conhecimento já acumulado nesta sessão (sem nova auditoria do legado) para produzir a matriz final de domínios e um roadmap de poucos épicos grandes, cada um fechando um domínio inteiro. Levantamento do legado considerado encerrado a partir daqui — sessões futuras executam um épico deste roadmap sem redescoberta.

### Matriz Final de Domínios

| Domínio | Já existe | Falta | Generalizável | Macros que passam a reutilizar | Prioridade |
|---|---|---|---|---|---|
| **Foundation API (meta)** | `TarefaContinua`/`MotorDeTick`/`Bot.registrarTarefa`, `GerenciadorDeComandos`/`Comando`, `MacroUtils` (8 métodos), `AbridorDeBau` | — | — | todas | **CONCLUÍDO** |
| **Mundo** | `Mundo`/`Bloco`/`tracarRaio`, `RegistroDeBlocos`, `RegistroDePortais`/`localizarPortal` | — | — | todas | **CONCLUÍDO** |
| **Entidades/Combate (Cat. 2)** | `EntidadesDoMundo` (`porId`/`porUuid`/`mobMaisProximo`), `interagirComEntidade`/`UseEntityPacket`, `olharParaEntidade`, `jogadorVisivelMaisProximo` | — | — | Mob, UseBow, futuras | **CONCLUÍDO** (Categoria 1/KillAura permanece excluída por política, DEC-38 — não é gap) |
| **Inventário** | `InventarioDoJogador`/`JanelaDeSlots`, `clicarSlot`/`shiftClique`/`trocarSlots`/`pegarItem`/`largarItem`/`moverItem`, `selecionarSlotDaHotbar`, `MacroUtils.selecionarSlotComItem`/`selecionarMelhorFerramenta`/`itemNaMao` | — | — | todas | **CONCLUÍDO** |
| **Containers** (baú/fornalha/brewing/anvil/beacon/hopper/dispenser) | `Janela`/`JanelaDeSlots`, `abrirJanela`/`fecharJanela`/`confirmarTransacao`, `AbridorDeBau` | — | — | AutoFish, Mob, futuras | **CONCLUÍDO** (genérico por `tipo` string, DEC-37) |
| **Itens/NBT** | `TagNBT`/`NbtCodec`, `ItemStack.nbt()` | leitura de regra específica (ex. "tem Flame?") é composição, não infra | — | futuras (composição) | **CONCLUÍDO** (infra) |
| **Efeitos** | `SessaoDeJogo.aplicarEfeito`/`removerEfeito`/`efeito` (próprio bot), mirror em `EntidadeRemota` | resolver `amplifierCeleridade`/`amplifierFadiga` (DEC-28) é composição futura sobre `efeito(id)`, não infra nova | — | Mineração (bônus de haste), futuras | **CONCLUÍDO** (infra) |
| **Chat** | `enviarMensagem`, `ultimaMensagemDeChat`, `ParserDeChatComponent`/`textoPlanoDaUltimaMensagemDeChat` | resolução de `translate` para texto de sistema (sem consumidor real) | — | futuras macros reativas a texto | **CONCLUÍDO** (infra; item sem consumidor fica de fora por design) |
| **Movimento** | `mover`/`olhar`/`moverEOlhar`/`pular`/`aplicarFisica` (vertical+horizontal) | `MoveQueue`/`Movement`-por-yaw | não se aplica — **decisão já tomada (DEC-36), não reabrir** | `CommandMove`/`CommandRetard` permanecem bloqueados por política, não por lacuna | **FORA DE ESCOPO** (DEC-36) |
| **Pathfinding** | `BuscadorDeCaminho`/`GuiaDeCaminho`/`criarCaminhoPara` | — | — | Goto, Follow, Portal, futuras | **CONCLUÍDO** |
| **Equipamentos** | `EntidadeRemota.equipamento` (mirror de outros) | bot nunca veste armadura própria no legado — sem gap real | — | — | **CONCLUÍDO** (sem necessidade) |
| **Economia/NPC** | `MacroUtils.clicarPlacaAgachando` (placa de compra/venda), Container genérico (cobriria trade real se algum dia usado) | Trade List (`0x30` CB, trade real de Villager) — **zero consumidor no legado** (só placas) | — | — | **CONCLUÍDO** (sem necessidade comprovada) |
| **Interface com Servidor** | `enviarMensagem` (comandos de chat/teleporte), ciclo de vida (DEC-19/30/31/33) | — | — | Mob, AutoFish, MobTeleport | **CONCLUÍDO** |
| **Scripting/Orquestração** | `GerenciadorDeComandos.executar` (dispatch externo por texto) | **um `Comando` não tem como invocar outro `Comando` registrado por alias/args programaticamente** (`CommandScript` do legado orquestra `$comando1; $comando2...`) | sim — é o único domínio com capacidade de composição genuinamente ausente | `CommandScript` (Fase 3); qualquer macro futura que precise compor comandos já registrados | **ÚNICO ÉPICO REAL REMANESCENTE** |

**Permanentemente fora de escopo, não são gap de infraestrutura (decisão de política já tomada, não reabrir):** `CommandKillAura` (DEC-38, Categoria 1), `CommandInvCaptcha` (bypass de detecção de bot/CAPTCHA — recusa permanente, mesma categoria de política que KillAura, não uma lacuna técnica), `CommandGive`/`CommandProxy` (sem equivalente de domínio, DEC-23), `CommandMove`/`CommandRetard` (DEC-36).

**Já implementadas integralmente, não voltam ao backlog:** AntiAFK, AutoFish/Pesca, AutoMiner, Herbalismo, Follow, Portal, Mob/Solk, Twerk, UseEntity, Goto, Reconectar, LimparChat, Ajuda, ListarJogadores, DropAll/LargarTudo, ClicarItemDaHotbar, todos os comandos de bloco/container/inventário.

**Candidatos remanescentes que já são 100% composição hoje (sem nenhuma infraestrutura nova — só precisam da sessão "implemente a macro X"):** `Solk.CommandMobTeleport` (mesma família de `CommandMob`, mecânica específica a confirmar só no momento da implementação, não bloqueia o roadmap), `CommandUseBow` limitado a mirar/carregar/soltar contra alvo já identificado (`usarItemNaMao`+`soltarItemEmUso`+Categoria 2/DEC-38 já bastam).

### Roadmap Definitivo (Épicos)

**EPIC-I2 — Motor de Orquestração de Comandos (único épico de Foundation API remanescente).** Domínio: Scripting. Objetivo: permitir que um `Comando` (ou uma `TarefaContinua`) invoque outro `Comando` já registrado por alias+argumentos, fechando o pré-requisito de `CommandScript`. Escopo esperado (a refinar só na sessão de implementação, sem nova auditoria): expor `GerenciadorDeComandos` (ou uma abstração equivalente) como colaborador injetável de um novo `ComandoScript`, decidindo layering (`interfaces.comando` já depende de `GerenciadorDeComandos`? verificar na hora — primeiro ponto real de "Comando chama Comando", pode exigir uma DEC pequena e aditiva, mesmo padrão da DEC-21/23). Produz: capacidade reutilizável por qualquer automação futura que precise compor comandos já prontos, não só `CommandScript`.

**Backlog de macros (pura composição, sem épico de infraestrutura — cada uma é uma sessão "implemente a macro X"):** `CommandMobTeleport`, `CommandUseBow`, e (após o EPIC-I2) `CommandScript`.

**Conclusão do levantamento:** com o EPIC-I2, a plataforma atinge o estado objetivo — qualquer macro remanescente do legado (exceto as permanentemente fora de escopo por política/DEC já listadas acima) é composição pura de Foundation APIs já existentes. Nenhum outro domínio tem lacuna de infraestrutura real pendente.

---

# 6. Escopo Implementado

Atualmente existem apenas componentes estruturais.

Implementado:

- Estrutura Maven
- Spring Boot
- Organização dos módulos (camadas `domain`/`application`/`infrastructure`/`interfaces`, DEC-12)
- Documentação
- Arquitetura
- Entidade de domínio `Bot` e Value Objects associados (`IdentificadorBot`, `EnderecoServidor`, `CredenciaisBot`, `SessaoBot`, `EstadoSessao`)
- Casos de Uso: `CasoDeUsoCriarBot`, `CasoDeUsoConectarBot`, `CasoDeUsoDesconectarBot`
- Arquitetura da camada de comunicação (contratos): `domain.protocol` (`Packet`, `Codec`, `LeitorDePacote`, `EscritorDePacote`, `EstadoConexao`, `VersaoProtocolo`, `PacketHandler`, `EventoDeProtocolo`), `domain.network` (`ConexaoMinecraft`, `SessaoDeRede`), `application.port` (`ConexaoBotPort`), `infrastructure.protocol` (`RegistroDePacotes`, `ProtocolDispatcher`)
- Packets concretos do protocolo 1.8 (Handshake e Login Start): `domain.protocol.v1_8` (`HandshakePacket`, `LoginStartPacket`, `HandshakeCodec`, `LoginStartCodec`), `infrastructure.protocol` (`BufferLeitorDePacote`, `BufferEscritorDePacote`), `infrastructure.protocol.v1_8` (`RegistroDePacotesV1_8`)
- Estado LOGIN completo do protocolo 1.8 (Encryption Request/Response, Set Compression, Login Success): `domain.protocol.v1_8` (`EncryptionRequestPacket`, `EncryptionResponsePacket`, `SetCompressionPacket`, `LoginSuccessPacket` + Codecs), `domain.protocol.SentidoDoPacote` (DEC-16), `domain.protocol` `readByteArray`/`writeByteArray` (DEC-15)
- Infraestrutura de transporte TCP: `infrastructure.protocol` (`CodificadorDeFrame`, `DecodificadorDeFrame`), `infrastructure.network` (`TransporteSocket` — implementa `ConexaoMinecraft`, usa `java.net.Socket` + Virtual Threads DEC-03)
- Pipeline de protocolo validada de ponta a ponta: `domain.protocol.v1_8` (6 `PacketHandler`s concretos + 6 Records de Evento), `ProtocolDispatcher` (`infrastructure.protocol`, testado)
- Integração Application ↔ Infraestrutura de comunicação: `application.usecase.CasoDeUsoConectarBot` (dependente de `ConexaoBotPort`), `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8` (primeira implementação de `ConexaoBotPort`, DEC-17), `ConexaoMinecraft.avancarEstado` (DEC-17) — fluxo completo validado sobre socket loopback local
- Fábrica de produção de conexão real: `infrastructure.network.v1_8.FabricaDeConexaoMinecraftV1_8` (abre `java.net.Socket` real; `HandshakePacket` validado sobre Socket local e, manualmente, contra o servidor Minecraft 1.8 real Olimpo/Craftlandia — ver Incremento 7A)
- Fluxo real de LOGIN até a primeira resposta do servidor: `AdaptadorConexaoBotV1_8.connect()` (Handshake + LoginStart reais, `EstadoConexao` avançando corretamente para `PLAY` no sucesso ou permanecendo em `LOGIN` quando o servidor exige Encryption/Compression), validado manualmente contra o servidor Minecraft 1.8 real Olimpo/Craftlandia (respondeu com `SetCompressionPacket`, recusado corretamente) — ver Incremento 7B
- Robustez do `TransporteSocket.readLoop` contra falhas de decodificação: `RuntimeException` originada ao decodificar/despachar um pacote (ex.: id não registrado) encerra a Virtual Thread de forma controlada e libera `input`/`output`, em vez de propagar como exceção não tratada — ver Incremento 7C
- Compressão zlib integrada ao fluxo de LOGIN e validada contra servidor real: `ConexaoMinecraft.ativarCompressao` (DEC-18), `CodificadorDeFrame`/`DecodificadorDeFrame` com sobrecargas de compressão (`java.util.zip.Deflater`/`Inflater` — Incremento 8A), `AdaptadorConexaoBotV1_8` ativando a compressão ao receber `SetCompressionPacket` e prosseguindo o login normalmente (Incremento 8B). Validado manualmente em 2026-07-16 contra o servidor real Olimpo/Craftlandia (`olimpo.clmc.com.br:3737`) pelo caminho oficial completo (`CasoDeUsoConectarBot → ConexaoBotPort → AdaptadorConexaoBotV1_8 → TransporteSocket`): servidor em modo OFFLINE, `compressionThreshold=1024`, `connect()` concluiu com sucesso — primeira conexão ponta a ponta bem-sucedida contra um servidor real neste projeto (Incremento 8C).
- Retenção da Sessão de Jogo e roteamento de eventos PLAY (DEC-19, Milestone 5 Incremento 1): `domain.bot.SessaoDeJogo`, `domain.protocol.ReceptorDeEvento<T>`, `infrastructure.protocol.RoteadorDeEventos`; `ConexaoBotPort.connect()` retornando `SessaoDeJogo`.
- Primeiros pacotes do estado PLAY (Milestone 5 Incremento 2): Keep Alive/Resposta Keep Alive, Join Game, Player Position And Look/Confirmação de Posição, Disconnect Play — `domain.protocol.v1_8` (Packets, Codecs, Handlers, Eventos, Receptores), `SessaoDeJogo` com `responderKeepAlive`/`registrarEntradaNoJogo`/`atualizarPosicao`/`encerrarPorDesconexaoDoServidor`.
- Chat recebido do servidor (Milestone 5 Incremento 3): `domain.protocol.v1_8.ChatMessagePacket`/`ChatMessageCodec`/`ChatMessageHandler`/`EventoChatMessage`/`ReceptorChatMessage` (PLAY, id `0x02`, CLIENTBOUND); `SessaoDeJogo.registrarMensagemDeChat` — fluxo completo `Servidor → TransporteSocket → Codec → Handler → Evento → Receptor → SessaoDeJogo → Bot` validado com teste de integração real. Texto do chat mantido como JSON cru do `ChatComponent` (parser de formatação/tradução do C# deliberadamente não reconstruído nesta etapa).
- Estado de vida do jogador — Update Health/Respawn (Milestone 5 Incremento 4): `domain.protocol.v1_8.UpdateHealthPacket`/`UpdateHealthCodec`/`UpdateHealthHandler`/`EventoUpdateHealth`/`ReceptorUpdateHealth` (PLAY, id `0x06`, CLIENTBOUND) e `RespawnPacket`/`RespawnCodec`/`RespawnHandler`/`EventoRespawn`/`ReceptorRespawn` (PLAY, id `0x07`, CLIENTBOUND); `SessaoDeJogo` com `health`/`food`/`saturation` (`atualizarVida`) e `registrarRespawn` (reaproveita `dimension`/`gamemode` já existentes). Envio automático de respawn (`PacketClientStatus`) do C# deliberadamente não replicado (automação fora de escopo).
- Inventário do jogador — Window Items/Set Slot/Held Item Change (Milestone 5 Incremento 5): `domain.bot.InventarioDoJogador` (novo agregado dedicado, não bounded context — 45 slots + slot ativo), referenciado por `SessaoDeJogo.inventario()`; `domain.protocol.v1_8.ItemStack`/`ItemStackCodec` (estrutura "Slot" do protocolo, NBT consumido com segurança mas nunca exposto); `WindowItemsPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x30`), `SetSlotPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x2F`), `HeldItemChangePacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x09`), todos CLIENTBOUND. `windowId != 0` (baús/janelas não-jogador) tratado como no-op por construção.
- Fundação do ciclo de vida de entidades — Spawn Player/Spawn Mob/Destroy Entities (Milestone 5 Incremento 6.1): `domain.bot.EntidadeRemota` (sealed, não é Aggregate Root), `EntidadeJogadorRemoto`/`EntidadeMob`, `EntidadesDoMundo` (coleção única, não bounded context), referenciada por `SessaoDeJogo.entidades()`; `SpawnPlayerPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x0C`), `SpawnMobPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x0F`), `DestroyEntitiesPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x13`), todos CLIENTBOUND. Correção estrutural registrada: a hierarquia `Entity`/`EntityManager` do C# não corresponde ao que o nome sugere (`Entity.cs` é a física do próprio bot; `EntityManager` é um catálogo estático de tipos, não uma coleção viva) — ver Fase de Planejamento do Incremento 6.
- Movimentação e velocidade de entidades (Milestone 5 Incrementos 6.2 e 6.3): `EntityRelativeMovePacket` (id `0x15`), `EntityLookPacket` (id `0x16`), `EntityLookAndRelativeMovePacket` (id `0x17`), `EntityTeleportPacket` (id `0x18`), `EntityHeadLookPacket` (id `0x19`), `EntityVelocityPacket` (id `0x12`) — todos PLAY/CLIENTBOUND, cada um com Codec/Handler/Evento/Receptor. `EntidadeRemota` ganha `velocityX`/`velocityY`/`velocityZ`. Bug do C# (Entity Look só encontrava jogadores) corrigido por construção pela coleção única do Incremento 6.1. Receptores tratam `entityId` desconhecido como no-op (primeira vez que esse contrato foi exercitado). Bypass de knockback do C# (aplicação automática de velocidade ao próprio bot) deliberadamente não portado — automação de combate, fora de escopo.
- Estado visual de entidades (Milestone 5 Incremento 6.4): `EntityEquipmentPacket` (id `0x04`), `AnimationPacket` (id `0x0B`), `EntityStatusPacket` (id `0x1A`), `EntityEffectPacket` (id `0x1D`), `RemoveEntityEffectPacket` (id `0x1E`) — todos PLAY/CLIENTBOUND, cada um com Codec/Handler/Evento/Receptor. `EntidadeRemota` ganha `equipamento` (Map por slot), `ultimaAnimacao`, `ultimoStatus`, `efeitosAtivos` (Map de `EfeitoAtivo`). 4 dos 5 pacotes não têm precedente no C# — formato de fio validado contra a especificação oficial do protocolo 47. Entity Effect aplicado a qualquer entidade rastreada (C# só aplicava ao próprio bot). Entity Metadata (`0x1C`) deliberadamente não implementado como pipeline próprio — permanece coberto pelo mecanismo de descarte seguro da DEC-20.
- Implementação integral da DEC-20 (Milestone 7 Incremento 7.0): `infrastructure.protocol.PacoteNaoRegistradoException` (novo, subtipo de `IllegalArgumentException`); `RegistroDePacotesV1_8.localizarCodec`/`localizarId` lançando esse tipo; `TransporteSocket.readLoop` descartando (log WARN + continua) um pacote PLAY sem Codec registrado em vez de encerrar a conexão, mantendo o comportamento estrito em HANDSHAKING/STATUS/LOGIN e em qualquer falha de `codec.decode(...)`. Decisão já aprovada em 2026-07-16, cuja implementação estava pendente até esta etapa.
- Fundação do Mundo — Chunk Data, Block Change, Multi Block Change (Milestone 7 Incrementos 7.1 e 7.2): `domain.bot.Mundo` (novo agregado, não bounded context — `ConcurrentHashMap<PosicaoDeChunk, Chunk>`), `Chunk` (Entity interna, 16 `SecaoDeChunk`), `PosicaoDeChunk` (record), referenciado por `SessaoDeJogo.mundo()`; `domain.protocol.v1_8.Bloco` (Value Object, id de 12 bits preservado sem o truncamento para byte presente no C#); `ChunkDataPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x21`, formato little-endian id+metadata por bloco, bioma/skylight/blocklight consumidos sem expor, coluna completa substitui o chunk inteiro e coluna parcial preserva seções não alteradas), `MultiBlockChangePacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x22`) e `BlockChangePacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x23`, campo Position empacotado em `long`). `Mundo.definirBloco` trata chunk desconhecido como no-op, mesmo contrato de `EntidadesDoMundo` para entidade desconhecida.
- Map Chunk Bulk (Milestone 7 Incremento 7.3): `domain.protocol.v1_8.SecoesDeChunkCodec` (novo utilitário compartilhado, extraído de `ChunkDataCodec`, reaproveitado pelos dois pacotes), `ColunaDeChunk` (record); `MapChunkBulkPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x26`, `skyLightSent` único para todas as colunas do pacote, cabeçalhos lidos antes dos dados, sempre coluna completa); `Mundo.registrarColunaCompleta` (novo, reaproveitado também por `ReceptorChunkData`).
- Explosion (Milestone 7 Incremento 7.4): `domain.protocol.v1_8.RegistroDeExplosao` (record); `ExplosionPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x27`, CLIENTBOUND; `radius` consumido sem ser exposto, count lido como `int` de 4 bytes e não VarInt); `SessaoDeJogo` ganha `motionX`/`motionY`/`motionZ` + `aplicarImpulsoDeExplosao` (primeira modelagem de motion do jogador local). Flag `Client.MapAndPhysics` do legado (gate que pode pular o pacote inteiro) não tem equivalente na modelagem Java — mutação aplicada incondicionalmente, consistente com os demais Receptores de Mundo já implementados.
- Change Game State (Milestone 7 Incremento 7.5): `domain.protocol.v1_8.ChangeGameStatePacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x2B`, CLIENTBOUND); `SessaoDeJogo.atualizarGamemode` (novo). Só `reason==3` (mudança de gamemode) produz mutação, fiel ao legado; as demais 9 `reason` são decodificadas mas não têm efeito. Time Update (`0x03`) e Spawn Position (`0x05`) avaliados e **deliberadamente não implementados** — nenhuma versão do C# legado (4 projetos auditados) jamais os leu ou usou; permanecem cobertos pelo descarte seguro da DEC-20, reproduzindo com fidelidade o "ignorado por completo" do legado.
- DEC-21 — Papel do Caso de Uso em Ações Iniciadas pelo Bot (Milestone 8, Incremento 8.1): formaliza o fluxo `CasoDeUso → SessaoDeJogo → ConexaoMinecraft → Packet → Servidor`, complementar ao fluxo reativo da DEC-19. Puramente documental, sem código.
- Lista de Jogadores — Player List (Milestone 8, Incremento 8.2): `domain.bot.ListaDeJogadores`/`JogadorConhecido` (novo agregado, não bounded context), referenciado por `SessaoDeJogo.listaDeJogadores()`; `domain.protocol.v1_8.ItemDeListaDeJogadores` (sealed, 5 variantes) + `PlayerListItemPacket`/Codec/Handler/Evento/Receptor (PLAY, id `0x38`, CLIENTBOUND). Divergência documentada: bug de alinhamento do legado na ação 3 (UUID desconhecido) não replicado — Codec sempre lê o formato de fio completo, fiel ao precedente "Codec nunca pula bytes obrigatórios por estado de domínio".
- Chat Enviado pelo Bot (Milestone 8, Incremento 8.3): `domain.protocol.v1_8.EnvioDeChatPacket`/Codec/Handler/Evento (PLAY, id `0x01`, SERVERBOUND, sem Receptor — mesmo precedente de `RespostaKeepAlivePacket`/`ConfirmacaoDePosicaoPacket`); `SessaoDeJogo.enviarMensagem` (trunca em 99 caracteres, fiel ao off-by-one do legado; no-op em mensagem vazia/nula); `application.usecase.CasoDeUsoEnviarMensagemDeChat` (primeiro Caso de Uso do Play State, conforme DEC-21). Primeiro pacote SERVERBOUND de Play State iniciado pelo bot (não por reação a protocolo).
- DEC-22 — Ações Fundamentais do Jogador: Movimentação e Rotação (Milestone 9, Incremento 9.1): formaliza que a Milestone 9 implementa só o envio explícito e sob demanda de posição/rotação (sem Tick loop/física); decide reaproveitar `ConfirmacaoDePosicaoPacket` (id `0x06`) para qualquer combinado posição+rotação futuro em vez de nova classe (colisão de chave em `RegistroDePacotesV1_8`); decide mutação otimista de estado. Puramente documental, sem código.
- Movimentação do Jogador — Player Position (Milestone 9, Incremento 9.2): `domain.protocol.v1_8.PlayerPositionPacket`/Codec/Handler/Evento (PLAY, id `0x04`, SERVERBOUND, sem colisão com `EntityEquipmentPacket` id `0x04` CLIENTBOUND); `SessaoDeJogo.mover(double,double,double,boolean)` (mutação otimista de x/y/z); `application.usecase.CasoDeUsoMoverJogador` (conforme DEC-21/DEC-22).
- Rotação do Jogador — Player Look (Milestone 9, Incremento 9.3): `domain.protocol.v1_8.PlayerLookPacket`/Codec/Handler/Evento (PLAY, id `0x05`, SERVERBOUND); `SessaoDeJogo.olhar(float,float,boolean)` (mutação otimista de yaw/pitch); `application.usecase.CasoDeUsoRotacionarJogador` (mesmo padrão). Envio explícito e sob demanda apenas — nenhum Tick loop/física/automação implementado (motor de física continua fora de escopo, DEC-22).
- Milestone 10 (Incremento 10.1, planejamento): nenhuma DEC nova necessária — DEC-21/DEC-22 já cobrem integralmente o padrão de ação iniciada pelo bot para as interações do jogador com o mundo; puramente documental, sem código.
- Swing Arm (Milestone 10, Incremento 10.2): `domain.protocol.v1_8.BalancarBracoPacket`/Codec/Handler/EventoBalancarBraco (PLAY, id `0x0A`, SERVERBOUND, sem campos — nome em português para evitar colisão de classe Java com `AnimationPacket` clientbound, que reutiliza o mesmo nome oficial "Animation"); `SessaoDeJogo.balancarBraco()`; `application.usecase.CasoDeUsoBalancarBraco`.
- Player Digging (Milestone 10, Incremento 10.3): `domain.protocol.v1_8.PlayerDiggingPacket`/Codec/Handler/EventoPlayerDigging (PLAY, id `0x07`, SERVERBOUND, sem colisão com `RespawnPacket` clientbound); `SessaoDeJogo.iniciarQuebraDeBloco`/`cancelarQuebraDeBloco`/`finalizarQuebraDeBloco` (status 0/1/2); `application.usecase.CasoDeUsoIniciarQuebraDeBloco`/`CasoDeUsoCancelarQuebraDeBloco`/`CasoDeUsoFinalizarQuebraDeBloco`. Mineração automática (AutoMiner do legado, Tick loop) deliberadamente não portada; a fórmula de força de quebra (`DiggingHelper`) foi portada depois, como primitiva pura, na Milestone 17.
- Player Block Placement (Milestone 10, Incremento 10.4): `domain.protocol.v1_8.PlayerBlockPlacementPacket`/Codec/Handler/EventoPlayerBlockPlacement (PLAY, id `0x08`, SERVERBOUND, sem colisão com `PlayerPositionAndLookPacket` clientbound; Codec com extensão de sinal do campo y para fidelidade ao sentinela -1/-1/-1 "usar item na mão"); `SessaoDeJogo.colocarBloco`; `application.usecase.CasoDeUsoColocarBloco`. Lógica de inventário automático (seleção de item, reenvio automático para itens especiais) deliberadamente não portada.
- Movimentação e Rotação Combinadas do Jogador — Move And Look (Milestone 11, Incremento 11.1): reaproveita `ConfirmacaoDePosicaoPacket`/Codec (PLAY, id `0x06`, SERVERBOUND, já registrado desde a Milestone 5) conforme decidido pela DEC-22 — nenhum Packet/Codec/Handler novo; `SessaoDeJogo.moverEOlhar` (mutação otimista de x/y/z/yaw/pitch); `application.usecase.CasoDeUsoMoverEOlharJogador` (conforme DEC-21).
- Arquitetura de Execução de Comandos do Bot (Milestone 12, Incremento 12.1, DEC-23): novo subpacote `interfaces.comando` (camada `interfaces` aprovada desde a DEC-12, populada pela primeira vez) — `Comando`/`ResultadoComando`/`GerenciadorDeComandos`, contrato mínimo single-shot sem `Tick`/`Toggle`/`isMacro`; 8 comandos concretos (`ComandoMover`/`ComandoOlhar`/`ComandoMoverEOlhar`/`ComandoBalancarBraco`/`ComandoIniciarQuebraDeBloco`/`ComandoCancelarQuebraDeBloco`/`ComandoFinalizarQuebraDeBloco`/`ComandoColocarBloco`), todos delegando a Casos de Uso já aprovados das Milestones 9–10, sem nenhum pacote/Port/agregado novo. `CommandHelp`/`CommandPlayerList`/`CommandMove`/`CommandGoto`/`CommandSneak` e os comandos de automação/inventário/combate do legado avaliados e deliberadamente não portados nesta milestone (ver subseção da Milestone 12 para o motivo específico de cada um).
- Raycast Fiel ao Legado sobre Mundo (Milestone 13, Incremento 13.1, DEC-24): `domain.protocol.v1_8.Bloco.solido()` (porte de `Blocks.IsSolid`); `domain.bot.Mundo.tracarRaio(...)` (porte de `World.RayCast`, incluindo o quirk do bloco de destino nunca testado e a semântica de `permitirAgua`); `domain.bot.ResultadoDoRaio` (novo record, porte de `HitResult`); `domain.bot.SessaoDeJogo.tracarRaioParaBlocos(alcance)` (porte de `RayCastBlocks`/`GetLookVector`/`CalculateLookVector`); correção de bounds-check em `Mundo.blocoEm` fiel a `World.GetBlock` (evita `ArrayIndexOutOfBoundsException` para y fora de `[0,256)`). Sem Packet/Port/Use Case novo. `CanSeeEntity` genérico (altura por tipo de mob) deliberadamente não portado — ver subseção da Milestone 13.
- Ações de Bloco com Auto-Mira (Milestone 14, Incremento 14.1, DEC-25): `domain.bot.SessaoDeJogo.olharParaBloco(x,y,z)` (porte de `Entity.LookTo`/`LookToBlock`, sem o jitter aleatório do legado) e `usarItemNaMao(item)` (porte do sentinela `-1`/`-1`/`-1` de `PacketBlockPlace(ItemStack)`, fecha a lacuna do Incremento 10.4); `application.usecase.CasoDeUsoOlharParaBloco`/`CasoDeUsoUsarItemNaMao`; `interfaces.comando.ComandoClicarBloco` (porte de `CommandClickBlock`), `ComandoQuebrarBloco` (porte do caminho base de `CommandBreakBlock`, sem `ncp`/`at`), `ComandoColocarBlocoAutoMira` (porte de `CommandPlaceBlock`, alias distinto de `ComandoColocarBloco`/Milestone 12 para não colidir com "placeblock"). Sem Packet/Port/agregado novo — consome integralmente o raycast entregue pela DEC-24.
- Infraestrutura de Saída de Mensagens para o Operador (Milestone 15, Incremento 15.1, DEC-26): `domain.bot.SaidaDoOperador` (porte de `ChatMessages`/`MaximumChatLines`, regime permanente de 151 mensagens); `Bot` ganha o campo `saidaDoOperador`; `GerenciadorDeComandos.executar` ganha as mensagens de fallback de `CommandManagerNew.RunCommand`; `interfaces.comando.ComandoAjuda` (porte de `CommandHelp`) e `ComandoListarJogadores` (porte de `CommandPlayerList`). Sem Packet/Port/agregado/bounded context novo, nenhuma interface pública alterada.
- Algoritmo de Busca de Caminho sobre Mundo (Milestone 16, Incremento 16.1, DEC-27): `domain.bot.PontoDeCaminho` (novo, porte de `PathPoint`) e `domain.bot.BuscadorDeCaminho` (novo, package-private, porte de `PathFinder`+`Path`, sem o branch inalcançável `NodeType._2`); `Mundo.criarCaminhoPara(...)` (porte de `World.CreatePathTo`); `SessaoDeJogo.criarCaminhoPara(destX,destY,destZ)` (conveniência com os valores fixos do único call site do legado). `PathGuide` (execução do caminho tick a tick, dependente de física/Motion que não existe no domínio Java) e todos os seus consumidores (`CommandGoto`/`CommandFollow`/`CommandPortal`/`AutoMiner`) permanecem deliberadamente fora de escopo — a DEC-22 (motor de física fora de escopo) não é reaberta. Sem Packet/Port/Caso de Uso/`Comando`/bounded context novo.
- Registro de Blocos e Calculadora de Força de Quebra (Milestone 17, Incremento 17.1, DEC-28): `domain.protocol.v1_8.RegistroDeBlocos` (novo, package-private, porte de `Blocks`/`Block` — só `Hardness`/`Material`/`HarvestTools`, os únicos 3 campos com consumidor comprovado) e `CalculadoraDeQuebraDeBloco` (novo, público, porte completo de `DiggingHelper` — `podeColher`/`forcaDaFerramenta`/`forcaDoJogador`/`forcaDeQuebra`); `SessaoDeJogo.estaSubmerso()` (novo, porte de `Entity.IsUnderWater`). Nível de Efficiency da ferramenta, amplifier de Haste e de Mining Fatigue do próprio bot viram parâmetros explícitos (sentinela `0`/`-1`) por não terem estado equivalente no domínio Java ainda (NBT de item não exposto desde a Milestone 5.5; efeitos do próprio bot não modelados desde a Milestone 5.6.4). `AutoMiner`/`CommandMiner`/`CommandBreakBlock` (Tick loop) permanecem deliberadamente fora de escopo. Sem Packet/Port/Caso de Uso/`Comando`/bounded context novo.
- Update Sign e encerramento formal da Milestone 7 (Milestone 18, Incremento 18.1): `domain.protocol.v1_8.UpdateSignPacket`/Codec/Handler/EventoUpdateSign/ReceptorUpdateSign (PLAY, id `0x33`, CLIENTBOUND); `Mundo.registrarPlaca`/`placaEm` (novo dicionário interno independente do mapa de chunks, sem no-op para chunk não carregado — divergência fiel ao `World.Signs` do legado). World Border (`0x44`), Update Block Entity (`0x35`), Block Action (`0x24`) e Block Break Animation (`0x25`) avaliados e confirmados **sem nenhum precedente** em todo o `Handler_v18.cs`/`Projeto Adv 2.4.5` — cobertos pelo descarte seguro da DEC-20, mesmo tratamento de Time Update/Spawn Position. Bounded context de Mundo (Milestone 7) encerrado oficialmente. Sem DEC nova.
- Linha de Visão contra Jogador Remoto (Milestone 19, Incremento 19.1): `domain.bot.SessaoDeJogo.podeVerJogador(EntidadeJogadorRemoto)` (novo, porte de `Entity.CanSeePlayer`), segundo consumidor de `Mundo.tracarRaio`/DEC-24. `CanSeeEntity` genérico para mobs permanece fora de escopo (depende de altura por tipo de mob, dado ainda não levantado). Sem Packet/Port/Caso de Uso/`Comando`/agregado/bounded context novo, sem DEC nova.
- Entity Action — Sneak (Milestone 20, Incremento 20.1): `domain.protocol.v1_8.EntityActionPacket`/Codec/Handler/EventoEntityAction (PLAY, id `0x0B`, SERVERBOUND, sem colisão com `AnimationPacket` clientbound; sem Receptor, mesmo precedente de `EnvioDeChatPacket`); `SessaoDeJogo.agachar()`/`pararDeAgachar()` (`actionId` 0/1, `jumpBoost` sempre 0); `application.usecase.CasoDeUsoAgachar`/`CasoDeUsoPararDeAgachar`; `interfaces.comando.ComandoAgachar`/`ComandoPararDeAgachar`. `ActionID` 2 (leave bed) e 5 (jump boost) confirmados como código morto (nenhum call site no legado); `ActionID` 3/4 (sprint) e `Player` bare (`0x03`, `PacketUpdate`) confirmados como Tick-loop-only (único call site de cada um dentro do loop de física do `MinecraftClient.cs`) — mesmo motivo de exclusão da DEC-22, não ausência de precedente. Sem Packet/Port/agregado/bounded context novo, sem DEC nova.
- Fundação da Engine de Execução Contínua (Milestone 21, Incremento 21.1, DEC-29): `domain.bot.EstadoExecucao` (novo enum `PARADO`/`EXECUTANDO`/`PAUSADO`, sem precedente no legado); `Bot` ganha ciclo de vida (`iniciar`/`pausar`/`retomar`/`parar`) e registro de tarefas contínuas (`registrarTarefa`/`removerTarefa`/`tarefasContinuas`); `domain.bot.TarefaContinua` (nova interface funcional, `void executar(Bot)`, zero implementações reais); `application.port.AgendadorDeTarefasPort` (novo, segundo Port do projeto); `infrastructure.execucao.AgendadorDeTarefasVirtualThread` (implementa o Port sobre Virtual Threads, DEC-03) e `MotorDeTick` (percorre bots `EXECUTANDO` e invoca suas `TarefaContinua`, isolando falha por tarefa, protegido contra reentrância). Mecanismo puro de agendamento/iteração — zero leitura/escrita de Motion/OnGround/física; nenhuma macro/automação registrada; não reabre DEC-22/DEC-23/DEC-27 (ver DEC-29). Sem Packet/Comando/bounded context novo.
- Propagação de Perda de Conexão para o Ciclo de Vida do Bot (Milestone 22, Incremento 22.1, DEC-30): `domain.network.ConexaoMinecraft.estaAberta()` (novo método `default`, preserva os 65 fakes de teste existentes); `infrastructure.network.TransporteSocket.estaAberta()` (sobrescreve); `domain.bot.SessaoDeJogo.estaEncerrada()`/`encerrarVoluntariamente()` novos; `domain.bot.Bot.registrarDesconexao()` novo; `infrastructure.execucao.MotorDeTick.tick()` passa a detectar e reagir a sessão de jogo encerrada antes do filtro de `EstadoExecucao`; `infrastructure.network.v1_8.AdaptadorConexaoBotV1_8.disconnect(Bot)` implementado de fato (antes lançava `UnsupportedOperationException` desde a Milestone 4); `application.usecase.CasoDeUsoDesconectarBot` integrado a `ConexaoBotPort` pela primeira vez. Poll pelo `MotorDeTick`, não callback/EventBus — fiel ao único padrão de propagação de estado de conexão que o legado de fato usa (busca exaustiva confirma zero eventos de conexão reais no C#). Nenhuma interface já aprovada teve assinatura alterada. 737 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- KeepAlive/AutoReconnect/Retomada de Tarefas/Proxy por Bot/Rotação de Proxy (Milestone 22, Incrementos 22.2-22.6, DEC-31): `domain.bot.SessaoDeJogo.keepAliveExpirou`/`LIMITE_KEEP_ALIVE` (fiel a 750 ticks/37500ms do legado); `domain.bot.PoliticaDeReconexao`/`PoliticaDeReconexaoComJitter` (intervalo de reconexão como política configurável, DEC-31 — regra de retry incondicional até sucesso preservada, jitter evita tempestade de reconexão); `infrastructure.execucao.GerenciadorDeReconexao` (novo, mesma categoria de `MotorDeTick`, ativa `SessaoBot.autoReconnect` pela primeira vez desde a Milestone 3, chama `bot.iniciar()` no sucesso para retomar `tarefasContinuas` automaticamente); `domain.bot.ConfiguracaoProxy`/`TipoDeProxy`/`PoolDeProxies` (proxy por bot via extensão aditiva de `EnderecoServidor`, rotação round-robin fiel a `ProxyList.NextProxy()`); `FabricaDeConexaoMinecraftV1_8` roteando via `java.net.Proxy` nativo do JDK em vez de reconstruir o handshake manual SOCKS4/5/HTTP do legado (`Proxy.cs`, divergência de implementação documentada, não de comportamento). Zero novo Port/agregado/bounded context; zero interface já aprovada alterada. 766 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Wiring de Produção/DI (Milestone 22, Incremento 22.7, encerramento da Milestone 22): `infrastructure.config.ConfiguracaoDeConexao`/`ConfiguracaoDeExecucao` (novos, `@Configuration`) compõem todos os beans de `MotorDeTick`/`AgendadorDeTarefasPort`/`GerenciadorDeReconexao`/`ConexaoBotPort`/Casos de Uso; `infrastructure.config.CicloDeVidaDoMotorDeExecucao` (novo, `SmartLifecycle`) liga o tick (50ms) e a verificação de reconexão (1000ms) a um agendamento real no start do Spring, e encerra `AgendadorDeTarefasPort` de forma limpa no stop. `infrastructure.config` deixa de estar vazio. Registro/remoção de bots individuais permanece sem call site de produção — nenhum CLI/API existe ainda (DEC-02). Zero novo Port/agregado/bounded context/interface alterada. 794 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Épico Fase 2 "execução de caminho" (Milestones 27-29): `domain.bot.SessaoDeJogo.caminhoAtual()`/`definirCaminho`/`limparCaminho` (novos) tornam `GuiaDeCaminho`/DEC-35 consumível em produção, ticado incondicionalmente por `infrastructure.execucao.MotorDeTick` (mesmo lugar onde o legado tica `CurrentPath`); `interfaces.comando.ComandoGoto` (porte de `CommandGoto`, Milestone 27) é o menor consumidor real; `domain.bot.TarefaFollow`/`interfaces.comando.ComandoFollow` (porte de `CommandFollow`, Milestone 28, terceira Macro real) segue um jogador remoto olhando para ele (`SessaoDeJogo.olharParaEntidade`, novo) e recalculando o caminho a cada 2.5 blocos de deslocamento do alvo, resolvido via `ListaDeJogadores.uuidPorNome`/`EntidadesDoMundo.porUuid` (novos); `domain.bot.RegistroDePortais`/`PosicaoDePortal`/`SessaoDeJogo.localizarPortal` + `interfaces.comando.ComandoPortal` (porte de `CommandPortal`, Milestone 29) localizam portais do servidor CraftLandia por nome (dado estático hardcoded, 25 minijogos) ou por varredura de blocos (`"**"`). `AutoMiner` continua bloqueado (mecanismo `MoveQueue`/`Movement`/`MoveRelative` por yaw, DEC própria futura, não herdada da DEC-35). Cálculo de caminho síncrono em todos os três consumidores (divergência de execução registrada, não de resultado) — o legado despacha via `Task.Factory.StartNew`, sem precedente de wrapper assíncrono no domínio Java. Zero novo Port/bounded context/interface já aprovada alterada. 856 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Épico "Mineração" (Milestone 30, DEC-36): `domain.bot.TarefaMineracao`/`interfaces.comando.ComandoMinerar` (porte de `AutoMiner`/`CommandMiner`, quarta Macro real) — busca o bloco-alvo de menor custo num cubo de raio 8 ao redor dos pés (pedra/terra, únicos candidatos de fábrica sem `Program.Config`), desloca-se via `GuiaDeCaminho` (mesmo mecanismo de `ComandoGoto`/`TarefaFollow`/`ComandoPortal`, sem o mecanismo adicional `MoveQueue`/`Movement`/`MoveRelative` por yaw — DEC-36 decide não portá-lo, sem consumidor real no modelo de execução síncrona já adotado) e quebra blocos no alcance do raio de 6 acumulando `CalculadoraDeQuebraDeBloco.forcaDeQuebra` (DEC-28) por tick. Fecha o item 4 pendente da DEC-35, encerrando o Épico "Mineração" nesta milestone única. Zero novo Port/bounded context/agregado/interface já aprovada alterada. 865 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- EPIC-I1 — Container Framework (Milestone 32, DEC-37): `domain.bot.Janela`/`JanelaDeSlots` (novo agregado + interface, também implementada por `InventarioDoJogador`); `OpenWindowPacket`(CB 0x2D)/`CloseWindowPacket`(CB 0x2E)/`FecharJanelaPacket`(SB 0x0D)/`ClickWindowPacket`(SB 0x0E)/`ConfirmTransactionPacket`(CB 0x32)/`ConfirmarTransacaoPacket`(SB 0x0F), cada um com Codec/Handler/Evento (+Receptor onde há consumo real); `ReceptorSetSlot`/`ReceptorWindowItems` estendidos para `windowId != 0` (fecha a lacuna da Milestone 5.5/31); `SessaoDeJogo` ganha a API completa de janela (`abrirJanela`/`fecharJanela`/`clicarSlot`/`shiftClique`/`trocarSlots`/`pegarItem`/`largarItem`/`moverItem`/`confirmarTransacao`/`localizarItemNaJanela`/`localizarEspacoLivreNaJanela`) sobre indexação unificada de slot (janela + inventário do bot numa única numeração, elimina por construção o bug `isChestOpen` do legado); 8 Casos de Uso; `ComandoClicarSlot`/`ComandoFecharJanela`. Zero regra de macro nesta camada — desbloqueia a Milestone 31 e todo o roadmap de infraestrutura da Fase 2 (EPIC-I2 em diante). Zero interface pré-existente alterada. 963 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Domínio Inventory/Container — auditoria e fechamento (Milestone 33): `JanelaDeSlots.localizarTodosOsItens` (Item Search) e `SessaoDeJogo.limparCursor` (Inventory Utils) novos; 7 `Comando`s novos conectando Casos de Uso do Container Framework sem `Comando` correspondente; Equipment/Armadura confirmado sem lacuna funcional (legado nunca veste armadura). Zero DEC nova. 994 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Domínio Interação com o Mundo — auditoria e fechamento (Milestone 34): `UseEntityPacket`/Codec (novo, PLAY `0x02` SERVERBOUND, porte de `PacketUseEntity`); `SessaoDeJogo.interagirComEntidade` novo; `CasoDeUsoOlharParaEntidade`/`CasoDeUsoInteragirComEntidade` novos; `interfaces.comando.ComandoUseEntity` (porte de `CommandUseEntity` — nick ou `@any`, atacar/interagir). Blocos/portas/alavancas/botões/placas/estruturas/uso-de-item confirmados 100% cobertos desde as Milestones 14/18/25/29/32 — nenhuma classe de porta/alavanca/botão/NPC tem precedente próprio no legado (tudo compõe sobre o clique genérico de bloco já portado). Zero DEC nova. 1007 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Backlog "Composição Pura" em lote (Milestone 35): `SaidaDoOperador.limpar()`/`ComandoLimparChat`; `SessaoDeJogo.soltarItemEmUso` (status 5, sem consumidor); `TarefaLargarTudo`/`ComandoLargarTudo` (DropAll); `CasoDeUsoSelecionarSlotDaHotbar`/`ComandoClicarItemDaHotbar` (HotbarClick); `TarefaTwerk`/`ComandoTwerk`; `CasoDeUsoReconectarBot`/`ComandoReconectar` (Reco, sem suporte a `[IP:porta]` alternativo); `TarefaAutoFish`/`ComandoAutoFish` (Solk/CommandPesca/AutoFish, maior peça do lote — achado de fidelidade: `"$"` no legado é auto-invocação local de outro comando, não chat real). Zero DEC nova, zero Packet/Port/agregado/bounded context novo, zero interface pré-existente alterada. 1036 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.
- Foundation APIs — Milestones 36-39 (ver DEC-38/DEC-39): fecham Container/Combate-Categoria-2/Chat/NBT/Efeitos-do-próprio-bot como capacidades reutilizáveis. **EPIC-APP1 — Primeira API Pública REST+WebSocket (Milestone 40, DEC-40)** e **EPIC-APP2 — Cobertura completa da API para o React (Milestone 41, DEC-41)**: ~13 Controllers REST sob `/api/v1`, WebSocket cru (`/ws/bots/{id}/events`, `/ws/events`), `ApiKeyFilter`, CORS, Swagger UI — ver DEC-40/DEC-41 para o detalhamento completo (as linhas "Serviços"/"APIs REST" abaixo, em "Não implementado", estão desatualizadas desde a Milestone 40; mantidas aqui só por não ter sido objeto desta sessão revisar o restante da lista).
- **EPIC-FRONT-01 — Persistência PostgreSQL/Testes de Integração REST/CORS (Milestone 42, DEC-42):** adapters JPA (`RepositorioDeContasJpa`/`RepositorioDeServidoresJpa`/`RepositorioDeProxiesJpa`) substituem os in-memory atrás das portas `RepositorioDeContas`/`RepositorioDeServidores` e da porta nova `RepositorioDeProxies`; migration Flyway (`contas`/`servidores`/`proxies`); `PoolDeProxies` com write-through; 6 testes de integração REST novos (primeiros do projeto); bug de CORS corrigido em `ApiKeyFilter` (preflight `OPTIONS` barrado por engano). Zero DEC de protocolo/domínio reaberta, zero contrato de API alterado. 1104 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente.

Não implementado:

- Serviços (parcialmente desatualizado — ver EPIC-APP1/EPIC-APP2/EPIC-FRONT-01 acima)
- ~~Repositórios~~ Conta/Servidor/Proxy persistidos via JPA/PostgreSQL desde a Milestone 42 (EPIC-FRONT-01); persistência de Bot/`GerenciadorDeBots` continua em memória (fora do escopo pedido nesta sessão)
- APIs REST (desatualizado — ver EPIC-APP1/EPIC-APP2 acima, Milestones 40-41)
- Scheduler distribuído (multi-nó/multi-JVM) — o scheduler single-JVM em processo já existe desde a Milestone 21 (`AgendadorDeTarefasPort`/`MotorDeTick`), sem nenhuma tarefa real registrada ainda
- Rede/Protocolo Minecraft (criptografia AES/RSA real e autenticação Mojang para servidores em modo online, Session Server, Status State)
- Demais pacotes do estado PLAY (Entity Metadata `0x1C` — coberto apenas pelo descarte seguro da DEC-20, sem semântica exposta —, combate, XP/Experience — sem precedente no C# auditado, mesmo tratamento de Time Update/Spawn Position); bounded context de Mundo (Milestone 7) **encerrado oficialmente na Milestone 18** — Chunk Data, Block Change, Multi Block Change, Map Chunk Bulk, Explosion, Change Game State e Update Sign implementados; Time Update (`0x03`), Spawn Position (`0x05`), World Border (`0x44`), Update Block Entity (`0x35`), Block Action (`0x24`) e Block Break Animation (`0x25`) avaliados e confirmados sem nenhum precedente no C# legado — cobertos com fidelidade pelo descarte seguro da DEC-20 (ver Incrementos 7.5 e 18.1). Player List/tab list e Chat Enviado pelo Bot **implementados** — ver Milestone 8. Movimentação (Player Position `0x04`) e rotação (Player Look `0x05`) do jogador **implementadas** — ver Milestone 9. Swing Arm (`0x0A`), Player Digging (`0x07`) e Player Block Placement (`0x08`) **implementados** — ver Milestone 10; mineração automática, física de quebra e "usar item na mão" (sentinela -1/-1/-1, Codec já suporta) continuam fora de escopo. Combinação posição+rotação iniciada pelo bot **implementada** — ver Milestone 11 (reaproveita `ConfirmacaoDePosicaoPacket`, DEC-22); Tick loop/motor de física automático continua fora de escopo. Entity Action `0x0B` serverbound **implementado para o subconjunto sneak** (`actionId` 0/1) — ver Milestone 20; leave bed/jump boost são código morto comprovado no legado, sprint e `Player` bare `0x03` só têm call site dentro do Tick loop de física (mesmo motivo de exclusão da DEC-22).
- Crafting (mesa de trabalho/fornalha como receita — a janela em si é genérica e já suportada desde a Milestone 32/EPIC-I1, mas nenhuma regra de receita/resultado de crafting foi modelada); `Window Property` (`0x31` — progresso de fornalha/mesa de encantamento/bigorna, sem precedente de uso no legado); `Creative Inventory Action` (`0x10` — só serve `CommandGive`, já excluído desde a DEC-23)
- NBT de itens (encantamentos, nome customizado, lore) — consumido do protocolo mas descartado, nunca exposto como dado de domínio
- Parser de `ChatComponent` (cores, `translate`, extras) — texto do chat permanece como JSON cru
- Automações (ex.: respawn automático ao detectar `health <= 0`)
- `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` (não integrados)
- EventBus
- Macros
- Cliente Minecraft completo

---

# 7. Restrições do Projeto

É proibido:

- Converter automaticamente código C#
- Criar regras de negócio sem validação
- Alterar comportamento do legado
- Implementar funcionalidades fora da milestone atual

Toda evolução deve ocorrer de forma incremental.

---

# 8. Decisões Arquiteturais Congeladas

## Plataforma

Java 21 LTS

## Framework

Spring Boot

## Build

Maven

## Frontend

React

## Banco

PostgreSQL

## Arquitetura

Clean + Hexagonal

Estas decisões somente podem ser alteradas mediante nova ADR.

---

# 9. Pendências Conhecidas

Lista das atividades ainda não iniciadas.

Exemplo:

- Modelagem do domínio
- Definição dos Bounded Contexts
- Estratégia de persistência
- Migração dos bots
- Migração do protocolo Minecraft
- Scheduler distribuído

---

# 10. Próxima Milestone

Nome

**Atualização mais recente (Milestone Frontend 06, sessão 2026-07-27): EPIC-FRONT-06 — Bot
Details — CONCLUÍDA.** Núcleo restante da Fase 2, `features/bots/details/` completa: 5 sub-abas
roteadas sob `/bots/:id` (Console com logs em tempo real via WS + chat + comandos, Ações de
movimento/interação, Inventário/Equipamento/Janela, Mundo/estado/bloco/entidades/jogadores, Macros do
bot cruzadas com o Catálogo global). Canal WS por bot (`connectBotEventsSocket`) conectado pela
primeira vez nesta sessão. 3 GAPs de backend confirmados manualmente contra o backend real (não
contornados): (1) `estado`/`mundo/*`/`inventario/*` exigem sessão PLAY ativa (409 quando
desconectado); (2) `MacroResponse.tipo` (única info de macro ativa) frequentemente não bate com
`nome`/`aliases` do Catálogo; (3) `DELETE /macros/{alias}` responde 200 mas pode não remover a macro
da lista com o bot desconectado. Ver subseção "Milestone Frontend 06" (Seção 5) para o detalhamento
completo. Próximo passo: EPIC-FRONT-07 em diante implementam Monitoramento (Fase 3) e, se o backend
resolver os GAPs pendentes, as features da Fase 4 (Spammer/Ferramentas/Viewer 3D/Minerador).

Atualização anterior (Milestone Frontend 05, sessão 2026-07-27): EPIC-FRONT-05 — Feature Bots —
CONCLUÍDA (listagem, CRUD exceto edição, ações de execução/sessão, troca de proxy, auto-reconnect,
ações em lote — ver subseção "Milestone Frontend 05", Seção 5).

Atualização anterior (Milestone Frontend 04, sessão 2026-07-27): EPIC-FRONT-04 "Fundação
Administrativa" — CONCLUÍDA (Contas/Servidores, Catálogo de Comandos/Macros, Configurações — ver
linha de log correspondente na Seção 9, sem subseção dedicada).

Atualização anterior (Milestone Frontend 03, sessão 2026-07-27): EPIC-FRONT-03 — Feature
Proxy — CONCLUÍDA.** Segunda feature funcional, `features/proxy/` completa e pronta para produção:
listar/criar/editar/excluir com confirmação, busca e paginação client-side, tudo via hooks gerados
pelo Orval. Achado de backend documentado ao final (não bloqueou a entrega): `ProxyController` não
tem `PUT` (proxy não tem identidade própria por entrada) — "editar" implementado como composição
client-side de remover+adicionar, usando as mesmas funções geradas. Validado contra o backend Java
já em execução (IntelliJ): criação/edição/exclusão reais confirmadas via `curl` direto no backend.
Ver subseção "Milestone Frontend 03" (Seção 5) para o detalhamento completo.

Atualização anterior (Milestone Frontend 02, sessão 2026-07-27): EPIC-FRONT-02 — Feature Dashboard
— CONCLUÍDA (ver subseção "Milestone Frontend 02", Seção 5).

Atualização anterior (Milestone Frontend 01, sessão 2026-07-27): EPIC-FRONT-01 (Frontend) —
Fundação do projeto React — CONCLUÍDA (ver subseção "Milestone Frontend 01", Seção 5).

Atualização anterior (Milestone 42, sessão 2026-07-27): EPIC-FRONT-01 (Backend) — Persistência
PostgreSQL/Testes de Integração REST/CORS — CONCLUÍDA. Mudança de fase declarada pelo responsável:
migração funcional C#→Java considerada concluída dentro do escopo aprovado, Java passa a ser a
única fonte de verdade, prioridade agora é evolução do produto (não mais auditoria do legado). Ver
subseção "Milestone 42" (Seção 5) e **DEC-42** para o detalhamento completo.

Atualização anterior (Milestone 41, EPIC-APP2 — ver nota de continuidade acima da Seção "Roadmap
Definitivo" e **DEC-41**).

**Estratégia de trabalho mudou (2026-07-23): a unidade de trabalho não é mais milestone/épico por sessão, é o domínio da plataforma** (ver "Pivô de Estratégia: Unidade de Trabalho passa a ser o Domínio", início da Milestone 33). Nenhuma milestone obrigatória em aberto — Milestones 5 a 30 concluídas (Milestone 22 encerrada com o Incremento 22.7, wiring de produção/DI; Milestone 23 — DEC-32, primeira Macro real, AntiAFK; Milestone 24 — DEC-33, abertura da Fase 2/Macros, wiring de produção do ciclo de vida do bot ao `MotorDeTick`; Milestone 25 — Herbalismo, segunda Macro real, e DEC-34, correção de fidelidade de altura de olho; Milestone 26 — DEC-35, reabertura parcial da DEC-22 para motor de física horizontal mínimo + `GuiaDeCaminho`, execução de caminho; Milestones 27-29 — Épico Fase 2 "execução de caminho": `ComandoGoto`, `TarefaFollow`/`ComandoFollow` e `ComandoPortal`; Milestone 30 — DEC-36, Épico "Mineração", `TarefaMineracao`/`ComandoMinerar`). Milestone 31 (Sistema de Pesca/AutoFish) foi BLOQUEADA por depender de um agregado novo (`Janela`/Container) — o mesmo achado motivou um **pivô de estratégia** do responsável do projeto: em vez de continuar portando macro por macro, a Fase 2 passa a construir primeiro um roadmap de épicos de infraestrutura reutilizável (EPIC-I1 a EPIC-I13, ver "Pivô de Estratégia da Fase 2" acima da Milestone 32). **Milestone 32 — EPIC-I1 (Container Framework, DEC-37) CONCLUÍDA**, desbloqueando formalmente a Milestone 31. **Milestone 33 — Domínio Inventory/Container (auditoria + fechamento) CONCLUÍDA**: confirma Container/Item Transfer 100% cobertos, fecha Item Search/Inventory Utils, confirma Equipment sem lacuna funcional real (legado nunca veste armadura). **Milestone 34 — Domínio Interação com o Mundo (auditoria + fechamento) CONCLUÍDA**: confirma blocos/portas/alavancas/botões/placas/estruturas/uso-de-item 100% cobertos desde as Milestones 14/18/25/29/32 (nenhuma classe de porta/alavanca/botão/NPC tem precedente próprio no legado — tudo já compõe sobre o clique genérico de bloco); fecha a única lacuna real, interação/ataque manual a entidades (`UseEntityPacket`/`ComandoUseEntity`, porte de `CommandUseEntity`). **Pivô de estratégia (mesma sessão da Milestone 34, 2026-07-23): unidade de trabalho passa de "domínio auditado por sessão" para "backlog global auditado uma única vez"** — produzido um artifact externo (não commitado em `docs-reescrita`) com a matriz completa de todo o legado `Projeto Adv 2.4.5`, classificando cada macro/capacidade em 5 Épicos (ver memória `project_backlog_definitivo`). **Milestone 35 — Backlog "Composição Pura" em lote CONCLUÍDA**: todo item do Épico 1 (zero capacidade nova) e Épico 2 (retoque trivial de 1 método) implementado numa única sessão sem pausa entre itens, incluindo a macro AutoFish que motivou o bloqueio original da Milestone 31. Próximo passo: Épicos 3 (dado de altura por mob, precisa levantamento no legado), 4 (decisão de política de combate — Solk/CommandMob e UseBow) e 5 (motor de scripting, Fase 3) — nenhum atende aos critérios "zero DEC/decisão/mudança arquitetural" da Milestone 35, todos aguardam o responsável.

Histórico (Milestones 5 a 22, todas concluídas — mantido por rastreabilidade)

- Milestone 5 — Play State: fase de planejamento (DEC-19, DEC-20), Incrementos 1 a 5 (retenção de sessão/roteamento; Keep Alive/Join Game/Player Position And Look/Disconnect; Chat Message recebido; Update Health/Respawn; Inventário — Window Items/Set Slot/Held Item Change) e Incremento 6 completo (6.1 fundação de entidades, 6.2 movimentação, 6.3 velocidade, 6.4 estado visual) — ver subseções da Milestone 5.
- Milestone 7 — Modelagem do Mundo: Incrementos 7.0 (DEC-20 implementada), 7.1 (Chunk Data), 7.2 (Block Change/Multi Block Change), 7.3 (Map Chunk Bulk), 7.4 (Explosion), 7.5 (Change Game State) — ver subseções da Milestone 7.
- Milestone 8: Incremento 8.1 (DEC-21 — papel do Caso de Uso em ações iniciadas pelo bot), 8.2 (Player List `0x38`), 8.3 (Chat Enviado pelo Bot, primeiro Caso de Uso do Play State) — ver subseções da Milestone 8.
- Milestone 9: Incremento 9.1 (DEC-22 — movimentação/rotação, mutação otimista de estado), 9.2 (Player Position `0x04`), 9.3 (Player Look `0x05`) — ver subseções da Milestone 9.
- Milestone 10: Incremento 10.1 (planejamento — nenhuma DEC nova necessária), 10.2 (Swing Arm `0x0A`), 10.3 (Player Digging `0x07` — início/cancelamento/término, sem mineração automática), 10.4 (Player Block Placement `0x08`, sem lógica de inventário automático) — ver subseções da Milestone 10.
- Milestone 11: Incremento 11.1 (Movimentação e Rotação Combinadas — reaproveita `ConfirmacaoDePosicaoPacket`/Codec `0x06` conforme DEC-22, nenhuma DEC nova, nenhum Packet/Codec/Handler novo) — ver subseção da Milestone 11.
- Milestone 12: Incremento 12.1 (DEC-23 — Arquitetura de Execução de Comandos do Bot; `interfaces.comando` populado pela primeira vez; `Comando`/`ResultadoComando`/`GerenciadorDeComandos`; 8 comandos concretos sobre Casos de Uso já aprovados; nenhum pacote/Port/agregado novo) — ver subseção da Milestone 12.
- Milestone 13: Incremento 13.1 (DEC-24 — Raycast Fiel ao Legado sobre Mundo; `Mundo.tracarRaio` porta `World.RayCast`, `Bloco.solido()` porta `Blocks.IsSolid`, `SessaoDeJogo.tracarRaioParaBlocos` porta `RayCastBlocks`; correção de bounds-check em `Mundo.blocoEm`; nenhum Packet/Port/Use Case novo) — ver subseção da Milestone 13.
- Milestone 14: Incremento 14.1 (DEC-25 — Ações de Bloco com Auto-Mira; `SessaoDeJogo.olharParaBloco`/`usarItemNaMao` novos; `ComandoClicarBloco`/`ComandoQuebrarBloco`/`ComandoColocarBlocoAutoMira` portam `CommandClickBlock`/`CommandBreakBlock`/`CommandPlaceBlock` consumindo o raycast da DEC-24; nenhum Packet/Port/agregado novo) — ver subseção da Milestone 14.
- Milestone 15: Incremento 15.1 (DEC-26 — Infraestrutura de Saída de Mensagens para o Operador; `domain.bot.SaidaDoOperador` porta `ChatMessages`/`MaximumChatLines` do legado; `Bot` ganha o campo `saidaDoOperador`; `GerenciadorDeComandos.executar` ganha mensagens de fallback do `CommandManagerNew`; `ComandoAjuda`/`ComandoListarJogadores` portam `CommandHelp`/`CommandPlayerList`; nenhum Packet/Port/agregado/bounded context novo, nenhuma interface pública alterada) — ver subseção da Milestone 15.
- Milestone 16: Incremento 16.1 (DEC-27 — Infraestrutura de PathFinding: Algoritmo Puro vs. Execução por Tick; `domain.bot.PontoDeCaminho`/`BuscadorDeCaminho` portam `PathPoint`/`PathFinder`+`Path` do legado; `Mundo.criarCaminhoPara`/`SessaoDeJogo.criarCaminhoPara` novos; `PathGuide` e seus consumidores permanecem fora de escopo sem reabrir a DEC-22; branch morto `NodeType._2` e parâmetro morto `canDrown` comprovados e não portados; nenhum Packet/Port/Caso de Uso/`Comando`/bounded context novo) — ver subseção da Milestone 16.
- Milestone 17: Incremento 17.1 (DEC-28 — Calculadora de Força de Mineração: Fórmula Pura com Parâmetros Explícitos para Estado Ausente; `domain.protocol.v1_8.RegistroDeBlocos` (package-private, porte de `Blocks`/`Block` — só `Hardness`/`Material`/`HarvestTools`) e `CalculadoraDeQuebraDeBloco` (público, porte completo de `DiggingHelper`) novos; `SessaoDeJogo.estaSubmerso()` novo; nível de Efficiency/amplifier de Haste/amplifier de Mining Fatigue viram parâmetros explícitos (sentinela `-1`/`0`) por não terem estado equivalente no domínio Java ainda; nenhum Packet/Port/Caso de Uso/`Comando`/bounded context novo) — ver subseção da Milestone 17.
- Milestone 18: Incremento 18.1 (Update Sign `0x33` — porte completo de `Handler_v18.cs case 51`; `Mundo.registrarPlaca`/`placaEm` novos; World Border/Update Block Entity/Block Action/Block Break Animation confirmados sem nenhum precedente no legado após busca exaustiva; bounded context de Mundo/Milestone 7 encerrado oficialmente; sem DEC nova) — ver subseção da Milestone 18.
- Milestone 19: Incremento 19.1 (Linha de Visão contra Jogador Remoto — `SessaoDeJogo.podeVerJogador` porta `Entity.CanSeePlayer`, segundo consumidor de `Mundo.tracarRaio`/DEC-24; `CanSeeEntity` genérico para mobs permanece fora de escopo por falta de dado de altura por tipo de mob; sem DEC nova) — ver subseção da Milestone 19.
- Milestone 20: Incremento 20.1 (Entity Action `0x0B` serverbound, subconjunto sneak — `SessaoDeJogo.agachar`/`pararDeAgachar`, `ComandoAgachar`/`ComandoPararDeAgachar` novos, conforme DEC-21; `ActionID` 2/5 confirmados código morto, `ActionID` 3/4 (sprint) e `Player` bare `0x03` confirmados Tick-loop-only no legado — mesmo motivo de exclusão da DEC-22; sem DEC nova) — ver subseção da Milestone 20.
- Milestone 21: Incremento 21.1 (DEC-29 — Fundação da Engine de Execução Contínua; `domain.bot.EstadoExecucao`/ciclo de vida em `Bot` (`iniciar`/`pausar`/`retomar`/`parar`) e registro de tarefas (`TarefaContinua`/`registrarTarefa`/`removerTarefa`); `application.port.AgendadorDeTarefasPort` (segundo Port do projeto) e `infrastructure.execucao.AgendadorDeTarefasVirtualThread`/`MotorDeTick` novos; mecanismo genérico de agendamento/tick sobre Virtual Threads (DEC-03), sem nenhuma leitura/escrita de física e sem nenhuma tarefa real registrada — DEC-29 resolve explicitamente que isso não reabre DEC-22/DEC-23/DEC-27; pivô de escopo — primeira milestone desde a 4 que entrega infraestrutura de execução em vez de capacidade isolada de protocolo) — ver subseção da Milestone 21.
- Milestone 22 (ENCERRADA nesta sessão): Incremento 22.1 (DEC-30 — Propagação de Perda de Conexão), Incrementos 22.2-22.6 (DEC-31 — AutoReconnect/keep-alive/retomada de tarefas/proxy por bot/rotação de proxy) e Incremento 22.7 (Wiring de Produção/DI — `infrastructure.config` deixa de estar vazio, sem DEC nova) — ver subseção da Milestone 22.
- Milestone 23: DEC-32 — Reabertura Parcial da DEC-22 (Motor de Física Vertical Mínimo) e primeira Macro real (`TarefaAntiAFK`/`ComandoAntiAFK`), portando `CommandAntiAFK` do legado — ver subseção da Milestone 23.
- Milestone 24: DEC-33 — Abertura da Fase 2 (Macros/Automações): análise exaustiva do framework de macros do legado (`ICommand`/`Tick`, AutoMiner/AutoFish/AutoAttack/Herbalism/Solk/Follow/Portal) e inventário completo da arquitetura Java; achado central (bot criado/conectado pelos Casos de Uso reais nunca era registrado no `MotorDeTick`) corrigido via `application.port.MotorDeExecucaoPort` — ver subseção da Milestone 24.
- Milestone 25: Herbalismo (segunda Macro real, `TarefaHerbalismo`/`ComandoHerbalismo`, mesmo par toggle de AntiAFK) — porta `CommandHerbalism` do legado; novo `SelecionarSlotDaHotbarPacket`/`SessaoDeJogo.selecionarSlotDaHotbar`; **DEC-34** corrige `olharParaBloco`/`tracarRaioParaBlocos`/`podeVerJogador` para usar altura de olho (1.62) na origem do próprio bot, fiel a `Entity.PosY` do legado — ver subseção da Milestone 25.
- Milestone 26: **DEC-35** — Reabertura Parcial da DEC-22 (Motor de Física Horizontal Mínimo): `SessaoDeJogo.aplicarFisicaVertical()` generalizado para `aplicarFisica()` (colisão 3-eixos, sem assistência de "step"); novo `domain.bot.GuiaDeCaminho` (porte de `PathGuide`, execução de caminho tick a tick sobre `SessaoDeJogo.criarCaminhoPara`/DEC-27). Desbloqueia `CommandGoto`/`CommandFollow`/`CommandPortal` (dependem só de execução de caminho); `AutoMiner` continua bloqueado por um mecanismo adicional (`MoveQueue`/`Movement`/`MoveRelative` por yaw, DEC própria futura) — ver subseção da Milestone 26.
- Milestones 27-29 (Épico Fase 2 "execução de caminho", ENCERRADO): `SessaoDeJogo.caminhoAtual()`/`definirCaminho`/`limparCaminho` (novos) + `MotorDeTick` ticando o caminho ativo incondicionalmente tornam `GuiaDeCaminho` consumível; `ComandoGoto` (M27, menor consumidor real); `TarefaFollow`/`ComandoFollow` (M28, terceira Macro real, `SessaoDeJogo.olharParaEntidade`/`ListaDeJogadores.uuidPorNome`/`EntidadesDoMundo.porUuid` novos); `ComandoPortal` (M29, `domain.bot.RegistroDePortais`/`PosicaoDePortal`/`SessaoDeJogo.localizarPortal` novos, dado estático dos 25 minijogos do CraftLandia hardcoded a partir do resource do legado). Nenhuma DEC nova nas três; cálculo de caminho síncrono em todas (divergência de execução registrada, não de resultado) — ver subseções das Milestones 27 a 29.
- Milestone 30 (Épico "Mineração", ENCERRADO): **DEC-36** decide não portar `MoveQueue`/`Movement`/`MoveRelative` por yaw (sem consumidor real no modelo de execução síncrona já adotado desde a DEC-35); `TarefaMineracao`/`ComandoMinerar` (quarta Macro real) portam `AutoMiner`/`CommandMiner` — busca do bloco de menor custo em cubo de raio 8, deslocamento via `GuiaDeCaminho`, quebra via `CalculadoraDeQuebraDeBloco`/DEC-28. Fecha o item pendente da DEC-35. Zero Port/bounded context/agregado novo — ver subseção da Milestone 30.
- Milestone 31 (Sistema de Pesca/AutoFish, BLOQUEADA — Fase de Planejamento apenas): análise exaustiva de `CommandPesca`/`MacroUtils` (`Projeto Adv 2.4.5`) confirma que o núcleo de captura já está coberto por `usarItemNaMao`/DEC-25, mas 3 dos 5 estados restantes da máquina de estados (`BUSCAR_VARA`/`GUARDAR_ITENS`/parte de `REPARAR`) dependem de abrir/ler/clicar/fechar baús — capacidade inexistente hoje (`ReceptorSetSlot` já trata `windowId != 0` como no-op deliberado). Acionado o gatilho de parada "novo agregado" (`Janela`/Container) — nenhum código implementado, nenhuma DEC criada, aguarda decisão do responsável (candidata a DEC-37) — ver subseção da Milestone 31. **Desbloqueada pela Milestone 32.**
- Milestone 32 (EPIC-I1 — Container Framework, DEC-37, CONCLUÍDA): `Janela`/`JanelaDeSlots` (novo agregado + interface, `domain.bot`); 6 famílias de Packet/Codec/Handler/Evento (`OpenWindow`/`CloseWindow`/`FecharJanela`/`ClickWindow`/`ConfirmTransaction`/`ConfirmarTransacao`); `ReceptorSetSlot`/`ReceptorWindowItems` estendidos para `windowId != 0`; API completa de janela em `SessaoDeJogo` sobre indexação unificada de slot; 8 Casos de Uso; `ComandoClicarSlot`/`ComandoFecharJanela`. Genérico para qualquer Window do protocolo 1.8 (baú/fornalha/mesa de encantamento/trade/cavalo/bigorna etc), sem regra de macro. Desbloqueia a Milestone 31 e o restante do roadmap de infraestrutura da Fase 2 — ver "Pivô de Estratégia da Fase 2" e subseção da Milestone 32.
- Milestone 33 (Domínio Inventory/Container, auditoria + fechamento, CONCLUÍDA): primeira sessão sob a nova estratégia "unidade de trabalho = domínio" (ver "Pivô de Estratégia: Unidade de Trabalho passa a ser o Domínio"). Confirma Container+Item Transfer 100% cobertos desde a Milestone 32; implementa as lacunas reais restantes — `JanelaDeSlots.localizarTodosOsItens` (Item Search), `SessaoDeJogo.limparCursor` (Inventory Utils), 7 `Comando`s novos conectando Casos de Uso já aprovados sem `Comando` correspondente. Equipment/Armadura analisado e confirmado **sem nenhuma lacuna funcional** — o legado nunca veste armadura em nenhuma macro, construir auto-equipe seria inventar regra de negócio sem validação. Zero DEC nova (crescimento aditivo) — ver subseção da Milestone 33.
- Milestone 34 (Domínio Interação com o Mundo, auditoria + fechamento, CONCLUÍDA): segunda sessão sob a estratégia "unidade de trabalho = domínio". Confirma blocos (clicar/quebrar/colocar), estruturas (portal) e uso de item/uso de item em bloco 100% cobertos desde as Milestones 14/29; confirma que portas/alavancas/botões/NPCs **não têm nenhum precedente próprio no legado** (toda interação física desse tipo compõe sobre o clique genérico de bloco, já portado) — não implementados, por ausência de regra a preservar, não por lacuna. Fecha a única lacuna real: `UseEntityPacket`/`SessaoDeJogo.interagirComEntidade`/`ComandoUseEntity` (porte de `CommandUseEntity`, interação/ataque manual e pontual a uma entidade). Zero DEC nova (crescimento aditivo) — ver subseção da Milestone 34.
- Milestone 35 (Backlog "Composição Pura" em lote, CONCLUÍDA): pivô de estratégia — unidade de trabalho passa de domínio-por-sessão para backlog-global-auditado-uma-vez (artifact externo com 5 Épicos, memória `project_backlog_definitivo`). Implementa todo o Épico 1 (zero capacidade nova) e Épico 2 (retoque trivial) numa única sessão contínua, sem pausa entre itens: `ComandoLimparChat`, `SessaoDeJogo.soltarItemEmUso`, `TarefaLargarTudo`/`ComandoLargarTudo` (DropAll), `CasoDeUsoSelecionarSlotDaHotbar`/`ComandoClicarItemDaHotbar` (HotbarClick), `TarefaTwerk`/`ComandoTwerk`, `CasoDeUsoReconectarBot`/`ComandoReconectar` (Reco), e **`TarefaAutoFish`/`ComandoAutoFish`** (Solk/CommandPesca — a macro que motivara o bloqueio original da Milestone 31, agora composição pura). Achado de fidelidade: `"$"` no início de uma mensagem do legado é auto-invocação LOCAL de outro comando (`CmdManager.RunCommand`), não chat real — muda a leitura de `"$move j 40"` em toda a base Solk. Zero DEC nova — ver subseção da Milestone 35.

Candidatos (ver subseções correspondentes na Seção 5 para objetivo/escopo/dependências/riscos/critérios de aceite completos de cada um; nenhuma DEC pendente bloqueia qualquer um deles, exceto onde indicado)

- Altura de olho por tipo de mob (`EntityProperty.Height` do legado) — desbloquearia um `CanSeeEntity` genérico sobre `Mundo.tracarRaio` para combate contra mobs (jogadores já cobertos desde a Milestone 19 via `podeVerJogador`/constante `1.62`; levantar a altura por tipo de mob é um levantamento de dados maior, avaliado e deixado de fora desde a Milestone 13). Independente da frente de execução contínua.
- Criptografia AES-CFB8/modo online — candidata desde o encerramento da Milestone 4, ainda não escolhida (desbloquearia também um `ComandoReco` de verdade contra servidores em modo online).
- Decisão de transporte para o operador (CLI ou API, DEC-02 ainda não decidida) — único jeito de efetivamente consumir `SaidaDoOperador` (Milestone 15) e o ciclo de vida do bot (Milestone 21/22/23) fora dos testes; o wiring de produção em si já está pronto desde o Incremento 22.7 (`infrastructure.config`), só falta o transporte que crie/conecte um bot de verdade.
- Um `Comando`/macro de mineração completa (`AutoMiner`) que consuma `CalculadoraDeQuebraDeBloco` (Milestone 17) + `GuiaDeCaminho` (Milestone 26) — a execução de caminho já existe desde a DEC-35, mas `AutoMiner` é o único consumidor identificado que também usa `Player.MoveQueue`/`Movement`/`MoveRelative` por yaw (caminhada exploratória própria, distinta de `GuiaDeCaminho`) — mecanismo ainda não portado, exige nova DEC de escopo próprio antes de qualquer implementação (não herda a aprovação da DEC-35, que cobriu só execução de caminho).
- Suporte a NBT real em `ItemStack` (encantamentos/nome customizado/lore) — desbloquearia `nivelEficiencia` real em `CalculadoraDeQuebraDeBloco.forcaDoJogador` em vez do parâmetro `0` (Milestone 17/DEC-28); também serviria outros consumidores (crafting, display).
- Rastreamento de efeitos do próprio bot (Haste/Mining Fatigue e demais) — hoje `EntidadeRemota.efeitosAtivos` só cobre entidades remotas (`ReceptorEntityEffect`, Milestone 5.6.4); desbloquearia `amplifierCeleridade`/`amplifierFadiga` reais em `CalculadoraDeQuebraDeBloco.forcaDoJogador` em vez do sentinela `-1` (Milestone 17/DEC-28) — decisão arquitetural própria (`SessaoDeJogo` vs. reaproveitar `EntidadeRemota`), ainda não tomada.
- **AutoFish** (`CommandPesca`) — **DESBLOQUEADA pela Milestone 32/DEC-37 e pela Milestone 34**: o agregado `Janela`/Container, os 6 Packets de container e a interação com bloco/uso de item já existem (`abrirJanela`/`clicarSlot`/`shiftClique`/`moverItem`/`fecharJanela` da Milestone 32; `usarItemNaMao`/`ComandoClicarBloco` da Milestone 14). Candidata pronta para implementação direta — os 3 estados que dependiam de container (`BUSCAR_VARA`/`GUARDAR_ITENS`/parte de `REPARAR`) e a placa de compra/venda de linha (right-click genérico) agora só precisam compor primitivas já prontas, sem nova infraestrutura.
- **Solk** (análise completa na Milestone 24) — o alias legado `"solk"` mapeia para uma macro incompleta no próprio C# (`CommandMobTeleport`, loop de ataque comentado); a macro "Solk" completa de fato é `CommandMob`, que inclui um loop de ataque sustentado contra mobs. **Pergunta em aberto, não decidida nesta sessão:** esse loop de combate conta como "combate/automação (KillAura e afins)", já permanentemente fora de escopo por política? Requer decisão explícita do responsável antes de qualquer análise de implementação.
- Combate/automação (`CommandKillAura`/AutoAttack e afins) permanece fora de escopo por política do projeto, não por dependência técnica (ver "Fora de escopo" abaixo).

Candidatos encerrados nas Milestones 18-26 (não reabrir sem novo motivo): checagem de linha de visão contra entidade (Milestone 19); `Player` bare/Entity Action → `ComandoSneak` (Milestone 20 — subconjunto sneak implementado, leave bed/jump boost são código morto comprovado, sprint e `Player` bare são Tick-loop-only, todos fora de escopo por DEC-22, não por lacuna de precedente); restante da Milestone 7 (Milestone 18 — bounded context de Mundo encerrado oficialmente); fundação de execução contínua/scheduler/tick engine (Milestone 21 — DEC-29 entrega o mecanismo); ciclo de vida contínuo — reconexão/keep-alive/proxy/retomada de tarefas/wiring de produção (Milestone 22, completa, incluindo 22.7); wiring de produção do ciclo de vida do bot ao `MotorDeTick` (Milestone 24 — DEC-33); **Herbalismo** (Milestone 25 — `TarefaHerbalismo`/`ComandoHerbalismo`, DEC-34); **motor de física horizontal mínimo + execução de caminho** (Milestone 26 — `SessaoDeJogo.aplicarFisica()`/`GuiaDeCaminho`, DEC-35 — reabre a DEC-22 só para este subconjunto; não reabrir de novo para `CommandGoto`/`Follow`/`Portal`, que já podem consumir `GuiaDeCaminho` diretamente); **`ComandoGoto`/`TarefaFollow`+`ComandoFollow`/`ComandoPortal`** (Milestones 27-29 — os três consumidores de produção de `GuiaDeCaminho`, nenhuma DEC nova; não reabrir para os mesmos comandos, só para features novas como `AutoMiner`).

Fora de escopo por política do projeto (não candidatos): combate/automação (KillAura e afins), lógica de inventário automático — não reconsiderados a cada milestone, permanecem bloqueados por decisão de escopo, não por dependência técnica. Água/lava/escada/sprint e o mecanismo `Player.MoveQueue`/`Movement`/`MoveRelative` por yaw (exclusivo de `AutoMiner`) continuam bloqueados pela DEC-22 (não reabertos — a DEC-32/Milestone 23 abriu exceção só para o eixo vertical de `TarefaAntiAFK`, e a DEC-35/Milestone 26 estendeu a exceção só para colisão horizontal via `GuiaDeCaminho`/execução de caminho); qualquer novo pedido que precise desses restantes (ex.: `AutoMiner` completo, KillAura) exige nova análise "isso reabre a DEC-22, e para qual escopo?", não herda as aprovações já concedidas. Desde a DEC-29 (Milestone 21), o *mecanismo* de scheduler/tick engine deixou de estar nesta lista — só o *conteúdo* físico/automático não coberto por uma DEC própria permanece bloqueado.

---

# 11. Próximo Prompt Esperado

Resumo da próxima atividade que deverá ser executada pela IA.

**Atualização mais recente (Milestone 42, sessão 2026-07-27): EPIC-FRONT-01 — CONCLUÍDA.** Sessão
marcou virada de fase: migração funcional C#→Java considerada concluída dentro do escopo aprovado;
a partir de agora, não auditar o legado C# salvo pedido explícito ou bug específico que exija
comparação, e priorizar evolução do produto. Escopo pedido: substituir persistência in-memory de
Conta/Servidor/Proxy por PostgreSQL usando as portas já existentes (`RepositorioDeContas`/
`RepositorioDeServidores`) mais uma porta nova só onde não havia (`RepositorioDeProxies`, fechando a
lacuna de `PoolDeProxies` nunca ter tido persistência), adapters JPA sem alterar o domínio,
migrations Flyway, testes de integração REST (primeiros do projeto), validação de
serialização/CORS/WebSocket, sem novos endpoints além do indispensável. Entregue: 3 entidades
JPA + 3 `JpaRepository` + 3 adapters (`infrastructure.persistence`/`.jpa`), migration
`V1__contas_servidores_proxies.sql`, `RepositorioDeProxies` (porta nova) com write-through em
`ProxyController`/`PoolDeProxies`, `EntityScan`/`EnableJpaRepositories` explícitos em
`AdvancedBotApplication` (achado técnico: `AutoConfigurationPackages` não segue `scanBasePackages`
customizado), 6 testes de integração REST novos (`ContaControllerTest`/`ServidorControllerTest`/
`ProxyControllerTest`, contra PostgreSQL real dedicado a testes via `@Transactional`), e correção de
um bug de CORS em `ApiKeyFilter` (preflight `OPTIONS` era barrado com 401 antes do CORS do Spring
MVC rodar, quebrando toda requisição não-simples a partir de um browser). Zero DEC de
protocolo/domínio reaberta, zero contrato de API existente alterado (só o meio de persistência por
trás mudou) — ver **DEC-42** para o detalhamento completo. 1098→1104 testes automatizados (+6), 0
falhas, 0 erros, 3 skipped deliberadamente; validação manual adicional via `mvn spring-boot:run`
real contra PostgreSQL de desenvolvimento (Flyway migra e depois confirma idempotência; CRUD via
`curl`; preflight CORS confirmado corrigido; WebSocket `/ws/events` conectado via Node.js nativo).
**Próximo passo: decisão do responsável** entre continuar o roadmap de épicos de fechamento
(EPIC-I2 — Motor de Orquestração de Comandos, único épico de Foundation API remanescente, ver
"Roadmap Definitivo Pós-Milestone 39" acima) ou abrir uma nova frente de evolução de produto agora
que o backend está pronto para produção (persistência de bots/`GerenciadorDeBots`, autenticação
multi-usuário, o próprio frontend React).

Atualização anterior (Milestone 35, sessão 2026-07-23): Backlog "Composição Pura" em lote — CONCLUÍDA.** Terceira mudança de estratégia do responsável no mesmo dia: em vez de auditar domínio por domínio (Milestones 33-34), produzir um artifact único com o backlog GLOBAL de todo o legado `Projeto Adv 2.4.5` (matriz macro-por-macro + catálogo de capacidades + 5 Épicos, ver memória `project_backlog_definitivo` — não commitado em `docs-reescrita` por instrução explícita), e então implementar TUDO que se enquadra em "zero DEC nova, zero decisão do responsável, zero mudança arquitetural, infraestrutura já pronta" (Épicos 1 e 2) numa única sessão contínua, sem tratar cada item como milestone/macro isolada. Entregue: `SaidaDoOperador.limpar()`/`ComandoLimparChat` (ClearChat); `SessaoDeJogo.soltarItemEmUso` (Player Digging status 5, sem consumidor de produção ainda); `TarefaLargarTudo`/`ComandoLargarTudo` (DropAll, laço pautado sobre `largarItem`/M32); `CasoDeUsoSelecionarSlotDaHotbar` (novo)/`ComandoClicarItemDaHotbar` (HotbarClick); `TarefaTwerk`/`ComandoTwerk`; `CasoDeUsoReconectarBot` (novo)/`ComandoReconectar` (Reco, sem `[IP:porta]` alternativo); e a peça maior, **`TarefaAutoFish`/`ComandoAutoFish`** (Solk/CommandPesca, "solkpesca" — a macro AutoFish que motivara o bloqueio original da Milestone 31, hoje pura composição sobre Container/M32 + Interação com o Mundo/M34). Achado de fidelidade não documentado antes: `MinecraftClient.SendMessage` do legado intercepta qualquer mensagem começada com `"$"` e a roteia como auto-invocação LOCAL de outro `ICommand` (`CmdManager.RunCommand`), nunca como chat real — isso muda a leitura de `"$move j 40"` (usado em quase toda transição de estado das macros Solk) de "comando de chat" para "chamada local a `CommandMove`", que por sua vez depende do mecanismo `MoveQueue`/`Movement` por yaw (DEC-36, não portado). Substituído por `pular()+aplicarFisica()` (mesma composição de `TarefaAntiAFK`/DEC-32). Duas divergências de fidelidade documentadas diretamente no código de `TarefaAutoFish` (não nesta seção, para não duplicar): `deveGuardarItem` sentinela `false` (depende de NBT de item, lacuna já registrada desde a Milestone 17/DEC-28) e o fallback "sem baú com espaço" simplificado (só o primeiro candidato é tentado, sem o `MoveQueue` do legado). Zero DEC nova em todo o lote, zero Packet/Port/agregado/bounded context novo, zero interface pré-existente alterada. 1007→1036 testes automatizados (+29), 0 falhas, 0 erros, 3 skipped deliberadamente. **Backlog de composição pura esgotado** — só restam os Épicos 3 (dado de altura por mob, precisa levantamento no legado), 4 (decisão de política de combate, Solk/CommandMob e o resto de UseBow) e 5 (motor de scripting, Fase 3 greenfield), nenhum dos quais atende aos critérios desta sessão. Próximo passo: decisão do responsável sobre qual Épico (3, 4 ou 5) atacar em seguida.

Atualização anterior (Milestone 34, sessão 2026-07-23): Domínio Interação com o Mundo — auditoria + fechamento — CONCLUÍDA.** Segunda sessão sob a estratégia "unidade de trabalho = domínio" (ver Milestone 33). Escopo pedido: auditar toda interação genérica do bot com o mundo (blocos, portas/alavancas/botões, placas, NPCs, entidades, uso de item, uso de item em bloco, estruturas), classificar em já implementado/parcial/ausente, implementar só primitivas reutilizáveis (nunca macro), extrair API só quando 3+ consumidores no legado a usam. Auditoria confirmou blocos (clicar/quebrar/colocar com auto-mira, Milestone 14), estruturas (portal, Milestone 29) e uso de item/uso de item em bloco (`usarItemNaMao`/Milestone 14) 100% cobertos. Achado central: **portas/alavancas/botões/NPCs (trade) não têm nenhum precedente próprio no legado** — busca exaustiva em `AdvancedBot.Client.Commands/*.cs` não encontra nenhuma classe dedicada; toda interação desse tipo no legado (`MacroUtils.RightclickBlock`/`openChest`, usada por placas de compra/venda) é só right-click genérico de bloco, já coberto desde a Milestone 14 — não implementado por ausência de regra a preservar (CLAUDE.md proíbe inventar regra sem precedente), não por lacuna real. Escrita de placa (Update Sign SERVERBOUND) também sem nenhum call site no legado — não implementada. Única lacuna real encontrada e fechada: **`PacketUseEntity`/`CommandUseEntity.cs`** (interagir ou atacar uma entidade uma única vez) nunca tinha sido portado — novo `UseEntityPacket`/`UseEntityCodec` (PLAY `0x02` SERVERBOUND), `SessaoDeJogo.interagirComEntidade`, `CasoDeUsoOlharParaEntidade` (faltava o wrapper para `olharParaEntidade`, que já existia desde a Milestone 28 só com consumidor direto em `TarefaFollow`), `CasoDeUsoInteragirComEntidade`, `interfaces.comando.ComandoUseEntity` (porte fiel de `CommandUseEntity` — nick ou `@any`/sem argumento para o mais próximo visível, modo atacar/interagir com a mesma inversão de índice do legado). Omissão documentada: o filtro `IsBot` do legado (exclui outros bots do mesmo processo da busca `@any`) não tem equivalente — não existe registro de múltiplas instâncias de `Bot` no domínio Java, mesma categoria da omissão do toggle `"retard"` em `ComandoFollow` (Milestone 28). Macros avaliadas só como consumidoras, não implementadas: `CommandUseBow`/`CommandKillAura` (combate, fora de escopo por política), `CommandTwerk`/`CommandRetard`/`CommandInvCaptcha`/`CommandReco` (compõem sobre primitivas já existentes ou pertencem a outro domínio). Zero DEC nova (crescimento aditivo, mesmo padrão da DEC-19/DEC-21). 994→1007 testes automatizados (+13), 0 falhas, 0 erros, 3 skipped deliberadamente. **Domínio Interação com o Mundo esgotado** — qualquer macro futura que interaja com bloco/entidade/estrutura (AutoFish/M31, um eventual porte não-combativo, uma futura automação de trade) é pura composição, sem infraestrutura nova. Próximo passo: escolha do responsável do próximo domínio a esgotar (AutoFish/M31 agora duplamente desbloqueada — Container + Interação com o Mundo — é a candidata mais óbvia).

Atualização anterior (Milestone 33, sessão 2026-07-23): Domínio Inventory/Container — auditoria + fechamento — CONCLUÍDA. Primeira sessão sob a nova estratégia do responsável: a unidade de trabalho deixa de ser milestone/épico por sessão e passa a ser o **domínio da plataforma** (pode conter vários épicos; esgotar tudo que houver reaproveitamento arquitetural antes de parar; nunca implementar macro diretamente, só capacidade reutilizável; documentação/testes só ao final do domínio). Auditoria completa (C# legado vs. árvore Java atual, não só o mapeamento pré-Milestone-32) confirmou que Container+Item Transfer já estavam 100% cobertos desde a DEC-37 (Milestone 32). Lacunas reais encontradas e fechadas: `JanelaDeSlots.localizarTodosOsItens` (Item Search, porte de `CommandMob.buscarSlotsCom`); `SessaoDeJogo.limparCursor` (Inventory Utils, porte generalizado de `CommandMob.limparCursor`); 7 `Comando`s novos (`ShiftClique`/`TrocarSlots`/`PegarItem`/`LargarItem`/`MoverItem`/`ConfirmarTransacao`/`LimparCursor`) conectando Casos de Uso que já existiam desde a Milestone 32 mas não tinham comando correspondente. Equipment/Armadura analisado exaustivamente e confirmado **sem nenhuma lacuna funcional** — o legado nunca veste armadura em nenhuma macro (só há precedente de auto-seleção para ferramentas, `MacroUtils.selecionarMelhorItem`, sem equivalente para armadura); vestir/trocar armadura manualmente já funciona hoje via os primitivos genéricos de clique sobre os slots 5-8. Construir uma lógica de auto-equipe-melhor-armadura seria inventar regra de negócio sem validação no legado — registrado como candidato dependente de decisão do responsável, não como bloqueio. Zero DEC nova (crescimento aditivo, mesmo padrão da DEC-19). 963→994 testes automatizados (+31), 0 falhas, 0 erros, 3 skipped deliberadamente. **Domínio Inventory/Container esgotado** — qualquer macro futura (AutoFish/M31, Solk, reparo) que precise de container/transferência/busca/limpeza de cursor é pura composição, sem infraestrutura nova. Próximo passo: escolha do responsável do próximo domínio a esgotar.

Atualização anterior (Milestone 32, sessão 2026-07-23): EPIC-I1 — Container Framework (DEC-37) — CONCLUÍDA, desbloqueia a Milestone 31. O bloqueio da Milestone 31 (análise exaustiva de `CommandPesca.cs`/`MacroUtils.cs`, achado: 3 dos 5 estados restantes da máquina dependem de abrir/ler/clicar/fechar baús, capacidade inexistente) motivou um pivô de estratégia do responsável: em vez de continuar porte-macro-por-macro, a Fase 2 primeiro constrói um roadmap de épicos de infraestrutura reutilizável (ver "Pivô de Estratégia da Fase 2", mapeamento completo de primitivas do legado agrupadas em ~24 capacidades arquiteturais com prioridade/risco/DEC por capacidade). Instruído a implementar o épico raiz (EPIC-I1) imediatamente, em sessão única. Entregue: `domain.bot.Janela`/`JanelaDeSlots` (novo agregado + interface, genérico para qualquer Window do protocolo 1.8 — não só baú); 6 famílias de Packet/Codec/Handler/Evento (`OpenWindow` CB 0x2D, `CloseWindow` CB 0x2E/`FecharJanela` SB 0x0D, `ClickWindow` SB 0x0E, `ConfirmTransaction` CB 0x32/`ConfirmarTransacao` SB 0x0F — pares CB/SB em classes distintas por direção, mesmo precedente da DEC-16); `ReceptorSetSlot`/`ReceptorWindowItems` estendidos para `windowId != 0` (fecha a lacuna documentada desde a Milestone 5.5/31); API completa de janela em `SessaoDeJogo` (`abrirJanela`/`fecharJanela`/`clicarSlot`/`shiftClique`/`trocarSlots`/`pegarItem`/`largarItem`/`moverItem`/`confirmarTransacao`/`localizarItemNaJanela`/`localizarEspacoLivreNaJanela`) sobre uma indexação UNIFICADA de slot (janela + inventário do bot numa só numeração, eliminando por construção a classe de bug `isChestOpen`/argumento-trocado que existia nas 4 variantes quase-duplicadas de `MacroUtils.moverItemXXX` do legado); simulação local de clique fiel a `Inventory.Click` (com correção do bug `ItemStack.IsSameItem` do legado); 8 Casos de Uso; `ComandoClicarSlot`(porte de `CommandInvClick`)/`ComandoFecharJanela`. Decisões documentadas na DEC-37: cursor de clique por sessão (não `static` como no legado), `trocarSlots`/mode 2 sem precedente no legado (adição aditiva), `esperarJanela`/`esperarFechamento` do pedido original implementados como consulta não-bloqueante (`janelaAtual()`) em vez de espera síncrona (violaria a restrição de não-paralelismo do `MotorDeTick`), `Window Property`/`Creative Inventory Action` deliberadamente fora de escopo (sem precedente de uso real no legado). Zero interface pré-existente alterada. 865→963 testes automatizados (+98), 0 falhas, 0 erros, 3 skipped deliberadamente. Próximo passo: escolha do responsável entre retomar a Milestone 31 (AutoFish, agora desbloqueada) ou continuar o roadmap de infraestrutura (EPIC-I2 em diante) — vale checar antes quais EPIC-I* já estão de fato cobertos pelas Milestones 23-30 (física/pathfinding/targeting/follow/portal), já que o mapeamento de primitivas foi feito só contra o C# legado, não contra a árvore Java atual. Demais candidatos inalterados (ver Seção 10): `Solk`/`CommandMob` segue com a pergunta em aberto sobre o loop de combate; decisão de transporte CLI/API (DEC-02), altura de olho por tipo de mob, criptografia/modo online, NBT real em `ItemStack`, rastreamento de efeitos do próprio bot.

Histórico (Milestones 5 a 29 concluídas — ver Seção 5 (subseções por milestone) e Seção 10 (histórico condensado) acima. **Épico Fase 2 "execução de caminho" (Milestones 27-29) encerrado nesta sessão: `ComandoGoto`, `TarefaFollow`/`ComandoFollow` e `ComandoPortal` — os três consumidores de produção de `GuiaDeCaminho`/DEC-35 previstos ao final da Milestone 26.** Escopo desta sessão: `SessaoDeJogo.caminhoAtual()`/`definirCaminho`/`limparCaminho` (novos) tornam o caminho ativo persistível por bot; `infrastructure.execucao.MotorDeTick.tick()` passa a tickar esse caminho incondicionalmente (mesmo ponto onde o legado tica `CurrentPath`, antes do laço de `TarefaContinua`) — mecanismo de infraestrutura compartilhado, não uma `TarefaContinua`. `ComandoGoto` (M27, porte de `CommandGoto`) prova o ciclo `Comando → GuiaDeCaminho → SessaoDeJogo`. `TarefaFollow`/`ComandoFollow` (M28, terceira Macro real, porte de `CommandFollow`) segue jogador remoto olhando para ele (`SessaoDeJogo.olharParaEntidade`, novo) e recalculando o caminho a cada 2.5 blocos de deslocamento do alvo, resolvido via `ListaDeJogadores.uuidPorNome`/`EntidadesDoMundo.porUuid` (novos) — toggle via registro/remoção de `TarefaContinua`, mesmo padrão de AntiAFK/Herbalismo. `ComandoPortal` (M29, porte de `CommandPortal`) localiza portais do servidor CraftLandia via `domain.bot.RegistroDePortais`/`PosicaoDePortal`/`SessaoDeJogo.localizarPortal` (novos — dado estático dos 25 minijogos do resource `Resources.portals` do legado, decodificado e hardcoded, sem parser JSON/dependência nova) ou por varredura de blocos (`"**"`). Nenhuma DEC nova nas três milestones (mesmo padrão já previsto na Seção 10 ao final da Milestone 26); cálculo de caminho síncrono nos três consumidores — divergência de execução registrada, não de resultado (o legado despacha via `Task.Factory.StartNew`, sem precedente de wrapper assíncrono no domínio Java desde a DEC-27); omissão registrada do toggle `"retard"` do legado (`CommandFollow`/`CommandPortal` o desligam, mas `CommandRetard` nunca foi portado). 819→856 testes automatizados (+37: +8 Goto, +17 Follow, +12 Portal), 0 falhas, 0 erros, 3 skipped deliberadamente; zero interface já aprovada com assinatura alterada. Próximo passo: nenhuma milestone obrigatória em aberto — escolha do responsável entre os candidatos independentes da Seção 10: **`AutoMiner`** (exige DEC própria nova para `MoveQueue`/`Movement`/`MoveRelative` por yaw, não herdada da DEC-35 nem das Milestones 27-29); **AutoFish/`CommandPesca`** (sem bloqueio de DEC-22, mas precisa de suporte a container/janela, lacuna nova ainda não coberta por DEC); **Solk** (pergunta em aberto — o loop de ataque contra mobs de `CommandMob` conta como "combate/automação" já excluído por política?); mais os candidatos já registrados antes da Fase 2 (decisão de transporte CLI/API — DEC-02 —, altura de olho por tipo de mob, criptografia/modo online, NBT real em `ItemStack`, rastreamento de efeitos do próprio bot). Riscos documentados, não resolvidos: `MotorDeTick`/`GerenciadorDeReconexao` seguem sequenciais por ciclo, sem paralelismo por bot/tarefa (Seção 5, Milestone 22); interação entre `EstadoExecucao.PAUSADO` e reconexão automática não analisada (DEC-33, Consequências/Negativas); ausência de assistência de "step" em `aplicarFisica()` pode exigir revisão se um consumidor real (`AutoMiner`/`Follow` sobre terreno irregular) provar que o pulo constante é um problema prático (DEC-35, Consequências/Negativas); cálculo de caminho síncrono em `ComandoGoto`/`TarefaFollow`/`ComandoPortal` pode introduzir latência perceptível por ciclo de `MotorDeTick` se `BuscadorDeCaminho` se mostrar custoso em mapas grandes/muitos bots simultâneos (nenhum wrapper assíncrono existe no domínio Java hoje).

---

# 12. Histórico de Execuções

| Data | Milestone | Resultado |
|-------|-----------|-----------|
| 2026-07-14 | Fundação da Plataforma | ✔ |
| 2026-07-15 | Correção Java 21 | ✔ |
| 2026-07-15 | Milestone 3 (incremento 1): Bot, Value Objects e CasoDeUsoCriarBot | ✔ |
| 2026-07-15 | Milestone 3 (incremento 2): EstadoSessao e SessaoBot | ✔ |
| 2026-07-15 | Milestone 3 (incremento 3, encerramento): CasoDeUsoConectarBot e CasoDeUsoDesconectarBot | ✔ |
| 2026-07-15 | Milestone 4 (incremento 1): Arquitetura da Camada de Comunicação (DEC-13) | ✔ |
| 2026-07-15 | Milestone 4 (incremento 2): Packets concretos de Handshake e Login Start 1.8 (DEC-14) | ✔ |
| 2026-07-15 | Milestone 4 (incremento 3): Conclusão da modelagem do estado LOGIN 1.8 — Encryption Request/Response, Set Compression, Login Success (DEC-15, DEC-16) | ✔ |
| 2026-07-15 | Milestone 4 (incremento 4): Infraestrutura de Transporte TCP — CodificadorDeFrame, DecodificadorDeFrame, TransporteSocket (Virtual Threads, DEC-03) | ✔ |
| 2026-07-16 | Milestone 4 (incremento 5): Validação integrada da pipeline de protocolo — PacketHandlers concretos, Eventos e ProtocolDispatcher testado | ✔ |
| 2026-07-16 | Milestone 4 (incremento 6): Integração Application ↔ Infraestrutura de comunicação — AdaptadorConexaoBotV1_8, ConexaoMinecraft.avancarEstado (DEC-17) | ✔ |
| 2026-07-16 | Milestone 4 (incremento 7A): Fábrica de produção (Socket real); Handshake validado localmente e contra servidor Minecraft 1.8 real (Olimpo/Craftlandia) | ✔ |
| 2026-07-16 | Milestone 4 (incremento 7B): Fluxo real de LOGIN (Handshake+LoginStart) via AdaptadorConexaoBotV1_8; correção de EstadoConexao->PLAY; validado contra Olimpo/Craftlandia (respondeu SetCompression) | ✔ |
| 2026-07-16 | Milestone 4 (incremento 7C): Robustez do TransporteSocket.readLoop — RuntimeException de decodificação encerra a Virtual Thread com segurança e libera recursos | ✔ |
| 2026-07-16 | Milestone 4 (incremento 8A): Fundação de compressão zlib — ConexaoMinecraft.ativarCompressao (DEC-18), CodificadorDeFrame/DecodificadorDeFrame com Deflater/Inflater, isolado do fluxo de LOGIN | ✔ |
| 2026-07-16 | Milestone 4 (incremento 8B): Integração da compressão ao fluxo de LOGIN — AdaptadorConexaoBotV1_8 ativa compressão ao receber SetCompression e continua o login normalmente | ✔ |
| 2026-07-16 | Milestone 4 (incremento 8C): Validação manual contra Olimpo/Craftlandia pelo caminho oficial completo — modo OFFLINE, threshold=1024, connect() CONNECTED (primeira conexão real ponta a ponta bem-sucedida) | ✔ |
| 2026-07-16 | Milestone 4: ENCERRAMENTO OFICIAL — 12 incrementos, 6 DECs (13–18), 110 testes automatizados, 3 validações manuais contra servidor real, auditoria arquitetural sem bloqueadores | ✔ |
| 2026-07-16 | Milestone 5 (Fase de Planejamento): desenho arquitetural do Play State apresentado, revisado e aprovado pelo responsável do projeto | ✔ |
| 2026-07-16 | Milestone 5 (Fase de Planejamento): DEC-19 (Retenção da Sessão de Jogo e Roteamento de Eventos) e DEC-20 (Tolerância a Pacotes PLAY Não Registrados) formalizadas; Incremento 1 autorizado | ✔ |
| 2026-07-16 | Milestone 5 (incremento 1): SessaoDeJogo, ReceptorDeEvento, RoteadorDeEventos; ConexaoBotPort.connect() retornando SessaoDeJogo (DEC-19) | ✔ |
| 2026-07-16 | Milestone 5 (incremento 2): Keep Alive/Resposta Keep Alive, Join Game, Player Position And Look/Confirmação de Posição, Disconnect Play — Packets/Codecs/Handlers/Eventos/Receptores | ✔ |
| 2026-07-16 | Milestone 5 (incremento 3): Chat Message recebido do servidor — ChatMessagePacket/Codec/Handler/EventoChatMessage/ReceptorChatMessage, SessaoDeJogo.registrarMensagemDeChat; nenhum Caso de Uso/Port/DEC novo; 184 testes automatizados, 0 falhas | ✔ |
| 2026-07-16 | Milestone 5 (incremento 4): Estado de vida do jogador — UpdateHealthPacket/Codec/Handler/Evento/Receptor (PLAY 0x06 CLIENTBOUND) e RespawnPacket/Codec/Handler/Evento/Receptor (PLAY 0x07 CLIENTBOUND); SessaoDeJogo.atualizarVida/registrarRespawn; ambiguidade de id 0x06 resolvida por leitura do C#; nenhum agregado/Caso de Uso/Port/DEC novo; 206 testes automatizados, 0 falhas | ✔ |
| 2026-07-16 | Milestone 5 (incremento 5): Inventário do jogador — InventarioDoJogador (novo agregado dedicado, não bounded context), ItemStack/ItemStackCodec (NBT consumido com segurança, nunca exposto), WindowItemsPacket (0x30)/SetSlotPacket (0x2F)/HeldItemChangePacket (0x09), todos PLAY/CLIENTBOUND; nenhuma DEC nova; 241 testes automatizados, 0 falhas | ✔ |
| 2026-07-18 | Milestone 5 (Fase de Planejamento do Incremento 6): desenho arquitetural das Entidades do Mundo apresentado, revisado e aprovado — correção estrutural sobre Entity/EntityManager do C#, mapeamento completo dos pacotes PLAY de entidade, modelagem proposta (EntidadeRemota/EntidadesDoMundo, sem novo agregado raiz/EntityManager/bounded context); escopo adicional aprovado (Incremento 6.4) | ✔ |
| 2026-07-18 | Milestone 5 (incremento 6.1): Fundação do ciclo de vida de entidades — EntidadeRemota/EntidadeJogadorRemoto/EntidadeMob/EntidadesDoMundo (domain.bot); SpawnPlayerPacket (0x0C)/SpawnMobPacket (0x0F)/DestroyEntitiesPacket (0x13), todos PLAY/CLIENTBOUND; divergências documentadas (coleção única, headYaw dedicado, rotação simétrica no spawn de mob, divisão double correta); nenhuma DEC nova; 281 testes automatizados, 0 falhas | ✔ |
| 2026-07-18 | Milestone 5 (incrementos 6.2 e 6.3): Movimentação e velocidade de entidades — EntityRelativeMovePacket (0x15)/EntityLookPacket (0x16)/EntityLookAndRelativeMovePacket (0x17)/EntityTeleportPacket (0x18)/EntityHeadLookPacket (0x19)/EntityVelocityPacket (0x12), todos PLAY/CLIENTBOUND; bug de Entity Look (C# só via PlayerManager.Players) corrigido por construção; bypass de knockback do C# deliberadamente não portado; entidade desconhecida tratada como no-op; nenhuma DEC nova; 327 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 5 (incremento 6.4, encerramento do Incremento 6): Estado visual das entidades — EntityEquipmentPacket (0x04)/AnimationPacket (0x0B)/EntityStatusPacket (0x1A)/EntityEffectPacket (0x1D)/RemoveEntityEffectPacket (0x1E), todos PLAY/CLIENTBOUND; 4 dos 5 pacotes sem precedente no C#, validados contra a especificação oficial do protocolo 47; Entity Effect aplicado a qualquer entidade rastreada (C# só ao próprio bot); Entity Metadata (0x1C) deliberadamente não implementado como pipeline próprio, coberto pela DEC-20; nenhuma DEC nova; 374 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 7 (Fase de Planejamento): desenho arquitetural da Modelagem do Mundo (Chunks e Blocos) apresentado, revisado e aprovado — mapeamento dos pacotes PLAY candidatos ao bounded context Mundo, modelagem proposta (Mundo/Chunk/Bloco, sem novo bounded context/Port/Caso de Uso); achado crítico: DEC-20 aprovada mas nunca implementada, registrada como pré-requisito bloqueante (Incremento 7.0) | ✔ |
| 2026-07-20 | Milestone 7 (incremento 7.0): Implementação integral da DEC-20 — PacoteNaoRegistradoException (infrastructure.protocol); RegistroDePacotesV1_8 lançando o tipo específico; TransporteSocket.readLoop descartando pacote PLAY não registrado (log WARN + continua) em vez de encerrar a conexão, HANDSHAKING/STATUS/LOGIN inalterados; nenhuma DEC nova (execução de decisão já aprovada); 377 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 7 (incrementos 7.1 e 7.2): Fundação do Mundo e mutação de blocos — domain.bot.Mundo/Chunk/SecaoDeChunk/PosicaoDeChunk (novo agregado interno de SessaoDeJogo, não bounded context); domain.protocol.v1_8.Bloco (Value Object, id de 12 bits preservado sem truncar); ChunkDataPacket (0x21, coluna completa/parcial, bioma/luz descartados), MultiBlockChangePacket (0x22), BlockChangePacket (0x23), todos PLAY/CLIENTBOUND; chunk desconhecido tratado como no-op; nenhuma DEC nova; 420 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 7 (incremento 7.3): Map Chunk Bulk — MapChunkBulkPacket/Codec/Handler/EventoMapChunkBulk/ReceptorMapChunkBulk (0x26, PLAY/CLIENTBOUND, skyLightSent único para todas as colunas, cabeçalhos antes dos dados, sempre coluna completa); SecoesDeChunkCodec extraído de ChunkDataCodec e Mundo.registrarColunaCompleta extraído de ReceptorChunkData, ambos reaproveitados; nenhuma DEC nova; 432 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 7 (incremento 7.4): Explosion — RegistroDeExplosao/ExplosionPacket/Codec/Handler/EventoExplosion/ReceptorExplosion (0x27, PLAY/CLIENTBOUND; radius descartado sem ser exposto, count lido como int de 4 bytes); SessaoDeJogo ganha motionX/motionY/motionZ + aplicarImpulsoDeExplosao; flag Client.MapAndPhysics do legado sem equivalente Java, mutação aplicada incondicionalmente; nenhuma DEC nova | ✔ |
| 2026-07-20 | Milestone 7 (incremento 7.5): Change Game State — ChangeGameStatePacket/Codec/Handler/EventoChangeGameState/ReceptorChangeGameState (0x2B, PLAY/CLIENTBOUND, só reason==3 muta gamemode, fiel ao legado); SessaoDeJogo.atualizarGamemode (novo); achado crítico: Time Update (0x03) e Spawn Position (0x05) auditados em 4 projetos C# e confirmados como nunca implementados no legado — deliberadamente mantidos não registrados, cobertos pela DEC-20; nenhuma DEC nova; 453 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 8 (incremento 8.1): DEC-21 — Papel do Caso de Uso em Ações Iniciadas pelo Bot no Estado PLAY, formalizando o fluxo CasoDeUso→SessaoDeJogo→ConexaoMinecraft→Packet→Servidor complementar à DEC-19; puramente documental | ✔ |
| 2026-07-20 | Milestone 8 (incremento 8.2): Lista de Jogadores — ListaDeJogadores/JogadorConhecido (domain.bot, novo agregado interno de SessaoDeJogo); ItemDeListaDeJogadores (sealed, 5 variantes)/PlayerListItemPacket/Codec/Handler/EventoPlayerListItem/ReceptorPlayerListItem (0x38, PLAY/CLIENTBOUND); divergência documentada: bug de alinhamento do legado na ação 3 (UUID desconhecido) não replicado, Codec sempre fiel ao formato de fio; nenhuma DEC nova (DEC-21 já cobre o padrão) | ✔ |
| 2026-07-20 | Milestone 8 (incremento 8.3): Chat Enviado pelo Bot — EnvioDeChatPacket/Codec/Handler/EventoEnvioDeChat (0x01, PLAY/SERVERBOUND, sem Receptor, mesmo precedente de RespostaKeepAlive/ConfirmacaoDePosicao); SessaoDeJogo.enviarMensagem (trunca em 99 caracteres, fiel ao off-by-one do legado; no-op em mensagem vazia/nula); CasoDeUsoEnviarMensagemDeChat (primeiro Caso de Uso do Play State, conforme DEC-21); nenhum Port novo; 492 testes automatizados, 0 falhas | ✔ |
| 2026-07-20 | Milestone 9 (incremento 9.1): DEC-22 — Ações Fundamentais do Jogador (Movimentação e Rotação); formaliza envio explícito sob demanda (sem Tick loop/física); decide reaproveitar ConfirmacaoDePosicaoPacket (id 0x06) para futuro combinado posição+rotação em vez de nova classe (colisão de chave em RegistroDePacotesV1_8); decide mutação otimista de estado; puramente documental | ✔ |
| 2026-07-20 | Milestone 9 (incremento 9.2): Movimentação do Jogador — PlayerPositionPacket/Codec/Handler/EventoPlayerPosition (0x04, PLAY/SERVERBOUND, sem colisão com EntityEquipmentPacket 0x04 CLIENTBOUND); SessaoDeJogo.mover (mutação otimista de x/y/z); CasoDeUsoMoverJogador (conforme DEC-21/DEC-22); nenhum Port novo | ✔ |
| 2026-07-20 | Milestone 9 (incremento 9.3, encerramento): Rotação do Jogador — PlayerLookPacket/Codec/Handler/EventoPlayerLook (0x05, PLAY/SERVERBOUND); SessaoDeJogo.olhar (mutação otimista de yaw/pitch); CasoDeUsoRotacionarJogador (mesmo padrão); nenhum Port novo; nenhuma física/Tick loop/automação; 511 testes automatizados, 0 falhas, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 10 (incremento 10.1, planejamento): análise arquitetural das interações do jogador com o mundo — DEC-21/DEC-22 já cobrem integralmente o padrão de ação iniciada pelo bot; nenhuma DEC nova, nenhum Port novo, nenhum agregado novo; puramente documental | ✔ |
| 2026-07-21 | Milestone 10 (incremento 10.2): Swing Arm — BalancarBracoPacket/Codec/Handler/EventoBalancarBraco (0x0A, PLAY/SERVERBOUND, sem campos, nome em português para evitar colisão com AnimationPacket clientbound); SessaoDeJogo.balancarBraco(); CasoDeUsoBalancarBraco; nenhum Port novo | ✔ |
| 2026-07-21 | Milestone 10 (incremento 10.3): Player Digging — PlayerDiggingPacket/Codec/Handler/EventoPlayerDigging (0x07, PLAY/SERVERBOUND, sem colisão com RespawnPacket clientbound); SessaoDeJogo.iniciarQuebraDeBloco/cancelarQuebraDeBloco/finalizarQuebraDeBloco (status 0/1/2); 3 Casos de Uso; mineração automática (AutoMiner/DiggingHelper do legado) deliberadamente não portada | ✔ |
| 2026-07-21 | Milestone 10 (incremento 10.4, encerramento): Player Block Placement — PlayerBlockPlacementPacket/Codec/Handler/EventoPlayerBlockPlacement (0x08, PLAY/SERVERBOUND, sem colisão com PlayerPositionAndLookPacket clientbound; Codec com extensão de sinal do campo y para o sentinela -1/-1/-1); SessaoDeJogo.colocarBloco; CasoDeUsoColocarBloco; lógica de inventário automático deliberadamente não portada; 548 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 11 (incremento 11.1, encerramento): Movimentação e Rotação Combinadas — reaproveita ConfirmacaoDePosicaoPacket/Codec (0x06, PLAY/SERVERBOUND) já existente desde a Milestone 5, conforme decidido pela DEC-22 (nenhum Packet/Codec/Handler/Evento novo); SessaoDeJogo.moverEOlhar (mutação otimista de x/y/z/yaw/pitch); CasoDeUsoMoverEOlharJogador (conforme DEC-21); nenhuma DEC nova, nenhum Port novo; 552 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 12 (incremento 12.1, encerramento): Arquitetura de Execução de Comandos do Bot (DEC-23) — novo subpacote interfaces.comando (camada aprovada desde a DEC-12, populada pela primeira vez); Comando/ResultadoComando/GerenciadorDeComandos, contrato mínimo single-shot sem Tick/Toggle/isMacro; 8 comandos concretos (ComandoMover/ComandoOlhar/ComandoMoverEOlhar/ComandoBalancarBraco/ComandoIniciarQuebraDeBloco/ComandoCancelarQuebraDeBloco/ComandoFinalizarQuebraDeBloco/ComandoColocarBloco) delegando a Casos de Uso já aprovados das Milestones 9-10; CommandHelp/CommandPlayerList/CommandMove/CommandGoto/CommandSneak e comandos de automação/inventário/combate do legado avaliados e deliberadamente não portados (motivo documentado por comando na DEC-23); nenhum Packet/Codec/Port/agregado novo; 583 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 13 (incremento 13.1, encerramento): Raycast Fiel ao Legado sobre Mundo (DEC-24) — Bloco.solido() porta Blocks.IsSolid; Mundo.tracarRaio(...) porta World.RayCast por completo (quirk do bloco de destino nunca testado, semântica de permitirAgua preservada); novo record ResultadoDoRaio; SessaoDeJogo.tracarRaioParaBlocos(alcance) porta RayCastBlocks/GetLookVector/CalculateLookVector; correção de bounds-check em Mundo.blocoEm fiel a World.GetBlock (evita ArrayIndexOutOfBoundsException para y fora de [0,256)); CanSeeEntity genérico (altura por tipo de mob) deliberadamente não portado; nenhum Packet/Port/Use Case novo; 597 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 14 (incremento 14.1, encerramento): Ações de Bloco com Auto-Mira (DEC-25) — SessaoDeJogo.olharParaBloco(x,y,z) porta Entity.LookTo/LookToBlock (sem o jitter aleatório do legado); SessaoDeJogo.usarItemNaMao(item) porta o sentinela -1/-1/-1/direction -1 de PacketBlockPlace(ItemStack); CasoDeUsoOlharParaBloco/CasoDeUsoUsarItemNaMao novos; ComandoClicarBloco (porta CommandClickBlock), ComandoQuebrarBloco (porta o caminho base de CommandBreakBlock, opções rp/rt; ncp/at fora de escopo), ComandoColocarBlocoAutoMira (porta CommandPlaceBlock; branch de item especial de PlaceCurrentBlock não portado por ser inalcançável a partir deste comando; alias próprio para não colidir com ComandoColocarBloco/Milestone 12); nenhum Packet/Port/agregado novo, consome o raycast da DEC-24; 623 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 15 (incremento 15.1, encerramento): Infraestrutura de Saída de Mensagens para o Operador (DEC-26) — domain.bot.SaidaDoOperador porta MinecraftClient.ChatMessages/MaximumChatLines/PrintToChat (regime permanente de 151 mensagens preservado); Bot ganha o campo saidaDoOperador (sem mudança de construtor); GerenciadorDeComandos.executar ganha as mensagens de fallback de CommandManagerNew.RunCommand (mesma assinatura pública); ComandoAjuda (porta CommandHelp) e ComandoListarJogadores (porta CommandPlayerList) novos, fecham a lacuna aberta desde a DEC-23; nenhum Packet/Port/agregado/bounded context novo, nenhuma interface pública alterada; 640 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 16 (incremento 16.1, encerramento): Algoritmo de Busca de Caminho sobre Mundo (DEC-27) — domain.bot.PontoDeCaminho (porta PathPoint) e domain.bot.BuscadorDeCaminho (package-private, porta PathFinder+Path); Mundo.criarCaminhoPara (porta World.CreatePathTo por completo) e SessaoDeJogo.criarCaminhoPara (conveniência com os 4 valores fixos do único call site do legado) novos; PathGuide e seus consumidores (CommandGoto/CommandFollow/CommandPortal/AutoMiner) permanecem fora de escopo sem reabrir a DEC-22 (motor de física); dois achados de código morto comprovados e não portados (canDrown nunca true em nenhuma chamada do legado; NodeType._2 inalcançável); nenhum Packet/Port/Caso de Uso/Comando/bounded context novo; 652 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 17 (incremento 17.1, encerramento): Registro de Blocos e Calculadora de Força de Quebra (DEC-28) — domain.protocol.v1_8.RegistroDeBlocos (package-private, porta Blocks/Block do legado, só dureza/material/ferramentasDeColeta, 236 blocos extraídos do blocks.json embutido no .resx) e CalculadoraDeQuebraDeBloco (público, porta DiggingHelper por completo, tabela de ferramentas do materials.json embutida, 43 entradas) novos; SessaoDeJogo.estaSubmerso() novo (porta Entity.IsUnderWater, derivável hoje sem lacuna); nível de Efficiency/amplifier de Haste/amplifier de Mining Fatigue/OnGround viram parâmetros explícitos (sentinela 0/-1) por falta de estado equivalente no domínio Java (NBT de item e efeitos do próprio bot continuam não modelados, lacunas já aceitas em milestones anteriores; motor de física fora de escopo desde a DEC-22); AutoMiner/CommandMiner/CommandBreakBlock (Tick loop) permanecem fora de escopo; nenhum Packet/Port/Caso de Uso/Comando/bounded context novo; 683 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 18 (incremento 18.1, encerramento): Update Sign e Encerramento Formal da Milestone 7 — domain.protocol.v1_8.UpdateSignPacket/Codec/Handler/EventoUpdateSign/ReceptorUpdateSign novos (PLAY, id 0x33, CLIENTBOUND, porta Handler_v18.cs case 51 por completo); Mundo.registrarPlaca/placaEm novos (dicionário interno independente do mapa de chunks, sem no-op para chunk não carregado, fiel ao World.Signs do legado; expurgação por distância e gate Client.MapAndPhysics do legado não portados, mesma categoria de otimização específica do C# já registrada na DEC-27); busca exaustiva em Handler_v18.cs (35 case enumerados) e em classes Packet* de todo o Projeto Adv 2.4.5 confirma ausência total de precedente para World Border (0x44), Update Block Entity (0x35), Block Action (0x24) e Block Break Animation (0x25) — cobertos pelo descarte seguro da DEC-20, mesmo tratamento de Time Update/Spawn Position; bounded context de Mundo (Milestone 7) encerrado oficialmente; nenhuma DEC nova; 693 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 19 (incremento 19.1, encerramento): Linha de Visão contra Jogador Remoto — SessaoDeJogo.podeVerJogador(EntidadeJogadorRemoto) novo, porta Entity.CanSeePlayer (raio dos pés do bot até o olho do alvo, y+1.62, mesma constante já aceita na Milestone 13), segundo consumidor de Mundo.tracarRaio/DEC-24; CanSeeEntity genérico para mobs permanece fora de escopo (depende de EntityProperty.Height por tipo de mob, dado ainda não levantado); nenhum Packet/Port/Caso de Uso/Comando/agregado/bounded context novo; nenhuma DEC nova; 695 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 20 (incremento 20.1, encerramento): Entity Action — Sneak — domain.protocol.v1_8.EntityActionPacket/Codec/Handler/EventoEntityAction novos (PLAY, id 0x0B, SERVERBOUND, sem colisão com AnimationPacket clientbound, sem Receptor); SessaoDeJogo.agachar()/pararDeAgachar() novos (actionId 0/1, jumpBoost sempre 0); CasoDeUsoAgachar/CasoDeUsoPararDeAgachar (conforme DEC-21) e ComandoAgachar/ComandoPararDeAgachar novos — toggle único do legado (CommandSneak) modelado como 2 comandos single-shot simétricos, já que Comando não tem Toggle() desde a DEC-23, mesmo padrão da separação de PlayerDigging na Milestone 10; rastreamento completo de todo call site de PacketEntityAction no legado confirma ActionID 2 (leave bed) e 5 (jump boost) como código morto e ActionID 3/4 (sprint) como Tick-loop-only (MinecraftClient.cs, mesmo motivo da DEC-22); Player bare (0x03, PacketUpdate) investigado como candidato irmão e confirmado Tick-loop-only pelo mesmo motivo; nenhum Packet/Port/agregado/bounded context novo; nenhuma DEC nova; 710 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 21 (incremento 21.1, encerramento): Fundação da Engine de Execução Contínua (DEC-29) — pivô de escopo (infraestrutura de execução, não capacidade isolada de protocolo); domain.bot.EstadoExecucao (PARADO/EXECUTANDO/PAUSADO, sem precedente no legado) novo; Bot ganha iniciar()/pausar()/retomar()/parar() e registrarTarefa/removerTarefa/tarefasContinuas; domain.bot.TarefaContinua (interface funcional, zero implementações reais) novo; application.port.AgendadorDeTarefasPort (segundo Port do projeto) novo; infrastructure.execucao.AgendadorDeTarefasVirtualThread (Virtual Threads, DEC-03) e MotorDeTick (percorre bots EXECUTANDO, isola falha por tarefa, protegido contra reentrância) novos; DEC-29 resolve explicitamente que o mecanismo não reabre DEC-22/DEC-23/DEC-27 (zero leitura/escrita de física, zero tarefa real registrada — mesma distinção mecanismo/conteúdo da própria DEC-27); nenhum Packet/Comando/bounded context novo, nenhuma interface existente alterada; 726 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 22 (Fase de Planejamento): análise arquitetural completa da frente de ciclo de vida contínuo — levantamento exaustivo do legado (Auto Reconnect, login automático, delay/tentativas/backoff, proxy/rotação, detecção de disconnect, eventos de conexão/perda, scheduler/Tick, AutoMiner/AutoAttack/AutoFish/AntiAFK/Follow/Herbalism/Repair/Solk, preservação de estado, plugins, dependências entre automações); riscos identificados (MotorDeTick sequencial/não-paralelo por ciclo, tensão fidelidade-vs-escala em backoff de reconexão, mecanismos redundantes de reconexão no legado, máquinas de estado que travam após reconexão, pool de proxy global sem isolamento por bot); dividida em 7 incrementos pequenos (22.1–22.7); nenhuma DEC pendente bloqueia qualquer um | ✔ |
| 2026-07-21 | Milestone 22 (incremento 22.1): Propagação de Perda de Conexão para o Ciclo de Vida do Bot (DEC-30) — poll de SessaoDeJogo.estaEncerrada() pelo MotorDeTick.tick() (não callback/EventBus, fiel ao único padrão real do legado: polling de beingTicked, zero eventos de conexão encontrados no C#); ConexaoMinecraft.estaAberta() (novo método default, preserva 65 fakes de teste); TransporteSocket.estaAberta() (sobrescreve); SessaoDeJogo.estaEncerrada()/encerrarVoluntariamente() novos; Bot.registrarDesconexao() novo; AdaptadorConexaoBotV1_8.disconnect(Bot) implementado de fato (antes UnsupportedOperationException desde a Milestone 4); CasoDeUsoDesconectarBot integrado a ConexaoBotPort pela primeira vez; extensão 100% aditiva, nenhuma interface já aprovada teve assinatura alterada; 737 testes automatizados (726→737), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 22 (incrementos 22.2-22.6, DEC-31, na mesma sessão sem pausa por instrução explícita do responsável): DEC-31 formaliza que o AutoReconnect preserva a regra funcional do legado (retry incondicional até sucesso, sem cap) mas o intervalo entre tentativas vira política configurável (PoliticaDeReconexao/PoliticaDeReconexaoComJitter, com jitter para evitar tempestade de reconexão em centenas de bots); 22.2 SessaoDeJogo.keepAliveExpirou/LIMITE_KEEP_ALIVE (fiel a 750 ticks/37500ms, MinecraftClient.cs:773-777); 22.3 GerenciadorDeReconexao (novo, mesma categoria de MotorDeTick, ativa SessaoBot.autoReconnect existente desde a M3 pela primeira vez, reaproveita CasoDeUsoConectarBot/DEC-21); 22.4 bot.iniciar() no sucesso da reconexão retoma tarefasContinuas automaticamente (já sobreviviam por construção desde a M21); 22.5 ConfiguracaoProxy/TipoDeProxy novos, EnderecoServidor ganha proxy via extensão aditiva (construtor de 2 args preservado por sobrecarga), FabricaDeConexaoMinecraftV1_8 roteia via java.net.Proxy nativo do JDK em vez de reconstruir o handshake manual SOCKS4/5/HTTP de Proxy.cs (divergência de implementação documentada, não de comportamento); 22.6 PoolDeProxies (round-robin fiel a ProxyList.NextProxy(), AtomicInteger em vez do índice não sincronizado do legado) com rotação em toda falha de reconexão e no motivo de kick "muitas contas conectadas"; zero novo Port/agregado/bounded context em todo o lote, zero interface já aprovada alterada; 766 testes automatizados (737→766), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-21 | Milestone 23 (encerramento): primeira Macro real do projeto — AntiAFK. Análise exclusiva de CommandAntiAFK do legado comprovou dependência integral de motor de física (gravidade/colisão/OnGround) bloqueado desde a DEC-22 e reafirmado por DEC-23/27/28/29/30/31; sessão interrompida e justificativa apresentada ao responsável, conforme instrução explícita; responsável decidiu reabrir a DEC-22 de forma restrita — **DEC-32** (motor de física vertical mínimo, sem movimento horizontal/água/lava/escada; fidelidade de forma de bloco "alturas parciais simples" — slab/snow layer/soul sand/carpet, sem shapes multi-caixa). Entregue: Bloco.alturaSuperficie() novo; SessaoDeJogo.onGround()/pular()/aplicarFisicaVertical() novos (gravidade -0.08/tick, drag *0.98, impulso de pulo 0.42, porte de AABB.ClipYCollide restrito ao eixo Y, só envia PlayerPositionPacket via mover() quando y muda); TarefaAntiAFK novo (primeiro consumidor real de MotorDeTick/TarefaContinua desde a DEC-29, guarda contra sessaoDeJogo==null — lacuna já prevista pela própria DEC-29); ComandoAntiAFK novo (toggle sem CasoDeUso dedicado, registra/remove TarefaAntiAFK diretamente em Bot já que nenhum Packet é enviado pelo Comando; delay inválido rejeitado sem ligar nada — bug de toggle-antes-de-validar do legado não portado). Zero novo Port/agregado/bounded context/Packet; zero interface já aprovada alterada; movimento horizontal permanece bloqueado pela DEC-22 inalterada. 791 testes automatizados (766→791), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-22 | Milestone 22 (incremento 22.7, ENCERRAMENTO FORMAL DA MILESTONE 22): Wiring de Produção (DI) do MotorDeTick/AgendadorDeTarefasPort/GerenciadorDeReconexao/ConexaoBotPort — infrastructure.config.ConfiguracaoDeConexao/ConfiguracaoDeExecucao (novos, @Configuration) compõem todos os beans já aprovados nas DEC-29/30/31; infrastructure.config.CicloDeVidaDoMotorDeExecucao (novo, SmartLifecycle) agenda MotorDeTick.tick() a 50ms e GerenciadorDeReconexao.verificar() a 1000ms no start, encerra AgendadorDeTarefasPort de forma limpa no stop; infrastructure.config deixa de estar vazio. Sem DEC nova (puramente wiring); valores de infraestrutura sem precedente no legado documentados (jitter de reconexão 1000ms, timeout de conexão TCP 10000ms). Registro/remoção de bots individuais permanece sem call site de produção — sem CLI/API ainda (DEC-02), fora de escopo desta milestone por instrução explícita. Zero novo Port/agregado/bounded context/interface alterada. 794 testes automatizados (791→794), 0 falhas, 0 erros, 3 skipped deliberadamente; validação adicional via `mvn spring-boot:run` real confirma boot e shutdown gracioso via SpringApplicationShutdownHook fora de JUnit | ✔ |
| 2026-07-22 | Milestone 24 (incremento 24.1, abertura da Fase 2 — Macros/Automações): análise exaustiva do framework de macros do legado (ICommand/Tick classe abstrata, CommandManagerNew, thread de tick ~50ms; AutoMiner/AutoFish(CommandPesca)/AutoAttack(CommandKillAura)/Herbalism/Solk/Follow/Portal e outras automações descobertas) e inventário completo da arquitetura Java atual (MotorDeTick/TarefaContinua/SessaoDeJogo/Mundo/BuscadorDeCaminho/CalculadoraDeQuebraDeBloco/Comando); achado central: CasoDeUsoCriarBot/CasoDeUsoConectarBot nunca chamavam MotorDeTick.registrar/Bot.iniciar() — mesmo TarefaAntiAFK (Milestone 23) nunca executava fora de teste, gap evidenciado pelo próprio BootstrapDeProducaoTest (22.7) contornando os Casos de Uso reais. **DEC-33** corrige via application.port.MotorDeExecucaoPort (novo, mesma categoria de ConexaoBotPort/AgendadorDeTarefasPort): CasoDeUsoCriarBot registra o bot criado, CasoDeUsoConectarBot.connect() chama bot.iniciar() no sucesso (efeito colateral correto: reconexão automática via GerenciadorDeReconexao volta a marcar EXECUTANDO). Conclusão de suficiência arquitetural: arquitetura atual suficiente sem lacuna de framework para macros sem movimento horizontal/sem container (Herbalism, candidato concreto); AutoMiner/Follow/Portal seguem bloqueadas pela DEC-22 (nova DEC de reabertura necessária, escopo próprio, não herda a DEC-32); AutoFish precisa de suporte a container/janela (lacuna nova); Solk tem ambiguidade não resolvida vs. política de combate/automação; AutoAttack(KillAura) permanece permanentemente fora de escopo. Zero bounded context novo, zero DEC redefinida, zero interface pré-existente alterada (exceto construtor de CasoDeUsoCriarBot, sem call site de produção fixo antes desta DEC). 798 testes automatizados (791→798), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-22 | Milestone 25 (incremento 25.1, encerramento): Herbalismo — segunda Macro real do projeto, escolhida pelo responsável entre os candidatos da Milestone 24 por não depender de movimento horizontal nem de lacuna nova de framework. Porta CommandHerbalism (legado) integralmente: TarefaHerbalismo/ComandoHerbalismo (novo par toggle, mesmo padrão de AntiAFK); SelecionarSlotDaHotbarPacket/SelecionarSlotDaHotbarCodec (novo, PLAY 0x09 SERVERBOUND, coexiste com HeldItemChangePacket CLIENTBOUND já registrado no mesmo id, mesma disciplina da DEC-16) e SessaoDeJogo.selecionarSlotDaHotbar (porta o setter MinecraftClient.HotbarSlot). Replantar reproduz o branch "double-send" de PlaceCurrentBlock para o item 338 (cana) — branch que a Milestone 14/DEC-25 tinha comprovado mas não portado por ser inalcançável a partir de ComandoColocarBlocoAutoMira (exige item<256), e que aqui É alcançado. Achado de fidelidade durante a validação: Entity.PosY do legado (origem de LookTo/RayCastBlocks/CanSeePlayer) é altura do olho (Entity.cs:326, PosY=AABB.MinY+1.62), não dos pés — SessaoDeJogo.olharParaBloco/tracarRaioParaBlocos(DEC-24/25)/podeVerJogador(Milestone 19) usavam pés sem essa soma, divergência invisível nos 3 comandos já em produção (miram longe) mas categórica para Herbalismo (mira o próprio bloco dos pés, raio saía para cima em vez de para baixo). Apresentado com alternativas antes de implementar; responsável decidiu corrigir globalmente — **DEC-34**, muda ângulos/coordenadas exatas de teste dos 3 comandos existentes (ComandoClicarBloco/ComandoQuebrarBloco/ComandoColocarBlocoAutoMira), não o comportamento observável. Zero bounded context novo; zero interface pré-existente com assinatura alterada. 814 testes automatizados (798→814), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-22 | Milestone 26 (incremento 26.1, encerramento): análise arquitetural completa do motor de movimento horizontal do legado (Entity.Move/MoveRelative/AABB.Clip*Collide, PathGuide, CommandGoto/CommandFollow/CommandPortal/AutoMiner), diferenciando mecanismo de movimento, física completa (fora de escopo), execução de caminho (PathGuide) e consumidores/macros; confirmado que Goto/Follow/Portal dependem só de execução de caminho, nunca de MoveQueue/Movement/MoveRelative por yaw (mecanismo exclusivo de AutoMiner). **DEC-35** — reabertura parcial da DEC-22 (mesmo precedente da DEC-32/vertical): SessaoDeJogo.aplicarFisicaVertical() generalizado para aplicarFisica() (colisão 3-eixos Y/X/Z, reaproveitando Bloco.alturaSuperficie() também para X/Z; sem a assistência de "step"/stepHeight do legado — qualquer resistência horizontal é colisão cheia, resolvida pelo pulo automático); colididoHorizontalmente()/velocidadeHorizontal()/somarMotionHorizontal (novos, pacote, unificam a fórmula de velocidade duplicada 3x no legado). domain.bot.GuiaDeCaminho novo (porte de PathGuide — criar/finalizado/tick, consome SessaoDeJogo.criarCaminhoPara já existente desde a DEC-27); bônus de pulo por poção de Speed do legado não portado (efeitos do próprio bot não modelados, mesma lacuna da DEC-28). Zero Comando/CasoDeUso/TarefaContinua/Port/bounded context/agregado novo — GuiaDeCaminho fica sem consumidor de produção nesta sessão, mesmo precedente da DEC-27/BuscadorDeCaminho; ComandoGoto/TarefaFollow/ComandoPortal ficam como candidatos imediatos, AutoMiner segue bloqueado por mecanismo adicional (MoveQueue/Movement/MoveRelative por yaw, DEC própria futura). Rename interno aplicarFisicaVertical→aplicarFisica com comportamento idêntico para o único consumidor pré-existente (TarefaAntiAFK), provado pelos testes mantidos verdes. 819 testes automatizados (814→819), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Milestone 27 (incremento 27.1, Épico Fase 2 "execução de caminho"): ComandoGoto — porte de CommandGoto, menor consumidor real de GuiaDeCaminho/DEC-35. SessaoDeJogo.caminhoAtual()/definirCaminho/limparCaminho (novos) + MotorDeTick.tick() ticando o caminho ativo incondicionalmente (mesmo lugar onde o legado tica CurrentPath, antes do laço de TarefaContinua) tornam GuiaDeCaminho consumível em produção pela primeira vez. Cálculo de caminho síncrono (legado despacha via Task.Factory.StartNew, sem precedente de wrapper assíncrono no domínio Java) — divergência de execução registrada, não de resultado. Mensagem de falha preserva o typo ("possivel") e a cor incorreta (§a) do CommandGoto original. Sem DEC nova (mesmo padrão de ComandoMover/DEC-23). Zero Port/bounded context/interface já aprovada alterada. 827 testes automatizados (819→827), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Milestone 28 (incremento 28.1, Épico Fase 2 "execução de caminho"): TarefaFollow/ComandoFollow — porte de CommandFollow, terceira Macro real, mesmo par toggle de TarefaAntiAFK/TarefaHerbalismo (registro/remoção de TarefaContinua substitui o campo nullable interno persistente do legado, mesmo resultado observável). SessaoDeJogo.olharParaEntidade (novo, porte de Entity.LookTo); ListaDeJogadores.uuidPorNome/EntidadesDoMundo.porUuid (novos) resolvem nick → EntidadeJogadorRemoto em dois passos, mesma composição de PlayerManager.GetPlayerByNick do legado. Recalcula GuiaDeCaminho a cada 2.5 blocos de deslocamento do alvo (sentinela Y=-555.0 do legado reproduzido fielmente). Omissão registrada: toggle do comando "retard" do legado sem equivalente (CommandRetard nunca foi portado). Sem DEC nova. Zero Port/bounded context/interface já aprovada alterada. 844 testes automatizados (827→844), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Milestone 29 (incremento 29.1, encerramento do Épico Fase 2 "execução de caminho"): ComandoPortal — porte de CommandPortal, localiza portais de minijogos do servidor CraftLandia por nome conhecido ou por varredura de blocos ("**"). domain.bot.RegistroDePortais/PosicaoDePortal (novos) hardcoded a partir do resource embutido Resources.portals do legado (JSON base64 decodificado, 25 minijogos), sem parser JSON/dependência nova no projeto (mesmo racional de RegistroDeBlocos/DEC-28); SessaoDeJogo.localizarPortal (novo) reproduz FindPortal/BruteForcePortalFinder, incluindo a ordem exata dos laços da varredura por blocos. Sem DEC nova. Épico Fase 2 "execução de caminho" (Milestones 26-29) encerrado: GuiaDeCaminho (capacidade pura, M26) agora tem três consumidores de produção. Zero Port/bounded context/interface já aprovada alterada. 856 testes automatizados (844→856), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Milestone 30 (incremento 30.1, encerramento, Épico "Mineração"): DEC-36 decide não portar MoveQueue/Movement/MoveRelative por yaw (sem consumidor real no modelo de execução síncrona já adotado desde a DEC-35), fechando o item 4 pendente daquela DEC. TarefaMineracao/ComandoMinerar (quarta Macro real) portam AutoMiner/CommandMiner: busca do bloco de menor custo (pedra/terra, sentinelas de fábrica) num cubo de raio 8 ao redor dos pés, deslocamento via GuiaDeCaminho (sem o nudge MoveQueue), quebra no alcance do raio 6 via CalculadoraDeQuebraDeBloco/DEC-28 acumulada por tick, timeout de 15s fiel. Zero Port/bounded context/agregado/interface já aprovada alterada. 865 testes automatizados (856→865), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Milestone 31 (Fase de Planejamento, BLOQUEADA): Sistema de Pesca (AutoFish/CommandPesca) — análise exaustiva de CommandPesca.cs/MacroUtils.cs (único alias registrado "solkpesca") aprofunda o achado da Milestone 24: núcleo de captura já 100% coberto por SessaoDeJogo.usarItemNaMao/DEC-25, mas BUSCAR_VARA/GUARDAR_ITENS/parte de REPARAR (3 dos 10 estados da máquina) dependem de abrir/ler/clicar/fechar baús — nenhum Packet de container existe hoje, ReceptorSetSlot já trata windowId != 0 como no-op deliberado. Acionado o gatilho de parada "novo agregado" (Janela/Container) — sem código, sem DEC nova nesta sessão; aguarda decisão do responsável sobre DEC própria (candidata a DEC-37) antes de qualquer implementação | ⏸ |
| 2026-07-23 | Pivô de Estratégia da Fase 2: responsável decide construir primeiro um roadmap de épicos de infraestrutura reutilizável (EPIC-I1 a EPIC-I13) em vez de continuar porte-macro-por-macro; mapeamento exaustivo do legado produzido, agrupando toda primitiva reutilizável em ~24 capacidades arquiteturais com prioridade/risco/DEC por capacidade | ✔ |
| 2026-07-23 | Milestone 32 (EPIC-I1 — Container Framework, DEC-37, encerramento): Janela/JanelaDeSlots (novo agregado + interface, domain.bot, genérico para qualquer Window do protocolo 1.8); OpenWindowPacket(CB 0x2D)/CloseWindowPacket(CB 0x2E)/FecharJanelaPacket(SB 0x0D)/ClickWindowPacket(SB 0x0E)/ConfirmTransactionPacket(CB 0x32)/ConfirmarTransacaoPacket(SB 0x0F), cada um com Codec/Handler/Evento; ReceptorSetSlot/ReceptorWindowItems estendidos para windowId != 0 (fecha a lacuna da Milestone 5.5/31); SessaoDeJogo ganha abrirJanela/fecharJanela/clicarSlot/shiftClique/trocarSlots/pegarItem/largarItem/moverItem/confirmarTransacao/localizarItemNaJanela/localizarEspacoLivreNaJanela sobre indexação unificada de slot (elimina por construção o bug isChestOpen das 4 variantes de MacroUtils.moverItemXXX do legado); cursor de clique por sessão, não static (bug do legado não replicado); 8 Casos de Uso; ComandoClicarSlot(porte de CommandInvClick)/ComandoFecharJanela. trocarSlots(mode 2) e Window Property/Creative Inventory Action fora de escopo, decisões documentadas na DEC-37. Zero interface pré-existente alterada. Desbloqueia a Milestone 31. 963 testes automatizados (865→963, +98), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-23 | Pivô de Estratégia: unidade de trabalho passa de milestone/épico por sessão para domínio da plataforma (pode conter vários épicos; esgotar tudo que houver reaproveitamento antes de parar; nunca implementar macro diretamente, só capacidade reutilizável) — primeiro domínio escolhido: Inventory/Container | ✔ |
| 2026-07-23 | Milestone 33 (Domínio Inventory/Container, auditoria + fechamento): confirmado Container+Item Transfer 100% cobertos desde a Milestone 32/DEC-37 (nada a reimplementar); JanelaDeSlots.localizarTodosOsItens novo (Item Search, porte de CommandMob.buscarSlotsCom); SessaoDeJogo.limparCursor novo (Inventory Utils, porte generalizado de CommandMob.limparCursor); 7 Comandos novos (ShiftClique/TrocarSlots/PegarItem/LargarItem/MoverItem/ConfirmarTransacao/LimparCursor) conectando Casos de Uso já aprovados desde a Milestone 32 sem Comando correspondente. Equipment/Armadura analisado e confirmado sem lacuna funcional real — legado nunca veste armadura em nenhuma macro, construir auto-equipe seria regra de negócio sem validação (candidato dependente de decisão do responsável, não bloqueio). Zero DEC nova, zero Packet/Port/agregado/bounded context novo. 994 testes automatizados (963→994, +31), 0 falhas, 0 erros, 3 skipped deliberadamente. Domínio Inventory/Container esgotado | ✔ |
| 2026-07-23 | Milestone 34 (Domínio Interação com o Mundo, auditoria + fechamento): confirmado blocos (clicar/quebrar/colocar, Milestone 14), estruturas (portal, Milestone 29) e uso de item/uso de item em bloco (Milestone 14) 100% cobertos; confirmado que portas/alavancas/botões/NPCs não têm nenhum precedente próprio no legado (interação física genérica já coberta pelo clique de bloco), escrita de placa sem call site no legado — nenhum dos dois implementado, por ausência de regra a preservar. UseEntityPacket/UseEntityCodec novo (PLAY 0x02 SERVERBOUND, porte de PacketUseEntity); SessaoDeJogo.interagirComEntidade novo; CasoDeUsoOlharParaEntidade (faltava o wrapper, olharParaEntidade já existia desde a Milestone 28)/CasoDeUsoInteragirComEntidade novos; ComandoUseEntity novo (porte de CommandUseEntity — nick ou @any, atacar/interagir, filtro IsBot do legado não portado por ausência de registro de múltiplas instâncias de Bot). Zero DEC nova, zero Packet/Port/agregado/bounded context novo. 1007 testes automatizados (994→1007, +13), 0 falhas, 0 erros, 3 skipped deliberadamente. Domínio Interação com o Mundo esgotado | ✔ |
| 2026-07-23 | Milestone 35 (Backlog "Composição Pura" em lote): pivô de estratégia para backlog-global-auditado-uma-vez (artifact com 5 Épicos). Implementado todo Épico 1+2 numa sessão contínua sem pausa entre itens: ComandoLimparChat/SaidaDoOperador.limpar; SessaoDeJogo.soltarItemEmUso (status 5, sem consumidor); TarefaLargarTudo/ComandoLargarTudo (DropAll); CasoDeUsoSelecionarSlotDaHotbar/ComandoClicarItemDaHotbar (HotbarClick); TarefaTwerk/ComandoTwerk; CasoDeUsoReconectarBot/ComandoReconectar (Reco); TarefaAutoFish/ComandoAutoFish (Solk/CommandPesca, desbloqueia a Milestone 31). Achado: "$" no legado é auto-invocação local de outro comando, não chat real - "$move j 40" substituído por pular()+aplicarFisica() (DEC-32). deveGuardarItem sentinela false (NBT ausente, lacuna já registrada desde DEC-28); fallback de baú sem espaço simplificado (só 1º candidato, sem MoveQueue). Zero DEC nova, zero Packet/Port/agregado/bounded context novo, zero interface pré-existente alterada. 1036 testes automatizados (1007→1036, +29), 0 falhas, 0 erros, 3 skipped deliberadamente. Backlog de composição pura esgotado - restam só os Épicos 3/4/5, todos dependentes de dado/decisão/DEC do responsável | ✔ |
| 2026-07-25 | Milestone 36 (DEC-38, Épico 4 — política de combate): reabre parcialmente a DEC-23 (mesmo padrão da DEC-32/DEC-35 sobre a DEC-22, nenhum texto da DEC-23 alterado). Formaliza Categoria 1 (automação de combate genérica — KillAura/Aura, ataque automático indiscriminado, PvP automático, ataque através de parede — permanece fora de escopo, sem exceção) separada da Categoria 2 (ataque a entidade específica já identificada, como etapa de uma automação de negócio maior — passa a candidata, aprovação individual macro por macro), fundamentada no precedente já em produção desde a Milestone 34 (UseEntityPacket/interagirComEntidade/ComandoUseEntity, tratados como interação com o mundo, não combate). Único gap de infraestrutura genérica fechado: EntidadesDoMundo.mobMaisProximo(x,y,z,raio) (novo — primitiva pura, mesma categoria de Mundo.tracarRaio/DEC-24, sem checagem de linha de visão, fiel ao CanSeeEntity desativado no legado). Solk/CommandMob/CommandMobTeleport/CommandPesca/CommandUseBow continuam não implementados, aguardando aprovação individual antes de qualquer código de macro. Zero Packet/Port/agregado/bounded context novo, zero interface pré-existente alterada. 1041 testes automatizados (1036→1041, +5), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-25 | Milestone 37 (Solk/CommandMob, Categoria 2/DEC-38): TarefaMob/ComandoMob (aliases solkmob/mob) - porte de Solk/CommandMob.cs, 12 sub-estados, 100% composição sobre capacidades já aprovadas (mobMaisProximo/M36, interagirComEntidade/M34, Container/DEC-37, pular+aplicarFisica). Duas capacidades genéricas extraídas de TarefaAutoFish (regra de três, reaproveitadas pelas duas macros): MacroUtils (clicarBlocoComBotaoDireito/clicarPlacaAgachando/contarSlotsVagos/slotUnificadoDoInventario/itemNaMao) e AbridorDeBau (retentativa de abrir baú). Divergências documentadas: FileLock multi-conta sem equivalente (DEC-09), VERIFICAR_POCAO inalcançável com config default, filtro Flame por NBT (lacuna já registrada DEC-28), limpeza de espadas extras sem retentativa real (mesma simplificação DEC-37), $move j N vira pular+aplicarFisica, interrupções reativas a chat não portadas (parser ChatComponent, lacuna M5), 850 hits distribuídos em ataques gated a 100ms (não bloqueante). Zero DEC nova, zero Packet/Port/agregado/bounded context novo, zero interface pré-existente alterada. Candidatos remanescentes do Épico 4: CommandMobTeleport, CommandUseBow (mira em alvo já identificado). 1051 testes automatizados (1041→1051, +10), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-25 | Milestone 38 (Pivô: unidade de trabalho = Capacidade Fundamental/Foundation API): matriz de 16 domínios funcionais levantada sem nova auditoria macro por macro. Achado central - projeto já quase inteiramente fechado a nível de primitiva (Container Framework/DEC-37 cobre Furnace/Brewing/Beacon/Anvil/Dispenser/Hopper genericamente; Blocos/Entidades-Cat.2/Inventário/Portais/Mundo/Combate-Cat.2 fechados). 4 Foundation APIs novas por extração de duplicação real: SessaoDeJogo.jogadorVisivelMaisProximo (generalizado de ComandoUseEntity, simétrico a mobMaisProximo/M36), MacroUtils.selecionarMelhorFerramenta (extraído de TarefaMineracao, fiel a AutoMiner.SelectBestTool), MacroUtils.selecionarSlotComItem (generalizado de TarefaHerbalismo), MacroUtils.itemNaMao ganha 3º/4º consumidor (TarefaMineracao/TarefaHerbalismo refatoradas). Domínios parcialmente fechados, exigem decisão própria fora desta sessão: Chat (parser ChatComponent/JSON, lacuna M5), Efeitos (auto-buff do bot, lacuna DEC-28), Itens (NBT, lacuna DEC-28), Movimento (MoveQueue, DEC-36). Zero DEC nova, zero macro implementada/conectada, zero Packet/Port/agregado novo. 1051 testes automatizados (mesma contagem de M37 - sessão de refatoração/extração, 0 falhas, 0 erros, 3 skipped) | ✔ |
| 2026-07-25 | Milestone 39 (DEC-39, fecha Chat/NBT/Efeitos do próprio bot): TagNBT/NbtCodec novos (11 tipos de tag, extraído do descarte já existente em ItemStackCodec desde M10); ItemStack ganha nbt (4º componente, aditivo - construtor de 3 args preservado, nbt=null). SessaoDeJogo ganha aplicarEfeito/removerEfeito/efeito/efeitosAtivos (reaproveita EntidadeRemota.EfeitoAtivo); ReceptorEntityEffect/ReceptorRemoveEntityEffect roteiam para o próprio bot quando entityId bate (antes: no-op silencioso). ParserDeChatComponent novo (extração de texto plano de ChatComponent JSON, parser próprio - jackson não é dependência real do projeto); SessaoDeJogo.textoPlanoDaUltimaMensagemDeChat novo. translate/with não resolvem texto de tradução (fora de escopo). Movimento/MoveQueue deliberadamente não tocado (DEC-36 já resolvida). Zero macro implementada/conectada, zero Packet/Port/agregado novo, zero interface pré-existente quebrada (extensões 100% aditivas). 1072 testes automatizados (1051→1072, +21), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-25 | Milestone 40 (EPIC-APP1, DEC-40, encerramento): mudança de fase — de "portar funcionalidades" para "executar bots reais através da aplicação Java". Primeira camada web do projeto (`interfaces.rest`/`interfaces.websocket`), REST + WebSocket sob `/api/v1`, construída inteiramente sobre Casos de Uso e a camada `interfaces.comando` (DEC-23) já aprovados, sem regra de negócio nova; legado C# deliberadamente não consultado nesta etapa (instrução explícita, sem precedente de API no legado). `application.registry.GerenciadorDeBots` (novo, registry por id, lacuna que nenhuma peça anterior cobria) + `CasoDeUsoRemoverBot`/`CasoDeUsoIniciarBot`/`CasoDeUsoPararBot`/`CasoDeUsoPausarBot`/`CasoDeUsoRetomarBot` (novos, cascas finas sobre `Bot`); `CasoDeUsoCriarBot` ganha `GerenciadorDeBots` no construtor (único ajuste de assinatura pré-existente). `ConfiguracaoDeComandos` conecta pela primeira vez em produção o `GerenciadorDeComandos`/~38 `Comando*` aprovados desde a Milestone 12/DEC-23 (nunca antes instanciados fora de teste) — macros expostas via `POST/DELETE /bots/{id}/macros/{alias}` sem nenhuma lógica de macro nova. `Conta`/`PerfilDeServidor` (novos, records com identidade) + `RepositorioDeContas`/`RepositorioDeServidores` (portas novas, adapters in-memory — PostgreSQL/DEC oficial do `CLAUDE.md` deliberadamente fora de escopo); `PoolDeProxies` ganha `adicionar`/`remover`/`listar`. `application.registry.NotificadorDeEventos` (novo, pub/sub por bot + global) alimentado por dois hooks opcionais aditivos (`SaidaDoOperador.definirOuvinte`/`Bot.definirOuvinteDeEstado`), consumido por `BotEventsWebSocketHandler` (WebSocket cru, sem STOMP, `/ws/bots/{id}/events` e `/ws/events`). Autenticação por `X-API-Key` (`ApiKeyFilter`) — decisão deliberada de não introduzir Spring Security/OAuth (YAGNI, sem consumidor externo real ainda). 11 Controllers REST (Bot/Ação/Inventário/Mundo/Macro/Comando/Conta/Servidor/Proxy/Log/Métricas), DTOs em `interfaces.rest.dto`, `GlobalExceptionHandler` (400/404/409/500 padronizados), paginação offset/limit. Validado end-to-end via `mvn spring-boot:run` real (criar servidor/conta → bot → iniciar → macro → evento WebSocket em tempo real → métricas → pausar → 409 → remover → 404 → 401 sem chave). Zero DEC anterior reaberta/contradita. 1089 testes automatizados (1072→1089, +17), 0 falhas, 0 erros, 3 skipped deliberadamente | ✔ |
| 2026-07-25 | Milestone 41 (EPIC-APP2, DEC-41, encerramento): cobertura completa da API para as 12 telas do React mapeadas pelo responsável, sobre a base do EPIC-APP1/DEC-40. `BotResponse` enriquecido (proxy/macrosAtivas/posição/vida/autoReconnect/msDesdeUltimoKeepAlive numa única chamada de lista). Equipamento (`InventarioDoJogador` ganha 5 getters nomeados de armadura/mão) e Container/Janela (`SessaoDeJogo.janelaAtual()`/DEC-37, nunca antes exposta) via `InventarioController`. `EstadoMundoResponse` ganha `chunkAtual`; `/mundo/entidades` ganha filtro `?tipo=mob|jogador`. Catálogos globais novos `GET /commands`/`GET /macros` (reaproveitam metadado de `GerenciadorDeComandos.comandos()`, sem duplicar). `PUT /bots/{id}/proxy` expõe `Bot.trocarProxy` (existia desde a Milestone 22, sem caminho de produção); `PUT /bots/{id}/auto-reconnect` exigiu 2 métodos aditivos novos (`SessaoBot.comAutoReconnect`/`Bot.definirAutoReconnect` — antes não havia nenhuma forma alcançável de ligar reconexão automática por bot); `GET /configuracao/reconnect-policy` só-leitura sobre o bean `PoliticaDeReconexaoComJitter` (tipo de retorno do bean mudado de interface para concreto). `MetricasResponse` ganha `porEstadoDeSessao`/`memoria`/`uptimeMs`/`cpuLoad` (via `java.lang.management`, sem Actuator) e `motorDeTick` (instrumentação nova em `MotorDeTick`, aditiva, sem alterar lógica de tick). `ComandoController`/`MacroController` publicam evento `"comando"` via `NotificadorDeEventos` já existente. **4 limitações documentadas como decisão consciente, não implementadas**: bioma (descartado no decode desde a Milestone 7, exigiria reabrir codec de protocolo), NPCs (protocolo 1.8 não distingue NPC de mob/player), ping real/RTT (nunca medido pelo client — exposto como `msDesdeUltimoKeepAlive`, nome honesto, não fabricado), separação chat/erro/comando no console (`SaidaDoOperador` é buffer único por design, DEC-26 — retag exigiria dezenas de call sites). Zero DEC anterior reaberta/contradita (protocolo/domínio intocados). 1098 testes automatizados (1089→1098, +9), 0 falhas, 0 erros, 3 skipped deliberadamente; validação adicional end-to-end via `mvn spring-boot:run` real + WebSocket (Node.js nativo) | ✔ |
| 2026-07-27 | Milestone 42 (EPIC-FRONT-01, DEC-42, encerramento): mudança de fase declarada pelo responsável — migração funcional C#→Java concluída dentro do escopo aprovado, Java é a única fonte de verdade, sem novas auditorias do legado salvo pedido explícito/bug específico. Preparação do backend para produção: `RepositorioDeContasJpa`/`RepositorioDeServidoresJpa`/`RepositorioDeProxiesJpa` (`infrastructure.persistence`) substituem os adapters in-memory (removidos) atrás das portas `RepositorioDeContas`/`RepositorioDeServidores` já existentes e da porta nova `RepositorioDeProxies` (mesmo padrão, fecha a lacuna de `PoolDeProxies` nunca ter tido persistência própria — `PoolDeProxies` continua como cache em memória usado por `GerenciadorDeReconexao`, carregado do repositório no startup, escrito em write-through por `ProxyController`). 3 `@Entity`/`JpaRepository` novos (`infrastructure.persistence.jpa`), domínio sem nenhuma anotação de framework. Schema via Flyway (`db/migration/V1__contas_servidores_proxies.sql`, `hibernate.ddl-auto=validate`); tabela `proxies` deliberadamente sem `UNIQUE` (fiel ao `PoolDeProxies` in-memory, que sempre aceitou duplicatas). Credenciais via variável de ambiente (mesmo padrão de `advancedbot.api.key`), role/banco dedicados criados num PostgreSQL 18 já instalado na máquina. Achado técnico: `EntityScan`/`EnableJpaRepositories` precisaram ser explícitos em `AdvancedBotApplication` — `AutoConfigurationPackages` não segue `scanBasePackages` customizado. 6 testes de integração REST novos (`ContaControllerTest`/`ServidorControllerTest`/`ProxyControllerTest`, primeiros testes de Controller HTTP do projeto — lacuna aberta desde a DEC-40), `@SpringBootTest`+`MockMvc` contra PostgreSQL real dedicado a testes (`advancedbot_test`), isolamento via `@Transactional`. Bug de CORS encontrado e corrigido em `ApiKeyFilter`: preflight `OPTIONS` era barrado com 401 antes do CORS do Spring MVC rodar (preflight nunca carrega `X-API-Key` por spec do navegador), quebrando CORS para qualquer requisição não-simples a partir de um browser. Zero DEC de protocolo/domínio reaberta, zero contrato de API existente alterado. 1098→1104 testes automatizados (+6), 0 falhas, 0 erros, 3 skipped deliberadamente; validação manual adicional via `mvn spring-boot:run` real contra PostgreSQL de desenvolvimento (Flyway migra e depois confirma idempotência; CRUD via `curl`; preflight CORS confirmado corrigido; WebSocket `/ws/events` conectado via Node.js nativo) | ✔ |
| 2026-07-27 | Milestone Frontend 01 (EPIC-FRONT-01 Frontend, encerramento): arquitetura do frontend já congelada em sessão anterior (`docs-reescrita/docs/12-Interface/06-Plano-Construcao-Frontend.md`/`07-Matriz-Frontend-Backend.md`, zero decisão em aberto); esta sessão implementa integralmente a fundação, sem nenhuma tela de negócio. Projeto `advancedbot-frontend/` (Vite + React 19 + TypeScript) criado na raiz, irmão de `advancedbot-java/`. Tailwind v4 com tema claro/escuro por classe (slice Zustand `uiStore` persistido); React Router v7 (data router, `AppShell` + rota de fundação + 404); TanStack Query (`QueryCache`/`MutationCache.onError` centralizados, todo erro vira `AppError` tipado e Toast automático); Zustand (`uiStore`, `toastStore`); Axios (`httpClient` único, interceptors `X-API-Key`/`ErrorResponse`→`AppError`); mitt (`wsBus`, `ManagedSocket` com reconexão por backoff exponencial, canais `/ws/events` global e `/ws/bots/{id}/events` por bot, parser único do envelope `EventoDeBot`/`IdentificadorBot` incluindo o formato `{"value": uuid}` da DEC-40). Orval configurado contra `/v3/api-docs` (springdoc, já presente no backend) e executado com sucesso contra o backend real: 13 controllers, 57 paths, 71 arquivos gerados em `shared/api/generated` — nenhum DTO de REST escrito à mão a partir de agora. Design System base: `Button`/`Input`/`NumberInput`/`SearchBox`/`Card`/`Badge`/`Tooltip`/`Spinner`/`Skeleton`/`ThemeToggle` (atômicos), `Modal`/`ConfirmDialog`/`DataTable` (moleculares, virtualização própria sem lib externa acima de 200 linhas), `Toast`/`EmptyState`/`ErrorState`/`ErrorBoundary` (feedback), `Sidebar`(genérico, recebe `items` por propriedade)/`TopBar`/`Workspace`/`AppShell`/`PageContainer`/`PageHeader` (layout). ESLint (flat config) + Prettier configurados; Vitest + React Testing Library + MSW (`src/test/setup.ts`/`server.ts`). Nota de nomenclatura: o rótulo `EPIC-FRONT-01` já havia sido usado na Milestone 42 para o backend (persistência/CORS) — mantido pelo responsável do projeto para esta milestone de frontend também, registrado explicitamente para não confundir as duas. 19 testes automatizados (Vitest), 0 falhas. Validação: `tsc -b --noEmit` sem erros, `eslint .` sem erros/avisos, `prettier --check .` sem divergências, `vitest run` 19/19, `npm run build` sucesso (bundle ~373 KB/~120 KB gzip); validação manual no navegador com backend real rodando (JDK 21, PostgreSQL 18 local) — `AppShell` renderiza, tema dark por padrão, toggle de tema e colapso de sidebar funcionam, WebSocket global conectado sem erros de console. Limitação conhecida: `npm audit` reporta uma vulnerabilidade alta em `react-router-dom` (GHSA-qwww-vcr4-c8h2, modo RSC/framework, não exercido por esta SPA em modo biblioteca) presente até a última versão publicada — documentada, não corrigível upstream ainda | ✔ |
| 2026-07-27 | Milestone Frontend 02 (EPIC-FRONT-02, encerramento): primeira feature funcional do frontend sobre a fundação da Milestone Frontend 01 — `features/dashboard/` completa (components/hooks/services/pages/tests), consumindo exclusivamente hooks gerados pelo Orval, zero chamada Axios manual na feature. Achado/correção antes da implementação: `orval.config.ts` tinha `override.query: {useQuery:true, useMutation:true}` — com os dois juntos o Orval 8.23 inverte a geração (GET vira `useMutation`, POST/PUT/DELETE viram `useQuery`), defeito que só apareceu ao consumir o primeiro hook GET de verdade (`useMetricas`); corrigido para `{signal:true}` (Orval decide pelo verbo HTTP, padrão correto), `shared/api/generated` regenerado e conferido nos 13 controllers. Entregue: `services/formatters.ts`/`deriveMetrics.ts` (funções puras, sem HTTP); `hooks/useDashboardMetrics` (wrap de `useMetricas` + `refetchInterval` 5s de fallback + invalidação imediata via `wsBus` no evento `"estado"` do canal `/ws/events`); `hooks/useDashboardTotals` (wrap de `useListar2`/`useListar`/`useListar1` de contas/servidores/proxies, `staleTime` 5min — nenhum dos três tem WS); `hooks/useMetricsHistory` (acumula até 30 amostras reais de CPU/TPS em memória para os gráficos, sem endpoint de série histórica no backend); componentes `MetricCard`/`StateBreakdownCard`/`MetricTrendChart` (sparkline SVG sem lib nova)/`DashboardGapNotice`; `DashboardPage` com 12 `MetricCard`, 2 `StateBreakdownCard`, 2 `MetricTrendChart`, loading (skeleton de página inteira), error (`ErrorState` com retry), grid responsivo, tema claro/escuro herdado. 3 GAPs registrados explicitamente na própria tela (não simulados): Threads da JVM (ausente no backend), Proxy em uso (`ProxyResponse` sem campo de associação a bot, só total cadastrado), Heap vs. memória não-heap (backend só expõe heap — "Memória JVM" e "Heap" pedidos como cards separados colapsam num único card real). `DashboardPage` virou rota raiz (`/`), `FoundationPage` removida (código morto). 44 testes automatizados (19→44, +25: `formatters` 13, `deriveMetrics` 6, `MetricCard` 2, `DashboardPage` 3 com MSW). Validação: `tsc -b --noEmit`/`eslint .`/`prettier --check .` sem erros, `vitest run` 44/44, `npm run build` sucesso; validação manual contra backend Java real (JDK 21, PostgreSQL 18 local) incluindo criar/iniciar um bot via `curl` e observar a tela atualizar sozinha (Bots totais 0→1, distribuição por estado refletindo Executando/Desconectado) sem recarregar a página, tema escuro alternado, viewport mobile (375px) sem overflow horizontal, console sem erros em todos os passos; dados de teste removidos do banco de desenvolvimento ao final | ✔ |
| 2026-07-27 | Milestone Frontend 03 (EPIC-FRONT-03, encerramento): segunda feature funcional do frontend, `features/proxy/` completa (components/hooks/services/pages/tests), backend já em execução via IntelliJ durante todo o desenvolvimento. Entregue: `Select` atômico novo no Design System (`shared/components/atoms/Select.tsx` — já especificado desde a Fase 0, implementado agora para o campo `tipo`); `services/proxyValidation.ts` (host/porta 1-65535/tipo dentre HTTP-SOCKS4-SOCKS5, sem HTTP); `services/filterProxies.ts` (busca/paginação client-side, backend não tem esses parâmetros em `GET /proxies`); `hooks/useProxyList` (wrap `useListar1`, `staleTime` 5min); `hooks/useProxyTableState` (busca+página, estado de UI local); `hooks/useProxyMutations` (`useCreateProxy`/`useDeleteProxy` wrap de `useAdicionar`/`useRemover1` gerados + invalidação + toast; `useUpdateProxy` mutation composta chamando `remover1`+`adicionar` gerados em sequência); componentes `ProxyTable` (especialização de `DataTable`, sem coluna de latência — GAP conhecido) e `ProxyFormModal` (criar/editar, reaproveita `Modal`/`ConfirmDialog`); `ProxyPage` com `PageHeader`+`SearchBox`+`ProxyTable`+paginação+modal+confirmação de exclusão; rota `/proxy` lazy-loaded (primeiro uso real de code-splitting por rota, decisão travada #5); correção incidental no `MutationCache.onError` global (`shared/lib/queryClient.ts`) para preservar a mensagem de erros que não são `AppError` (necessário para o erro específico de `useUpdateProxy`, beneficia qualquer mutation composta futura). **Achado de backend (documentado ao final, não bloqueou a entrega)**: `ProxyController.java` não tem `PUT` — proxy não tem identidade própria por entrada (chave natural host+port+tipo, comentário no próprio código confirma que é deliberado); "editar" implementado como remover+adicionar client-side com as mesmas funções geradas, sem endpoint novo; se a remoção for bem-sucedida e a criação falhar a entrada original já foi perdida (mensagem de erro específica avisa o operador); backend permite duplicatas exatas (sem `UNIQUE`) e `remover()` remove só a primeira ocorrência — com duplicatas na lista, editar/excluir uma linha pode afetar a outra ocorrência idêntica (limitação herdada do modelo de dados do backend). 64 testes automatizados (44→64, +20: `proxyValidation` 4, `filterProxies` 6, `ProxyFormModal` 3, `ProxyTable` 2, `ProxyPage` 5 com MSW). Validação: `tsc -b --noEmit`/`eslint .`/`prettier --check .` sem erros (1 aviso pré-existente não bloqueante), `vitest run` 64/64, `npm run build` sucesso com `ProxyPage` confirmado como chunk separado; validação manual contra o backend já em execução — criação/edição (porta 1080→1081 sem duplicar)/exclusão de uma proxy real confirmadas via `curl` direto no backend a cada passo, viewport mobile (375px) sem overflow, console do navegador sem erros (um erro de HMR transitório de um refactor de nome em andamento não se repetiu após reiniciar o servidor de desenvolvimento e recarregar) | ✔ |
| 2026-07-27 | Milestone Frontend 04 (EPIC-FRONT-04 "Fundação Administrativa", encerramento): quatro features independentes da Fase 1 implementadas na mesma sessão (Contas, Servidores, Catálogo Comandos/Macros, Configurações), reutilizando integralmente a fundação das Milestones Frontend 01-03, backend já em execução via IntelliJ durante todo o desenvolvimento. `shared/lib/pagination.ts` (`paginate`/`totalPages`) extraído de `features/proxy/services/filterProxies.ts` no momento em que Contas/Servidores precisaram da mesma lógica (regra de promoção do Design System, 2º consumidor real); `shared/components/navigation/Tabs.tsx` novo, promovido direto para `shared/components` porque nasceu com dois consumidores já no mesmo épico (Contas/Servidores e Catálogo alternam sub-recurso na mesma rota). `features/contas-servidores/` (CRUD completo de Conta e Servidor, uma rota `/contas-servidores` com abas `ContasPanel`/`ServidoresPanel`): `services/contaValidation.ts`/`servidorValidation.ts`, `services/filterEntities.ts` (busca client-side, mesmo padrão de Proxy), `hooks/useContaList`/`useServidorList` (`GET /contas`/`GET /servidores` com `offset=0&limit=500`, paginação client-side sobre o lote completo — decisão explícita do responsável mesmo o backend suportando `offset`/`limit` real), `hooks/useContaMutations`/`useServidorMutations` (`useCreateConta`/`useCreateServidor`, `useDeleteConta`/`useDeleteServidor`, `useUpdateConta`/`useUpdateServidor` mutation composta remover+criar, mesmo padrão de `useUpdateProxy` — ver GAP abaixo), componentes `ContaTable`/`ContaFormModal`/`ServidorTable`/`ServidorFormModal`. `features/catalogo/` (somente leitura, rota `/catalogo` com abas `ComandosPanel`/`MacrosPanel` sobre `GET /commands`/`GET /macros`): `services/filterCatalogo.ts`, `hooks/useCatalogoComandos`/`useCatalogoMacros`/`useCatalogoTableState` (reused por ambas as abas, mesmo DTO `ComandoCatalogoResponse`), componente `CatalogoTable` reutilizado pelas duas abas; agrupamento avaliado e não aplicado — `ComandoCatalogoResponse` não expõe nenhuma categoria/taxonomia, agrupar seria inventar estrutura ausente no backend. `features/configuracoes/` (rota `/configuracoes`, somente leitura sobre `GET /configuracao/reconnect-policy`): `hooks/useConfiguracao`, componente `ReconnectPolicyPanel` (fica local à feature, único consumidor). `AppShell`/`router.tsx` atualizados com as 3 novas rotas lazy-loaded e itens de navegação; badge de versão do `TopBar` atualizado para `EPIC-FRONT-04`. **GAP de backend confirmado (documentado, não implementado como solução de contorno)**: `conta-controller.ts`/`servidor-controller.ts` não têm `PUT` (só `listar*/criar*/detalhar*/remover*`) — mesma limitação estrutural já registrada para Proxy na Milestone Frontend 03, mas agravada aqui porque Conta/Servidor têm identidade própria por `id` (UUID gerado pelo backend): "editar" via remover+criar client-side troca o `id` da entidade (diferente de Proxy, cuja chave é natural); se a remoção for bem-sucedida e a criação falhar, a entidade original já foi perdida (mesma mensagem de erro específica de `useUpdateProxy`). 102 testes automatizados (64→102, +38: `pagination` 3, `Tabs` 2, `contaValidation` 5, `servidorValidation` 5, `filterEntities` 6, `filterCatalogo` 4, `ContasServidoresPage` 7 com MSW, `CatalogoPage` 4 com MSW, `ConfiguracoesPage` 2 com MSW). Validação: `tsc -b --noEmit`/`eslint .` sem erros (4 avisos pré-existentes de fast-refresh em `router.tsx`, não bloqueantes, mesma categoria já registrada), `vitest run` 102/102, `npm run build` sucesso com `ContasServidoresPage`/`CatalogoPage`/`ConfiguracoesPage`/`Tabs`/`pagination` confirmados como chunks separados; validação manual completa contra o backend real via `.claude/launch.json` (`npm --prefix advancedbot-frontend run dev`) e Browser pane — criar/editar/excluir uma Conta e um Servidor reais (CRUD completo nas duas abas), listar e buscar no Catálogo de Comandos (3 páginas reais) e Macros (8 itens reais) com paginação e busca funcionando, ler a Política de Reconexão real (2000ms/1000ms), console do navegador sem erros em todos os passos; dados de teste (1 conta, 1 servidor) removidos do banco de desenvolvimento ao final | ✔ |
| 2026-07-27 | Milestone Frontend 05 (EPIC-FRONT-05 "Feature Bots", encerramento): núcleo da Fase 2 do roadmap, `features/bots/` completa (components/hooks/services/pages/tests), reutilizando integralmente a fundação e os padrões das Milestones Frontend 01-04, backend já em execução via IntelliJ durante todo o desenvolvimento. Promovido para `shared` no momento em que a 2ª necessidade real apareceu (troca de proxy de bot usa o mesmo formato host+porta+tipo do cadastro de Proxy): `shared/types/proxy.ts` (`TIPOS_DE_PROXY`/`TipoDeProxy`) e `shared/lib/proxyFormValidation.ts` (`validateProxyForm`/`isProxyFormValid`/`ProxyFormValues`), ambos antes só em `features/proxy/services/proxyValidation.ts` (mantido como reexport, nenhum import existente quebrou). `features/bots/services/botState.ts` (`EstadoDeExecucao`/`EstadoDeSessao` tipados manualmente como view-model — `estadoExecucao`/`estadoSessao` são `string` cru no OpenAPI, valores reais confirmados em `EstadoExecucao.java`/`EstadoSessao.java`), `services/botValidation.ts` (validação pura refletindo a regra real de `POST /api/v1/bots`: conta por `contaId` OU credenciais inline, servidor por `servidorId` OU host+porta inline, confirmado em `BotController.resolverCredenciais`/`resolverEndereco`), `services/filterBots.ts` (busca client-side). `hooks/useBotList` (wrap `useListar3`, `offset=0&limit=500`, paginação client-side, `staleTime` 10s + invalidação via `wsBus` no evento `"estado"` do canal global), `hooks/useBotTableState`, `hooks/useBotMutations` (`useCreateBot`/`useDeleteBot`, sem `useUpdateBot` — GAP abaixo), `hooks/useBotActions` (uma mutation por ação: iniciar/parar/pausar/retomar/conectar/desconectar/reconectar/trocarProxy/definirAutoReconnect; `useBatchBotAction` compõe ações em lote via `Promise.allSettled` sobre as funções cruas geradas). Componentes `BotStatusBadge` (execução e sessão como eixos independentes), `BotTable` (especialização de `DataTable`, botões de ação contextuais por estado — evita oferecer ação inválida ao operador), `BotFormModal` (reusa `Tabs` para alternar conta/servidor existente vs. inline), `BotProxyModal`. `BotsPage` com `PageHeader`+`SearchBox`+barra de ações em lote+`BotTable`+modais+confirmação de exclusão; rota `/bots` lazy-loaded, item "Bots" na `Sidebar`; badge de versão do `TopBar` atualizado para `EPIC-FRONT-05`. **GAP de backend confirmado (documentado, não implementado como solução de contorno)**: sem "editar" bot — diferente de Proxy/Conta/Servidor, a composição client-side remover+criar não é segura aqui porque `BotResponse` nunca devolve a senha usada na criação (`CriarBotRequest.password` é write-only); a Feature Bots oferece só criar e excluir. Ações em lote continuam sendo N chamadas paralelas client-side (sem `connect-batch` nativo, GAP §8 já registrado). 108 testes automatizados (102→108, +6: `BotsPage` com MSW — listagem, empty state, criação com credenciais e host/porta manuais, ação de iniciar com atualização de estado, exclusão com confirmação, ação em lote). Validação: `tsc -b --noEmit`/`eslint .` sem erros (5 avisos pré-existentes de fast-refresh em `router.tsx`, mesma categoria já registrada, não bloqueantes), `prettier --check .` sem divergências após `--write` (17 arquivos reformatados, nenhuma mudança de conteúdo/lógica), `vitest run` 108/108, `npm run build` sucesso com `BotsPage` confirmado como chunk separado; validação manual completa contra o backend real via Browser pane — criar um bot real com credenciais e host/porta manuais, iniciar (estado mudou para Executando com botões contextuais corretos), trocar proxy real (badge atualizada), excluir com confirmação (voltou ao EmptyState), console do navegador sem erros em todos os passos | ✔ |
| 2026-07-27 | Milestone Frontend 06 (EPIC-FRONT-06 "Bot Details", encerramento): núcleo restante da Fase 2 do roadmap, `features/bots/details/` completa (hooks/services/components/pages/tests), reutilizando integralmente a fundação e os padrões das Milestones Frontend 01-05, backend já em execução via IntelliJ durante todo o desenvolvimento. 5 sub-abas roteadas sob `/bots/:id` (rotas aninhadas, decisão travada #5): Console (`hooks/useRealtimeLogs` — busca inicial via `useListar5` + buffer WS `"log"` concatenados na leitura, sem sincronizar `query.data` para `useState`; `hooks/useBotConsole` — chat via `useEnviarMensagemDeChat`, comando via `useExecutar`, histórico duplo REST+WS `"comando"`; componente `ConsoleLogViewer` novo), Ações (`hooks/useBotActionsPanel`, 14 mutations de `acao-controller`, invalida `getEstadoQueryKey` em ações de movimento/câmera/postura; página `AcoesPage`), Inventário (`hooks/useInventario`/`useInventarioActions`, 3 queries + 10 mutations de `inventario-controller`; componentes novos `ItemSlot`/`InventorySlotGrid` — sem grade no Design System ainda), Mundo (`hooks/useEstadoMundo`/`useMundoEntidades`/`useBlocoLookup`, polling leve sem WS dedicado; página `MundoPage` com `DataTable` reusado para entidades/jogadores), Macros (`hooks/useBotMacros`, cruza `useListar4`/`useAtivar`/`useDesativar` do `macro-controller` com `useCatalogoMacros` do Catálogo global — 2º consumidor real do catálogo de macros; invalida também `getDetalhar2QueryKey(id)` para manter `BotResponse.macrosAtivas` sincronizado). Canal WS por bot (`connectBotEventsSocket`, infraestrutura da Fundação nunca antes exercitada) conectado por `hooks/useBotEventsSocket` no layout (`BotDetailsPage`), mantido vivo entre troca de abas; `BotDetailsPage` também usa `hooks/useBotDetail` (wrap de `useDetalhar2` + invalidação no evento WS global `"estado"` filtrado por `botId`) para o estado compartilhado do bot selecionado. `BotTable` da listagem (EPIC-FRONT-05) ganhou link de navegação (`username` → `/bots/{id}/console`). **3 GAPs de backend confirmados manualmente contra o backend real (documentados, não contornados)**: (1) `GET /estado`, todo `/mundo/*` e todo `/inventario/*` respondem `409 "Bot não está em uma sessão de jogo ativa (PLAY)"` quando o bot não está conectado a um servidor — confirmado criando um bot e nunca conectando-o, tratado como `ErrorState`+toast normal; (2) `MacroResponse.tipo` (única informação devolvida para macros ativas) frequentemente não corresponde a `nome`/`aliases` do Catálogo — ativar `antiafk` e reconsultar `GET /macros` devolveu `{"tipo":"TarefaAntiAFK"}`, valor derivado internamente pelo backend, não o alias original; a correspondência com o catálogo cai para o valor cru quando não há match; (3) `DELETE /bots/{id}/macros/{alias}` usando `tipo` como `alias` (única opção dado o GAP #2) respondeu `200 OK` duas vezes em teste manual, mas `GET /macros` continuou devolvendo a mesma macro ativa depois — confirmado via `read_network_requests` na Browser pane, sem efeito observável com o bot desconectado. 116 testes automatizados (108→116, +8: `BotDetailsPage` com MSW cobrindo as 5 sub-abas — cabeçalho, navegação por `Tabs`, Console chat+comando+limpar logs, Ações mover, Inventário grid+janela vazia, Mundo estado+bloco+listas vazias, Macros ativar); corrigido incidentalmente `BotsPage.test.tsx` (EPIC-FRONT-05), que não tinha `MemoryRouter` e quebrou quando `BotTable` ganhou `<Link>`. Validação: `tsc -b --noEmit`/`eslint .` sem erros (11 avisos pré-existentes de fast-refresh em `router.tsx`, cresceu de 5 para 11 só por mais rotas lazy, mesma categoria já registrada), `prettier --check .` sem divergências, `vitest run` 116/116, `npm run build` sucesso com `BotDetailsPage`/`ConsolePage`/`AcoesPage`/`InventarioPage`/`MundoPage`/`MacrosPage` confirmados como chunks separados; validação manual completa contra o backend real via Browser pane — bot real criado e navegado até o Console pelo link da listagem, comando `help` executado com resposta e saída completa chegando via evento WS `"log"` em tempo real (canal per-bot confirmado ponta a ponta), Mundo/Inventário exibindo corretamente o `409` real do backend, catálogo real de macros carregado (8 macros) e `antiafk` ativada com sucesso (descobrindo os GAPs #2/#3 nesse mesmo passo), aba Ações renderizada sem erro, bot de teste excluído ao final, console do navegador sem erros em nenhum passo | ✔ |
| 2026-07-28 | Milestone 43 (EPIC-PROD-01, DEC-43, encerramento): mudança de fase — de "portar funcionalidades" para "produto utilizável no dia a dia". Único objeto do domínio ainda 100% em memória (`Bot`/`GerenciadorDeBots`, confirmado por auditoria prévia contra a implementação Java real, legado C# deliberadamente não consultado) ganha persistência completa, seguindo o mesmo padrão de `RepositorioDeContas`/`RepositorioDeServidores`/`RepositorioDeProxies` (DEC-42): porta nova `RepositorioDeBots` (`application.port`) + adapter `RepositorioDeBotsJpa` (`infrastructure.persistence`) + `BotJpaEntity`/`BotMacroJpaEntity` (`infrastructure.persistence.jpa`, domínio sem nenhuma anotação de framework) + `db/migration/V2__bots.sql` (tabelas `bots`/`bot_macros`, FK `ON DELETE CASCADE`). Diferente de Conta/Servidor/Proxy (CRUD simples), `Bot` é um agregado mutável com >10 Casos de Uso alterando estado em produção — decisão de design: em vez de um único `salvar(Bot)`, a porta expõe 4 operações (`criar`/`atualizarConfiguracao`/`sincronizarMacros`/`remover`), cada uma chamada pelo Caso de Uso que já realiza a mutação em memória correspondente (`CasoDeUsoCriarBot`/`IniciarBot`/`PararBot`/`PausarBot`/`RetomarBot`/`DefinirAutoReconnect`/`TrocarProxy`/`ConectarBot`/`DesconectarBot`/`RemoverBot`, todos ganham `RepositorioDeBots` como colaborador novo, nenhuma assinatura pública alterada; `MacroController` ganha o mesmo colaborador para sincronizar `Bot.getTarefasContinuas()` após cada ativação/desativação de macro). `CasoDeUsoCriarBot.criar` ganha 2 overloads aditivos (`(EnderecoServidor,CredenciaisBot,UUID contaId,UUID servidorId)` e `(IdentificadorBot,EnderecoServidor,CredenciaisBot,UUID,UUID)`) — o overload de 2 argumentos original permanece intacto para os call sites/testes existentes; o overload com `IdentificadorBot` explícito é o que permite ao restaurador reaproveitar o id original em vez de gerar um novo. Reconstrução no boot: `RestauradorDeBots` (`application.bootstrap`, `ApplicationRunner` novo — único componente sem Caso de Uso pré-existente por trás, porque nenhuma peça anterior orquestrava "ler tudo do repositório e recriar", só orquestração, zero regra de negócio nova) lê `RepositorioDeBots.listar()` no boot e, para cada bot persistido, chama exatamente os mesmos Casos de Uso que a API usa em operação normal: `CasoDeUsoCriarBot.criar(id,...)` (registra em `GerenciadorDeBots`/`MotorDeExecucaoPort`/`GerenciadorDeReconexao`, os 3 registries que hoje só eram populados via API), depois `CasoDeUsoDefinirAutoReconnect` se aplicável, depois replay de cada macro persistida via `GerenciadorDeComandos.executar(bot, alias)` (mesmo comando que a ativação normal usaria), depois `CasoDeUsoConectarBot.connect(bot)` se o estado desejado era `EXECUTANDO`/`PAUSADO` (reconstitui a sessão de rede de verdade; falha de conexão é tolerada e logada — se `autoReconnect=true` o bot já está registrado em `GerenciadorDeReconexao` e a reconexão automática assume dali, mesmo caminho de uma queda em produção). **Limitação documentada, decisão consciente**: macro `Follow` (alias `follow`) é a única das 8 macros deliberadamente excluída da restauração — depende de um `entityId` de jogador só válido dentro da `SessaoDeJogo` em que foi criada (`TarefaFollow`), não sobrevive a uma reconexão de jeito nenhum, com ou sem persistência; as outras 7 (`antiafk`/`herbalismo`/`miner`/`mob`/`autofish`/`dropall`/`twerk`) são restauradas com os argumentos-padrão de cada comando (não com os argumentos exatos usados na ativação original — `TarefaContinua` não expõe accessor de configuração hoje, mesmo GAP já registrado na Milestone Frontend 06 item 2 sobre `MacroResponse.tipo`; persistir os argumentos exigiria uma mudança de contrato fora deste escopo). **Bug real encontrado e corrigido durante a validação manual, não durante os testes automatizados**: `BotMacroSpringDataRepository.deleteByBotId` (query derivada Spring Data com prefixo `delete`) lançava `TransactionRequiredException` sempre que havia ao menos uma linha para remover, porque — ao contrário dos métodos herdados de `SimpleJpaRepository` (`save`/`deleteById`), que já nascem `@Transactional` — um método de query derivada customizado só executa `EntityManager.remove()` dentro de transação se isso for pedido explicitamente; o teste automatizado inicial não pegou o bug porque rodava com `@Transactional` de classe (mascarando a ausência de transação própria do método); corrigido com `@Transactional` explícito no método, e o teste de integração (`RepositorioDeBotsJpaTest`) foi reescrito sem `@Transactional` de classe (limpeza manual via `@AfterEach`) justamente para não mascarar esse tipo de regressão de novo. Zero Port pré-existente alterada, zero contrato REST alterado, zero arquitetura alterada. 1104→1109 testes automatizados (+5: `RepositorioDeBotsJpaTest` com 4 cenários incluindo o de regressão do bug de transação, mais os ajustes de fakes nos testes pré-existentes de Casos de Uso de Bot). Validação manual obrigatória executada via `mvn spring-boot:run` real (JDK 21, PostgreSQL 18 local) + frontend real: 3 bots criados (host/porta fictícios, sem servidor Minecraft real disponível no ambiente), 2 iniciados, 1 com `autoReconnect` ligado, macros `antiafk`/`twerk` ativadas, API derrubada via `taskkill` no processo real (não um restart gracioso) e religada do zero — confirmado após o restart: os 3 bots continuam cadastrados com os mesmos `id`s, `autoReconnect`/macros idênticos ao estado anterior, log `RestauradorDeBots` confirmando as 3 restaurações e a tentativa automática de reconexão para o bot com `autoReconnect=true` (falha esperada, host fictício), frontend (`/bots`) exibindo os mesmos 3 bots sem novo cadastro; exclusão de todos os bots ao final confirmada com `204` (incluindo os bots com macro ativa, validando o fix do bug de transação) | ✔ |
| 2026-07-28 | Milestone 44 (EPIC-PROD-02 "Experiência Operacional", encerramento): mudança de foco dentro da fase "produto utilizável no dia a dia" — de infraestrutura (Milestone 43) para reduzir passos do fluxo operacional existente, sem feature nova, sem alterar arquitetura/domínio/protocolo (projeto C# deliberadamente não consultado). Auditoria prévia (read-only, contra a aplicação real rodando) confirmou o estado antes da mudança: criar um Bot com conta/servidor ainda não cadastrados exigia 3 telas (`/contas-servidores` aba Contas → aba Servidores → `/bots`) mais 2 ações pós-criação separadas (`BotProxyModal`, `/bots/:id/macros`); quando conta/servidor eram informados inline no `POST /bots`, o backend nunca os persistia (ficavam presos ao Bot, `contaId`/`servidorId` nulos no snapshot) — nenhum Caso de Uso de criação existia para Conta/Servidor, a lógica estava inline em `ContaController`/`ServidorController`. **Backend**: `CasoDeUsoCriarConta`/`CasoDeUsoCriarServidor` novos (`application.usecase`, mesmo padrão casca-fina de `CasoDeUsoTrocarProxy`/`CasoDeUsoDefinirAutoReconnect`), registrados em `ConfiguracaoDeCasosDeUso`; `ContaController`/`ServidorController` passam a chamar esses Casos de Uso em vez de montar `Conta.nova`/`PerfilDeServidor.novo` inline (zero mudança de rota/contrato). Dois helpers estáticos novos em `interfaces.rest` (mesmo padrão de `BotLookup`): `MacroAtivacaoSupport.ativar` (extraído de `MacroController.ativar`/`desativar`, agora compartilhado) e `ProxySupport.resolver` (extraído de `BotController.trocarProxy`, agora compartilhado). `CriarBotRequest` ganha 6 campos opcionais aditivos: `nomeServidor`, `proxyHost`/`proxyPort`/`proxyTipo`, `autoReconnect`, `macroInicial`. `BotController.criar` (fluxo composto): quando `contaId`/`servidorId` não vêm na requisição, `resolverCredenciais`/`resolverEndereco` agora persistem uma Conta/`PerfilDeServidor` novos via os Casos de Uso acima (`nomeServidor` opcional, default = `host`) em vez de descartar os dados; proxy é aplicado direto no `EnderecoServidor` da criação (`EnderecoServidor` já suportava proxy no construtor desde sempre, sem mudança de domínio); `autoReconnect`/`macroInicial`, quando informados, disparam `CasoDeUsoDefinirAutoReconnect`/`MacroAtivacaoSupport.ativar` logo após a criação — confirmado que `GerenciadorDeComandos.executar` não tem gate de `EstadoSessao.PLAY`, então a macro inicial ativa mesmo com o bot ainda desconectado. **Frontend**: `BotFormModal` (já usava `Tabs` para alternar conta/servidor existente vs. inline desde o EPIC-FRONT-05) ganha os 3 campos que faltavam no mesmo modal — toggle "Usar proxy" (reaproveita `ProxyFormValues`/`validateProxyForm` de `shared/lib/proxyFormValidation.ts`, zero duplicação de validação), `Select` "Macro inicial" (reaproveita `useCatalogoMacros`, 3º consumidor real do catálogo) e checkbox "Auto reconnect"; `botValidation.ts` estende `BotFormValues`/`validateBotForm` de forma aditiva. `BotTable`: coluna "Macros ativas" ganha `Tooltip` com os nomes reais (antes só mostrava a contagem), sem navegação extra até `/bots/:id/macros`. Campo "versão" (Minecraft) deliberadamente **não** incluído no formulário — decisão validada com o responsável: domínio não tem nenhum conceito de versão (protocolo fixo `domain.protocol.v1_8`), incluir o campo exigiria tocar domínio/protocolo, fora de escopo deste épico (GAP registrado abaixo). Zero DEC nova (consolidação de lógica já existente atrás de Casos de Uso, mesmo padrão arquitetural, não decisão arquitetural nova), zero Port pré-existente alterada, zero contrato REST pré-existente quebrado (só campos aditivos), zero teste automatizado pré-existente quebrado (`ContaControllerTest`/`ServidorControllerTest` continuam verdes via `@SpringBootTest`/`MockMvc`, sem instanciação manual de controller). **Passos eliminados (cenário real medido, conta/servidor 100% novos)**: antes — 3 telas (`/contas-servidores`×2 abas + `/bots`), 3 submits de formulário (Conta, Servidor, Bot) mais 2 ações pós-criação (Proxy, Macro inicial) = **5 submits/ações + 3 navegações de tela**; depois — 1 tela (`/bots`, modal "Novo bot"), **1 submit** cobrindo nickname/senha/servidor/proxy/macro inicial/auto-reconnect, **0 navegações prévias**. Validado manualmente via `mvn spring-boot:run` real (JDK 21, PostgreSQL 18 local) + frontend real (Browser pane): Bot #1 criado com conta ("Steve2") e servidor ("mc.exemplo.com", `nomeServidor` default = host) 100% novos, proxy (SOCKS5), macro inicial (`antiafk`) e auto-reconnect num único `POST /api/v1/bots` → `201`, resposta confirmando `autoReconnect:true`, `proxy` preenchido e `macrosAtivas:["TarefaAntiAFK"]`; Conta e Servidor confirmados persistidos via `GET /contas`/`GET /servidores` (reaproveitáveis); Bot #2 criado reaproveitando a mesma Conta/Servidor pelos `Select` "existente" — `GET /contas`/`GET /servidores` confirmados em `total:2` antes e depois (zero duplicação); ambos os bots excluídos ao final via `ConfirmDialog` (`204`), console do navegador sem erros em nenhum passo. `mvn -o compile` limpo, `tsc -b --noEmit`/`eslint .` sem erros. **3 GAPs registrados para EPIC-PROD-03 (Viewer Operacional)**: (1) campo "versão"/multi-protocolo sem suporte de domínio (protocolo fixo 1.8); (2) vínculo Bot→Conta/Servidor/Proxy não exposto na UI — `Bot` (domínio em memória) não guarda `contaId`/`servidorId`, só o snapshot de persistência guarda, expor exigiria tocar domínio ou I/O extra por bot (`BotResponse` é documentado para não fazer I/O); (3) painel unificado de estado — hoje o estado de um bot continua fragmentado entre listagem/cabeçalho de detalhe/sub-abas (`/bots/:id/console`, `/acoes`, `/inventario`, `/mundo`, `/macros`), fora de escopo deste épico por virar Viewer | ✔ |
| 2026-07-28 | Milestone 45 (EPIC-PROD-03 "Painel Operacional Unificado", encerramento): resolve o GAP #3 registrado na Milestone 44 (painel unificado de estado). Auditoria prévia (read-only) confirmou que **todo** dado pedido (status/conta/servidor/proxy/autoReconnect/macro ativa/console/vida/fome/coordenadas/dimensão/inventário/equipamento/janela/entidades/jogadores/bloco) já era servido por endpoints REST + eventos WS existentes, cada um já consumido por um hook próprio nas 5 sub-abas de `/bots/:id` (Console/Ações/Inventário/Mundo/Macros) — **épico 100% frontend, zero mudança de backend**. Nova rota `/bots/:id/painel` (`features/bots/details/pages/PainelOperacionalPage.tsx`), inserida como 6ª sub-aba e promovida a `DEFAULT_BOT_DETAILS_TAB` (`services/botDetailsNav.ts` — antes `'console'`, agora `'painel'`; o `index` redirect de `BotDetailsPage` já usa essa constante, sem mudança extra), reúne coluna esquerda (status/conta/servidor/proxy/autoReconnect/macros ativas/últimos comandos, via `useBotDetail`+`useBotConsole`), centro (console completo, componente novo `ConsolePanel` — ver abaixo) e coluna direita (vida/fome/coordenadas/dimensão via `useEstadoMundo`, equipamento via `ItemSlot`, inventário via `InventorySlotGrid`+`useInventarioActions.clicarSlot`, janela condicional) num grid CSS responsivo (`xl:grid-cols-[280px_1fr_320px]`), mais rodapé (entidades próximas/jogadores online via `DataTable`+`useMundoEntidades`, consulta de bloco via `useBlocoLookup`). 100% composição de hooks/componentes das 5 sub-abas já existentes — nenhum hook gerado novo, nenhuma chamada Axios manual nova. Único componente novo de fato: `features/bots/details/components/ConsolePanel.tsx`, extraído do corpo de `ConsolePage` (logs+chat+comando+histórico) pra ser reusado por `ConsolePage` e pelo Painel sem duplicar a composição — ambos os consumidores agora recebem `logs` (`useRealtimeLogs`) e `botConsole` (`useBotConsole`) como props injetadas pelo próprio chamador (nunca chamados dentro do `ConsolePanel`), decisão deliberada pra evitar que o Painel assine os canais WS `bot:{id}` (`"log"`/`"comando"`) duas vezes — a coluna de status e o console central compartilham a mesma instância de `useBotConsole` (mesmo array `history` alimenta os "últimos comandos" à esquerda e o histórico completo no centro). **GAP confirmado, não implementado (proibido pelo escopo — "não adicionar funcionalidade nova ao domínio")**: XP não existe em lugar nenhum do backend — `EstadoMundoResponse.java` não tem campo `xp`, `SessaoDeJogo` nunca decodifica o pacote `SetExperience` (0x1F CB, protocolo 1.8); exibido como "—" com `Tooltip` explicando a ausência, sem parsing de pacote novo. Zero DEC nova, zero endpoint/DTO backend tocado, zero Port alterada. **5 abas antigas tornam-se secundárias** após o Painel — continuam existindo e funcionando sem regressão, mas passam a ser usadas só para ações específicas/avançadas fora do escopo do painel: Ações (formulários completos de movimento/câmera/bloco/entidade), Inventário (controles de transação de janela — `confirmarTransacao`/`limparCursor`/`fecharJanela`, deliberadamente fora do Painel pra manter densidade), Mundo (estado completo com `onGround`/`estaSubmerso`/chunk), Macros (ativação/desativação com argumentos customizados, o Painel só lista as macros já ativas). Console perde o `ConfirmDialog`/botão "Limpar logs" do corpo compartilhado (ficam só na página standalone `ConsolePage`, que continua com o próprio cabeçalho — no Painel a limpeza de logs continua alcançável navegando pra aba Console). 117 testes automatizados (116→117: `BotDetailsPage.test.tsx` ganha `describe('Painel Operacional')` cobrindo render simultâneo de status+console+jogador+entidades e execução de comando via console, reaproveitando os mesmos mocks MSW já existentes no arquivo — nenhum endpoint novo mockado). Validação: `tsc -b --noEmit`/`eslint .` sem erros, `vitest run` 117/117, `mvn -o compile` limpo (build de confirmação, já que o épico não tocou Java); validação manual completa contra backend real (`mvn spring-boot:run`, JDK 21, PostgreSQL 18 local) + frontend real via Browser pane — bot criado reaproveitando Conta/Servidor existentes com macro inicial `antiafk`, `/bots/:id` já abre direto no Painel (link da listagem confirmado apontando pra `/painel`), coluna de status mostrando `TarefaAntiAFK` ativa e o log real `§6AntiAFK §aON` simultaneamente, `GET /estado`/`/mundo/*` retornando `409` (bot desconectado) tratado como `EmptyState` sem quebrar a tela, comando `help` executado via console do Painel com saída completa via WS e "Últimos comandos" atualizado em tempo real na coluna de status sem reload, consulta de bloco alvo retornando `409` tratado graciosamente, navegação pra aba Macros confirmando o mesmo estado (`TarefaAntiAFK`) sem duplicar cadastro, bot de teste excluído ao final, console do navegador sem erros em nenhum passo | ✔ |

| 2026-07-28 | MacroMobSimples (porte de `docs-reescrita/docs/10-Macros/10.1-Macro-Mob-Simplificada.md`, solicitado direto pelo responsável para uso de teste em produção no bot Apolo, sem milestone/épico formal — composição pura sobre capacidade já aprovada, nenhuma decisão arquitetural nova): `domain.bot.TarefaMobSimples` (7 estados lineares — `INICIALIZAR`/`BUSCAR_E_ATACAR_MOB`/`VENDER_ITEMS`/`CHECAR_ESPADA`/`IR_PARA_BAU_ESPADAS`/`TROCAR_ESPADA_NO_BAU`/`RETORNAR_FARM` — fiel à especificação simplificada, não ao legado C#/`TarefaMob`; sem sub-estados de venda por tempo/ciclos, sem contagem fixa de hits, sem checagem de poção/Flame) + `interfaces.comando.ComandoMobSimples` (alias `mobsimples`, mesmo padrão toggle de `ComandoMob`/`ComandoMinerar`, registrado em `ConfiguracaoDeComandos`). Reaproveita 100% de primitivas já existentes: `EntidadesDoMundo.mobMaisProximo` (DEC-38), `interagirComEntidade`/`olharParaEntidade`, `Janela`/Container (DEC-37), `AbridorDeBau`. Duas funções antes privadas de `TarefaMob` extraídas para `MacroUtils` (2º consumidor, mesma regra de três já aplicada nesta base) para eliminar duplicação: `localizarBauProximo` (varredura de baú, agora parametrizada por IDs de bloco) e `localizarItemNaJanela` (generalização de "achar espada substituta na janela aberta"); `TarefaMob` refatorada para reusar ambas, comportamento inalterado (testes pré-existentes continuam verdes). Requisito de resiliência da Seção 4 da especificação implementado literalmente: sem espada substituta disponível no baú, a macro avisa no console do operador e se autorremove (`bot.removerTarefa(this)`, mesmo padrão de `TarefaHerbalismo`) em vez de atacar desarmado — única divergência de robustez em relação a `TarefaMob`, que não trata esse caso. Reconexão/morte resetando a FSM para `INICIALIZAR` **não** implementado (mesma lacuna documentada de `TarefaMob`, que também não reseta — omissão preexistente, não nova). Zero DEC nova, zero Packet/Port/agregado novo, zero interface pré-existente quebrada. 1117→1127 testes automatizados (+10: 8 em `TarefaMobSimplesTest` cobrindo teleporte inicial/busca-e-ataque dentro e fora do raio/transição pra venda com inventário cheio/desativação segura sem espada no baú/troca bem-sucedida de espada, 2 em `ComandoMobSimplesTest` ligar/desligar), 0 falhas, 0 erros, 3 skipped deliberadamente (`mvn -o package`, suíte completa 1127 testes). Não testado ao vivo contra o servidor Craftlandia/bot Apolo nesta sessão (fora do escopo desta implementação) | ✔ |

| 2026-07-28 | Correção do Painel Operacional/Mundo (bug reportado pelo responsável ao vivo: "Entidades próximas" mostrava nome cru da classe Java em vez de nick/nome do mob, inventário só mostrava ID numérico do item sem nome/ícone, sem agrupamento de mob por tipo — sem milestone/épico formal, dado de exibição extraído do legado autorizado): 3 registros novos de dado estático extraídos do `.resx` legado (`Projeto Adv 2.4.5/AdvancedBot.Properties.Resources.resx`, mesma origem/padrão de extração já usado por `RegistroDeBlocos`/`blocks.json`) — `domain.protocol.v1_8.RegistroDeItens` (336 itens de `Resources.items`, id→displayName + variações por damage para os poucos itens cujo nome depende de metadata: carvão/carvão vegetal, dye, cabeças, vaso de flores, peixe, maçã dourada encantada), `domain.bot.RegistroDeMobs` (35 tipos de mob de `Resources.entities`, id byte de `SpawnMobPacket`→displayName), e `advancedbot-frontend/src/features/bots/details/services/itemSpriteCoords.ts` (630 recortes de sprite sheet 16x16, de `itemmap_csv`/`texturemap_csv`, mesmo `.resx`) + `public/sprites/item-icons.png` (spritesheet 1024x320 real do legado, extraído do byte[] serializado do `.resx`, assinatura PNG localizada manualmente dentro do blob de serialização binária do .NET). **Backend**: `EntidadeResponse.de` ganha 2º parâmetro `ListaDeJogadores` — `tipo` agora é nick do jogador (via `ListaDeJogadores.porUuid`, já existia, nunca fora usado por este endpoint) ou nome real do mob (`RegistroDeMobs.nome`), nunca mais `getClass().getSimpleName()`; `ItemStackDto` ganha campo aditivo `nome` (`RegistroDeItens.nome(itemId, damage)`), populado em `ItemStackDto.de` e portanto em todo endpoint que já reusa esse DTO (Inventário/Equipamento/Janela) sem tocar nenhum deles. **Frontend**: `ItemSlot` renderiza o ícone real (crop do spritesheet via `background-position`, `image-rendering: pixelated`, 3x scale) atrás do contador, mais o nome real no tooltip (`formatItemLabel` trocado de `#id:damage` pro `nome` do backend com fallback pro id cru se desconhecido); nova função pura `agruparEntidades` (`features/bots/details/services/entidades.ts`) agrupa por `tipo` (funciona sem campo de categoria explícito — nicks de jogador nunca colidem, só mobs da mesma espécie repetem, mesmo efeito da Craftlandia "Zombie ×5"), reaproveitada por `MundoPage`/`PainelOperacionalPage` (2º consumidor, elimina a duplicata de `ENTIDADE_COLUMNS` que já existia entre as duas). Tipos gerados (`orval`) regenerados contra o backend real rodando localmente (Postgres 18 já ativo) — só `ItemStackDto`/demais modelos ganham o campo aditivo, nenhum contrato quebrado. Zero DEC nova (dado de exibição/protocolo público 1.8, não regra de negócio), zero Port alterada, zero endpoint REST novo. 1127→1131 testes automatizados no backend (+4: `RegistroDeItensTest`/`RegistroDeMobsTest`/`EntidadeResponseTest`/`ItemStackDtoTest`) e 120→129 no frontend (+9: `entidades.test.ts`/`itemStack.test.ts`), 0 falhas, `mvn -o package`/`tsc -b --noEmit`/`eslint .`/`vitest run` limpos. Validação manual parcial: `mvn spring-boot:run` real + frontend real via Browser pane confirmaram o Painel do bot `Solk`/Apolo (`apolo.clmc.com.br:3636`, desconectado) sem erro de console e com `ErrorState`/`EmptyState` corretos nas 3 seções (bot sem sessão ativa, `409` esperado); **não validado contra dado real de mob/inventário em jogo** — os 2 bots persistidos (Solk/Apolo, Apolo é o mesmo citado pelo responsável para o teste de farm multi-dias da `MacroMobSimples`) estavam desconectados nesta sessão, conectar um bot real a um servidor ao vivo só para validar uma correção de UI não foi feito sem autorização explícita | ✔ |

| 2026-07-28 | Tela de configuração de macro (pedido direto do responsável: mudar as homes da `MacroMobSimples` sem recompilar e conferir a configuração atual de um bot, com decisão explícita de perfil compartilhado + framework genérico para qualquer macro futura, não só MobSimples — sem milestone/épico formal, novo agregado com identidade própria seguindo o mesmo idioma já usado por Conta/PerfilDeServidor/EPIC-APP1): novo agregado `domain.bot.ConfiguracaoDeMacro` (id/nome/tipoDeMacro/parametros — mapa livre chave→valor, não campos tipados, porque o mesmo agregado serve qualquer `TarefaContinua` que implemente a nova interface `domain.bot.TarefaComConfiguracao`) + `application.port.RepositorioDeConfiguracoesDeMacro` + adapter JPA (`ConfiguracaoDeMacroJpaEntity`/`ConfiguracaoDeMacroParametroJpaEntity`, linha filha por par chave/valor, mesmo padrão flat de `bot_macros`, `@Transactional` explícito no delete derivado — mesma classe de bug já corrigida na Milestone 43) + migration `V3__configuracoes_de_macro.sql` + 2 Casos de Uso finos (criar/atualizar) + `ConfiguracaoDeMacroController` (CRUD completo, `/api/v1/configuracoes-macro`, com `PUT` de verdade, diferente de Conta/Servidor que não têm). `TarefaMobSimples` ganha `Configuracao` (record aninhado: `homeMob`/`homeBau`/`cmdVenderMobs`, default = comportamento hardcoded de antes) + construtor aceitando `Configuracao`+`UUID` opcional do perfil aplicado; `ComandoMobSimples` ganha `RepositorioDeConfiguracoesDeMacro` como colaborador (mesmo idioma de `argumentos[0]` já usado por `ComandoAntiAFK`/delay, mas resolvendo um perfil persistido por id em vez de um valor cru). `MacroResponse` ganha `configuracaoDeMacroId`/`configuracaoAtual` (populados via `instanceof TarefaComConfiguracao`, nenhuma outra macro afetada); `AtivarMacroRequest` ganha `configuracaoDeMacroId` opcional (prioridade sobre `argumentos` em `MacroController.ativar`). **Bug real encontrado e corrigido durante a validação manual, não coberto por nenhum teste automatizado até então**: `CatalogoController.CLASSES_DE_MACRO` (allowlist hardcoded de `GET /api/v1/macros`) nunca foi atualizada quando `ComandoMobSimples` foi registrado em sessão anterior — a macro funcionava perfeitamente via `GerenciadorDeComandos`/`POST /macros/mobsimples`, mas ficava invisível em qualquer tela que dependesse do catálogo (Select de ativação em `MacrosPage`, Select de `tipoDeMacro` na tela nova); corrigido adicionando `"ComandoMobSimples"` ao `Set`, `CatalogoControllerTest` novo (regressão). **Frontend**: nova feature `features/configuracoes-macro/` (`ConfiguracoesMacroPage` — CRUD com editor genérico de linhas chave/valor para "parametros", sem formulário tipado por macro; rota `/configuracoes-macro` + item na Sidebar), `MacrosPage` ganha Select opcional de perfil (filtrado por `tipoDeMacro`= alias escolhido, só aparece depois de escolher a macro — bug de UX encontrado e corrigido na própria validação manual: mostrava perfis de qualquer tipo antes de qualquer macro ser selecionada) e a tabela de macros ativas ganha colunas "Perfil aplicado" (resolve id→nome) e "Configuração atual" (valores já resolvidos, com fallback pro padrão). **Bug de correção de estado (React) encontrado e corrigido durante os testes automatizados, não na validação manual**: primeira versão do `ConfiguracaoDeMacroFormModal` inicializava `tipoDeMacro` num `useState` no mount a partir do catálogo (query assíncrona separada, sem ordem garantida) e sincronizava depois via `useEffect` chamando `setState` direto no corpo do efeito — ESLint (`react-hooks/set-state-in-effect`) rejeitou como anti-padrão de cascata de renders; corrigido derivando o valor efetivo a cada render (`tipoDeMacroEscolhido ?? tiposDeMacroDisponiveis[0] ?? ''`), sem efeito nenhum. Tipos gerados (`orval`) regenerados contra o backend real — como o novo controller é alfabeticamente anterior a `BotController`/`LogController`/`MacroController` na spec OpenAPI, os sufixos numéricos de colisão de nome do Spring (`useListar3`→`useListar4`, `useListar4`→`useListar5`, `useListar5`→`useListar6`, `useCriar2`→`useCriar3`, `useDetalhar2`→`useDetalhar3`, `useRemover3`→`useRemover4`) mudaram em cascata para TODOS os hooks pré-existentes que os referenciavam (`useBotDetail`/`useBotMacros`/`useRealtimeLogs`/`useBotActions`/`useBotList`/`useBotMutations`) — efeito colateral mecânico já esperado ao acrescentar um controller cedo no alfabeto, sem nenhuma mudança de comportamento real, só renomeação. Zero DEC nova (dado de configuração de exibição/operação, não regra de negócio nova), zero Port pré-existente alterada, zero contrato REST pré-existente quebrado (só campos aditivos em `MacroResponse`/`AtivarMacroRequest`). 1154 testes automatizados no backend (1131→1154, +23: `ConfiguracaoDeMacroControllerTest`/`RepositorioDeConfiguracoesDeMacroJpaTest`/`CatalogoControllerTest` novos, mais testes novos em `TarefaMobSimplesTest`/`ComandoMobSimplesTest`/`MacroResponseTest`) e 133 no frontend (129→133, +4: `ConfiguracoesMacroPage.test.tsx`), 0 falhas, `mvn -o package`/`tsc -b --noEmit`/`eslint .`/`vitest run` limpos. Validado manualmente de ponta a ponta contra o backend real (`mvn spring-boot:run`, Postgres 18 local) + frontend real via Browser pane: perfil "Craftlandia - Apolo" (`homeMob=/home apolomob`, `homeBau=/home apolobau`) criado na tela nova, aplicado na ativação da `MobSimples` no bot Apolo (`apolo.clmc.com.br:3636`, desconectado nesta sessão) via o Select de perfil, tabela de macros ativas confirmando "Perfil aplicado: Craftlandia - Apolo" e "Configuração atual: homeBau=/home apolobau, homeMob=/home apolomob, cmdVenderMobs=/vender mobs" (mistura correta de valor customizado + default), macro desativada e perfil de teste removido ao final (`DELETE` real, `204`), console do navegador sem erros novos em nenhum passo | ✔ |

| 2026-07-29 | EPIC-VIEWER-01 (evolução do Painel Operacional, escopo restrito pelo responsável a um único bot — Swarm Bar/visão multi-bot deliberadamente fora desta sessão): auditoria prévia obrigatória (contra código atual, não contra a proposta original em `docs-reescrita/docs/12-Interface/08-Proposta-Viewer-Operacional.md`) confirmou que o Painel Operacional (Milestone 45) já reunia todo o estado pedido (status/conta/servidor/proxy/autoReconnect/macros ativas/últimos comandos/console/vida/fome/coordenadas/dimensão/inventário/equipamento/janela/entidades/jogadores/bloco) e que a correção de exibição de 2026-07-28 já resolvia nome real de mob/jogador e ícone/nome real de item — nenhuma melhoria proposta para este épico já estava implementada, e nenhuma se tornou obsoleta; as lacunas reais eram só: `msDesdeUltimoKeepAlive` (`BotResponse`/`EstadoMundoResponse`) nunca lido em nenhum componente, nenhum indicador de "travamento aparente", direção (yaw/pitch) nunca exibida (só coordenadas x/y/z). **100% frontend, zero mudança de backend, zero endpoint/evento WS novo, zero DEC.** Entregue: `features/bots/details/services/painelSaude.ts` novo (puro, sem I/O) com `classificarSaudeConexao`/`classificarAtividade`/`classificarTravamento`/`formatMsAtras` — thresholds heurísticos documentados em comentário, não vêm do backend; `features/bots/details/hooks/useUltimaAtividade.ts` novo, captura o campo `timestamp` do envelope WS `EventoDeBot` (já emitido pelo backend, já trafegando por `wsBus`, nunca antes capturado por `useBotConsole`/`useRealtimeLogs`) no mesmo canal `bot:{id}` já aberto por `useBotEventsSocket` — nenhuma assinatura WS nova; tick de 1s interno para manter "tempo desde a última atividade" andando sem depender de evento novo (`Date.now()` lido só dentro de efeitos, por causa da regra de pureza do React — duas iterações até o ESLint parar de reclamar: 1ª tentativa chamava `Date.now()` direto no corpo do hook, 2ª chamava `setState` síncrono no corpo do efeito, versão final usa `useState(() => Date.now())` como valor inicial + `setInterval` dentro do efeito). `PainelOperacionalPage.tsx`: nova linha de 3 `MetricCard` (componente já existente do Dashboard, reaproveitado sem alteração) — Saúde de conexão, Atividade recente, Travamento aparente — e campo "Direção (yaw/pitch)" somado ao card Jogador já existente; nenhum componente novo de UI, nenhuma página nova, nenhuma sub-aba nova. GAPs documentados, não implementados: `msDesdeUltimoKeepAlive` continua sendo snapshot de rede, não RTT real (mesma limitação já coberta por DEC-41); "Travamento aparente" é heurística client-side (macro ativa + sessão conectada + nenhum evento WS há mais de 30s), marcada como estimativa explícita na UI (`hint`), sem confirmação de backend; Swarm Bar/visão multi-bot permanece fora de escopo. 17 testes automatizados novos no frontend (`painelSaude.test.ts`, funções puras) mais extensão do `describe('Painel Operacional')` em `BotDetailsPage.test.tsx` cobrindo os 3 indicadores e o novo campo de direção — 151/151 testes passando (30 arquivos). Validação: `tsc -b --noEmit`/`eslint .` sem erros (1 aviso pré-existente em `BotProxyModal.tsx`, não relacionado), `vitest run` 151/151, `vite build` sucesso; `mvn -o compile` do backend limpo (build de confirmação, nenhum arquivo Java tocado). Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21, PostgreSQL 18 local) + frontend real (Vite dev server) via Browser pane: Painel Operacional do bot `Solk` (`apolo.clmc.com.br:3636`, já persistido de sessão anterior, desconectado nesta sessão) mostrando corretamente os 3 novos indicadores no estado desconectado ("Desconectado"/"Sem eventos ainda"/"N/A — Sem macro ativa ou bot desconectado") e o card Jogador em `EmptyState` (coerente, sem sessão ativa — campo Direção não aparece), sem erros de console. **Validação com bot conectado a servidor Minecraft real não foi executada nesta sessão** — mesmo critério já aplicado na correção de 2026-07-28 (conectar a conta `Solk` a um servidor de terceiros ao vivo só para validar UI não foi feito sem confirmação explícita adicional do responsável neste turno); estados "Saudável"/"Ativo agora"/"Possível travamento" ficam cobertos pelos 18 testes unitários de `painelSaude.test.ts`, mas não pela observação ao vivo de um bot em jogo | ✔ |

| 2026-07-29 | EPIC-VIEWER-02 (Estágio 1, Timeline de Eventos do Painel Operacional, incorporada ao Painel existente, sem página nova): auditoria prévia confirmou as fontes de evento já disponíveis no frontend — canal WS por bot (`bot:{id}`, aberto por `useBotEventsSocket`) com só 3 tipos reais emitidos pelo backend (`"log"` de `SaidaDoOperador`, `"estado"` de mudança de `EstadoExecucao` — não é posição/mundo, confirmado em `GerenciadorDeBots.registrar` — e `"comando"` de `ComandoExecutadoPayload`, também usado por ativação/desativação de macro via `MacroAtivacaoSupport.ativar`, sem tipo próprio, GAP herdado do doc 08 §1.1); `useBotConsole`/`useRealtimeLogs` já consomem `"comando"`/`"log"` mas descartam o `timestamp` do envelope; `wsBus` já emite um canal `status` (transporte do `ManagedSocket`: `connecting`/`open`/`closed`/`reconnecting`) desde o EPIC-APP1, nunca antes consumido por nenhum hook de domínio — única fonte real de conexão/desconexão/reconexão disponível hoje (reflete o transporte WS do navegador, não o `EstadoDeSessao` do bot no Minecraft, mesma distinção já registrada para `msDesdeUltimoKeepAlive`); confirmado que `Alert`/`ScrollArea` não existem no Design System (só `Toast`/`EmptyState`/`ErrorState`/`ErrorBoundary` em `feedback/`), substituídos por composição de tokens de cor já usados e pelo mesmo padrão `overflow-y-auto` do `ConsoleLogViewer`, sem construir componentes novos fora do escopo (mesmo critério do `ProgressBar` recusado no OPX-05); nenhuma Timeline/histórico cronológico existia antes ("últimos comandos" do Painel é só os 5 últimos itens de `botConsole.history`, sem cruzamento com log/estado/conexão). **100% frontend, zero mudança de backend, zero endpoint/evento WS novo, zero DEC, zero migração de polling para WebSocket.** Entregue: `features/bots/details/services/timelineEventos.ts` novo (puro) — tipos (`TimelineEntrada`, tipo `log`/`comando`/`estado`/`conexao`), classificação de severidade por texto (`classificarSeveridadeDeTexto`, heurística sobre `falha`/`erro`/`exception`/`§4`/`§c` etc., mesmo espírito de `painelSaude.ts`) e por status de conexão (`classificarSeveridadeDeConexao`), buffer FIFO limitado (`adicionarEntrada`, `TIMELINE_MAX_ENTRADAS = 200` — evita crescimento infinito de memória); `features/bots/details/hooks/useBotTimeline.ts` novo, assina `wsBus` `message` (`bot:{id}`, tipos `log`/`estado`/`comando`) e `wsBus` `status` (mesmo canal) sem abrir nenhuma conexão WS nova (reaproveita o socket já aberto por `useBotEventsSocket`), reseta o buffer ao trocar de bot via ajuste de estado durante o render (padrão React recomendado, não num efeito — ESLint `react-hooks/set-state-in-effect` rejeitou a primeira versão que chamava `setEntradas([])` dentro do efeito); `features/bots/details/components/EventTimeline.tsx` novo (apresentacional, recebe `timeline` injetado pelo chamador, mesmo padrão de `ConsolePanel`) — filtro por tipo (`Select` existente), botão "Limpar" (`Button` ghost existente), aviso fixo de "histórico desta sessão" (tokens de cor existentes, não um `Alert` novo), lista com `role="log"` (semântica ARIA de feed de mensagens) reaproveitando o padrão de scroll do `ConsoleLogViewer`, `EmptyState`/`Badge` existentes para vazio/severidade. `PainelOperacionalPage.tsx`: `EventTimeline` inserido em novo `Card` "Timeline de eventos" logo abaixo do Console, mesma coluna central — nenhuma página nova, nenhuma sub-aba nova. GAPs documentados, não implementados: macro ativa/desativa continua sem tipo WS próprio (aparece como `"Comando"` genérico); `conexao` reflete transporte WS do navegador, não sessão real do bot; severidade é heurística de texto, não categoria do backend; sem persistência (perdido a cada reload por design do Estágio 1, aviso fixo avisa isso na própria UI); `Alert`/`ScrollArea` continuam inexistentes como átomos, substituídos por composição. 12 testes automatizados novos no frontend (`timelineEventos.test.ts`, funções puras) mais um teste novo em `BotDetailsPage.test.tsx` cobrindo ordem cronológica, filtro por tipo e "Limpar" (usando `role="log"` pra escopar a busca e não colidir com o texto duplicado que também aparece no Console) — 164/164 testes passando (31 arquivos). Validação: `tsc -b --noEmit`/`eslint .` sem erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vitest run` 164/164, `vite build` sucesso; `mvn -o compile` do backend limpo (nenhum arquivo Java tocado). Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21, PostgreSQL 18 local) + frontend real via Browser pane: comando `help` executado pelo Console do bot `Solk` (desconectado) — Timeline mostrou, em ordem, a entrada "Comando" (`help → SUCESSO`) seguida da entrada "Console" com a saída completa, ambas com horário real; indicador "Atividade recente" (EPIC-VIEWER-01) atualizou para "Ativo agora" no mesmo instante, confirmando dois indicadores diferentes reaproveitando o mesmo evento WS; filtro "Estado" escondeu corretamente as entradas existentes (`EmptyState` "Nenhum evento desse tipo nesta sessão"); "Limpar" esvaziou o buffer (`EmptyState` "Nenhum evento chegou ainda nesta sessão" de volta); sem erros de console em nenhum passo. **Validação durante execução de macro contra servidor Minecraft real não foi executada** — nenhum bot estava conectado a um servidor real nesta sessão; conectar a conta `Solk` a um servidor de terceiros ao vivo só para esta validação não foi feito sem confirmação explícita adicional do responsável neste turno (mesmo critério de sessões anteriores) — o cruzamento cronológico comando→log já foi confirmado com o bot desconectado, e a emissão do evento `"comando"` por ativação de macro real fica coberta pela leitura de código (`MacroAtivacaoSupport.ativar`), não pela observação ao vivo | ✔ |

| 2026-07-29 | EPIC-VIEWER-04 (Etapa 1, infraestrutura mínima do Viewer 2D dentro do Painel Operacional — nova aba "Debug 2D", plano top-down, marcador de posição e seta de direção; roadmap completo do épico apresentado numa sessão anterior de planejamento, esta é só a primeira etapa isolada dele): auditoria prévia confirmou que posição/direção já existiam e já eram exibidas em texto (`EstadoMundoResponse.x/y/z/yaw/pitch` via `GET /estado`, `hooks/useEstadoMundo.ts`, polling 3s, sem WS dedicado — GAP §8 do doc 06, herdado); `pages/MundoPage.tsx` confirmado como referência de padrão de loading/erro/vazio (`Skeleton`/`ErrorState`/`EmptyState`), não duplicado; nenhum componente de canvas/SVG de mundo existia em `shared/components` (busca direta antes de implementar) — único precedente de SVG customizado no projeto é `features/dashboard/components/MetricTrendChart.tsx` (sparkline sem lib externa), confirmando SVG sem biblioteca nova como abordagem já aceita (arquitetura congelada); abas de `BotDetailsPage` confirmadas como rotas aninhadas em `services/botDetailsNav.ts` (`BOT_DETAILS_TABS`) + `app/router/router.tsx` (lazy import por aba), único ponto de integração necessário, sem duplicar `useBotEventsSocket` (continua aberto uma única vez no layout pai). **100% frontend, zero mudança de backend, zero endpoint/evento WS novo, zero polling adicional, zero DEC.** Entregue: `features/bots/details/components/BotDebugViewer2D.tsx` novo (apresentacional, puro) — SVG fixo (`viewBox` 300x300), plano top-down centrado no bot (sem grid de blocos), marcador de posição (círculo central), seta de direção calculada do `yaw` (convenção Minecraft, vetor de planta `(dx,dz) = (-sin(yaw), cos(yaw))`, documentado em comentário); `pitch` sem representação visual em top-down, exibido só como texto junto de x/y/z; prop `overlays?: ReactNode` (slot vazio nesta etapa, não usado) preparando o mesmo `<svg>`/sistema de coordenadas para camadas futuras (blocos/entidades/path) sem recriar a projeção. `features/bots/details/pages/DebugViewerPage.tsx` novo — reaproveita `useEstadoMundo` sem alteração, mesmos estados de loading/erro/vazio do Design System já existentes. `services/botDetailsNav.ts`: nova entrada `{ key: 'debug-2d', label: 'Debug 2D' }`. `app/router/router.tsx`: rota filha `debug-2d`, lazy-loaded, mesmo padrão das demais abas. Nenhum arquivo de backend tocado — nenhum DTO/endpoint/evento WS/contrato REST ou WS alterado. Escopo desta etapa deliberadamente não inclui: blocos, entidades, raycast, mineração, combate, follow, overlays de alcance/raio, timeline, viewer 3D — todos ficam para etapas seguintes do mesmo épico. Teste novo `describe('Debug 2D')` em `BotDetailsPage.test.tsx` cobrindo o SVG (`role="img"`, nome acessível) e os textos de posição/yaw/pitch a partir do mesmo mock de `GET /estado` já usado pelas demais abas — 165/165 testes passando (31 arquivos, +1 novo). Validação: `tsc -b --noEmit`/`eslint .` sem erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vitest run` 165/165, `vite build` sucesso (novo chunk lazy `DebugViewerPage-*.js`); `mvn -o compile` do backend limpo (nenhum arquivo Java tocado). Validação manual via Browser pane (Vite dev server local, sem backend Java rodando nesta sessão): navegação direta a `/bots/{id}/debug-2d` confirma a aba na navegação, página monta sem exceção JS (console sem erros), card exibe estado de carregamento/vazio esperado na ausência de backend. **Validação contra servidor Minecraft real (conectar bot, mover personagem, girar câmera, confirmar posição/direção acompanhando ao vivo) não foi executada nesta sessão** — nenhum backend Java nem bot conectado a um servidor real estava disponível no ambiente desta sessão (mesmo critério de transparência das entradas anteriores); fica como validação pendente a ser executada pelo responsável com o backend rodando e um bot conectado a um servidor real | ✔ |

| 2026-07-29 | EPIC-VIEWER-04 (Etapa 2, contexto visual ao redor do bot no Debug 2D — grade de blocos pequena e fixa + entidades próximas, somado à posição/direção da Etapa 1, arquitetura da Etapa 1 não refeita, só estendida): auditoria prévia confirmou `BotDebugViewer2D.tsx` (SVG puro, marcador+seta de yaw) já preparado com prop `overlays?: ReactNode` (slot vazio) pra camadas futuras; `hooks/useMundoEntidades.ts` já existente (usado por `MundoPage`, `GET /mundo/entidades`, polling 5s) confirmado como fonte pronta de entidades próximas; leitura direta de `MundoController.java` confirmou que `GET /mundo/entidades` (sem filtro `tipo`) já devolve mobs E jogadores remotos com `x/y/z` reais via `EntidadeResponse.de` — **sem GAP** para entidades; mesma leitura confirmou que `GET /mundo/jogadores` devolve só `JogadorResponse{nome,nomeDeExibicao}`, **sem posição** — não usado pra plotar nada, GAP documentado pra não ser reintroduzido por engano em telas futuras; confirmado (de novo) que `GET /mundo/bloco` continua só pontual, sem endpoint em lote/grade — GAP já conhecido da proposta original (doc 08 §5), grade implementada com N chamadas paralelas ao endpoint pontual, raio pequeno (2) por causa desse custo, decisão já prevista no roadmap da etapa. **100% frontend, zero mudança de backend, zero endpoint/evento WS novo, zero DEC.** Entregue: `features/bots/details/services/debugViewerGrade.ts` novo (puro) — `gerarCoordenadasDeGrade` enumera as 25 células da grade (`GRID_RAIO = 2`) ao redor da posição truncada do bot, com offset `dx/dz` em blocos; `features/bots/details/hooks/useBlocosGrade.ts` novo — busca cada célula em paralelo (`Promise.all` sobre o fetcher `bloco` já gerado pelo orval, sem endpoint novo), agregada numa única `useQuery` com chave nas coordenadas truncadas do centro (reaproveita cache enquanto o bot não atravessa pra outro bloco), `refetchInterval` 5s (mesma cadência de `useMundoEntidades`); `BotDebugViewer2D.tsx` ganhou prop aditiva `background?: ReactNode` (renderizada antes da seta/marcador, blocos ficam visualmente atrás) e passou a exportar `GRID_CELL_PX`/`VIEWER_CENTER_PX` pras camadas novas reaproveitarem a mesma projeção sem duplicar conta — `overlays` da Etapa 1 preservado, agora usado pelas entidades (primeiro plano); `DebugViewerGradeLayer.tsx`/`DebugViewerEntidadesLayer.tsx` novos (apresentacionais) — um `<rect>` por célula (sólido vs. vazio por cor, `<title>` com coordenada/blockId) e um `<circle>` por entidade (cor distinta do bot, `<title>` com tipo/id); `DebugViewerPage.tsx` passou a chamar `useBlocosGrade`/`useMundoEntidades` e compor as camadas, mais legenda mínima (4 itens, sem componente novo de Design System) e aviso de texto se a grade falhar, sem travar a exibição de posição/direção. Nenhum arquivo de backend tocado. 5 testes automatizados novos em `debugViewerGrade.test.ts` (funções puras) mais um teste novo em `BotDetailsPage.test.tsx` (`describe('Debug 2D')`) cobrindo grade completa (≥25 `<rect>`) e marcador de entidade (`<circle fill="var(--color-warning)">`) e legenda — 171/171 testes passando (32 arquivos). Validação: `tsc -b --noEmit`/`eslint .` sem erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vitest run` 171/171, `vite build` sucesso; `mvn -o compile` do backend limpo (nenhum arquivo Java tocado). Validação manual via Browser pane (Vite dev server local, sem backend Java rodando nesta sessão): navegação direta a `/bots/{id}/debug-2d` confirma a aba, página monta sem exceção JS, comportamento correto de estado vazio/erro na ausência de backend. **Validação contra servidor Minecraft real (mover o bot, mover entidades próximas, confirmar grade e marcadores acompanhando) não foi executada nesta sessão** — mesmo critério já registrado na Etapa 1 (nenhum backend Java nem bot conectado a servidor real disponível no ambiente); fica acumulada como validação pendente junto da Etapa 1 — critério de aceite conjunto: mover o bot desloca a grade junto (célula do bot sempre no centro) e mover uma entidade real move o marcador correspondente dentro do polling de 5s | ✔ |

| 2026-07-29 | EPIC-VIEWER-04 (Etapa 3, raycast real do domínio no Debug 2D — endpoint `GET /mundo/raycast` reaproveitando `SessaoDeJogo.tracarRaioParaBlocos`, linha do raio + contorno do bloco atingido + aresta da face; auditoria técnica feita numa sessão anterior sem código, esta sessão implementou sobre as conclusões já obtidas sem repetir a análise): zero algoritmo de raycast novo, zero alteração em `Mundo`/`SessaoDeJogo`/`TarefaMineracao`/macros, zero cálculo geométrico de raio no frontend, zero DEC. **Backend**: `interfaces/rest/dto/RaycastResponse.java` novo (record `atingiu,x,y,z,face,blockId,metadata,solido,distancia,alcance`, campos de alvo boxed/nuláveis, `alcance` sempre presente) com fábricas `semAlvo(alcance)` e `de(ResultadoDoRaio,Bloco,SessaoDeJogo,alcance)` (`distancia` = pés do bot → canto do bloco, só informativa, documentado que não é a distância usada pelo raycast em si); `MundoController.java` ganhou `GET /mundo/raycast` com `ALCANCE_RAYCAST=6.0` (mesmo valor já usado por `TarefaMineracao`/`TarefaMob`/`TarefaHerbalismo`/`TarefaAutoFish`, confirmado na auditoria — Viewer mostra exatamente o que as macros enxergam) — corpo do método só chama `sessaoDeJogo.tracarRaioParaBlocos(ALCANCE_RAYCAST)` e `mundo.blocoEm(...)` (mesma chamada de `GET /mundo/bloco`), nenhuma linha de domínio tocada; 409 sem sessão PLAY automático via `GlobalExceptionHandler` já existente, nenhum tratamento novo. **Frontend**: tipos regenerados via `npm run generate:api` contra backend real rodando localmente (`RaycastResponse`/`useRaycast` novos); `hooks/useDebugViewerRaycast.ts` novo (só `useRaycast` gerado + polling 5s, mesma cadência de `useMundoEntidades`/`useBlocosGrade`); `components/DebugViewerRaycastLayer.tsx` novo (apresentacional) — linha tracejada do centro até o bloco atingido, contorno do bloco (mesmo `GRID_CELL_PX` da Etapa 2, cor `--color-danger`), aresta da face só pras 4 faces horizontais (convenção `PlayerDiggingPacket`/`PlayerBlockPlacementPacket`, topo/base sem aresta representável em top-down, "quando possível"), retorna `null` quando `atingiu=false` — toda a matemática é projeção mundo→tela igual às camadas da Etapa 2, nenhum cálculo de raio; `BotDebugViewer2D.tsx` teve só `VIEW_SIZE` 300→520 (`CENTER` junto) porque o alcance real (6 blocos × 40px = 240px) não cabia no canvas dimensionado pra grade raio-2 da Etapa 2 — único ajuste, `GRID_CELL_PX`/projeção/`background`/`overlays`/camadas existentes intactos; `DebugViewerPage.tsx` compôs a camada nova dentro do slot `overlays` já existente (fragment junto de `DebugViewerEntidadesLayer`), legenda ganhou 1 item novo. Nenhum GAP de backend novo (auditoria prévia já havia confirmado que todo dado necessário já existia). `RaycastResponseTest` (2 casos) + `MundoControllerTest` novo (3 casos, sem contexto Spring, sessão de jogo real construída como em `SessaoDeJogoTest`, só `GerenciadorDeBots` mockado — acerto, sem alvo, `IllegalStateException` sem sessão) — `mvn -o test`: **1168 testes, 0 falhas, 3 skipped pré-existentes**, nenhuma regressão em `TarefaMineracaoTest`/`SessaoDeJogoTest`/`MundoTest`. `DebugViewerRaycastLayer.test.tsx` novo (3 casos) + 2 casos novos em `BotDetailsPage.test.tsx` (`describe('Debug 2D')`, com/sem alvo via mock MSW) — `vitest run`: **176/176 testes passando** (33 arquivos). `tsc -b --noEmit`/`eslint .` sem erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vite build` sucesso, `mvn -o compile` limpo. Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21 — `JAVA_HOME` precisou ser setado manualmente nesta sessão, ambiente sem JDK no PATH por padrão —, PostgreSQL 18 local, bot `Solk` persistido restaurado desconectado): confirmado via `curl` direto que `GET /mundo/raycast` retorna `409` real com o bot desconectado; confirmado via Browser pane que a aba Debug 2D contra o backend real (não mock) monta sem erro de console, a query de raycast dispara e falha com 409 visível em `read_network_requests`, e a UI degrada corretamente pro `EmptyState` sem travar. **Validação com bot em sessão PLAY real (olhando pra parede/céu, girando em quinas, comparando bloco destacado com bloco quebrado pela mineração) não foi executada** — nenhum bot conectado a servidor Minecraft real nesta sessão (mesmo critério de todas as sessões anteriores); fica acumulada com as pendências de validação ao vivo das Etapas 1 e 2 | ✔ |

| 2026-07-29 | EPIC-VIEWER-04 (Etapa 4, caminho real do domínio no Debug 2D — endpoint `GET /mundo/caminho-atual` reaproveitando `SessaoDeJogo.caminhoAtual()`/`GuiaDeCaminho`, polyline do trajeto calculado pelo pathfinding; auditoria rápida pedida explicitamente no início desta sessão, sem reconsultar o projeto C#): zero algoritmo de pathfinding novo, zero alteração em `BuscadorDeCaminho`/decisões de busca/macros, zero recálculo de caminho no frontend, zero DEC. Auditoria confirmou: caminho ativo mora em `SessaoDeJogo.caminhoAtual` (campo `volatile GuiaDeCaminho`, getter/setter/`limparCaminho()` já públicos, fiel a `MinecraftClient.CurrentPath` do legado); produzido por `BuscadorDeCaminho` (A*) via `Mundo.criarCaminhoPara`/`GuiaDeCaminho.criar`, consumido por `ComandoGoto`/`TarefaFollow`/`ComandoPortal`; atualizado a cada tick por `MotorDeTick` (`GuiaDeCaminho.tick()`, remove via `limparCaminho()` quando `finalizado()`); estrutura é `List<PontoDeCaminho>` (`x,y,z` inteiros, getters públicos) sem getter de leitura da lista completa (só `tick()`/`indiceMaisProximo()` liam internamente); nenhum evento WS de caminho existe (`NotificadorDeEventos` sem tipo dedicado, GAP aceito, fora de escopo desta etapa). **Backend**: `GuiaDeCaminho.java` ganhou só `pontos()` (retorna `List.copyOf(pontos)`, cópia defensiva porque a lista original é mutada por `tick()`) — única alteração no domínio, leitura pura, nenhuma decisão de busca tocada; `interfaces/rest/dto/PontoCaminhoResponse.java` e `CaminhoAtualResponse.java` novos (records, fábrica `de()`, mesmo padrão de `RaycastResponse`/`PosicaoResponse`) — `caminho == null` vira `{caminhoDisponivel: false, pontos: []}`, mesmo critério de nulidade de `RaycastResponse.semAlvo`; `MundoController.java` ganhou `GET /mundo/caminho-atual` — corpo só chama `sessao(id).caminhoAtual()`, nenhuma linha de domínio tocada; 409 sem sessão PLAY automático via `GlobalExceptionHandler` já existente (`IllegalStateException` → `HttpStatus.CONFLICT`), nenhum tratamento novo. **Frontend**: tipos regenerados via `npm run generate:api` contra backend real rodando localmente (`CaminhoAtualResponse`/`PontoCaminhoResponse`/`useCaminhoAtual` novos; campos `x/y/z` de `PontoCaminhoResponse` saíram opcionais no OpenAPI gerado, tratados com guarda `!== undefined` no componente, mesmo padrão de `DebugViewerEntidadesLayer`); `hooks/useDebugViewerCaminho.ts` novo (só `useCaminhoAtual` gerado + polling 5s, mesma cadência de `useDebugViewerRaycast`/`useMundoEntidades`); `components/DebugViewerCaminhoLayer.tsx` novo (apresentacional) — `<polyline>` ligando os pontos já projetados pro sistema de tela existente (`GRID_CELL_PX`/`VIEWER_CENTER_PX`), cor `--color-success` (nova na paleta de camadas, distinta de `--color-danger` do raycast e `--color-warning` das entidades), retorna `null` quando `caminhoDisponivel=false` ou `pontos` vazio — nenhum cálculo de rota, só projeção; `BotDebugViewer2D.tsx`/`DebugViewerGradeLayer.tsx`/`DebugViewerEntidadesLayer.tsx`/`DebugViewerRaycastLayer.tsx` não tocados (camadas existentes intactas, confirmado por regressão dos testes já existentes); `DebugViewerPage.tsx` compôs a camada nova dentro do slot `overlays` já existente, junto de raycast/entidades, e a legenda ganhou 1 item novo ("Caminho"). Nenhum GAP de backend novo. `MundoControllerTest` ganhou 3 casos novos (com caminho ativo — terreno plano reaproveitando o cenário de `MundoTest.criarCaminhoParaDeveEncontrarCaminhoRetoEmFaixaPlana`, sessão de jogo real, só `GerenciadorDeBots` mockado —, sem caminho ativo, `IllegalStateException` sem sessão) — `mvn -o test`: **1171 testes, 0 falhas, 3 skipped pré-existentes**, nenhuma regressão em `TarefaFollowTest`/`ComandoGotoTest`/`ComandoPortalTest`/`SessaoDeJogoTest`/`MundoTest`. `DebugViewerCaminhoLayer.test.tsx` novo (3 casos: sem caminho disponível, caminho disponível mas pontos vazios, polyline com pontos projetados corretamente) + 3 casos novos em `BotDetailsPage.test.tsx` (`describe('Debug 2D')`: polyline desenhada com pontos, nenhuma polyline sem caminho disponível, mock default de `/mundo/caminho-atual` somado ao `mockBotDetailsBackend`) — `vitest run`: **181/181 testes passando** (35 arquivos). `tsc -b --noEmit`/`eslint .` sem erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vite build` sucesso, `mvn -o compile` limpo. Validação manual contra backend real: servidor Java de dev encontrado já rodando na porta 8080 (processo anterior, sem o endpoint novo) precisou ser reiniciado (`mvnw spring-boot:run`, JDK 21, `JAVA_HOME` setado manualmente) pra servir o contrato atualizado antes de regenerar o cliente Orval; `curl` confirmou `caminho-atual` presente em `/v3/api-docs` real; nenhum bot persistido restaurado nesta sessão (`Restaurando 0 bot(s) persistido(s)`), sem alvo disponível pra exercitar o endpoint via Browser pane/curl com um bot real desta vez. **Validação contra servidor Minecraft real com bot em sessão PLAY navegando de fato (confirmar a polyline coincidindo com o trajeto percorrido, atualização em mudança de rota, desaparecimento ao chegar ao destino) não foi executada nesta sessão** — nenhum bot conectado a um servidor real disponível no ambiente; fica acumulada com as pendências de validação ao vivo das Etapas 1, 2 e 3 do mesmo épico | ✔ |

## Persistência de Bot (EPIC-PROD-01) — o que passou a persistir e o que continua só em memória

Registrado à parte da tabela acima para consulta rápida (ver Milestone 43 para o relato completo).

**Persistido em PostgreSQL (`bots`/`bot_macros`, sobrevive a restart) a partir desta milestone:**
- Identificador (`id`), nome/identidade (`username`), conta (`contaId` quando usada, senão email/username/password inline)
- Servidor (`servidorId` quando usado, senão host/port inline) e proxy associado (host/port/tipo)
- Política de AutoReconnect (ligado/desligado)
- Estado desejado (`estadoDesejado`: PARADO/EXECUTANDO/PAUSADO)
- Macros ativas por tipo (`bot_macros.tipo`, ex.: `TarefaAntiAFK`) — **sem** os argumentos exatos usados na ativação (ver limitação abaixo)

**Continua existindo só em memória, perdido a cada restart (por natureza, não por lacuna a fechar):**
- `SessaoDeJogo` (posição, vida, inventário, entidades conhecidas, chat) — só existe enquanto há conexão de rede ativa
- Argumentos de ativação de macro (ex.: delay customizado do AntiAFK, alvo do Follow) — `TarefaContinua` não expõe accessor de configuração hoje; a restauração reativa cada macro com os argumentos-padrão do respectivo comando
- Macro `Follow` em si — depende de um `entityId` de jogador só válido dentro da sessão em que foi criada, não é restaurável de forma alguma (nem com persistência de argumentos)
- Buffer de log do console (`SaidaDoOperador`) por bot

---

---

# 13. Critérios para Continuidade

Antes de iniciar qualquer nova tarefa é obrigatório verificar:

- Estado deste documento
- Plano de Migração
- Fundação Arquitetural
- Decisões Arquiteturais
- Baseline Tecnológica

Caso exista conflito, este documento deve ser atualizado antes da implementação.

---

## 14. Fase de Homologação (EPIC-QA-01)

Iniciada em 2026-07-28. A migração funcional é considerada concluída pelo responsável do projeto; esta fase não introduz novas funcionalidades — apenas compara, na prática, o comportamento real da aplicação Java (rodando com backend Spring Boot real, PostgreSQL real e servidor Minecraft real) contra o comportamento esperado, sem consultar o código C#.

Resultado: backlog técnico detalhado em [docs/21-Homologacao/01-Backlog-QA-Homologacao-Completa.md](../21-Homologacao/01-Backlog-QA-Homologacao-Completa.md), com achados classificados como BUG, GAP, Melhoria UX, Melhoria Performance e Melhoria Visual, agrupados em 9 épicos candidatos (EPIC-QA-01.1 a EPIC-QA-01.9).

Achado mais crítico desta rodada: desativar uma macro pela interface reporta sucesso mas não remove de fato a macro em execução no bot (mismatch de identificador entre ativação e desativação) — ver BUG-01 no documento de backlog.

### Sprint QA-01 (2026-07-28) — Correção dos BUGs críticos do backlog (BUG-01, BUG-02, BUG-04)

Escopo fechado: só os 3 BUGs críticos/altos do backlog de homologação. Nenhum GAP/MUX/MPF/MV foi tocado, nenhuma DEC nova aberta, nenhuma decisão arquitetural alterada.

- **BUG-01 (desativar macro não remove de fato)** — causa raiz confirmada no backend: `GET /macros` só devolve o nome de classe da `TarefaContinua` (ex. `TarefaMineracao`), e o frontend reenviava esse mesmo valor no `DELETE /macros/{alias}`; `GerenciadorDeComandos.localizar()` só resolve por chave de catálogo (ex. `miner`), então o DELETE nunca encontrava a tarefa mas respondia 200 do mesmo jeito. Corrigido só no backend (sem mudança no frontend): `MacroController.desativar()` agora resolve o alias de catálogo correto a partir do nome de classe da tarefa ativa antes de delegar a `MacroAtivacaoSupport.ativar()`, preservando o toggle (e efeitos colaterais, ex. `ComandoTwerk` parando de agachar) de cada `Comando`. Para isso, `Comando` ganhou um método default `tarefaContinua()` (retorna a classe de `TarefaContinua` que o comando liga/desliga), implementado nos 8 comandos de macro (`Follow`, `AntiAFK`, `Herbalismo`, `Minerar`, `Mob`, `AutoFish`, `LargarTudo`, `Twerk`). Validado end-to-end via UI real (Solk conectado a `olimpo.clmc.com.br:3737`): ativar Miner, clicar "Desativar", confirmar via `GET /macros` que a lista fica vazia.
- **BUG-02 (botão "Excluir" bot não executa nada)** — não reproduzido no código-fonte atual: `BotTable.tsx`/`BotsPage.tsx`/`ConfirmDialog.tsx` já têm o handler, o diálogo de confirmação e a chamada `DELETE /bots/{id}` corretamente ligados. Validado ao vivo na UI (criação de um bot descartável, clique em "Excluir", confirmação, bot removido da lista). Hipótese mais provável: o bug observado na homologação original refletia um build de frontend desatualizado (mesma classe de problema do GAP-06/GAP-08 de build não reproduzível), não um defeito de código-fonte. Nenhuma alteração de código foi necessária.
- **BUG-04 (troca de proxy não aplicada à conexão ativa; auto-reconnect rotaciona proxy sozinho)** — duas causas raízes distintas, ambas só no backend:
  1. `CasoDeUsoTrocarProxy.trocar()` só persistia a nova configuração, sem sinalizar a conexão ativa. Corrigido: se o bot já tem `SessaoDeJogo` em curso, `trocar()` agora força reconexão imediata via `CasoDeUsoReconectarBot` (mesmo caminho de `POST /reconnect`), sem conectar bots que estavam desconectados.
  2. `GerenciadorDeReconexao` chamava `rotacionarProxy()` incondicionalmente em qualquer falha de reconexão, não só no motivo de kick documentado (`MOTIVO_MUITAS_CONTAS`, fiel ao legado `MinecraftClient.cs:732-743`). Corrigido removendo essa chamada extra do bloco `catch`; a rotação legítima (por "muitas contas") continua intacta.
  - Validado ao vivo contra o bot real Solk: troca de proxy em bot conectado força reconexão de fato (posição do personagem mudou, confirmando reconexão real); troca para proxy SOCKS5 inválida cadastrada retornou erro real de conexão (409) em vez de sucesso silencioso; com auto-reconnect ligado e proxy inválida, 4 checagens consecutivas via `GET /bots/{id}` (~7s, cobrindo várias tentativas) confirmaram que a proxy **não** alterna mais sozinha entre as cadastradas. Estado do bot restaurado ao final (proxy removida, reconectado normalmente).

Build: recompilado e testes automatizados executados via Maven 3.9.9 embutido no plugin Maven do IntelliJ (`.../plugins/maven/lib/maven3`, já que o repositório continua sem Maven Wrapper — GAP-06 permanece em aberto). Dois testes que validavam o comportamento antigo e incorreto de rotação de proxy foram substituídos por testes que validam o comportamento corrigido; um teste novo cobre a reconexão forçada por troca de proxy. Suíte completa (1110+ testes) passou.

Itens correspondentes marcados como resolvidos em [docs/21-Homologacao/01-Backlog-QA-Homologacao-Completa.md](../21-Homologacao/01-Backlog-QA-Homologacao-Completa.md). BUG-03 e os demais GAP/MUX/MPF/MV do backlog seguem em aberto, fora do escopo desta sprint.

### Sprint QA-02 (2026-07-28) — Demais ajustes mapeados no backlog de homologação

Escopo: todos os itens restantes do backlog exceto GAP-03 (Configurações somente leitura) e MPF-01 (polling não consolidado), que o próprio backlog já classificava como dependentes de decisão de produto/arquitetura — adiados por decisão explícita do responsável, não implementados nesta sprint. Nenhuma DEC nova, nenhuma mudança de arquitetura.

- **GAP-06 (build não reprodutível)** — Maven Wrapper adicionado (`mvn wrapper:wrapper -Dmaven=3.9.9`, usando o Maven 3.9.9 embutido no plugin Maven do IntelliJ, já que a máquina não tinha Maven instalado nem acesso planejado a um). `.gitignore` criado (não existia) excluindo `target/`; os ~1500 arquivos de `target/` já versionados foram destrackeados via `git rm -r --cached target` (arquivos continuam em disco).
- **BUG-03 (`GET /commands` retorna 500 genérico)** — `GlobalExceptionHandler` não tinha handler para `HttpRequestMethodNotSupportedException`, caía no catch-all genérico (500). Adicionado handler dedicado retornando 405. Validado: `GET /commands` responde 405 com mensagem específica.
- **MV-01 (Twerk sem descrição)** — `ComandoTwerk.descricao()` preenchida.
- **GAP-04 (métricas JVM/proxy-bot/heap)** — Backend: `MetricasResponse.threadCount` (`ManagementFactory.getThreadMXBean()`), `MemoriaResponse.naoHeapUsadaMb` (`ManagementFactory.getMemoryMXBean()`), `ProxyResponse.botsEmUso` (join em `ProxyController.listar()` contra `GerenciadorDeBots`). Frontend: Dashboard ganhou cards "Memória (Non-Heap)" e "Threads da JVM" (removido `DashboardGapNotice`, agora obsoleto — as 3 lacunas que ele documentava foram fechadas); `ProxyTable` ganhou coluna "Em uso por". Tipos do frontend regenerados via `orval` contra o backend real rodando.
- **MUX-02 (Tipo não reseta no modal Nova Proxy)** — causa raiz: `key` do `ProxyFormModal` no modo create era sempre a mesma string literal, então React nunca remontava o componente (o `useState` interno reaproveitava o valor anterior). `ProxyPage` agora incrementa um contador a cada clique em "Nova proxy" e inclui no `key`.
- **MUX-03 (título genérico "Conflito de estado" pra qualquer 409)** — Backend (`FabricaDeConexaoMinecraftV1_8`) passou a anexar a causa raiz da falha de conexão na mensagem (timeout / host desconhecido / conexão recusada / sem rota até o host). Frontend (`errorMessages.ts`) usa esse prefixo estável pra escolher título específico, sem precisar de status/campo novo no contrato REST.
- **GAP-02 (slots de inventário sem aria-label)** — `ItemSlot` agora usa o texto do tooltip (já calculado) também como `aria-label`; todos os pontos de uso (inventário, hotbar, janela, cursor, equipamento) passam `label` explícito.
- **GAP-05 (semântica Iniciar vs Conectar)** — Tooltip explicativo nos botões "Iniciar"/"Conectar" da lista de bots. Nenhuma mudança de comportamento, só documentação inline.
- **MUX-01 (diálogo trocar proxy sem reaproveitar cadastradas/sem remover)** — `BotProxyModal` ganhou combobox "Proxy cadastrada" (com "Digitar manualmente" como fallback) e botão dedicado "Remover proxy" (host vazio, já tratado pelo backend desde a Sprint QA-01/`ProxySupport.resolver`).
- **GAP-01 (sem UI para abrir baú/clicar bloco)** — Formulário "Clicar bloco (abrir baú / interagir)" na aba Ações, reaproveitando o comando `ClicarBloco` já catalogado via o endpoint genérico `POST /commands` (sem endpoint REST novo). Ao concluir, invalida a query de `GET /inventario/janela` pra a seção "Janela" do Inventário atualizar sozinha.

Validação ao vivo contra o bot real Solk (`olimpo.clmc.com.br:3737`) e as 2 proxies já cadastradas: Dashboard mostrando threads/non-heap reais; tabela de Proxy com coluna "Em uso por"; modal de troca de proxy do bot com combobox e botão remover; Catálogo mostrando descrição do Twerk; aba Ações com o card "Clicar bloco" (testado com coordenada fora de alcance — retornou `{"resultado":"ERRO"}` via rede, sem efeito colateral no mundo real); aba Inventário com aria-label em todos os slots (ex. `"Slot 36: #3 ×24"`); tooltip de "Iniciar" confirmado via DOM. `GET /commands` retornando 405.

Build: `mvn -o package` via Maven Wrapper recém-adicionado; suíte completa do backend (1110+ testes) e do frontend (120 testes, 26 arquivos) passando; `tsc -b` sem erros.

### EPIC-PROD-04 (2026-07-28) — Auditoria de produtividade operacional e melhorias ALTA

Escopo: exclusivamente experiência de uso/produtividade operacional (Dashboard, Painel
Operacional, Lista de Bots, Sidebar, fluxo de criação de Bot, navegação, monitoramento,
escalabilidade visual para dezenas/centenas de bots) — não busca de bugs nem de problemas
arquiteturais (isso já foi feito no EPIC-QA-01/Sprints QA-01/QA-02 acima). Código C# não
consultado; só estado atual do projeto, documentação existente, backlog de homologação e
`docs-reescrita/docs/12-Interface/`. Nenhuma DEC aberta.

Auditoria completa com achados classificados ALTA/MÉDIA/BAIXA em
[docs/21-Homologacao/01-Backlog-QA-Homologacao-Completa.md](../21-Homologacao/01-Backlog-QA-Homologacao-Completa.md#6-epic-prod-04--auditoria-de-produtividade-operacional-2026-07-28)
(IDs `OPX-01` a `OPX-09`). Implementados só os ALTA sem mudança arquitetural, sem DEC nova,
reaproveitando componentes existentes do Design System:

- **OPX-01 (Dashboard sem atalho pra lista de Bots)** — `MetricCard`s de estado (conectados/
  executando/pausados/desconectados) e linhas do `StateBreakdownCard` (nova prop opcional
  `onItemClick`, retrocompatível) agora navegam para `/bots?estado=<ESTADO>`.
- **OPX-02 (lista de Bots sem filtro por estado)** — `Select` "Filtrar por estado" ao lado do
  `SearchBox` em `BotsPage`, usando nova `filterBotsByEstado` (`features/bots/services/filterBots.ts`),
  combinável com a busca textual já existente.
- **OPX-03 ("selecionar todos" só cobre a página atual de 10)** — botão "Selecionar todos os N bots
  filtrados" (e "Limpar seleção") quando a página inteira já está selecionada e o recorte filtrado
  é maior que a página, usando o novo `filteredIds` de `useBotTableState`.
- **OPX-04 (busca/filtro/página perdidos ao navegar pra um bot e voltar)** — `useBotTableState`
  reescrito pra persistir `q`/`estado`/`page` na URL via `useSearchParams` (em vez de `useState`
  local), o que também viabiliza o deep-link do Dashboard (OPX-01). `SearchBox` ganhou prop
  opcional `defaultValue` pra refletir o termo restaurado da URL. Único hook específico de Bots
  alterado — `useTableState` genérico (usado por Proxy/Contas/Servidores/Catálogo) não foi tocado.

OPX-05 (ações em lote sem limite de concorrência/progresso incremental) é ALTA mas não foi
implementado nesta sessão por exigir um componente do Design System ainda não construído
(`ProgressBar`), fora do critério "reaproveitar exclusivamente componentes existentes" definido
pra esta sessão — fica registrado pra sprint dedicada. OPX-06 a OPX-09 (MÉDIA/BAIXA) não
implementados, conforme escopo.

2 testes de `BotsPage.test.tsx` ajustados: o novo `Select` de filtro por estado introduz opções
com o mesmo texto (`Executando`/`Conectado`) que já apareciam como badge de estado na tabela,
causando `Found multiple elements`; os testes passaram a escopar a busca dentro da tabela
(`within(screen.getByRole('table'))`). `DashboardPage.test.tsx` passou a renderizar dentro de
`MemoryRouter` (novo uso de `useNavigate` no Dashboard).

Validação: `tsc -b` sem erros, `eslint .` sem erros (1 warning pré-existente não relacionado),
suíte completa do frontend (120 testes, 26 arquivos) passando. Validado ao vivo contra backend
real (Spring Boot + PostgreSQL, JDK 21 `ms-21.0.9`) e o bot real `Solk` conectado a
`olimpo.clmc.com.br:3737` (já reconectado automaticamente ao subir o backend, via
auto-reconnect): clique em "Bots conectados" no Dashboard navegou para `/bots?estado=CONNECTED`
com o `Select` já mostrando "Conectado" selecionado; navegação pro Painel Operacional do bot Solk
(console/vida/inventário/mundo reais) e volta via navegador preservou o filtro na URL sem
recarregar do zero; nenhum erro no console do navegador.

### EPIC-VIEWER-3D (2026-07-29) — Fundação do Viewer 3D (backend completo + Etapas 4-6 do frontend)

Sucessor do EPIC-VIEWER-04 (pausado, permanece como está - Debug 2D não foi tocado). Escopo desta
sessão: arquitetura definitiva do Viewer 3D (ver
[docs/21-EPIC-VIEWER-3D.md](../21-EPIC-VIEWER-3D.md) para a auditoria/plano completo produzidos
numa sessão anterior), implementada de ponta a ponta - backend (dirty-tracking, snapshot, canal WS
dedicado) e frontend (three.js/react-three-fiber: mesher com culling de face, entidades, HUD).
Nenhuma DEC nova aberta - a única decisão arquitetural desta sessão (canal WS dedicado em vez de
reaproveitar `/ws/bots/{id}/events`) já estava prevista e recomendada no próprio doc 21, seção 7.

**Backend:**
- `Mundo` ganhou dirty-tracking + versão monotônica: `chunksCarregadosPendentes`/
  `chunksDescarregadosPendentes`/`blocosAlteradosPendentes` (populados em `registrarChunk`/
  `descarregarColuna`/`definirBloco`, sem tocar nenhum Receptor de protocolo), drenados por
  `drenarChunksCarregados()`/`drenarChunksDescarregados()`/`drenarBlocosAlterados()`.
- `EntidadesDoMundo.remover` agora enfileira o id numa fila drenável (`drenarRemovidas()`) -
  entidades "atualizadas" continuam sendo reenviadas por inteiro a cada ciclo (payload pequeno,
  simplificação deliberada documentada em `ViewerDeltaResponse`).
- `Chunk`/`SecaoDeChunk` ganharam acessores de leitura bruta (`secaoPresente`/`idsDaSecao`/
  `metadataDaSecao`) para `ChunkSnapshotDto` serializar sem alocar um `Bloco` por índice.
- DTOs novos (`interfaces.rest.dto`): `ChunkPosicaoResponse`, `ChunkSnapshotDto` (binário compacto
  em Base64, ver comentário da classe), `BlocoAlteradoDto`, `EfeitoAtivoDto`, `EntidadeViewerDto`
  (superset de `EntidadeResponse` - velocidade/headYaw/equipamento/efeitos, tudo já existente no
  domínio, só sem DTO até agora), `ViewerSnapshotResponse`, `ViewerDeltaResponse`.
- `MundoController` ganhou `GET /api/v1/bots/{id}/viewer/snapshot?raioChunks=`.
- `NotificadorDeEventos` ganhou `temAssinantes(botId)` (opt-in gate); uma segunda instância
  (`notificadorDeEventosViewer`, bean `@Qualifier`, original marcado `@Primary`) isola o tráfego do
  Viewer do canal de log/estado existente.
- `ViewerWebSocketHandler` novo (`/ws/bots/{id}/viewer`) - snapshot completo automático ao
  conectar, depois deltas via `ViewerBroadcaster` (novo, `application.viewer`), agendado a cada
  150ms por `CicloDeVidaDoMotorDeExecucao` (terceira tarefa agendada, ao lado do tick de 50ms e da
  verificação de reconexão de 1s) - só serializa/transmite mundo para bots com assinante ativo no
  canal do Viewer, custo zero para os demais.
- Escopo deliberadamente fora desta sessão (documentado como limitação, não como pendência
  esquecida): metadata genérica de entidade (sneak/on-fire/nome customizado - só o codec existe,
  sem handler/receptor), biome/luz (descartados no parse), partículas, block entities além de
  placas, skin de jogador. Nenhum requer mudança de protocolo para o Viewer funcionar.

**Frontend** (`advancedbot-frontend`, dependências novas: `three`, `@react-three/fiber`,
`@react-three/drei`):
- Nova aba "Viewer 3D" (`Viewer3DPage`, rota `/bots/:id/viewer-3d`), mesmo padrão de tab do
  Debug 2D.
- `useViewerConnection` - abre o canal WS dedicado (`wsClient.connectViewerSocket`, novo), nunca
  faz polling REST. Dado pesado (chunks) vai direto para `ViewerWorldStore` (fora do estado React,
  lido pelo three.js via `useFrame`); só os campos pequenos (posição, inventário, entidades,
  caminho) entram em `useState`, para o React só re-renderizar o HUD.
- `chunkDecode.ts`/`ViewerWorldStore.ts`/`chunkMesher.ts` - decodificação do binário compacto,
  modelo de mundo do lado do cliente com o mesmo dirty-tracking do backend, e mesher com culling de
  face por vizinho (mesma família de algoritmo do `ChunkRenderer` do antigo ViewForm C#). Cor por
  bloco é determinística por id (sem atlas de textura/blockstate real ainda - Etapa 7 do roadmap,
  não bloqueia o valor operacional atual).
- `EntitiesLayer`/`PathOverlay`/`LookTargetOverlay` - entidades (caixa+nametag via
  `@react-three/drei` `Html`), caminho atual (linha, reaproveitando `CaminhoAtualResponse` já
  embutido em todo snapshot/delta) e bloco/entidade observados (raycast local contra o próprio
  `ViewerWorldStore`, sem chamar `/mundo/raycast` do backend - aproximação deliberada para o HUD,
  documentada em `lookRaycast.ts`).
- Câmera livre (`OrbitControls`, sempre centrada no bot) - o Viewer nunca envia nenhum comando ao
  bot, é puramente observador, conforme requisito do épico.

**Testes:** backend 1215 (0 falhas, 3 skips pré-existentes); frontend 207 (0 falhas), `tsc -b` e
`eslint .` limpos.

**Validação ao vivo:** backend real (Spring Boot + PostgreSQL) reiniciado com autorização explícita
do operador (processo já rodava com o bot real `Solk` conectado a `apolo.clmc.com.br:3636`;
auto-reconnect recuperou a sessão em segundos). `GET .../viewer/snapshot` e o WS
`/ws/bots/{id}/viewer` testados diretamente contra esse bot real (chunks/entidades/inventário reais
retornados, delta a cada ~150ms confirmado via client WS ad-hoc). Frontend verificado no navegador
contra esse mesmo backend: aba Viewer 3D renderizou o terreno real (água, árvores, relevo) do
servidor `apolo.clmc.com.br`, HUD mostrando posição/vida/dimensão reais do bot, nametag do Creeper
próximo renderizada, órbita de câmera funcional, sem erros no console.

Próximos passos (Etapas 7-9 do roadmap, upgrades de fidelidade visual, não bloqueiam o valor
operacional já entregue): parser real de blockstate/model JSON (geometria real de escada/cerca/
placa), biome/luz retidos no codec de chunk + água/lava com nível real, modelo de player/mob real
(hoje é caixa+nametag). Handler/receptor completo de Entity Metadata (sneak/fire/nome customizado)
também fica como candidato de sessão futura, fora do escopo original do épico mas identificado
durante a auditoria.

### EPIC-VIEWER-3D (2026-07-29) — Subsistema de Entidades e Câmera

Continuação direta da entrada anterior (mesmo dia): fecha os dois gaps que ficaram documentados
como "próximos passos" (Entity Metadata) e implementa o subsistema de câmera/overlays completo.
Auditoria prévia (domínio Java + legado C# `Projeto Adv 2.4.5`) confirmou que SpawnObject/XP Orb,
metadata de flags e todo o sistema de câmera multi-modo/animações/model rendering nunca existiram
no legado (`Handler_v152.cs`/`Handler_v18.cs` descartam esses pacotes ou nem têm `case` para eles;
`ViewForm.cs` só tinha toggle binário 1ª-pessoa/freecam, sem 3ª pessoa, sem mão do bot, sem
model renderer - só hitbox+nametag de texto) - trabalho é aditivo/novo, não há regra de negócio
herdada divergida. Nenhuma DEC nova aberta (mesmo raciocínio já registrado na entrada anterior:
todo pacote novo segue o padrão Packet→Codec→Handler→Evento→Receptor já estabelecido).

**Backend:**
- `EntidadeRemota` ganhou flags de metadata comuns a qualquer entidade (`FLAG_ON_FIRE`/
  `FLAG_AGACHADO`/`FLAG_CORRENDO`/`FLAG_INVISIVEL`, byte 0 do protocolo) e nome customizado
  (`definirNomeCustomizado`/`nomeCustomizado`/`nomeCustomizadoVisivel`); sealed permits estendido
  com duas subclasses novas: `EntidadeObjeto` (item dropado/minecart/projétil genérico via Spawn
  Object, com `itemStack` opcional só para tipo 2 = item dropado, populado via Entity Metadata
  índice 10) e `EntidadeXpOrb` (Spawn Experience Orb, campo `quantidade`).
- `MetadataDeEntidadeCodec` ganhou `decodificar()` (expõe valores por índice) ao lado do `pular()`
  já existente (mantido para SpawnMob/SpawnPlayer/SpawnObject, que só precisam do alinhamento de
  frame) - primeiro consumidor real do conteúdo de Entity Metadata desde a introdução do codec.
- 3 pacotes novos (5 arquivos cada, mesmo padrão Packet/Codec/Handler/Evento/Receptor):
  `SpawnObject` (0x0E), `SpawnExperienceOrb` (0x11), `EntityMetadata` (0x1C) - todos registrados em
  `RegistroDePacotesV1_8`/`AdaptadorConexaoBotV1_8`. Nenhum tinha precedente no legado (confirmado
  por auditoria - `Handler_v152.cs` descarta Spawn Object sem criar entidade, `Handler_v18/19.cs`
  nem tem `case` para ele); formato de fio validado direto contra a especificação do protocolo 47.
- `EntidadeViewerDto` estendido: `categoria` (JOGADOR/MOB/OBJETO/XP_ORB, `jogador` mantido por
  compatibilidade), `agachado`/`correndo`/`invisivel`/`comFogo`/`nomeCustomizado(Visivel)`,
  `tipoObjeto`/`itemStack` (só `EntidadeObjeto`), `quantidadeXp` (só `EntidadeXpOrb`).
  `EntidadeResponse.tipoDe` também ganhou os dois `case` novos (switch exaustivo sobre o sealed).

**Frontend** (`advancedbot-frontend`):
- Câmera extraída para módulo próprio `viewer3d/camera/` (`cameraMath.ts` - lógica pura testável,
  `CameraController.tsx`, `CameraModeSwitch.tsx`), substituindo a função `CameraFollow` antes
  embutida em `ViewerScene.tsx`. Três modos trocáveis em tempo real: `orbit` (comportamento
  original inalterado), `firstPerson` (câmera no olho do bot, rotação = yaw/pitch reais, near
  reduzido, oculta o próprio `SelfMarker`) e `thirdPerson` (atrás do bot a distância fixa,
  equivalente ao F5 vanilla). FOV mantido em 70° (default vanilla) nos três modos - sem
  dependência de sprint do próprio bot (fora de escopo, decisão já registrada na sessão anterior).
- `HeldItemView.tsx`/`assets/heldItemTexture.ts` - "mão do bot" em primeira pessoa: quad fixo no
  canto da câmera com a textura do item equipado, reaproveitando os sprite sheets 2D já existentes
  do HUD de inventário (recorte via `<canvas>`, mesma técnica de `ConstrutorDeAtlas`) em vez de
  novo asset.
- `EntitiesLayer.tsx` reescrito: categorias novas (item dropado = billboard do ícone do item, XP
  orb = billboard verde com tooltip da quantidade); visual de sneak (achata a caixa)/sprint
  (inclina)/fogo (overlay laranja translúcido)/invisibilidade (opacidade baixa, nunca some -
  ferramenta de debug); nome customizado no lugar do tipo quando visível; equipamento (armadura em
  4 caixas fincas sobrepostas + item na mão via mesmo helper de textura); animações de
  andar/atacar/morrer disparadas por transição de `velocityX/Z`/`ultimaAnimacao`/`ultimoStatus`
  (sem mudança de domínio, só consumo do que já era transmitido); interpolação de posição/rotação
  entre deltas (`useInterpolatedEntities.ts`, lerp de ângulo pelo caminho mais curto, função pura
  testável); hitbox de debug por entidade (toggle-controlada).
- `SelectedEntityOverlay.tsx` (realce da entidade observada, reaproveitando
  `calcularEntidadeObservada` já existente) e traço do raio de mira adicionado a
  `LookTargetOverlay.tsx` (antes só mostrava o bloco final).
- `DebugOverlaySettings.ts`/`DebugOverlayToggles.tsx` - 6 toggles individuais (raycast, bloco
  selecionado, entidade selecionada, pathfinding, hitboxes, gizmo de orientação via
  `GizmoHelper`/`GizmoViewport` do `@react-three/drei`, sem lib nova), estado em `Viewer3DPage`.

**Testes:** backend +9 arquivos de teste novos (`EntidadeObjetoTest`, `EntidadeXpOrbTest`, casos
novos em `EntidadeRemotaTest`/`EntidadeViewerDtoTest`, Codec+Handler+Receptor para os 3 pacotes
novos) - suíte completa (`mvn test`) verde. Frontend +3 arquivos de teste (`cameraMath.test.ts`,
`useInterpolatedEntities.test.ts`, casos novos cobrindo lerp de ângulo/posição/câmera 3ª pessoa) -
222 testes (`vitest run`) verdes, `tsc -b`/`eslint .` limpos. `heldItemTexture.ts` deliberadamente
sem teste dedicado (carregamento de imagem/canvas, mesma categoria não-testável de
`textureAtlas.ts`, que também não tem teste no projeto).

**Validação ao vivo:** backend Java reiniciado com autorização explícita do operador (havia uma
instância antiga já rodando na porta 8080, iniciada antes da sessão, servindo o schema OpenAPI
desatualizado) para regenerar os tipos Orval (`EntidadeViewerDto.ts` confirmado com todos os campos
novos). Frontend aberto no navegador contra esse backend e o bot real `Solk`: Viewer 3D carregou
sem erro de console, HUD/toolbars novos renderizaram (troca de câmera órbita → 1ª pessoa → 3ª
pessoa confirmada visualmente, gizmo de orientação reagindo à rotação, `SelfMarker` correto em 3ª
pessoa, toggle de overlay sem quebrar a cena). Bot estava desconectado do servidor Minecraft
durante o teste (0 entidades no mundo) - **não foi possível validar visualmente** mob/player real
com sneak/fogo/nome customizado/item dropado/XP orb nem os pacotes novos (SpawnObject/
SpawnExperienceOrb/EntityMetadata) contra tráfego de servidor real; cobertura desses caminhos fica
nos testes unitários/round-trip de codec, não em observação ao vivo.

Próximos passos: validar os 3 pacotes novos e a metadata de flags contra um servidor Minecraft real
com mobs/itens/XP orbs visíveis (a lacuna de validação ao vivo desta sessão); Etapas 7-9 do roadmap
seguem como upgrade de fidelidade futuro (blockstate real, biome/luz, modelo de player/mob real -
este último agora parcialmente mitigado pela caixa com armadura/item-na-mão/animações, mas ainda
não é um modelo 3D real de skin).

### EPIC-VIEWER-3D (2026-07-29) — Etapa 9: modelo real de player/mob (fecha o roadmap do épico)

Continuação direta da entrada anterior (mesmo dia). Usuário perguntou sobre skin real de player,
modelo do creeper e iluminação dos blocos - reauditoria descobriu que **Etapas 7 (parser real de
blockstate/model) e 8 (biome/luz real + água/lava com nível real) já estavam implementadas e
ligadas ao pipeline real** numa sessão anterior não refletida no meu contexto (`chunkMesher.ts` já
usa `modelResolver.ts`/`textureAtlas.ts`/`legacyBlocks.ts`/`biomeColors.ts`; backend já retém
block/sky light e bioma em `SecoesDeChunkCodec`/`Chunk`/`ChunkSnapshotDto` em vez de descartar).
Único item real do roadmap ainda pendente era a Etapa 9 - fechada nesta sessão. Sessão 100%
frontend, sem mudança de protocolo/domínio/DTO no backend.

**Fonte de textura:** skins de mob (`creeper.png`, `zombie.png` etc, formato padrão Minecraft
64x32/64x64) e skin padrão de player (`char.png`, estilo Steve) copiadas de
`anthack raiz\AntiHack_FULL\mob\` para `advancedbot-frontend/public/mob-skins/` (41 arquivos),
com autorização explícita do operador - mesmo critério já aceito para o `mc-assets` de blocos
(textura padrão Mojang extraída de client 1.8, uso local, sem redistribuição).

**Escopo consciente (decisão registrada no plano):** skin de PLAYER é sempre a padrão (`char.png`)
- buscar a skin real da conta via API da Mojang exigiria pipeline de rede/UUID novo, fora do que a
auditoria original do épico cobriu. Geometria biped (cabeça/tronco/braço/perna) é reaproveitada
para TODAS as entidades (players e mobs) - réplica exata do esqueleto de cada mob (aracnídeo pra
aranha, quadrúpede pra vaca) não é viável no orçamento desta sessão; o ganho real é sair de "caixa
sólida colorida" para "corpo articulado com a textura real do mob/player".

**Novo** (`advancedbot-frontend/src/features/bots/details/viewer3d/entityModel/`):
- `minecraftBoxUV.ts` - `retangulosUVDaCaixa` (função pura, testada) implementa o unwrap padrão de
  caixa Minecraft (cabeça em 0,0; tronco em 16,16; braço/perna em 40,16/0,16 - válido nos formatos
  64x32 e 64x64, já que a altura extra do 64x64 é só a camada de overlay, não usada aqui);
  `aplicarUVDeCaixaMinecraft` escreve isso no atributo `uv` de um `THREE.BoxGeometry` (ordem de
  grupo/vértice confirmada na fonte do three.js), com espelhamento horizontal pra reaproveitar a
  mesma região de textura no lado oposto do corpo (mesmo truque do formato clássico 64x32).
- `skinTextures.ts` - cache de textura completa por URL (mesmo padrão de `heldItemTexture.ts`, sem
  recorte); `SKIN_PADRAO_URL`; `MOB_SKIN_POR_NOME` (nome de exibição já resolvido pelo backend em
  `entidade.tipo` -> arquivo de skin; tipos sem arquivo disponível - Giant, Blaze, Magma Cube,
  Ender Dragon, Witch, Endermite, Guardian, Shulker, Horse, Rabbit - caem no fallback de cor sólida
  já existente, sem regressão).
- `EntityBipedModel.tsx` - biped de 6 caixas (cabeça/tronco/braço×2/perna×2) com pivôs de
  ombro/quadril expostos via `useImperativeHandle` para animação externa; textura real via
  `minecraftBoxUV` quando `textureUrl` resolve, fallback pra cor sólida quando não.

**Alterado:** `EntitiesLayer.tsx` - `EntityMarker`/`SelfMarker` trocam o `<boxGeometry>` único pelo
`<EntityBipedModel>`; animação de andar agora gira os pivôs de perna/braço em fase oposta (marcha
real) em vez de bob vertical do mesh inteiro; animação de ataque gira o braço direito pra frente em
vez de pulso de escala; convenção de coordenada do grupo raiz mudou de "centro da caixa" pra "pés"
(y=0), mesmo referencial usado pelo biped - `EquipamentoVisual` recalibrado pra essa nova pilha.
Sneak/fogo/invisibilidade/morte/hitbox continuam funcionando (atuam no grupo externo, agnósticos à
geometria interna).

**Testes:** `minecraftBoxUV.test.ts` (novo, `retangulosUVDaCaixa` contra valores conhecidos de
cabeça/tronco/braço) - 225 testes (`vitest run`) verdes, `tsc -b`/`eslint .` limpos. Sem teste
dedicado para `skinTextures.ts`/`EntityBipedModel.tsx` (carregamento de imagem/DOM, mesma categoria
não-testável de `heldItemTexture.ts`/`textureAtlas.ts`, convenção já estabelecida no projeto).

**Validação ao vivo:** backend real (porta 8080) e frontend reabertos contra o bot real `Solk`,
conectado e executando a macro na mobtrap (creeper por perto, mesma sessão de validação da entrada
anterior). Rede confirmou `GET /mob-skins/char.png` e `GET /mob-skins/creeper.png` retornando 200
assim que as entidades (bot + creeper) chegaram via WS - textura certa resolvida por categoria
(jogador vs. nome do mob). Câmera 3ª pessoa mostrou o `SelfMarker` com forma articulada e faixas de
cor distintas por segmento (não mais um bloco sólido), nametag e cone de direção funcionando, sem
erro de console. Cena permaneceu escura (mobtrap real sem luz - mesma condição já registrada na
validação anterior, não é regressão) - não foi possível fazer comparação pixel-a-pixel da skin
contra o cliente vanilla nesta sessão, mas o pipeline (textura certa por categoria, sem crash, UV
sem distorção visível nas partes que apareceram na captura) está confirmado ponta a ponta.

**Roadmap do EPIC-VIEWER-3D encerrado** (Etapas 1-9, ver `docs/21-EPIC-VIEWER-3D.md` seção 6).
Próximos passos, se houver interesse futuro: (a) validar os 3 pacotes novos da sessão anterior
(SpawnObject/SpawnExperienceOrb/EntityMetadata) contra tráfego real com item dropado/XP orb visível
- ainda não observado ao vivo; (b) skin real de player via API da Mojang (fora do escopo original);
(c) modelos específicos por família de mob (aracnídeo/quadrúpede) em vez do biped genérico
reaproveitado; (d) motor de propagação de luz para blocos alterados via delta (hoje aproximado,
documentado como limitação em `SecaoDeChunk`/`ViewerWorldStore`).

Itens marcados como resolvidos em [docs/21-Homologacao/01-Backlog-QA-Homologacao-Completa.md](../21-Homologacao/01-Backlog-QA-Homologacao-Completa.md). GAP-03 e MPF-01 seguem em aberto, adiados por decisão de produto/arquitetura pendente.

---

### TarefaMineracaoTrap (2026-08-03) — nova macro de mineração automática em trap de cobblestone

Implementação de `TarefaMineracaoTrap` a partir da especificação já aprovada em
`docs/10-Macros/10.2-Macro-Mineracao-Trap.md`. Sem equivalente no legado C# (Fonte da Verdade
`Projeto Adv 2.4.5` não contém nenhum `CommandMinerTrap`/`AutoMiner` de corredor) - mesma situação
de `TarefaMobSimples` (nova macro construída direto sobre a spec de docs-reescrita, sem conversão
de código legado).

**Novo** (`advancedbot-java/src/main/java/com/advancedbot/domain/bot/`):
- `TarefaMineracaoTrap.java` - FSM de 9 estados fiel à spec (`INICIALIZAR` até `RETORNAR_MINERACAO`),
  mesmo padrão estrutural de `TarefaMobSimples` (`Configuracao` record + `TarefaComConfiguracao`,
  `iniciarEspera`/`aguardando`, `AbridorDeBau`/`MacroUtils` reutilizados). Acúmulo de força de
  quebra reaproveita `CalculadoraDeQuebraDeBloco` no mesmo formato de `TarefaMineracao.quebrar()`.

**Decisões de implementação sem precedente direto (registradas aqui, sem DEC formal aberta -
escopo restrito a esta macro nova, sem alterar nenhuma primitiva de `SessaoDeJogo`):**
1. **Movimento sem `GuiaDeCaminho`/pathfinding**: como o túnel é reto ao longo do eixo X e a mira
   (Sul/Norte, eixo Z) é perpendicular ao deslocamento, a "caminhada" lateral é só o avanço/recuo
   direto de `x` em direção a `xFim`/`xInicio` via `moverEOlhar` (Z travado no valor registrado em
   cada extremidade), com velocidade constante própria (`VELOCIDADE_STRAFE_BLOCOS_POR_TICK = 0.13`)
   - nenhuma outra macro hoje anda sem pathfinding, `velocidadeHorizontal()` de `SessaoDeJogo`
   depende de `onGround` (motor de física vertical, DEC-32/35) nunca acionado por esta macro.
2. **Anti-stuck por posição estática**: sem precedente em nenhuma outra Tarefa (mais próximo:
   `TarefaMineracao.TIMEOUT_QUEBRA_MS`, que é timeout de quebra individual, não de posição) - 5s sem
   variação de `x` acima de um epsilon aciona re-teleporte via `homeInicio`/`homeFim` (Seção 5 da
   spec, "Bloqueio Físico no Corredor").
3. **Baú cheio ao descarregar**: a spec pede "tentar o próximo baú adjacente"; implementado como
   desativação segura da macro (mesmo padrão de "sem picareta nova no baú"), já que
   `MacroUtils.localizarBauProximo` não tem hoje uma variante "excluir o baú já tentado" - manter o
   fallback documentado é preferível a introduzir busca multi-baú sem consumidor validado.
4. **IDs de bloco/item literais** (cobblestone id 4, picareta de diamante id 278) - mesma convenção
   já usada em `TarefaMineracao`/`TarefaMobSimples` (ids vanilla hardcoded, sem enum de blocos/itens
   centralizado no domínio Java ainda).

**Alterado:**
- `interfaces/comando/ComandoMineracaoTrap.java` (novo) - toggle sobre `TarefaMineracaoTrap`, mesmo
  padrão de `ComandoMobSimples` (resolve `ConfiguracaoDeMacro` persistida por id/nome, alias
  `minerartrap`).
- `infrastructure/config/ConfiguracaoDeComandos.java` - registra `ComandoMineracaoTrap` no
  `GerenciadorDeComandos`.

**Validações executadas:** `mvnw compile` e `mvnw test` limpos (JDK 21, `.jdks/ms-21.0.9`) - suíte
completa existente sem regressão. Sem teste unitário dedicado para `TarefaMineracaoTrap` ainda
(pendente desta sessão) e **sem validação ao vivo contra servidor real** - o operador vai testar a
macro em jogo e ajustar comportamento a partir daí.

**Próximo passo recomendado:** testar `$minerartrap`/`/minerartrap` contra a trap real, ajustar
`VELOCIDADE_STRAFE_BLOCOS_POR_TICK`/`pitchMine`/`ALCANCE_MINERACAO` conforme o comportamento
observado, e então adicionar teste unitário cobrindo a FSM (mesmo padrão de
`TarefaMobSimplesTest`, se existir).

**Correção (mesma sessão, validação ao vivo):** primeiro teste real mostrou `xInicio`/`xFim`
idênticos e `MINERAR_SUL_DIREITA`/`NORTE_ESQUERDA` alternando infinitamente no mesmo tick (chegada
a &lt;1 bloco de si mesmo). Causa: `REGISTRAR_COORDS_INICIO`/`FIM` liam `sessaoDeJogo.x/y/z()`
só com base num timer fixo (`delayTeleporteMs`) após enviar o `/home`, sem confirmar que o
teleporte de fato já tinha sido processado pelo servidor - se a resposta demorasse mais que o
delay, capturava a posição ANTES do teleporte, duas vezes seguidas. Corrigido substituindo o gate
por tempo (`iniciarEspera`/`aguardando`) por `iniciarTeleporte`/`aguardandoTeleporte` em todo
teleporte que precisa da posição pós-chegada (calibração, entrada em mineração, re-teleporte
anti-stuck, ida pro baú): grava a posição ANTES de enviar o comando e só libera o próximo estado
quando `sessaoDeJogo.x/y/z()` de fato mudar além de 0.5 bloco (ou um teto de segurança de
`max(delayTeleporteMs*2, 15s)` estourar, com aviso ao operador nesse caso). Também adicionada
validação de distância mínima do túnel (`DISTANCIA_MINIMA_TUNEL = 5`) em `registrarCoordsFim` -
se início/fim caírem no mesmo ponto (homes mal configurados no servidor), a macro se desativa com
mensagem clara em vez de entrar no ping-pong. `mvnw compile`/`test` limpos após o ajuste.

**Segunda correção (mesma sessão):** com o ping-pong resolvido, andar funcionou mas a picareta
nunca quebrava nenhum bloco (inventário parado por 2min de teste ao vivo). Causa: mineração usava
`tracarRaioParaBlocos` com `pitchMine` fixo (~10°) - com o olho do bot 1.62 acima dos pés
(`ALTURA_DO_OLHO`) e a parede de 1 bloco de altura imediatamente ao lado (Z±1), um raio quase
horizontal passa por CIMA do topo da parede em vez de acertar a face lateral (só um pitch íngreme,
~50°+, alcançaria a face nessa distância - o que já não seria "olhar pro Sul/Norte"). Corrigido
trocando o raycast por mira direta no bloco-alvo calculado (coluna X atual, Z+1 Sul/Z-1 Norte) via
`olharParaBloco`, mesma primitiva que `TarefaMineracao` já usa - `minerarBloco()` substitui
`minerarParedeAFrente()`, e a caminhada agora só avança depois que o bloco à frente cai (antes
avançava e minerava em paralelo sem geometria válida). De caminho, `forcaDeQuebra` passou a ler
Haste/Mining Fatigue reais do próprio bot (`SessaoDeJogo.efeito(3)`/`efeito(4)`, Domínio "Efeitos"
DEC-39) em vez do sentinela `-1` fixo herdado de `TarefaMineracao` - atende o item 5 da spec
("Integração com Haste II"), que dependia do sinalizador ativo na chunk ser refletido na
velocidade de quebra. `mvnw compile`/`test` limpos após o ajuste; validação ao vivo ainda pendente
do lado do operador.

**Terceira investigação (mesma sessão) - "mundo vazio" descartado como bug de código.** Ainda sem
quebrar bloco, mira corrigida (feet+1) não resolveu - `MacroUtils.itemNaMao`/`contadorBalancoDeBraco`
confirmava via `GET /estado` que a mira nunca sequer executava `balancarBraco()`. Investigação via
`GET /mundo/bloco?x&y&z` revelou que TODA a área 5x5x5 ao redor da posição real do bot (confirmada
`CONNECTED` e se movendo de verdade via `GET /bots`) retornava ar (`blockId=0`), e blocos distantes
retornavam ids fora da faixa 0-255 (ex.: 1074, 3532) - inicialmente suspeito de bug de decodificação
de `SecoesDeChunkCodec`/`MapChunkBulkCodec` (formato não-vanilla).

Diagnóstico temporário (`System.err`, removido ao final - ver commits desta sessão) instrumentou
`ChunkDataCodec`/`MapChunkBulkCodec` com dump bruto (hex + tamanho calculado vs. real) em 3 rodadas
de reinício+reconexão do bot real. Resultado **descartou completamente a hipótese de bug**:
- IDs fora de 0-255 são decodificação CORRETA de um id de 12 bits não-vanilla (1074 = 67·16+2,
  documentado como comportamento deliberado no cabeçalho de `SecoesDeChunkCodec.java:9-12` - servidor
  customizado usa ids de bloco acima de 255, não é erro de shift).
- `bytesRestantes()` (leitor exposto temporariamente) confirmou **0 bytes sobrando** ao final de
  todo Map Chunk Bulk decodificado - a matemática de `calcularTamanho` bate byte-a-byte com o frame
  real, refutando a hipótese de desalinhamento de formato.
- A coluna do próprio bot (`chunkX=74219,chunkZ=0`) tinha o bit da seção 12 (y=192-207, cobre a
  posição real do bot em y=202) **setado na bitmask**, mas o dump bruto no offset exato dessa seção
  mostrou **32 bytes zerados de verdade, enviados assim pelo próprio servidor** - ar real, não
  artefato de leitura.

Conclusão: o cliente reflete fielmente o estado que o servidor está mandando; a "trap" na posição
onde o bot está simplesmente não tem bloco sólido ali segundo o servidor. Combinado com
`onGround` sempre `false` em toda a sessão e o Debug 2D mostrando só "vazio" ao redor - o bot está
genuinamente flutuando num vazio, não dentro de uma trap de cobblestone real. Hipóteses restantes,
fora do escopo de código (nenhuma decisão arquitetural pendente aqui): (a) `/home minerarinicio`/
`minerarfim` apontam pra coordenada errada/desatualizada no servidor; (b) a "trap" é um plot que
exige alguma ativação/posse não satisfeita por esta conta; (c) a posição salva pelo servidor pro
reconnect não é a mesma da trap. Instrumentação removida por completo (`ChunkDataCodec`/
`MapChunkBulkCodec`/`LeitorDePacote.bytesRestantes()` voltaram ao estado anterior a esta sessão) -
`mvnw compile`/`test` limpos. Próximo passo é do operador: confirmar visualmente no cliente Minecraft
real se há de fato cobblestone na posição reportada por `GET /estado`.

**Causa raiz confirmada e DEC aberta (mesma sessão, continuação):** operador confirmou via F3 do
cliente real (`Minecraft 1.8-forge1.8-11.14.4.1563/fml,forge`, 3 mods) bloco `minecraft:stone`
sólido exatamente na coordenada que nosso `Mundo.blocoEm` reportava como ar. Captura de pacote
bruta (Wireshark) decodificada em Python, independente do pipeline Java, confirmou 0 erros de
alinhamento/desalinhamento em ~7500 frames reais - a causa não é decodificação. O servidor-alvo é
Forge (ou híbrido), e o bot nunca implementou o handshake de Plugin Channel (`MC|Brand`/`FML|HS`,
pacote `0x3F`) que um cliente real completa no login - padrão reproduzido de forma idêntica em 2
contas distintas (`Solk`/`SwAFK_1`) e 2 dimensões (0/27): terreno natural sempre sincroniza,
conteúdo construído pelo jogador nunca sincroniza para o bot. Ver **DEC-44**
(`01-Decisoes-Arquiteturais.md`) para o diagnóstico completo, evidências e decisão de adiar a
implementação do handshake para sessão dedicada (fora de escopo desta sessão - mudança de protocolo
com risco de quebrar conexões existentes se malfeita). `TarefaMineracaoTrap` está com FSM/mira/
teleporte corretos mas **não pode ser validada ao vivo** neste servidor até a DEC-44 ser resolvida.
---

## Atualização 2026-08-04 — Causa raiz real encontrada e corrigida (supera o bloqueio de DEC-44)

Sessão de continuação da investigação acima. Objetivo inicial: provar experimentalmente (log
instrumentado, sem implementar nada até confirmar) a hipótese de corrida entre física local e chunk
ainda não carregado. Resultado: hipótese **refutada** (chunk sempre presente), mas revelou um bug
real e diferente — `SessaoDeJogo.atualizarPosicao` nunca zerava `motionX/Y/Z` do motor de física
local ao aplicar uma correção de posição do servidor; num servidor que reancora posição via
`PlayerPositionAndLook` repetido (padrão "cage"), gravidade acumulava silenciosamente atrás de cada
correção até se manifestar como queda de ~136 blocos sem oposição. Corrigido.

Investigação seguinte, motivada por relato do operador (bloco sólido real confirmado via F3 do
cliente oficial que o bot ainda reportava como ar mesmo pós-fix acima), **contradisse o diagnóstico
de DEC-44** (que havia atribuído o problema a ausência de handshake FML/Forge). Nova instrumentação
revelou que os "ids" retornados para blocos da trap, quando decompostos de volta para bytes brutos,
formavam um padrão de nibble repetido (`0x8888`, `0xAAAA`, `0xBBBB`) — assinatura de dado de **luz**
vazando para o parsing de id/metadata. Confirmado contra a especificação oficial do protocolo
(wiki.vg, Chunk Format): o layout real do wire é **por campo** (todos os ids de todas as seções,
depois toda luz de bloco, depois todo sky light) — não **por seção**, como `SecoesDeChunkCodec`
implementava. Os dois layouts têm o mesmo tamanho total em bytes, por isso nenhum teste de "bytes
restantes" (nem o desta sessão, nem o de DEC-44) jamais capturou o problema.

**Corrigido:** `SecoesDeChunkCodec.decode()`/`encode()` reescritos para o layout real (3 passes por
campo). **Isso supera DEC-44** — o handshake FML/Forge não é necessário para resolver a sincronização
de mundo. Ver **DEC-45** (`01-Decisoes-Arquiteturais.md`) para o diagnóstico completo, evidências e
validação. DEC-44 permanece registrada por completo (não editada), só recebeu uma nota de
atualização apontando para DEC-45.

Consequências diretas, também corrigidas nesta sessão:
- `TarefaMineracaoTrap.localizarAlturaDaParede` voltou a usar `feet+1` fixo (a mitigação cosmética de
  DEC-44, varredura ampla de Y, não é mais necessária com o layout de chunk corrigido) e ganhou
  varredura de distância em Z (1 a 4 blocos) — esta trap específica é um gerador de pedregulho cujo
  poço fica mais afastado do corredor de trânsito do que 1 bloco.
- **`TarefaMineracaoTrap` validada ao vivo pela primeira vez** contra `olimpo.clmc.com.br`: conta
  andando e quebrando pedregulho corretamente. Bloqueio de DEC-44 removido.
- Frontend (`advancedbot-frontend`): `chunkMesher.ts` corrigido para registrar `water_still`/
  `lava_still`/`water_flow`/`lava_flow` no atlas de textura (nunca eram carregados, água/lava sempre
  caíam no fallback "textura faltando" apesar dos PNGs existirem). Novo renderer de placa (id 63 em
  pé / 68 de parede) — `SignLayer.tsx`/`SignModel.tsx`/`assets/signBlocks.ts`/`assets/signTexture.ts`
  — mesmo padrão arquitetural de `ChestLayer.tsx`/`ChestModel.tsx` (block entity sem blockstate/
  model; texto da placa fora de escopo, só geometria/posição/orientação).

Validação: `mvn compile`/`mvn test` (backend, suíte completa) e `npm run typecheck`/`vitest run`
(frontend) limpos em todas as etapas. Nenhuma instrumentação temporária restou no código (removida
ao final de cada rodada, conforme protocolo de investigação acordado no início da sessão).

**Pendência registrada para sessão futura:** os testes de round-trip de
`ChunkDataCodecTest`/`MapChunkBulkCodecTest` são estruturalmente cegos à classe de bug corrigida
nesta sessão (testam `encode()`+`decode()` da mesma implementação, auto-consistentes mesmo quando os
dois erram do mesmo jeito). Recomenda-se um teste "golden file" com bytes reais de um Map Chunk Bulk
capturado ao vivo (Wireshark), decodificado só pelo `decode()`, para cobrir regressão futura desta
classe específica de erro.

---

## Atualização 2026-08-04 — GerenciadorDeMkb: infraestrutura reutilizável de armazenamento em baús (MKB)

**Motivo:** todas as macros que precisam guardar/retirar item de um ponto fixo de baús (mineração,
pesca etc.) reimplementavam essa lógica de forma ad-hoc (ex.: `TarefaAutoFish.detectarBaus`, varredura
dinâmica que só tenta o primeiro candidato). O responsável pediu um sistema MKB genérico e reutilizável:
fila de baús de reposição (servida sob demanda) + fileiras de baús de armazenamento que se expandem
lateralmente conforme lotam.

**Achado antes da implementação:** o legado C# de fonte da verdade (`Projeto Adv 2.4.5/
AdvancedBot.Client.Commands/Solk/MacroUtils.cs:79`, `LoadCheatsMKB`) só faz varredura dinâmica de baú
por proximidade (raio fixo em X/Y/Z) — não existe no legado nenhum conceito de fileira/coluna/baú de
reposição vs. baú de armazenamento. Não há também doc de spec prévia em `docs-reescrita/docs/
10-Macros`. Confirmado com o responsável (nesta sessão) que a implementação segue direto para código,
com este registro documentando o comportamento depois — mesma exceção já usada para
`TarefaMineracaoTrap` quanto à ausência de precedente, mas sem o passo de spec-doc prévia dessa vez.

**Entregue:**
- `Coordenada` (`domain.bot`, record `x,y,z`) — trio de coordenadas usado só para a aritmética de
  layout de MKB; deliberadamente distinto de `PontoDeCaminho` (que carrega estado mutável de
  pathfinding e não deve ser reaproveitado como carregador de posição genérico).
- `MkbConfiguracao` (`domain.bot`, record) — layout de uma "casa de baús": `homeComando`, base+passo+
  quantidade da fila de reposição, base+passo+baús-por-fileira+passo-de-fileira+limite-de-fileiras do
  armazenamento. `bauDeReposicao(indice)`/`bauDeArmazenamento(fileira, coluna)` resolvem a coordenada
  de qualquer baú da grade a partir da base + passos configurados.
- `GerenciadorDeMkb` (`domain.bot`) — componente poll-based reutilizável, mesmo padrão de
  `AbridorDeBau`: cada Tarefa dona de uma instância própria, chama `guardarItem(sessaoDeJogo, itemId)`
  ou `retirarItem(sessaoDeJogo, itemId, durabilidadeMaxima)` uma vez por tick (mesmos argumentos a cada
  chamada da mesma operação) até receber algo diferente de `AGUARDANDO`. Internamente compõe:
  `GuiaDeCaminho` (DEC-35, caminhada até o baú-alvo, dispensa reimplementar strafe manual) +
  `AbridorDeBau` (abertura) + transferência direta via `SessaoDeJogo.moverItem`/`localizarEspacoLivreNaJanela`
  (mesmo padrão de `TarefaMineracaoTrap.abrirBausDescarregar`, varredura de slots 9-35 sem tocar a
  hotbar). Aceita uma ou mais `MkbConfiguracao` (construtor vararg) para suportar múltiplas "casas de
  baú".

  Regras de negócio validadas com o responsável nesta sessão:
  1. Reposição só ocorre sob demanda (nunca automática ao passar pela MKB).
  2. Armazenamento vai sempre para o primeiro baú com espaço livre, varrendo a fileira atual da
     esquerda pra direita.
  3. Baú cheio avança coluna; fileira cheia avança fileira (design pensado para fileiras "infinitas" —
     o operador sempre constrói mais fileiras ao lado antes de esgotar as existentes).
  4. Ao atingir `limiteDeFileiras` (quando configurado > 0), tenta a próxima `MkbConfiguracao` da lista;
     se não houver mais nenhuma, reseta para a fileira 0 da primeira (baús antigos podem ter sido
     esvaziados manualmente).
  5. Salvaguarda `MAX_AVANCOS_POR_OPERACAO = 500` (não é regra de negócio, só teto de segurança contra
     loop sem fim em caso de configuração quebrada ou baús genuinamente esgotados em todas as MKBs).

**Nenhuma Tarefa existente foi alterada** — capacidade puramente aditiva, ainda sem nenhum consumidor
real (`TarefaAutoFish`/`TarefaMineracaoTrap` continuam com suas próprias lógicas de baú, inalteradas).

**Validação executada:** `mvnw compile`/`mvnw test-compile` limpos. Nenhum teste dedicado foi escrito
nesta sessão para `GerenciadorDeMkb`/`MkbConfiguracao`/`Coordenada`.

**Riscos/limitações conhecidas:**
- Sem teste automatizado e sem validação ao vivo contra um servidor real — comportamento (avanço de
  fileira, reposição cíclica, reset entre MKBs) só foi verificado por leitura de código.
- Nenhuma macro existente foi migrada para usar `GerenciadorDeMkb` — é infraestrutura pronta, não uma
  feature ponta-a-ponta ainda visível para o operador.

**Próximo passo recomendado:** escrever `GerenciadorDeMkbTest` cobrindo avanço de coluna/fileira,
transição entre múltiplas MKBs e o ciclo de reposição; e escolher a primeira Tarefa real para consumir
o componente (candidata natural: migrar `TarefaAutoFish.detectarBaus`/`GUARDAR_ITENS_ABRINDO_BAU` para
`GerenciadorDeMkb`, ou usá-lo diretamente na próxima macro nova que precisar de baú de reposição/
armazenamento estruturado).

---

## Atualização 2026-08-04 (continuação) — TarefaMobSimples migrada para GerenciadorDeMkb

**Motivo:** o responsável apontou que a troca de espada de `TarefaMobSimples` ainda buscava "um baú
específico" via `MacroUtils.localizarBauProximo` (varredura dinâmica de UM baú no cubo [-2,2) ao redor
dos pés, sem fila/substituta em mais de um lugar) - primeiro consumidor real de `GerenciadorDeMkb`.

**Entregue:**
- `GerenciadorDeMkb` ganhou um terceiro modo (`Modo.TROCAR_REPOSICAO`, método `trocarNaReposicao`):
  devolve o item gasto do slot de origem informado (se o itemId bater) e puxa uma substituta na mesma
  visita ao baú, fiel ao padrão `TROCAR_ESPADA_NO_BAU` original (o mesmo baú de reposição serve tanto
  de estoque quanto de descarte do item usado - diferente de `guardarItem`, que vai pra grade de
  armazenamento). Sem substituta na posição atual, avança a fila cíclica de reposição
  (`avancarReposicao`, já existente, reaproveitada sem alteração).
- `TarefaMobSimples`: estados `IR_PARA_BAU_ESPADAS`/`TROCAR_ESPADA_NO_BAU` (varredura dinâmica +
  `AbridorDeBau` manual) substituídos por um único `TROCANDO_ESPADA_NA_MKB`, que teleporta, captura a
  posição de pouso como base da fila de reposição e delega tudo a `gerenciadorDeMkb.trocarNaReposicao`.
  `Configuracao` ganhou `reposicaoOffsetX/Y/Z` (deslocamento do primeiro baú em relação ao pouso do
  teleporte, não mais uma varredura no cubo ao redor dos pés), `reposicaoPassoX/Y/Z` (deslocamento
  entre baús consecutivos da fila) e `quantidadeReposicao` - todos com chave própria em
  `apartirDeParametros`/`paraMapa`, PADRAO replica o layout de fila ao longo do eixo Z descrito pelo
  responsável nesta sessão. A base é recalculada a cada execução a partir da posição real de pouso
  (não gravada como coordenada absoluta no perfil), portátil entre servidores/contas.

**Validação executada:** `mvnw test` (suíte completa, 1278 testes) limpo. As duas tests de troca de
espada (`deveDesativarAMacroQuandoNaoHaEspadaSubstitutaNoBau`/`deveTrocarEspadaGastaPelaSubstitutaDoBau`)
foram adaptadas: como `GerenciadorDeMkb` agora depende de `GuiaDeCaminho`/`SessaoDeJogo.caminhoAtual()`
(só avançado por `MotorDeTick` em produção), os testes passaram a tickar manualmente o caminho ativo a
cada iteração (mesmo padrão de `GuiaDeCaminhoTest.ticarAteFinalizarOuLimite`) e a rodar múltiplas
chamadas de `executar()` até a operação de MKB se resolver, em vez de uma única chamada síncrona.

**Riscos/limitações conhecidas:** ainda sem validação ao vivo contra um servidor real (offset/passo
precisam ser calibrados manualmente por quem configurar o perfil `mobsimples`, valor padrão é só um
palpite razoável). `GerenciadorDeMkbTest` dedicado continua pendente (mesma pendência da atualização
anterior).

**Próximo passo recomendado:** validar ao vivo contra um servidor real com uma MKB configurada;
escrever `GerenciadorDeMkbTest`; avaliar migrar `TarefaMineracaoTrap`/`TarefaAutoFish` (mesmo padrão de
baú de reposição/descarga) para `GerenciadorDeMkb` quando fizer sentido.
