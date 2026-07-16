# 11 - Estado Atual da Migração

> Última atualização: 2026-07-15
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

Em andamento

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

Não implementado:

- Serviços
- Repositórios
- APIs REST
- Scheduler
- Rede/Protocolo Minecraft (compressão de frames zlib, criptografia AES/RSA real, Session Server/Mojang Authentication, Play State, Status State, `ProtocolDispatcher`/`PacketHandler`s concretos)
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

Milestone 4 (incremento 5) — Fluxo de Handshake/Login e ProtocolDispatcher/PacketHandlers

Objetivos (conforme Fase 3 do [07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md](07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md))

- Implementar `ProtocolDispatcher`/`PacketHandler`s concretos para os packets já modelados do estado LOGIN
- Orquestrar o fluxo de Handshake → Login Start → Encryption/Login Success usando o `TransporteSocket` e o `RegistroDePacotesV1_8`
- Avaliar se/quando a criptografia AES-CFB8 real (DEC-04, BouncyCastle) e a compressão funcional (zlib) serão necessárias, sempre usando o C# como fonte da verdade

Ainda não aprovada — aguardando plano resumido e aprovação explícita antes de iniciar, conforme processo do CLAUDE.md.

---

# 11. Próximo Prompt Esperado

Resumo da próxima atividade que deverá ser executada pela IA.

Apresentar plano resumido para a Milestone 4 (incremento 5 — `ProtocolDispatcher`/`PacketHandler`s concretos e orquestração do fluxo Handshake/Login) com base no código C# (`AdvancedBot.Client.MinecraftClient`, `AdvancedBot.Client.Handler`) e aguardar aprovação antes de implementar.

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

---

# 13. Critérios para Continuidade

Antes de iniciar qualquer nova tarefa é obrigatório verificar:

- Estado deste documento
- Plano de Migração
- Fundação Arquitetural
- Decisões Arquiteturais
- Baseline Tecnológica

Caso exista conflito, este documento deve ser atualizado antes da implementação.