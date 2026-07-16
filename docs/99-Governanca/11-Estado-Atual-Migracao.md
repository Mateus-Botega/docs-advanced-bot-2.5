# 11 - Estado Atual da Migração

> Última atualização: 2026-07-16
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

Em andamento — fase de planejamento concluída

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

Próximo passo autorizado

Incremento 1 — retenção da Sessão de Jogo e ciclo de vida da conexão após LOGIN, conforme DEC-19 — está autorizado a iniciar.

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

Não implementado:

- Serviços
- Repositórios
- APIs REST
- Scheduler
- Rede/Protocolo Minecraft (criptografia AES/RSA real e autenticação Mojang para servidores em modo online, Session Server, Play State, Status State)
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

Milestone 5 — Play State

Objetivos (conforme Fase 5 do [07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md](07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md))

- Entre as três frentes candidatas registradas ao encerrar a Milestone 4 (criptografia AES-CFB8/modo online, Play State, integração de disconnect/DI), o responsável do projeto escolheu **Play State**. Criptografia/modo online e a integração de `CasoDeUsoDesconectarBot`/`ConexaoBotPort.disconnect()` permanecem candidatas para uma milestone futura — não descartadas, apenas não escolhidas agora.
- Fase de planejamento concluída e aprovada (ver "Milestone 5 → Fase de Planejamento" acima): desenho arquitetural completo, DEC-19 (retenção da Sessão de Jogo e roteamento de eventos) e DEC-20 (tolerância a pacotes PLAY não registrados) formalizadas.
- Incremento 1 (retenção da Sessão de Jogo, conforme DEC-19) está **autorizado a iniciar**.
- Definir a estratégia de wiring/DI para a fábrica de conexão em produção (Spring `@Configuration` ou factory manual — nenhum padrão foi estabelecido ainda, `infrastructure.config` permanece vazio) continua em aberto; não bloqueia o Incremento 1.

Aprovada para início de implementação — plano resumido apresentado e aprovado pelo responsável do projeto, DEC-19 e DEC-20 formalizadas, conforme processo do CLAUDE.md.

---

# 11. Próximo Prompt Esperado

Resumo da próxima atividade que deverá ser executada pela IA.

Implementar o Incremento 1 da Milestone 5 — retenção da Sessão de Jogo e ciclo de vida da conexão após LOGIN, conforme [DEC-19](01-Decisoes-Arquiteturais.md): `SessaoDeJogo` (`domain.bot`), alteração do retorno de `ConexaoBotPort.connect()` para `SessaoDeJogo`, e os ajustes correspondentes em `CasoDeUsoConectarBot` e `AdaptadorConexaoBotV1_8`. Seguir o mesmo padrão de incremento pequeno, revisável e testado isoladamente já usado na Milestone 4.

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

---

# 13. Critérios para Continuidade

Antes de iniciar qualquer nova tarefa é obrigatório verificar:

- Estado deste documento
- Plano de Migração
- Fundação Arquitetural
- Decisões Arquiteturais
- Baseline Tecnológica

Caso exista conflito, este documento deve ser atualizado antes da implementação.