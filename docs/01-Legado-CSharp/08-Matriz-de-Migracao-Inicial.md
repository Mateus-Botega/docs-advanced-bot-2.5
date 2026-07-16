# 08 — Matriz de Migração Inicial

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-08                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Matriz de Rastreabilidade                        |
| **Escopo**         | Rastreabilidade completa C# → Java               |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-03, LEG-05, LEG-06, docs/99-Governanca/13-Matriz-de-Rastreabilidade.md |

---

## Legenda de Status

| Status | Significado |
|--------|-------------|
| `Não iniciado` | Levantado, aguardando início |
| `Em análise` | Em estudo e documentação |
| `Documentado` | Documentação de legado concluída |
| `Em desenvolvimento` | Implementação Java iniciada |
| `Implementado` | Código Java desenvolvido |
| `Testado` | Testes realizados |
| `Validado` | Equivalência funcional comprovada |
| `Concluído` | Item finalizado |
| `Bloqueado` | Dependência externa não resolvida |
| `Descartado` | Não será migrado para Java |

---

## Legenda de Criticidade

| Criticidade | Descrição |
|-------------|-----------|
| **BLOQUEANTE** | Bloqueia avanço da migração |
| **CRÍTICO** | Falha compromete todo o sistema |
| **ALTO** | Afeta múltiplos fluxos |
| **MÉDIO** | Afeta fluxo específico |
| **BAIXO** | Impacto limitado |
| **N/A** | Não aplicável (descartado) |

---

## Seção 1 — Infraestrutura de Rede e Protocolo

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| NET-001 | `MinecraftClient.cs` | `br.com.advancedbot.client.MinecraftClient` | BLOQUEANTE | Documentado |
| NET-002 | `PacketStream.cs` | `br.com.advancedbot.network.PacketStream` | BLOQUEANTE | Documentado |
| NET-003 | `PacketQueue.cs` | `br.com.advancedbot.network.PacketQueue` | CRÍTICO | Documentado |
| NET-004 | `ReadBuffer.cs` | `br.com.advancedbot.network.ReadBuffer` | CRÍTICO | Documentado |
| NET-005 | `WriteBuffer.cs` | `br.com.advancedbot.network.WriteBuffer` | CRÍTICO | Documentado |
| NET-006 | `MinecraftStream.cs` | `br.com.advancedbot.network.MinecraftStream` | ALTO | Documentado |
| NET-007 | `IPacket.cs` | `br.com.advancedbot.network.packet.IPacket` | CRÍTICO | Documentado |
| NET-008 | `ProtocolHandler.cs` | `br.com.advancedbot.network.handler.ProtocolHandler` | CRÍTICO | Documentado |
| NET-009 | `Handler_v152.cs` | `br.com.advancedbot.network.handler.HandlerV152` | ALTO | Documentado |
| NET-010 | `Handler_v17.cs` | `br.com.advancedbot.network.handler.HandlerV17` | ALTO | Documentado |
| NET-011 | `Handler_v18.cs` | `br.com.advancedbot.network.handler.HandlerV18` | ALTO | Documentado |
| NET-012 | `Handler_v19.cs` | `br.com.advancedbot.network.handler.HandlerV19` | MÉDIO | Documentado |
| NET-013 | `PacketStreamLegacy.cs` | `br.com.advancedbot.network.handler.PacketStreamLegacy` | MÉDIO | Documentado |
| NET-014 | `Proxy.cs` | `br.com.advancedbot.network.proxy.ProxyConnector` | ALTO | Documentado |
| NET-015 | `ProxyType.cs` | `br.com.advancedbot.network.proxy.ProxyType` | MÉDIO | Documentado |

---

