# Inventário de código próprio

## Núcleo e cliente

`Program`, `Main`, `Start`, formulários auxiliares, `MinecraftClient`, `PacketStream`, `PacketQueue`, `MinecraftStream`, `ReadBuffer`, `WriteBuffer`, `ClientVersion`, `IPacket`, `Utils`, `Statistics`, `SessionUtils`, `LoginCache`, `LoginResponse`, `Proxy`, `ProxyType`, `HttpConnection`, `HttpResponse`, `UUID`, `Vec3d`, `Vec3i`, `AABB`, `Movement`, `LookInterpolator`, `AutoMiner`, `DiggingHelper`, `ChatParser`.

## Mundo/entidades/itens

`Map/{World,Chunk,ChunkSection,BlockUtils,HitResult,SignTile}`, `Entity`, `MPPlayer`, `PlayerManager`, `PlayerNick`, `PlayerTab`, `Entitybase/{IEntity,EntityMob,EntityProperty,EntityManager}`, `Blocks`, `Block`, `Items`, `Item`, `ItemStack`, `Inventory`, `InventoryType`, `PathFinding/{Path,PathPoint,PathFinder,PathGuide}`.

## Protocolos/dados

`Handler/{ProtocolHandler,Handler_v152,Handler_v17,Handler_v18,Handler_v19,PacketStreamLegacy}`, todos os 26 `Packets/Packet*.cs`, `Crypto/{AesStream,CryptoUtils}`, `NBT/{Tag,NbtIO,DataInput,DataOutput,ByteTag,ShortTag,IntTag,LongTag,FloatTag,DoubleTag,StringTag,ByteArrayTag,IntArrayTag,ListTag,CompoundTag,EndTag}`.

## Extensões e automação

`Commands/ICommand`, `CommandManagerNew`, 27 comandos gerais e `Solk/{CommandPesca,CommandPescaV2,CommandMob,CommandMobPlus,CommandMobTeleport,ConfigMob,MacroUtils,FileLock}`, `Script/{ScriptParser,ScriptContext,JsScriptContext,Token,TokenType,Function,Variable,Expression,ParserException,ScriptException}`, `Plugins/{IPlugin,PluginManager,AdvancedBotAPI}`.

## Periferia

`ProxyChecker/*`, `Client.Bypassing/*`, `Crypto/*`, `Protection/HWIDChecker`, `Viewer/*`, `Viewer.Gui/*`, propriedades/resources e formulários. Bibliotecas vendorizadas (`Jint*`, `Ionic*`, `FastColoredTextBoxNS`, `MaxMind*`) devem ser substituídas por dependências, não reescritas.
