# Protocolo Minecraft

`ProtocolHandler.Create` escolhe `Handler_v152`, `Handler_v17`, `Handler_v18` ou `Handler_v19`. Cada handler decodifica IDs de entrada e serializa `IPacket` de saída para sua versão. `PacketStreamLegacy` implementa o formato 1.5.2; versões modernas usam VarInt e `PacketStream`.

Pacotes de saída próprios em `AdvancedBot.Client.Packets`: handshake/login, chat, keep-alive, posição/rotação, ações de entidade, dig/place/use, janelas/inventário, plugin message, confirmação e criptografia. `UnsafeDirectPacket` é escape hatch de bytes crus.

## Preservar e testar

- Seleção de versão e IDs por estado (handshake/login/play).
- Framing, comprimento, VarInt, strings UTF-8, UUID, NBT, compressão e AES.
- Decodificação de chunk, entidades, inventário, chat, keep-alive e teleporte.
- Não transportar IDs codificados para domínio; usar `PacketModel` canônico + adaptador por versão.

Criar fixtures binárias capturadas/anonimizadas por versão antes de reimplementar codecs.