## Seção 2 — Pacotes de Rede (Serializadores)

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| PKT-001 | `PacketHandshake.cs` | `br.com.advancedbot.network.packet.PacketHandshake` | CRÍTICO | Documentado |
| PKT-002 | `PacketLoginStart.cs` | `br.com.advancedbot.network.packet.PacketLoginStart` | CRÍTICO | Documentado |
| PKT-003 | `PacketEncryptionResponse.cs` | `br.com.advancedbot.network.packet.PacketEncryptionResponse` | BLOQUEANTE | Documentado |
| PKT-004 | `PacketClientSettings.cs` | `br.com.advancedbot.network.packet.PacketClientSettings` | ALTO | Documentado |
| PKT-005 | `PacketClientStatus.cs` | `br.com.advancedbot.network.packet.PacketClientStatus` | ALTO | Documentado |
| PKT-006 | `PacketPluginMessage.cs` | `br.com.advancedbot.network.packet.PacketPluginMessage` | MÉDIO | Documentado |
| PKT-007 | `PacketChatMessage.cs` | `br.com.advancedbot.network.packet.PacketChatMessage` | ALTO | Documentado |
| PKT-008 | `PacketPlayerPos.cs` | `br.com.advancedbot.network.packet.PacketPlayerPos` | CRÍTICO | Documentado |
| PKT-009 | `PacketPlayerLook.cs` | `br.com.advancedbot.network.packet.PacketPlayerLook` | CRÍTICO | Documentado |
| PKT-010 | `PacketPosAndLook.cs` | `br.com.advancedbot.network.packet.PacketPosAndLook` | CRÍTICO | Documentado |
| PKT-011 | `PacketUpdate.cs` | `br.com.advancedbot.network.packet.PacketUpdate` | ALTO | Documentado |
| PKT-012 | `PacketPlayerDigging.cs` | `br.com.advancedbot.network.packet.PacketPlayerDigging` | ALTO | Documentado |
| PKT-013 | `PacketBlockPlace.cs` | `br.com.advancedbot.network.packet.PacketBlockPlace` | ALTO | Documentado |
| PKT-014 | `PacketSwingArm.cs` | `br.com.advancedbot.network.packet.PacketSwingArm` | MÉDIO | Documentado |
| PKT-015 | `PacketHeldItemChange.cs` | `br.com.advancedbot.network.packet.PacketHeldItemChange` | ALTO | Documentado |
| PKT-016 | `PacketEntityAction.cs` | `br.com.advancedbot.network.packet.PacketEntityAction` | ALTO | Documentado |
| PKT-017 | `PacketUseEntity.cs` | `br.com.advancedbot.network.packet.PacketUseEntity` | ALTO | Documentado |
| PKT-018 | `PacketClickWindow.cs` | `br.com.advancedbot.network.packet.PacketClickWindow` | ALTO | Documentado |
| PKT-019 | `PacketConfirmTransaction.cs` | `br.com.advancedbot.network.packet.PacketConfirmTransaction` | MÉDIO | Documentado |
| PKT-020 | `PacketCloseWindow.cs` | `br.com.advancedbot.network.packet.PacketCloseWindow` | MÉDIO | Documentado |
| PKT-021 | `PacketCreativeInvAction.cs` | `br.com.advancedbot.network.packet.PacketCreativeInvAction` | BAIXO | Documentado |
| PKT-022 | `PacketKeepAlive.cs` | `br.com.advancedbot.network.packet.PacketKeepAlive` | CRÍTICO | Documentado |
| PKT-023 | `PacketTeleportConfirm.cs` | `br.com.advancedbot.network.packet.PacketTeleportConfirm` | ALTO | Documentado |
| PKT-024 | `PacketUseItem.cs` | `br.com.advancedbot.network.packet.PacketUseItem` | MÉDIO | Documentado |
| PKT-025 | `UnsafeDirectPacket.cs` | `br.com.advancedbot.network.packet.RawPacket` | BAIXO | Documentado |
| PKT-026 | `DiggingStatus.cs` | `br.com.advancedbot.network.packet.DiggingStatus` | MÉDIO | Documentado |

---

