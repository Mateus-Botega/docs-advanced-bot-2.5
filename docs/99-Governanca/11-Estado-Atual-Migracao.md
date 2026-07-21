# 11 - Estado Atual da Migração

> Última atualização: 2026-07-21
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

Não implementado:

- Serviços
- Repositórios
- APIs REST
- Scheduler distribuído (multi-nó/multi-JVM) — o scheduler single-JVM em processo já existe desde a Milestone 21 (`AgendadorDeTarefasPort`/`MotorDeTick`), sem nenhuma tarefa real registrada ainda
- Rede/Protocolo Minecraft (criptografia AES/RSA real e autenticação Mojang para servidores em modo online, Session Server, Status State)
- Demais pacotes do estado PLAY (Entity Metadata `0x1C` — coberto apenas pelo descarte seguro da DEC-20, sem semântica exposta —, combate, XP/Experience — sem precedente no C# auditado, mesmo tratamento de Time Update/Spawn Position); bounded context de Mundo (Milestone 7) **encerrado oficialmente na Milestone 18** — Chunk Data, Block Change, Multi Block Change, Map Chunk Bulk, Explosion, Change Game State e Update Sign implementados; Time Update (`0x03`), Spawn Position (`0x05`), World Border (`0x44`), Update Block Entity (`0x35`), Block Action (`0x24`) e Block Break Animation (`0x25`) avaliados e confirmados sem nenhum precedente no C# legado — cobertos com fidelidade pelo descarte seguro da DEC-20 (ver Incrementos 7.5 e 18.1). Player List/tab list e Chat Enviado pelo Bot **implementados** — ver Milestone 8. Movimentação (Player Position `0x04`) e rotação (Player Look `0x05`) do jogador **implementadas** — ver Milestone 9. Swing Arm (`0x0A`), Player Digging (`0x07`) e Player Block Placement (`0x08`) **implementados** — ver Milestone 10; mineração automática, física de quebra e "usar item na mão" (sentinela -1/-1/-1, Codec já suporta) continuam fora de escopo. Combinação posição+rotação iniciada pelo bot **implementada** — ver Milestone 11 (reaproveita `ConfirmacaoDePosicaoPacket`, DEC-22); Tick loop/motor de física automático continua fora de escopo. Entity Action `0x0B` serverbound **implementado para o subconjunto sneak** (`actionId` 0/1) — ver Milestone 20; leave bed/jump boost são código morto comprovado no legado, sprint e `Player` bare `0x03` só têm call site dentro do Tick loop de física (mesmo motivo de exclusão da DEC-22).
- Interação com inventário: clique, uso de itens, crafting, baús/janelas não-jogador (Open Window/Close Window/Confirm Transaction não implementados)
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

Milestone 22 — candidata, ainda não escolhida pelo responsável do projeto

Histórico (Milestones 5 a 21, todas concluídas — mantido por rastreabilidade)

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

Candidatos para Milestone 22 (nenhum escolhido ainda; nenhuma DEC pendente bloqueia qualquer um deles)

- Altura de olho por tipo de mob (`EntityProperty.Height` do legado) — desbloquearia um `CanSeeEntity` genérico sobre `Mundo.tracarRaio` para combate contra mobs (jogadores já cobertos desde a Milestone 19 via `podeVerJogador`/constante `1.62`; levantar a altura por tipo de mob é um levantamento de dados maior, avaliado e deixado de fora desde a Milestone 13).
- Criptografia AES-CFB8/modo online e integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` — candidatas desde o encerramento da Milestone 4, ainda não escolhidas (a segunda desbloquearia também um `ComandoReco`).
- Definir a estratégia de wiring/DI para a fábrica de conexão em produção e para o catálogo de comandos (Spring `@Configuration` ou factory manual — `infrastructure.config` permanece vazio) continua em aberto; não bloqueia nenhum candidato.
- Decisão de transporte para o operador (CLI ou API, DEC-02 ainda não decidida) — único jeito de efetivamente consumir `SaidaDoOperador` (Milestone 15) fora dos testes.
- Um `Comando`/macro de mineração completa que consuma `CalculadoraDeQuebraDeBloco` (Milestone 17) + `Mundo.criarCaminhoPara`/`SessaoDeJogo.criarCaminhoPara` (Milestone 16), registrada como `TarefaContinua` no `MotorDeTick` (Milestone 21) — as primitivas de cálculo/navegação e agora também o mecanismo de execução existem, mas nenhum consumidor de produção foi construído ainda (mesma situação inicial de `Mundo.tracarRaio` após a DEC-24); qualquer `TarefaContinua` real que leia/escreva estado de física precisará da mesma análise "isso reabre a DEC-22?" já resolvida pela DEC-29 para o mecanismo em si, feita agora para o conteúdo concreto.
- AutoReconnect — desbloqueado pela Milestone 21: `AgendadorDeTarefasPort.agendar(Runnable, Duration)` já cobre o mecanismo genérico que `autReconnectTimer` do legado precisaria; falta só a lógica de reconexão em si (fora de escopo da Milestone 21 por instrução explícita).
- Proxy por bot — candidata de infraestrutura de rede, independente da engine de execução; ainda não desenhada.
- Suporte a NBT real em `ItemStack` (encantamentos/nome customizado/lore) — desbloquearia `nivelEficiencia` real em `CalculadoraDeQuebraDeBloco.forcaDoJogador` em vez do parâmetro `0` (Milestone 17/DEC-28); também serviria outros consumidores (crafting, display).
- Rastreamento de efeitos do próprio bot (Haste/Mining Fatigue e demais) — hoje `EntidadeRemota.efeitosAtivos` só cobre entidades remotas (`ReceptorEntityEffect`, Milestone 5.6.4); desbloquearia `amplifierCeleridade`/`amplifierFadiga` reais em `CalculadoraDeQuebraDeBloco.forcaDoJogador` em vez do sentinela `-1` (Milestone 17/DEC-28) — decisão arquitetural própria (`SessaoDeJogo` vs. reaproveitar `EntidadeRemota`), ainda não tomada.
- DI de produção (wiring de `AgendadorDeTarefasPort`/`MotorDeTick`/`ConexaoBotPort` via Spring `@Configuration`) e CLI/API para o operador iniciar/pausar/parar um bot — `infrastructure.config` permanece vazio; explicitamente adiado pela Milestone 21.

Candidatos encerrados nas Milestones 18-21 (não reabrir sem novo motivo): checagem de linha de visão contra entidade (Milestone 19); `Player` bare/Entity Action → `ComandoSneak` (Milestone 20 — subconjunto sneak implementado, leave bed/jump boost são código morto comprovado, sprint e `Player` bare são Tick-loop-only, todos fora de escopo por DEC-22, não por lacuna de precedente); restante da Milestone 7 (Milestone 18 — bounded context de Mundo encerrado oficialmente); fundação de execução contínua/scheduler/tick engine (Milestone 21 — DEC-29 entrega o mecanismo; ver candidato de macro de mineração acima para o que continua em aberto).

Fora de escopo por política do projeto (não candidatos): conteúdo de física/automação dentro de qualquer `TarefaContinua` (leitura/escrita de Motion/OnGround/velocidade — e, por consequência, `PathGuide`/execução de caminho tick a tick — Milestone 16 —, mineração automática — Milestone 17 —, sprint e `Player` bare — Milestone 20), combate/automação, lógica de inventário automático — nenhum desses é reconsiderado a cada milestone; permanecem bloqueados por decisão de escopo, não por dependência técnica. Desde a DEC-29 (Milestone 21), o *mecanismo* de scheduler/tick engine deixou de estar nesta lista — só o *conteúdo* físico/automático permanece bloqueado; ver DEC-29 para a distinção completa.

---

# 11. Próximo Prompt Esperado

Resumo da próxima atividade que deverá ser executada pela IA.

Milestones 5 a 21 concluídas — ver Seção 5 (subseções por milestone) e Seção 10 (histórico condensado) acima. Milestone 21 (mais recente): Incremento 21.1 (DEC-29 — Fundação da Engine de Execução Contínua — pivô de escopo: pela primeira vez desde a Milestone 4, a entrega é infraestrutura de execução, não uma capacidade isolada de protocolo. Novos `domain.bot.EstadoExecucao` (`PARADO`/`EXECUTANDO`/`PAUSADO`, sem precedente no legado); `Bot` ganha `iniciar()`/`pausar()`/`retomar()`/`parar()` e `registrarTarefa`/`removerTarefa`/`tarefasContinuas`; `domain.bot.TarefaContinua` (interface funcional `void executar(Bot)`, zero implementações reais); `application.port.AgendadorDeTarefasPort` (segundo Port do projeto, `agendar(Runnable,Duration)`/`encerrar()`); `infrastructure.execucao.AgendadorDeTarefasVirtualThread` (relógio single-thread disparando cada execução em Virtual Thread nova, DEC-03) e `MotorDeTick` (percorre bots `EXECUTANDO`, invoca `TarefaContinua` isolando falha por tarefa, protegido contra reentrância). DEC-29 resolve explicitamente que o mecanismo — sem nenhuma leitura/escrita de física e sem nenhuma tarefa real registrada — não reabre o bloqueio de "Tick loop/motor de física automático" já firmado pela DEC-22/DEC-23/DEC-27 (mesma distinção "mecanismo vs. conteúdo" da própria DEC-27); nenhum Packet/Comando/bounded context novo, nenhuma interface existente alterada) — 726 testes automatizados, 0 falhas, 0 erros, 3 skipped deliberadamente. Nenhuma tarefa de código pendente para o Incremento 21.1. Próximo passo: candidatos não comprometidos listados na Seção 10 (altura de olho por tipo de mob que desbloquearia `CanSeeEntity` genérico; criptografia/modo online; integração de disconnect que desbloquearia `ComandoReco`; AutoReconnect real sobre `AgendadorDeTarefasPort`; Proxy por bot; um `Comando`/macro de mineração completa registrada como `TarefaContinua` no `MotorDeTick`, consumindo `CalculadoraDeQuebraDeBloco`+`criarCaminhoPara`; suporte a NBT real em `ItemStack` para `nivelEficiencia` real; rastreamento de efeitos do próprio bot para `amplifierCeleridade`/`amplifierFadiga` reais; DI de produção e decisão de transporte CLI/API — DEC-02 — para efetivamente consumir `SaidaDoOperador` e operar o ciclo de vida do bot) — a critério do responsável do projeto. Conteúdo de física/automação dentro de qualquer `TarefaContinua` (e `PathGuide`/mineração automática/sprint por consequência) e combate/automação permanecem fora de escopo por política do projeto — só o mecanismo de agendamento deixou de estar bloqueado (DEC-29). Nenhuma DEC pendente bloqueia os candidatos remanescentes.

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

---

# 13. Critérios para Continuidade

Antes de iniciar qualquer nova tarefa é obrigatório verificar:

- Estado deste documento
- Plano de Migração
- Fundação Arquitetural
- Decisões Arquiteturais
- Baseline Tecnológica

Caso exista conflito, este documento deve ser atualizado antes da implementação.