## Seção 3 — Domínio do Jogo

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| DOM-001 | `Entity.cs` | `br.com.advancedbot.domain.entity.PlayerEntity` | CRÍTICO | Documentado |
| DOM-002 | `AABB.cs` | `br.com.advancedbot.domain.physics.AABB` | CRÍTICO | Documentado |
| DOM-003 | `Vec3d.cs` | `br.com.advancedbot.domain.math.Vec3d` | CRÍTICO | Documentado |
| DOM-004 | `Vec3i.cs` | `br.com.advancedbot.domain.math.Vec3i` | ALTO | Documentado |
| DOM-005 | `World.cs` | `br.com.advancedbot.domain.world.World` | CRÍTICO | Documentado |
| DOM-006 | `Chunk.cs` | `br.com.advancedbot.domain.world.Chunk` | CRÍTICO | Documentado |
| DOM-007 | `ChunkSection.cs` | `br.com.advancedbot.domain.world.ChunkSection` | ALTO | Documentado |
| DOM-008 | `BlockUtils.cs` | `br.com.advancedbot.domain.world.BlockUtils` | ALTO | Documentado |
| DOM-009 | `HitResult.cs` | `br.com.advancedbot.domain.world.HitResult` | MÉDIO | Documentado |
| DOM-010 | `SignTile.cs` | `br.com.advancedbot.domain.world.SignTile` | BAIXO | Documentado |
| DOM-011 | `Inventory.cs` | `br.com.advancedbot.domain.inventory.Inventory` | ALTO | Documentado |
| DOM-012 | `ItemStack.cs` | `br.com.advancedbot.domain.inventory.ItemStack` | ALTO | Documentado |
| DOM-013 | `Item.cs` | `br.com.advancedbot.domain.inventory.Item` | MÉDIO | Documentado |
| DOM-014 | `Items.cs` | `br.com.advancedbot.domain.inventory.Items` | MÉDIO | Documentado |
| DOM-015 | `Blocks.cs` | `br.com.advancedbot.domain.world.Blocks` | ALTO | Documentado |
| DOM-016 | `Block.cs` | `br.com.advancedbot.domain.world.Block` | ALTO | Documentado |
| DOM-017 | `Movement.cs` | `br.com.advancedbot.domain.entity.Movement` | ALTO | Documentado |
| DOM-018 | `ClientVersion.cs` | `br.com.advancedbot.domain.protocol.ClientVersion` | ALTO | Documentado |
| DOM-019 | `InventoryType.cs` | `br.com.advancedbot.domain.inventory.InventoryType` | MÉDIO | Documentado |
| DOM-020 | `ReconnectType.cs` | `br.com.advancedbot.domain.client.ReconnectType` | BAIXO | Documentado |
| DOM-021 | `IEntity.cs` | `br.com.advancedbot.domain.entity.IEntity` | MÉDIO | Documentado |
| DOM-022 | `EntityMob.cs` | `br.com.advancedbot.domain.entity.EntityMob` | MÉDIO | Documentado |
| DOM-023 | `EntityProperty.cs` | `br.com.advancedbot.domain.entity.EntityProperty` | MÉDIO | Documentado |
| DOM-024 | `EntityManager.cs` | `br.com.advancedbot.domain.entity.EntityManager` | MÉDIO | Documentado |
| DOM-025 | `MPPlayer.cs` | `br.com.advancedbot.domain.entity.OtherPlayer` | MÉDIO | Documentado |
| DOM-026 | `PlayerManager.cs` | `br.com.advancedbot.domain.player.PlayerManager` | MÉDIO | Documentado |

---

## Seção 4 — Pathfinding e Navegação

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| NAV-001 | `PathFinder.cs` | `br.com.advancedbot.navigation.PathFinder` | ALTO | Documentado |
| NAV-002 | `Path.cs` | `br.com.advancedbot.navigation.Path` | ALTO | Documentado |
| NAV-003 | `PathPoint.cs` | `br.com.advancedbot.navigation.PathPoint` | ALTO | Documentado |
| NAV-004 | `PathGuide.cs` | `br.com.advancedbot.navigation.PathGuide` | ALTO | Documentado |

---

## Seção 5 — Criptografia

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| CRY-001 | `AesStream.cs` | `br.com.advancedbot.crypto.AesStream` | BLOQUEANTE | Documentado |
| CRY-002 | `CryptoUtils.cs` | `br.com.advancedbot.crypto.CryptoUtils` | BLOQUEANTE | Documentado |
| CRY-003 | `SessionUtils.cs` | `br.com.advancedbot.auth.SessionService` | CRÍTICO | Documentado |
| CRY-004 | `LoginCache.cs` | `br.com.advancedbot.auth.LoginCache` | MÉDIO | Documentado |
| CRY-005 | `LoginResponse.cs` | `br.com.advancedbot.auth.LoginResponse` | MÉDIO | Documentado |

---

## Seção 6 — NBT

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| NBT-001 | `Tag.cs` | `br.com.advancedbot.nbt.Tag` | ALTO | Documentado |
| NBT-002 | `CompoundTag.cs` | `br.com.advancedbot.nbt.CompoundTag` | ALTO | Documentado |
| NBT-003 | `ListTag.cs` | `br.com.advancedbot.nbt.ListTag` | ALTO | Documentado |
| NBT-004 | `NbtIO.cs` | `br.com.advancedbot.nbt.NbtIO` | MÉDIO | Documentado |
| NBT-005 | Demais Tags | `br.com.advancedbot.nbt.*Tag` | MÉDIO | Documentado |

---

## Seção 7 — Comandos e Macros

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| CMD-001 | `ICommand.cs` | `br.com.advancedbot.command.ICommand` | ALTO | Documentado |
| CMD-002 | `CommandManagerNew.cs` | `br.com.advancedbot.command.CommandManager` | ALTO | Documentado |
| CMD-003 | `CommandPesca.cs` | `br.com.advancedbot.macro.FishingMacro` | CRÍTICO | Documentado |
| CMD-004 | `CommandPescaV2.cs` | `br.com.advancedbot.macro.FishingMacroV2` | CRÍTICO | Documentado |
| CMD-005 | `CommandMob.cs` | `br.com.advancedbot.macro.MobMacro` | CRÍTICO | Documentado |
| CMD-006 | `CommandMobPlus.cs` | `br.com.advancedbot.macro.MobMacroPlus` | ALTO | Documentado |
| CMD-007 | `CommandMobTeleport.cs` | `br.com.advancedbot.macro.MobTeleportMacro` | MÉDIO | Documentado |
| CMD-008 | `MacroUtils.cs` | `br.com.advancedbot.macro.MacroUtils` | ALTO | Documentado |
| CMD-009 | `CommandAntiAFK.cs` | `br.com.advancedbot.command.AntiAfkCommand` | BAIXO | Documentado |
| CMD-010 | `CommandBreakBlock.cs` | `br.com.advancedbot.command.BreakBlockCommand` | MÉDIO | Documentado |
| CMD-011 | `CommandClickBlock.cs` | `br.com.advancedbot.command.ClickBlockCommand` | MÉDIO | Documentado |
| CMD-012 | `CommandDropAll.cs` | `br.com.advancedbot.command.DropAllCommand` | BAIXO | Documentado |
| CMD-013 | `CommandFollow.cs` | `br.com.advancedbot.command.FollowCommand` | MÉDIO | Documentado |
| CMD-014 | `CommandGoto.cs` | `br.com.advancedbot.command.GotoCommand` | MÉDIO | Documentado |
| CMD-015 | `CommandKillAura.cs` | `br.com.advancedbot.command.KillAuraCommand` | MÉDIO | Documentado |
| CMD-016 | `CommandMiner.cs` | `br.com.advancedbot.command.MinerCommand` | MÉDIO | Documentado |
| CMD-017 | `CommandPortal.cs` | `br.com.advancedbot.command.PortalCommand` | MÉDIO | Documentado |
| CMD-018 | `CommandScript.cs` | `br.com.advancedbot.command.ScriptCommand` | MÉDIO | Documentado |
| CMD-019 | `CommandReco.cs` | `br.com.advancedbot.command.ReconnectCommand` | BAIXO | Documentado |
| CMD-020 | `CommandSneak.cs` | `br.com.advancedbot.command.SneakCommand` | BAIXO | Documentado |
| CMD-021 | `CommandUseBow.cs` | `br.com.advancedbot.command.UseBowCommand` | MÉDIO | Documentado |
| CMD-022 | `CommandUseEntity.cs` | `br.com.advancedbot.command.UseEntityCommand` | MÉDIO | Documentado |
| CMD-023 | Outros Comandos | `br.com.advancedbot.command.*Command` | BAIXO | Documentado |

---

## Seção 8 — Plugins e Scripts

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| PLG-001 | `IPlugin.cs` | `br.com.advancedbot.plugin.IPlugin` | MÉDIO | Documentado |
| PLG-002 | `PluginManager.cs` | `br.com.advancedbot.plugin.PluginManager` | MÉDIO | Documentado |
| PLG-003 | `AdvancedBotAPI.cs` | `br.com.advancedbot.plugin.AdvancedBotAPI` | MÉDIO | Documentado |
| SCR-001 | `ScriptParser.cs` | `br.com.advancedbot.script.ScriptParser` | BAIXO | Documentado |
| SCR-002 | `ScriptContext.cs` | `br.com.advancedbot.script.ScriptContext` | BAIXO | Documentado |
| SCR-003 | `JsScriptContext.cs` | `br.com.advancedbot.script.JsScriptContext` | BAIXO | Documentado |

---

## Seção 9 — Utilitários

| ID | Origem C# | Destino Java (proposto) | Criticidade | Status |
|----|-----------|------------------------|-------------|--------|
| UTL-001 | `Utils.cs` | `br.com.advancedbot.util.Utils` | ALTO | Documentado |
| UTL-002 | `ChatParser.cs` | `br.com.advancedbot.util.ChatParser` | ALTO | Documentado |
| UTL-003 | `DiggingHelper.cs` | `br.com.advancedbot.util.DiggingHelper` | MÉDIO | Documentado |
| UTL-004 | `LookInterpolator.cs` | `br.com.advancedbot.util.LookInterpolator` | BAIXO | Documentado |
| UTL-005 | `UUID.cs` | `java.util.UUID` (nativo) | BAIXO | Documentado |
| UTL-006 | `HttpConnection.cs` | `java.net.http.HttpClient` (nativo) | MÉDIO | Documentado |
| UTL-007 | `ProxyUtils.cs` | `br.com.advancedbot.proxy.ProxyUtils` | MÉDIO | Documentado |
| UTL-008 | `NickGenerator.cs` | `br.com.advancedbot.util.NickGenerator` | BAIXO | Documentado |
| UTL-009 | `SrvResolver.cs` | `br.com.advancedbot.util.SrvResolver` | MÉDIO | Documentado |

---

## Seção 10 — Componentes Descartados (Não Migrar)

| ID | Origem C# | Motivo | Criticidade |
|----|-----------|--------|-------------|
| DES-001 | `ViewForm.cs` | Interface GUI Windows Forms | N/A |
| DES-002 | `ChunkRenderer.cs` | Renderizador OpenGL Windows | N/A |
| DES-003 | `WorldRenderer.cs` | Renderizador OpenGL Windows | N/A |
| DES-004 | `GL.cs` / `WGL.cs` | P/Invoke OpenGL Windows | N/A |
| DES-005 | `Font.cs` (Viewer) | Renderização de texto OpenGL | N/A |
| DES-006 | `Tessellator.cs` | Geometria OpenGL | N/A |
| DES-007 | `VBO.cs` | Vertex Buffer Object OpenGL | N/A |
| DES-008 | `TextureManager.cs` | Texturas OpenGL | N/A |
| DES-009 | `AsyncChunkBuilder.cs` | Builder OpenGL | N/A |
| DES-010 | `GuiCheckBox.cs` | GUI OpenGL | N/A |
| DES-011 | `GuiTrackBar.cs` | GUI OpenGL | N/A |
| DES-012 | `GuiOptions.cs` | GUI OpenGL | N/A |
| DES-013 | `Main.cs` | Formulário Windows Forms | N/A |
| DES-014 | `Start.cs` | Formulário Windows Forms | N/A |
| DES-015 | `MacroEditor.cs` | Editor Windows Forms | N/A |
| DES-016 | `Spammer.cs` | Formulário Windows Forms | N/A |
| DES-017 | `AccountChecker.cs` | Formulário Windows Forms | N/A |
| DES-018 | `BanCheck.cs` | Formulário Windows Forms | N/A |
| DES-019 | `ProxyCheckerForm.cs` | Formulário Windows Forms | N/A |
| DES-020 | `ProxyForm.cs` | Formulário Windows Forms | N/A |
| DES-021 | `Statistics.cs` | Formulário Windows Forms | N/A |
| DES-022 | `TestServer.cs` | Formulário Windows Forms | N/A |
| DES-023 | `FastColoredTextBoxNS/` | Componente WinForms editor | N/A |
| DES-024 | `Transitions.dll` | Animações WinForms | N/A |
| DES-025 | `HWIDChecker.cs` | Verificação HWID Windows (WMI) | N/A |
| DES-026 | `SkySurvival.cs` | Bypass servidor específico | BAIXO |
| DES-027 | `WorldCraftBP.cs` | Bypass servidor específico | BAIXO |

---

## Resumo Quantitativo

| Categoria | Total de Itens |
|-----------|---------------|
| Rede e Protocolo | 15 |
| Pacotes | 26 |
| Domínio do Jogo | 26 |
| Pathfinding | 4 |
| Criptografia e Auth | 5 |
| NBT | 5 |
| Comandos e Macros | 23 |
| Plugins e Scripts | 6 |
| Utilitários | 9 |
| **Subtotal a migrar** | **119** |
| Descartados | 27 |
| **Total geral** | **146** |

---

## Decisões Arquiteturais Pendentes

As seguintes decisões precisam ser tomadas antes de iniciar a migração:

| Código | Decisão | Impacto |
|--------|---------|---------|
| DEC-01 | Quais versões de protocolo serão suportadas na v1.0 Java? | Escopo de implementação dos Handlers |
| DEC-02 | Qual será a estratégia de interface do usuário Java? | Descarte ou substituição da UI |
| DEC-03 | Qual será o modelo de macros Java (async/await substituto)? | Arquitetura de todos os macros |
| DEC-04 | AES-CFB8 com provider nativo ou BouncyCastle? | Dependência crítica de rede |
| DEC-05 | O arquivo `conf.dat` (NBT) será mantido ou migrado para JSON/YAML? | Configuração |
| DEC-06 | Qual será o formato de plugins Java (.jar com interface)? | Sistema de extensibilidade |
| DEC-07 | O servidor alvo primário é 1.8 ou outro protocolo? | Prioridade de implementação dos Handlers |
| DEC-08 | Framework base: Spring Boot, Quarkus ou Java puro? | Arquitetura global do projeto Java |
