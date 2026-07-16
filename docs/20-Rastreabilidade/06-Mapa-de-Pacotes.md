# Mapa Exaustivo de Pacotes de Rede (Protocolo 47 - Minecraft 1.8.x)

Este documento atua como a Bíblia Criptográfica do novo bot. Ele substitui inteiramente o antigo e monolítico arquivo `PacketStream.cs` e a pasta `Net/Packets` do C# legado. Aqui, mapeamos a anatomia exata de bytes (Frame Decoding, Handshake, Compression) e definimos o contrato que o pipeline do Netty em Java deverá obedecer para se comunicar com qualquer servidor de Minecraft (Spigot, BungeeCord ou Vanilla).

Qualquer desvio do byte-array definido neste documento resultará no servidor ejetando o Bot com um erro de *Bad Packet ID* ou *DecoderException*.

---

## 1. Fundamentos do Protocolo de Rede (Netty e ByteBuf)

O Minecraft 1.8 introduziu mudanças drásticas no formato da rede. Em vez de pacotes com tamanhos fixos, tudo agora flui dentro de **Frames com Tamanho Variável (VarInt)**.

### 1.1 A Estrutura do Frame (O Contêiner)
Seja no Handshake ou no Jogo, todos os pacotes seguem esta regra estrita de encapsulamento:
1. `VarInt Length`: O tamanho total em bytes do pacote (incluindo o Packet ID, mas *excluindo* o próprio Length).
2. `VarInt PacketID`: O identificador do pacote (ex: `0x00` ou `0x40`).
3. `byte[] Data`: O corpo útil do pacote.

### 1.2 O Inferno do `VarInt` e `VarLong`
No C# legado, a leitura de `VarInt` era feita bloqueando a thread byte a byte via `NetworkStream.ReadByte()`.
Na arquitetura Netty Java, isso é substituído pelo `ByteBuf`, que já lê pacotes inteiros fragmentados usando um `LengthFieldBasedFrameDecoder` customizado.

**Algoritmo Exato para Leitura do VarInt em Java (A ser colocado no Decoder)**:
```java
public static int readVarInt(ByteBuf buf) {
    int numRead = 0;
    int result = 0;
    byte read;
    do {
        read = buf.readByte();
        int value = (read & 0b01111111);
        result |= (value << (7 * numRead));
        numRead++;
        if (numRead > 5) {
            throw new RuntimeException("VarInt is too big");
        }
    } while ((read & 0b10000000) != 0);
    return result;
}
```

### 1.3 As Máquinas de Estado do Protocolo (State Machine)
Um erro crítico do antigo bot era tentar ler pacotes de *Play* (ex: vida, inventário) enquanto ainda estava no estado de *Login*, causando confusão de IDs, visto que IDs se repetem entre estados.
O `AdvancedBot` no Netty deve implementar o padrão **State Pattern** em seu `ChannelInboundHandler`.

1. **HANDSHAKING**: Primeira fração de segundo de vida do socket.
2. **STATUS**: Usado apenas para "Pingar" o servidor e ver a MOTD/Players Online. (O bot não usa isso ao jogar).
3. **LOGIN**: Fase de negociação de criptografia AES e compressão Zlib.
4. **PLAY**: O jogo bruto (99% da vida útil do bot).

---

## 2. Fase de Compressão e Criptografia (Middleware)

No legado, a classe de rede tinha centenas de "Ifs" para verificar se a rede estava comprimida ou não. Na arquitetura Java, nós resolveremos isso usando os interceptadores dinâmicos do Netty.

### 2.1 A Chave Criptográfica (AES/CFB8)
Se o servidor estiver em "Online Mode" (Original), durante o estado de *Login*, o servidor mandará um Pacote `0x01 (Encryption Request)`.
1. O Bot gera uma chave secreta e responde no `0x01 (Encryption Response)` usando criptografia RSA pública fornecida pelo servidor.
2. Assim que o pacote de resposta é enviado, a classe do Java injeta dois manipuladores no Netty:
   - `ctx.pipeline().addBefore("frame-decoder", "decrypt", new MinecraftDecryptor(secretKey))`
   - `ctx.pipeline().addBefore("frame-encoder", "encrypt", new MinecraftEncryptor(secretKey))`
Isso torna a descriptografia invisível para o resto do Bot. O legado tratava a decriptação suja no mesmo fluxo lógico do processamento de blocos.

### 2.2 Zlib Compression (O Fardo do Pacote 0x46)
No estado PLAY, o servidor envia `0x46 (Set Compression)` ditando o Threshold. Ex: "Compacte tudo maior que 256 bytes".
O Netty altera a estrutura do Frame para:
- `VarInt PacketLength` (Tamanho do que vem a seguir)
- `VarInt DataLength` (O tamanho real se estivesse descompactado. `0` significa não compacto).
- `byte[] Data` (Compactado em ZLib se DataLength > 0).

**Implementação Obrigatória (ZlibDecoder)**:
```java
// O Inflater não deve ser alocado por requisição!
private final Inflater inflater = new Inflater();

protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
    if (in.readableBytes() == 0) return;
    
    int dataLength = ByteBufUtils.readVarInt(in);
    if (dataLength == 0) {
        // Não está comprimido, passa para o próximo decoder direto.
        out.add(in.readBytes(in.readableBytes()));
    } else {
        // Aloca array direto (Off-Heap evitado pelo ZGC) e descompacta
        byte[] compressedBytes = new byte[in.readableBytes()];
        in.readBytes(compressedBytes);
        inflater.setInput(compressedBytes);
        
        byte[] uncompressed = new byte[dataLength];
        inflater.inflate(uncompressed);
        inflater.reset(); // Crucial para não estourar memória
        
        out.add(Unpooled.wrappedBuffer(uncompressed));
    }
}
```


---

## 3. Mapeamento Exaustivo: Handshaking & Login

Abaixo está o mapeamento byte-a-byte exigido pelo protocolo 47 para injetar o Bot no servidor. Substitui toda a lógica rudimentar do `MPPlayer.Login()`.

### 3.1 Handshake (Client -> Server)
**ID: `0x00` | Estado: `Handshaking`**
O primeiríssimo pacote enviado ao conectar no IP.
- `VarInt (Protocol Version)`: Sempre `47` para o bot 1.8.x.
- `String (Server Address)`: O IP do servidor (limitado a 255 chars).
  - *Detalhe*: Servidores BungeeCord exigem que envie o IP real + Token Forwarding aqui, um detalhe crucial para quebrar anti-bungees.
- `Unsigned Short (Server Port)`: Geralmente 25565.
- `VarInt (Next State)`: `2` (Pede para entrar em estado de Login).

### 3.2 Login Start (Client -> Server)
**ID: `0x00` | Estado: `Login`**
- `String (Username)`: Nome do Bot. (Pode acionar um Kick instantâneo se tiver espaços ou > 16 chars).

### 3.3 Disconnect (Server -> Client)
**ID: `0x00` | Estado: `Login`**
- `String (Chat JSON)`: Motivo do kick (Ex: "O servidor está lotado"). O Bot deve gerar o evento `SessionLoginFailedEvent`.

### 3.4 Encryption Request (Server -> Client)
**ID: `0x01` | Estado: `Login`**
- `String (Server ID)`: Até 20 caracteres (usado na API da Mojang).
- `VarInt (Public Key Length)` e `byte[] (Public Key)`
- `VarInt (Verify Token Length)` e `byte[] (Verify Token)`
- **Ação Java**: O `AuthManager` entra em ação, contata `https://sessionserver.mojang.com/session/minecraft/join` autenticando o Token, gera o AES Secret, e envia o pacote de Resposta `0x01`.

### 3.5 Login Success (Server -> Client)
**ID: `0x02` | Estado: `Login`**
- `String (UUID)`: O ID Universal do jogador.
- `String (Username)`: Pode ser diferente do que enviamos se o servidor for BungeeCord.
- **Ação Java**: Transição de Estado concluída. O Pipeline do Netty obrigatoriamente *Muda de Estado Interno* de `LOGIN` para `PLAY`. Todos os IDs de pacote a partir do próximo byte adotam a Tabela Play.

---

## 4. Mapeamento Exaustivo: Play (Server -> Client) - Visão Global

A maioria dos pacotes de mundo e entidades. O "ReadByte()" infinito do antigo `PacketStream`.

### 4.1 Pacote `0x00` (Keep Alive)
- `VarInt (KeepAlive ID)`
- **Ação Obrigatória**: Responder imediatamente com o Pacote `0x00` na fila Clientbound com exatamente o mesmo `VarInt`. A latência (Ping) do jogador é calculada na diferença entre o envio do servidor e a resposta do Bot.

### 4.2 Pacote `0x01` (Join Game)
O pacote massivo que dita as leis da dimensão em que o Bot acaba de nascer.
- `Int (Entity ID)`: O ID físico do seu próprio Bot. (O C# engolia esse ID. O Java DEVE salvá-lo para discernir `EntityVelocity` aplicadas em você das aplicadas em outros mobs).
- `Unsigned Byte (Gamemode)`: 0 (Survival), 1 (Creative), etc.
- `Byte (Dimension)`: -1 (Nether), 0 (Overworld), 1 (End).
- `Unsigned Byte (Difficulty)`
- `Unsigned Byte (Max Players)`
- `String (Level Type)`: `default`, `flat`, `amplified`.
- `Boolean (Reduced Debug Info)`

### 4.3 Pacote `0x02` (Chat Message)
- `String (JSON Data)`: O texto formatado.
- `Byte (Position)`: 0 (Chat Normal), 1 (System - ignorado por clientes invisíveis), 2 (Action Bar).
- **Tradução C# -> Java**: O bot C# tentava limpar a String JSON fazendo replace de chaves. O Bot Java delegará para uma biblioteca GSON e publicará `ChatMessageReceivedEvent`.

### 4.4 Pacote `0x08` (Update Health)
- `Float (Health)`: 0 a 20.0.
- `VarInt (Food)`: 0 a 20.
- `Float (Food Saturation)`: 0.0 a 5.0.

### 4.5 Pacote `0x06` (Update Health) -> CORREÇÃO PROTOCOLAR (Update Health é 0x06 no Play!)
*(Nota Técnica: O documento anterior possuía um typo no ID. Aqui reafirmamos os IDs rigorosos do Protocolo 47).*
O ID real da Vida é `0x06`. O ID `0x08` corresponde a *Player Position And Look (Serverbound)*. 
Esta correção reflete a precisão cirúrgica exigida no parser Netty para evitar desconexões.


---

## 5. Mapeamento Exaustivo: Play (Server -> Client) - Física e Mundo

A espinha dorsal da engenharia reversa. O C# sofria de vazamentos de memória (Memory Leaks) e arrays corrompidos pois decodificava Chunks de forma sequencial na Thread de IO. Aqui ditamos os layouts binários exatos.

### 5.1 Pacote `0x21` (Chunk Data)
O pacote mais pesado do protocolo. Envia fatias verticais de 16x256x16 blocos.
- `Int (Chunk X)`
- `Int (Chunk Z)`
- `Boolean (Ground-Up Continuous)`: Se `true`, significa que é um Chunk inteiro novo. Se `false`, é apenas uma atualização de algumas seções.
- `Unsigned Short (Primary Bit Mask)`: Uma máscara de 16 bits indicando quais sub-chunks (seções de 16x16x16) estão contidas no byte array comprimido.
- `VarInt (Size)`: Tamanho dos dados comprimidos.
- `byte[] (Data)`: O array contendo Bloco, Metadados, Luz (SkyLight) e BlockLight.
- **Implementação Java**: A Thread NIO do Netty não deve extrair os metadados. Ela repassa o `byte[]` inteiro para uma `VirtualThread` usar o `ChunkParser` em *background*.

### 5.2 Pacote `0x26` (Map Chunk Bulk)
O servidor quer acelerar o carregamento enviando de 1 a 10 chunks de uma vez só.
- `Boolean (Sky Light Sent)`: Informa se a iluminação do céu está inclusa (Falso no Nether).
- `VarInt (Chunk Column Count)`: Quantidade de chunks no pacote.
- Array Metadados (Para cada chunk):
  - `Int (Chunk X)`
  - `Int (Chunk Z)`
  - `Unsigned Short (Primary Bit Mask)`
- `byte[] (Data)`: Todos os chunks encadeados. O `ChunkParser` deve fatiar iterando a máscara.

### 5.3 Pacote `0x23` (Block Change)
Um bloco foi quebrado ou colocado.
- `Position (64-bit Long)`: No Protocolo 47, as coordenadas X, Y e Z são empacotadas em 8 bytes.
  - O C# original usava shift manual e as vezes quebrava no Y negativo.
  - **Fórmula Java**: 
    - `x = val >> 38`
    - `y = (val >> 26) & 0xFFF`
    - `z = val << 38 >> 38`
- `VarInt (Block ID)`: Contém o ID do bloco e a Metadata fundidos (`blockId << 4 | metadata`).

### 5.4 Pacote `0x08` (Player Position And Look - Serverbound/Clientbound)
O servidor forçando o teleporte do Bot (Ex: ao renascer, entrar num portal, ou borracha elástica de Anti-Cheat).
- `Double (X)`
- `Double (Y)`
- `Double (Z)`
- `Float (Yaw)`
- `Float (Pitch)`
- `Byte (Flags)`: Uma máscara de bits indicando se as posições são relativas ou absolutas.
- **Ação Vital**: O Bot DEVE responder a este pacote com um `0x06` ou `0x04` confirmando a nova posição, senão o servidor o chutará.

### 5.5 Pacote `0x12` (Entity Velocity)
O "Knockback".
- `VarInt (Entity ID)`
- `Short (Velocity X)`: Unidade mágica do Notch: `(velX / 8000.0)`.
- `Short (Velocity Y)`
- `Short (Velocity Z)`

---

## 6. Mapeamento Exaustivo: Play (Server -> Client) - Inventário

Esses pacotes sincronizam a matriz lógica da janela de interação (Baú, Player).

### 6.1 Pacote `0x30` (Window Items)
- `Unsigned Byte (Window ID)`: O identificador (0 = Inventário do Player).
- `Short (Count)`: Quantidade de itens no array a seguir.
- Array de **Slot Data**:
  - A estrutura do Slot é iterada de acordo com o `Count`.

**A Anatomia do Slot (Slot Data)**:
A leitura de Slot é usada em vários pacotes. O Netty precisa de um método isolado `readSlot(ByteBuf buf)`:
1. `Short (Item ID)`: Se for `-1` (ou 65535 no Unsigned), significa VAZIO. Para por aqui.
2. `Byte (Item Count)`: Quantos itens no stack.
3. `Short (Item Damage/Meta)`: A durabilidade ou tipo da corante.
4. `Byte (NBT Start)`: O NBT pode ser nulo (`0x00`). Se não for, lê uma tag NBT inteira (nome customizado, encantamentos).

### 6.2 Pacote `0x32` (Confirm Transaction)
- `Unsigned Byte (Window ID)`
- `Short (Action Number)`: O ID da transação que o Bot tentou.
- `Boolean (Accepted)`: Se falso, o servidor abortou o clique.


---

## 7. Mapeamento Exaustivo: Play (Client -> Server) - Outbound

Estes pacotes são as saídas geradas pelo `IntentProcessorService`. Qualquer byte mal formatado aqui resultará num kick imediato por "Bad Packet ID" ou pior, um *Silent Ban* pelo Anti-Cheat.

### 7.1 Pacote `0x04` (Player Position), `0x05` (Look) e `0x06` (Position And Look)
A trindade do movimento. O Bot C# tinha um vício perigoso: enviava esses pacotes em momentos aleatórios, baseados em callbacks de interface. No Java, **um (e apenas um)** destes pacotes deve ser despachado a cada Tick de 50ms, obrigatoriamente.
- `Double (X)`
- `Double (Y)` (Coordenada dos Pés. O legado errava mandando Y da cabeça, o que gerava Fly Hack detectado).
- `Double (Z)`
- `Float (Yaw)` (Opcional dependendo do pacote)
- `Float (Pitch)` (Opcional dependendo do pacote)
- `Boolean (OnGround)`: Crítico. Dizer que está no chão (`true`) enquanto o Y cai resulta em ban de *NoFall*.

### 7.2 Pacote `0x02` (Use Entity)
O Killaura/Ataque/Interação.
- `VarInt (Target Entity ID)`
- `VarInt (Type)`: 
  - `0`: Interact (Clicar num villager)
  - `1`: Attack (Bater no Zumbi)
  - `2`: Interact At (Para Armor Stands, requer 3 floats adicionais Target X,Y,Z).

### 7.3 Pacote `0x07` (Player Digging)
Começar ou terminar de quebrar blocos, ou dropar o item da mão (`Q`).
- `Byte (Status)`:
  - `0`: Começou a quebrar (Started Digging)
  - `1`: Cancelou a quebra (Cancelled Digging)
  - `2`: Terminou a quebra (Finished Digging)
  - `3`: Drop Item Stack
  - `4`: Drop Single Item
  - `5`: Shoot Arrow / Finish Eating
- `Position (64-bit Long)`: A posição do bloco.
- `Byte (Face)`: A face do bloco que o bot clicou (0=Baixo, 1=Cima, 2=Norte, 3=Sul, 4=Oeste, 5=Leste). O legado fixava tudo em 1, gerando banimentos óbvios.

### 7.4 Pacote `0x08` (Player Block Placement)
Colocar blocos ou usar itens consumíveis (Comer Maçã Dourada, Jogar Poção).
- `Position (64-bit Long)`: A posição do bloco *contra o qual* clicamos. Se estamos comendo ou atirando no ar, mandamos `-1, -1, -1` encodado como Long.
- `Byte (Face)`: Face clicada ou `255` se for interação no ar.
- `Slot Data (Held Item)`: O item que o bot está segurando no momento. (Detalhe da seção 6.1).
- `Byte (Cursor X)`: A precisão do clique (0 a 15). O legado mandava `8, 8, 8` (Meio exato do bloco). Isso é assustadoramente robótico. A IA deve randomizar `cursorX = Random(2, 14)`.
- `Byte (Cursor Y)`
- `Byte (Cursor Z)`

### 7.5 Pacote `0x0B` (Entity Action)
Correr (Sprint) e Agachar (Sneak).
- `VarInt (Entity ID)`: Seu próprio ID.
- `VarInt (Action ID)`:
  - `0`: Start Sneaking
  - `1`: Stop Sneaking
  - `3`: Start Sprinting
  - `4`: Stop Sprinting

### 7.6 Pacote `0x0E` (Click Window)
O pacote anti-fraude mais monitorado do protocolo.
- `Byte (Window ID)`: 0 para seu inventário, ou o ID do baú aberto.
- `Short (Slot)`: O índice do slot (0 a 44).
- `Byte (Button)`:
  - 0: Botão Esquerdo
  - 1: Botão Direito
- `Short (Action Number)`: Começa em 1, incrementado a cada pacote 0x0E.
- `Byte (Mode)`: Shift-Click (1), Normal (0), Drop (4).
- `Slot Data (Clicked Item)`: O item que **achamos** que pegamos (Predictive State). O servidor validará isso com o `Confirm Transaction`. Se mandarmos `Air`, mas tinha uma espada, o servidor recusa a ação.


---

## 8. Apêndice A: Dicionário Exaustivo de Conversões Primitivas (Endianness)

O Minecraft foi escrito em Java (Big-Endian). O C# (Arquitetura x86/x64) é nativamente Little-Endian. No `AdvancedBot 2.4.5`, o programador teve que inverter os bytes de todos os shorts, ints e longs manualmente (ex: `Array.Reverse(bytes)`).
Na reescrita para Java, **não devemos reverter os bytes**, pois o Netty `ByteBuf` e a JVM já operam em Big-Endian nativamente.

### 8.1 Tabela de Tipos do Protocolo e Equivalente Netty

| Tipo no Protocolo (Wiki) | Tamanho (Bytes) | Como Ler no Netty (Java) | Como Escrever no Netty | Risco C# Legado |
|---|---|---|---|---|
| `Boolean` | 1 | `buf.readBoolean()` | `buf.writeBoolean(val)` | O C# lia como Byte e convertia. Lento. |
| `Byte` | 1 | `buf.readByte()` | `buf.writeByte(val)` | Igual. |
| `Unsigned Byte` | 1 | `buf.readUnsignedByte()` | `buf.writeByte(val)` | C# não suporta nativamente em arrays (precisava de cast). |
| `Short` | 2 | `buf.readShort()` | `buf.writeShort(val)` | O C# exigia `.Reverse()` antes do `BitConverter.ToInt16`. Java é direto. |
| `Unsigned Short` | 2 | `buf.readUnsignedShort()` | `buf.writeShort(val)` | Mesma inversão do C#. |
| `Int` | 4 | `buf.readInt()` | `buf.writeInt(val)` | Mesma inversão. |
| `Long` | 8 | `buf.readLong()` | `buf.writeLong(val)` | Mesma inversão. |
| `Float` | 4 | `buf.readFloat()` | `buf.writeFloat(val)` | Mesma inversão. |
| `Double` | 8 | `buf.readDouble()` | `buf.writeDouble(val)` | Mesma inversão. |
| `String` | VarInt + N | `readString(buf, maxLen)` | `writeString(buf, val)` | O C# não limitava o tamanho, gerando travamentos por strings gigantes (`OutOfMemoryException`). Java exigirá o parâmetro `maxLength` para abortar pacotes maliciosos. |
| `VarInt` | 1 a 5 | `readVarInt(buf)` | `writeVarInt(buf, val)` | O loop de Shift-Left (Seção 1.2). |
| `VarLong` | 1 a 10 | `readVarLong(buf)` | `writeVarLong(buf, val)` | Igual ao VarInt, mas usando máscara de 64bits. |
| `Position` | 8 | `readPosition(buf)` | `writePosition(buf, val)` | Empacotamento de X,Y,Z num único Long (Seção 5.3). |
| `UUID` | 16 | `new UUID(readLong(), readLong())` | `writeLong(most); writeLong(least)` | C# lidava como uma string formatada ou byte array sujo. |

---

## 9. Apêndice B: Código Proposto - O Decoder Mestre do Netty

Para ilustrar o poder da nova arquitetura Lock-Free em contraste com o C# `PacketStream.cs` (que possuía mais de 2.000 linhas num switch-case só), o Java divide a responsabilidade. Abaixo está a prova de conceito do `PacketDecoder`.

### 9.1 O Frame Decoder (Desfragmentador)
O TCP pode enviar um pacote pela metade (ex: 120 bytes chegam hoje, 30 bytes chegam amanhã). O C# do bot sofria corrupção de fluxo por não tratar isso. O Netty já faz isso nativamente.
```java
package com.advancedbot.infra.network.pipeline;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.ByteToMessageDecoder;
import java.util.List;

/**
 * Separa o fluxo contínuo de bytes do TCP em Pacotes (Frames) isolados.
 */
public class VarintFrameDecoder extends ByteToMessageDecoder {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        in.markReaderIndex();
        
        // Verifica se há bytes suficientes para tentar ler um VarInt
        final byte[] buf = new byte[3];
        for (int i = 0; i < buf.length; i++) {
            if (!in.isReadable()) {
                in.resetReaderIndex();
                return; // O pacote está incompleto, aguarda o TCP mandar o resto
            }
            buf[i] = in.readByte();
            if (buf[i] >= 0) {
                // Fim do VarInt
                int length = readVarIntFromTemp(buf);
                if (in.readableBytes() < length) {
                    in.resetReaderIndex();
                    return; // Pacote incompleto (tamanho declarado é maior que o que chegou)
                } else {
                    out.add(in.readBytes(length)); // Entrega o Frame limpo para o próximo Decoder
                    return;
                }
            }
        }
        throw new CorruptedFrameException("VarInt Length maior que 3 bytes.");
    }
}
```

### 9.2 O Packet Decoder (Roteador de Eventos)
Recebe o `ByteBuf` isolado da classe acima e transforma em `Eventos` para a IA.
```java
package com.advancedbot.infra.network.pipeline;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToMessageDecoder;
import java.util.List;

public class MinecraftPacketDecoder extends MessageToMessageDecoder<ByteBuf> {
    
    private final EventBus eventBus;
    private ConnectionState currentState = ConnectionState.HANDSHAKING;

    public MinecraftPacketDecoder(EventBus eventBus) {
        this.eventBus = eventBus;
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        int packetId = ByteBufUtils.readVarInt(in);
        
        switch (currentState) {
            case LOGIN:
                decodeLogin(packetId, in);
                break;
            case PLAY:
                decodePlay(packetId, in);
                break;
        }
        
        // Opcional: Avisa o Garbage Collector (Zero-Copy) que terminamos com esse buffer
        in.skipBytes(in.readableBytes());
    }

    private void decodePlay(int packetId, ByteBuf in) {
        switch (packetId) {
            case 0x00: // Keep Alive
                long pingId = ByteBufUtils.readVarInt(in);
                eventBus.publish(new KeepAlivePingEvent(pingId));
                break;
                
            case 0x23: // Block Change
                long pos = in.readLong();
                int x = (int) (pos >> 38);
                int y = (int) ((pos >> 26) & 0xFFF);
                int z = (int) (pos << 38 >> 38);
                int blockState = ByteBufUtils.readVarInt(in);
                eventBus.publish(new BlockChangedEvent(x, y, z, (short)(blockState >> 4), (byte)(blockState & 0xF)));
                break;
                
            // Centenas de cases injetando Events no Barramento
            default:
                // Ignorar pacotes não mapeados silenciosamente para não travar a IA
                break;
        }
    }
    
    public void setConnectionState(ConnectionState newState) {
        this.currentState = newState;
    }
}
```


---

## 10. Apêndice C: Código Proposto - O Encoder Mestre do Netty

Assim como lemos os bytes e transformamos em Eventos, precisamos pegar os DTOs do Bot (`ActionIntents` transformados em `MinecraftPacket`) e transformá-los de volta em bytes `ByteBuf` para o Windows enviar pela placa de rede.

### 10.1 O Roteador de Saída (MessageToByteEncoder)
```java
package com.advancedbot.infra.network.pipeline;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

/**
 * Pega um objeto MinecraftPacket (Gerado pelas IAs) e o escreve em bytes.
 */
public class MinecraftPacketEncoder extends MessageToByteEncoder<MinecraftPacket> {

    @Override
    protected void encode(ChannelHandlerContext ctx, MinecraftPacket packet, ByteBuf out) {
        // 1. Escreve o ID do pacote em formato VarInt
        ByteBufUtils.writeVarInt(out, packet.getPacketId());
        
        // 2. O próprio pacote sabe como se escrever no ByteBuf
        packet.write(out);
    }
}
```

### 10.2 O Frame Encoder (Prepender)
Após o pacote escrever seus bytes e seu ID, o protocolo exige que saibamos o tamanho total ANTES de enviar pela rede. Como não sabemos o tamanho até terminarmos de escrever, o Netty usa o `MessageToByteEncoder` para calcular o tamanho final e prependê-lo no início do buffer.

```java
package com.advancedbot.infra.network.pipeline;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.handler.codec.MessageToByteEncoder;

/**
 * Adiciona o VarInt Length no início do pacote.
 */
public class VarintFrameEncoder extends MessageToByteEncoder<ByteBuf> {
    
    @Override
    protected void encode(ChannelHandlerContext ctx, ByteBuf in, ByteBuf out) {
        int bodyLen = in.readableBytes();
        int headerLen = ByteBufUtils.getVarIntSize(bodyLen);
        
        // Garante que temos espaço suficiente para a operação zero-copy
        out.ensureWritable(headerLen + bodyLen);
        
        // Escreve o tamanho e depois injeta os bytes
        ByteBufUtils.writeVarInt(out, bodyLen);
        out.writeBytes(in, in.readerIndex(), bodyLen);
    }
}
```

---

## 11. Apêndice D: Tabela Mestra de IDs do Protocolo 47 (Minecraft 1.8)

Abaixo catalogamos de forma definitiva todos os pacotes válidos na fase `Play` (Jogando), tanto de entrada (Server -> Client) quanto de saída (Client -> Server). Se o bot tentar ler ou enviar um ID que não está nesta lista, ele causará uma descompressão corrompida.

### 11.1 Tabela Clientbound (O que o Servidor envia para o Bot)

| ID Hex | Nome do Pacote (Protocolo 47) | Nível de Prioridade / Impacto na Arquitetura Java |
|---|---|---|
| `0x00` | Keep Alive | **Crítico**. Roteia para `PingTrackerService`. Resposta imediata. |
| `0x01` | Join Game | **Crítico**. Define as constantes dimensionais do `WorldManager`. |
| `0x02` | Chat Message | Informativo. Despachado para o `WebConsoleService` e Plugins JS. |
| `0x03` | Time Update | Baixo. Usado para lógica de mobs queimando de dia. |
| `0x04` | Entity Equipment | Médio. Usado para Killaura calcular dano do inimigo pela espada. |
| `0x05` | Spawn Position | Baixo. Bússola. |
| `0x06` | Update Health | **Alto**. Despachado para `SurvivalAgent` (Comer/Fugir). |
| `0x07` | Respawn | Alto. Limpa o `WorldManager` inteiro. Mundo trocou. |
| `0x08` | Player Position And Look | **Crítico**. Requer resposta `0x06`. |
| `0x09` | Held Item Change | Baixo. Atualiza slot visual. |
| `0x0A` | Use Bed | Baixo. Ignorado. |
| `0x0B` | Animation | Baixo. Útil para Anti-Cheat (checar se o adversário balançou o braço). |
| `0x0C` | Spawn Player | **Alto**. Roteado para o Killaura / Radar. |
| `0x0D` | Collect Item | Baixo. Animação de pegar drop. |
| `0x0E` | Spawn Object | Médio. Setas, Barcos. |
| `0x0F` | Spawn Mob | **Alto**. Roteado para o Killaura (Zumbis, Creepers). |
| `0x10` | Spawn Painting | Ignorado. O Bot não renderiza imagens. |
| `0x11` | Spawn Experience Orb | Baixo. O Bot pode ser programado para caçar XP. |
| `0x12` | Entity Velocity | **Crítico**. Motor de Física (AntiKnockback). |
| `0x13` | Destroy Entities | **Alto**. Remoção do mapa no `EntityTrackerService`. |
| `0x14` | Entity | Baixo. |
| `0x15` | Entity Relative Move | Médio. Hitbox tracking. |
| `0x16` | Entity Look | Baixo. |
| `0x17` | Entity Look and Relative Move | Médio. Hitbox tracking. |
| `0x18` | Entity Teleport | Médio. O inimigo piscou (EnderPearl). |
| `0x19` | Entity Head Look | Ignorado. |
| `0x1A` | Entity Status | Baixo. Dano tomado. |
| `0x1B` | Attach Entity | Baixo. Montar no porco. |
| `0x1C` | Entity Metadata | **Alto**. Vidas boss, Invisibilidade, Fogo. |
| `0x1D` | Entity Effect | Baixo. Poção aplicada. |
| `0x1E` | Remove Entity Effect | Baixo. Poção acabou. |
| `0x1F` | Set Experience | Baixo. Atualiza nível na UI. |
| `0x20` | Entity Properties | Baixo. Velocidade do zumbi. |
| `0x21` | Chunk Data | **MASSIVO**. Offloaded para Virtual Threads (Zlib). |
| `0x22` | Multi Block Change | **Alto**. Atualiza o `WorldManager`. |
| `0x23` | Block Change | **Alto**. Atualiza o `WorldManager`. |
| `0x24` | Block Action | Baixo. Baú abrindo (som). |
| `0x25` | Block Break Animation | Baixo. Útil para saber se alguém está minerando. |
| `0x26` | Map Chunk Bulk | **MASSIVO**. Offloaded para Virtual Threads (Zlib). |
| `0x27` | Explosion | Médio. Aplica velocidade (Knockback) e quebra blocos locais. |
| `0x28` | Effect | Baixo. Som. |
| `0x29` | Sound Effect | Baixo. |
| `0x2A` | Particle | Ignorado. Bot Headless. |
| `0x2B` | Change Game State | Médio. Chuva começou/parou. |
| `0x2C` | Spawn Global Entity | Médio. Raio (Lightning Strike). |
| `0x2D` | Open Window | **Alto**. Inicia transação de inventário. |
| `0x2E` | Close Window | **Alto**. Limpa transação de inventário. |
| `0x2F` | Set Slot | **Alto**. Modifica `InventoryManager`. |
| `0x30` | Window Items | **Alto**. Sincronização em massa de baús. |
| `0x31` | Window Property | Baixo. Progresso da fornalha. |
| `0x32` | Confirm Transaction | **Crítico**. Libera o Semaphore de Inventário. |
| `0x33` | Update Sign | Médio. O texto da placa lido pelo bot. |
| `0x34` | Maps | Ignorado. Arte de mapa de papel. |
| `0x35` | Update Block Entity | Médio. Baús/Spawners extras. |
| `0x36` | Sign Editor Open | Baixo. |
| `0x37` | Statistics | Baixo. |
| `0x38` | Player List Item | Médio. A lista "Tab". (TAB Completer dependente). |
| `0x39` | Player Abilities | Médio. Avisa se o bot tem permissão de voar (Fly). |
| `0x3A` | Tab-Complete | Médio. Resposta da busca de nomes. |
| `0x3B` | Scoreboard Objective | Médio. UI Lateral. |
| `0x3C` | Update Score | Médio. Linhas do Scoreboard (Usado para achar dinheiro/Mcmmo). |
| `0x3D` | Display Scoreboard | Médio. |
| `0x3E` | Teams | Baixo. Anti-friendly fire em Killaura. |
| `0x3F` | Plugin Message | **Alto**. Canais customizados (Factions, BungeeCord, Mods). |
| `0x40` | Disconnect | **Fatal**. Gatilho para matar as threads e auto-reconectar. |
| `0x41` | Server Difficulty | Baixo. |
| `0x42` | Combat Event | Baixo. Mensagem de morte. |
| `0x43` | Camera | Ignorado. |
| `0x44` | World Border | Médio. Bot não pode andar além do X/Z recebido. |
| `0x45` | Title | Baixo. Letras gigantes na tela. |
| `0x46` | Set Compression | **Crítico Interceptor**. Insere o ZlibDecoder no Pipeline. |
| `0x47` | Player List Header/Footer | Baixo. Textos do TAB. |
| `0x48` | Resource Pack Send | Baixo. Requer recusa automática para evitar kick. |
| `0x49` | Update Entity NBT | Baixo. |

### 11.2 Tabela Serverbound (O que o Bot envia para o Servidor)

Se o bot tentar enviar um pacote fora de ordem, mais de 20 pacotes por segundo (PPS), ou com campos mal formatados, o Anti-Cheat `NoCheatPlus` ou `GrimAC` aplicará violações instantâneas.

| ID Hex | Nome do Pacote (Protocolo 47) | Nível de Risco Anti-Cheat / Frequência de Envio |
|---|---|---|
| `0x00` | Keep Alive | **Baixo**. Enviar apenas quando o servidor pedir. Não floodar. |
| `0x01` | Chat Message | **Alto**. Limite de 1 mensagem por segundo (Spam detection). Comandos (`/msg`) entram aqui. |
| `0x02` | Use Entity | **Extremo**. Se enviar pacotes de ataque em entidades além de 4.0 blocos (Reach), ou se a entidade estiver atrás da parede (Raytrace), o banimento é irreversível. |
| `0x03` | Player | **Extremo**. Usado para dizer `OnGround` sem alterar a posição real. |
| `0x04` | Player Position | **Crítico**. Enviado 20 vezes por segundo (1 por Tick). Se não enviar, gera Timeout. |
| `0x05` | Player Look | **Crítico**. Mesmo risco acima. A rotação de cabeça (Aimbot) deve ser interpolada humanamente (suave), nunca travando no alvo (Snap). |
| `0x06` | Player Position And Look | **Crítico**. Envio combinado de movimento e visão. |
| `0x07` | Player Digging | **Alto**. Tempo de quebra do bloco deve ser respeitado matematicamente (ex: Madeira sem machado demora 3s. Quebrar em 0.5s aciona o FastBreak). |
| `0x08` | Player Block Placement | **Médio**. FastPlace / FastBow são detectados aqui se enviar os pacotes em menos de 100ms de diferença. |
| `0x09` | Held Item Change | **Médio**. Mudar rápido de slot para usar poção e voltar a espada (AutoSoup). |
| `0x0A` | Animation | Baixo. Só pra dizer que balançou o braço ao quebrar ou bater. |
| `0x0B` | Entity Action | Médio. Usado para não tomar Knockback completo em alguns Anti-Cheats arcaicos. |
| `0x0C` | Steer Vehicle | Baixo. Pilotar porco ou barco (WASD traduzidos pra floats). |
| `0x0D` | Close Window | Baixo. |
| `0x0E` | Click Window | **Extremo**. Clique em baús. FastClick, AutoArmor e ChestStealer são pegos instantaneamente se a transação pular o limite de ping. |
| `0x0F` | Confirm Transaction | **Crítico**. Deve responder ao pacote `0x32` Clientbound. |
| `0x10` | Creative Inventory Action | Ignorado. Bot joga Survival/Factions. |
| `0x11` | Enchant Item | Baixo. Clicar no nível da mesa de encantamento. |
| `0x12` | Update Sign | Baixo. Escrever na placa após coloca-la. |
| `0x13` | Player Abilities | Baixo. O bot dizendo que começou a voar. |
| `0x14` | Tab-Complete | Baixo. Autocompletar um nome de jogador. |
| `0x15` | Client Settings | Baixo. Enviado no início do jogo (Linguagem, Distância de Render). |
| `0x16` | Client Status | Baixo. Respawn. |
| `0x17` | Plugin Message | **Alto**. Para responder a verificações customizadas do BungeeCord (ex: Bungee Perms) ou Anti-Hacks baseados em mods instalados no cliente falso. |
| `0x18` | Spectate | Ignorado. |
| `0x19` | Resource Pack Status | **Alto**. O bot deve mentir enviando "Accepted" e depois "Loaded" sem baixar nada, ou o servidor vai ejetar. |

---

## 12. Apêndice E: A Matemática do ZLib Threshold (Compressão Dinâmica)

Um dos maiores desafios que causavam falhas no bot legado era a mudança dinâmica de estado de compressão.
O servidor pode, no meio do jogo, enviar o pacote `0x46 Set Compression` com o valor `256`. 
O que isso muda na matemática do decodificador Java?

1. **Antes do Pacote `0x46`**:
   - Pacote `0x02` (Chat "Oi") chega:
   - Tamanho (`VarInt`): 4 bytes
   - ID (`VarInt`): 1 byte (`0x02`)
   - Dado (`String`): 3 bytes (Tamanho String = 2, Letras = 'O', 'i')
   - Total Físico na Placa de Rede: 4 + 1 + 3 = 8 bytes.

2. **Depois do Pacote `0x46` (Threshold = 256)**:
   - Pacote `0x02` (Chat "Oi") chega:
   - O Tamanho Descomprimido do conteúdo é 4 bytes.
   - 4 é MENOR que o Threshold 256.
   - O Netty deve formatar o pacote assim:
     - PacketLength (`VarInt`): 5 bytes (Soma do próximo campo + tamanho dos dados)
     - DataLength (`VarInt`): `0` (Zero significa que os bytes seguintes **NÃO ESTÃO COMPRIMIDOS**).
     - ID (`VarInt`): 1 byte (`0x02`)
     - Dado (`String`): 3 bytes.

3. **Depois do Pacote `0x46` (Threshold = 256) - Pacote Grande**:
   - Pacote `0x21` (Chunk Data) chega. Tamanho descomprimido = 10.000 bytes.
   - 10.000 é MAIOR que 256.
   - O Netty deve formatar o pacote assim:
     - PacketLength (`VarInt`): 1205 bytes. (Tamanho compactado + próximo campo).
     - DataLength (`VarInt`): 10000. (Valor Real).
     - ID + Dados: 1200 bytes empacotados em ZLib. O Netty precisa chamar `Inflater.inflate` para recuperar os 10.000 bytes e injetá-los no próximo estágio do pipeline.

*Essa lógica deve estar firmemente encapsulada no `MinecraftZlibEncoder` e `MinecraftZlibDecoder`. A camada de aplicação do bot (O EventBus) NÃO PODE FAZER IDEIA se o pacote foi comprimido ou não. A abstração tem que ser estanque.*

*Essa lógica deve estar firmemente encapsulada no `MinecraftZlibEncoder` e `MinecraftZlibDecoder`. A camada de aplicação do bot (O EventBus) NÃO PODE FAZER IDEIA se o pacote foi comprimido ou não. A abstração tem que ser estanque.*

---

## 13. Apêndice F: O Fluxo de Autenticação Premium (Yggdrasil)

O `AdvancedBot 2.4.5` suportava contas originais (Premium). Para isso, antes mesmo de abrir o Socket Netty, o Bot precisa interagir com a API Yggdrasil da Mojang via HTTP REST. O legado usava `HttpWebRequest` síncrono. O Java usará `java.net.http.HttpClient` assíncrono.

### 13.1 Fase 1: Autenticação Inicial (Client-Side)
O módulo `advancedbot-auth` faz um POST para `https://authserver.mojang.com/authenticate`:
```json
{
    "agent": { "name": "Minecraft", "version": 1 },
    "username": "bot@email.com",
    "password": "senha123",
    "clientToken": "uuid-gerado-aleatorio"
}
```
**Resposta (Salva no `AuthContext` do Bot)**:
Retorna o `AccessToken`, o `ClientToken` e o `Profile (UUID e Nome real)`. Se o IP do servidor VPS onde o bot roda estiver banido pela Mojang por muitas tentativas, retorna HTTP 403 (O bot deve logar este erro).

### 13.2 Fase 2: O Handshake de Sessão (Server-Side Join)
Quando o servidor Minecraft envia o Pacote `0x01 Encryption Request` (Veja Seção 3.4), o bot gera uma chave AES de 16 bytes.
Antes de mandar o `0x01 Encryption Response`, o bot DEVE fazer um POST para `https://sessionserver.mojang.com/session/minecraft/join`:
```json
{
    "accessToken": "token-obtido-na-fase-1",
    "selectedProfile": "uuid-sem-tracos",
    "serverId": "hash-sha1-gerado-pelo-java"
}
```
**A Matemática do ServerID Hash**:
O C# tinha uma função complexa para gerar o hash do servidor. O Java deve replicar isso perfeitamente:
`Hash = SHA-1(ServerIdString + SharedSecretAES + PublicKeyDoServidor)`.
Se a Mojang responder HTTP 204 (No Content), significa sucesso. O Netty agora tem permissão para enviar o pacote de resposta criptografado e o servidor o aceitará.

---

## 14. Apêndice G: Tabela de Metadados de Entidades (DataWatcher)

O Pacote `0x1C (Entity Metadata)` é um dos mais difíceis de decodificar. Ele não tem tamanho fixo. Ele envia um array de `Items`. Cada item tem um Tipo (Byte, Short, Int, Float, String, Item, Position, Rotation) e um Índice.

O C# usava uma classe genérica chamada `DataWatcher`. Se o DataWatcher falhasse ao decodificar um único byte, todo o resto do stream corrompia (Packet desync).

| Índice (Index) | Tipo Java (`ByteBuf` Read) | Significado Genérico (Entity) |
|---|---|---|
| `0` | `Byte` | **Flags de Estado**: `0x01`=Pegando fogo, `0x02`=Agachado, `0x08`=Sprinting, `0x20`=Invisível. Crítico para a IA ignorar inimigos invisíveis. |
| `1` | `Short` | **Air**: Fôlego debaixo d'água (300 a 0). |
| `2` | `String` | **Custom Name**: O nome flutuante em cima de um Mob ou ArmorStand (Usado em hologramas de servidores). |
| `3` | `Byte` | **Custom Name Visible**: `1` se o nome acima está visível, `0` se não. |
| `4` | `Byte` | **Silent**: `1` se a entidade não faz som. |
| `6` | `Float` | **Health**: A vida exata do Mob ou Jogador (usado pelo Killaura para priorizar os que estão morrendo). |
| `7` | `Int` | **Potion Color**: Cor das partículas flutuando sobre a entidade. |
| `8` | `Byte` | **Potion Ambient**: Se as partículas são opacas ou não. |
| `9` | `Byte` | **Arrows**: Quantidade de flechas presas no modelo do personagem. |

**Regra de Decodificação (O `0x7F` Final)**:
O array de metadados termina quando o Netty ler o byte `127` (`0x7F`). Sem essa checagem, o Java tentará ler além do pacote e causará uma `IndexOutOfBoundsException`.

---

## 15. Apêndice H: Anatomia dos Slots NBT (Named Binary Tag)

Um erro fatal na engenharia de pacotes é ignorar as NBT Tags. No Minecraft, um Item no inventário (Packet `0x30 Window Items` e `0x2F Set Slot`) pode conter uma gigantesca árvore de propriedades comprimida em formato binário (GZIP) anexada ao final de sua declaração.

O C# tinha a classe `NbtIO.cs` que usava Streams bloqueantes. Em Java, ler o NBT diretamente do `ByteBuf` do Netty requer um parser customizado ou o uso de bibliotecas de terceiros (como `JNBT` adaptado para Netty).

### 15.1 Tabela de Tipos NBT (Para Criação do Parser Java)

Quando o byte `NBT Start` for lido de um slot e não for `0x00`, o Java entra em um loop recursivo de leitura de tags. Abaixo estão os códigos hexadecimais de identificação de tags:

| ID Hex | Tipo da Tag (NBT) | Tamanho Lógico (Como ler do ByteBuf) | Aplicação em Itens de Jogo |
|---|---|---|---|
| `0x00` | `TAG_End` | 0 bytes | Sinaliza o fechamento de um bloco `TAG_Compound` ou de uma `TAG_List`. |
| `0x01` | `TAG_Byte` | 1 byte | Status de item inquebrável (`Unbreakable: 1b`). |
| `0x02` | `TAG_Short` | 2 bytes | Dano de uso de ferramentas antigas ou encantamentos. |
| `0x03` | `TAG_Int` | 4 bytes | Quantidade de durabilidade gasta ou cor de armadura de couro (RGB Hex). |
| `0x04` | `TAG_Long` | 8 bytes | Timestamp de criação ou UUIDs internos de atributos. |
| `0x05` | `TAG_Float` | 4 bytes | Multiplicadores de dano da espada. |
| `0x06` | `TAG_Double` | 8 bytes | Modificadores de velocidade de ataque. |
| `0x07` | `TAG_Byte_Array` | `Int Length` + N Bytes | Arte de mapas customizados, ou assinaturas de metadados obscuros. |
| `0x08` | `TAG_String` | `Short Length` + N Chars | Nomes customizados (`display.Name`) ou lore do item. **Muito usado em servidores Factions**. |
| `0x09` | `TAG_List` | `Byte TagID` + `Int Length` + N Tags | Lista de encantamentos aplicados na ferramenta. (Ex: Afiada 5, Inquebrável 3). |
| `0x0A` | `TAG_Compound` | N Tags até achar `0x00` | A raiz de todas as propriedades de um item. Como se fosse um objeto JSON, mas binário. |
| `0x0B` | `TAG_Int_Array` | `Int Length` + (N * 4 Bytes) | Arrays de blocos em estruturas complexas (pouco usado em itens normais). |

### 15.2 Cuidados de Segurança contra NBT Bomb
**Aviso Crítico de Arquitetura**: Servidores maliciosos (ou jogadores usando exploits) podem enviar pacotes com `TAG_Compound` profundamente aninhados (Ex: Uma caixa dentro de uma caixa dentro de uma caixa, 500 vezes).
Se o parser NBT recursivo no Java não tiver um **limite de profundidade (Depth Limit = 64)** e um **limite de bytes lidos (Quota = 2MB)**, o processo do Bot sofrerá um `StackOverflowError` ou estourará o Heap (ZipBomb/NBTBomb). O Java Netty Decoder DEVE implementar contadores rigorosos nestas leituras.

---

## 16. Apêndice I: Estratégias de Fuga de Anti-Cheat (Protocol Spoofing)

Muitos servidores implementam firewalls na camada de protocolo (Ex: Aegis, BungeeGuard, FlameCord) que tentam identificar bots analisando a exata ordem em que o `AdvancedBot` envia os pacotes durante o Login. O C# original era frequentemente bloqueado no Handshake por enviar dados rápido demais.

### 16.1 Ordem Canônica de Inicialização (O Padrão Vanilla)
Para que o bot Netty pareça perfeitamente idêntico ao Client Vanilla da Mojang, a ordem dos pacotes enviados imediatamente após a transição para o estado `PLAY` deve ser:

1. `0x15` (Client Settings): O Vanilla envia isso milissegundos após logar.
2. `0x17` (Plugin Message): O Vanilla anuncia seus canais (ex: `MC|Brand` enviando "vanilla").
3. Retardo Intencional: O bot **não deve** enviar nenhum pacote de Movimento (`0x04`) até receber o primeiro `0x08` (Player Position and Look) do Servidor. O C# legado errava nisso enviando X=0,Y=0 no primeiro tick, revelando ser um bot.

### 16.2 Spoofing do `MC|Brand`
Alguns Anti-Cheats kickam qualquer cliente cujo `MC|Brand` não seja conhecido (Vanilla, Forge, LabyMod).
Se o usuário configurar o Bot para se disfarçar de Forge, o pacote `0x17` (Plugin Message) deve registrar o canal `FML|HS` (Forge Mod Loader Handshake) e realizar as dezenas de trocas de pacotes do Forge antes de começar a andar. O Bot Java deverá ter uma classe `BrandSpooferService` ouvindo essas interações.

---

## 17. Apêndice J: Segurança de Buffer Leak (Zero-Copy)

No Netty, a memória (`ByteBuf`) costuma ser alocada Off-Heap (direto na RAM física do sistema operacional) para ignorar o Garbage Collector.
Se o Bot receber um pacote complexo (ex: Chunk) e a camada de aplicação se esquecer de chamar `buf.release()` após ler os dados, a RAM física da VPS do usuário vai encher silenciosamente até o sistema operacional matar o Java sem aviso (OOM Killer).

**A Regra de Ouro do Netty no Bot**:
O `PacketDecoder` descrito na seção 9 SEMPRE consome o buffer. Todas as IAs e Handlers devem extrair primitivas puras (`int, float, byte[]`) ou Strings do pacote **antes** de enviá-lo pelo EventBus. O EventBus NUNCA deve trafegar `ByteBuf`, pois isso causaria um *Buffer Leak* indetectável se um Agente não o consumisse.

Para evitar vazamentos acidentais durante o desenvolvimento, inicie a JVM com a flag `-Dio.netty.leakDetection.level=PARANOID`. O Netty avisará no console exatamente qual linha de código esqueceu de dar o `release()`.

---

## 18. Conclusão do Mapa de Pacotes

Este documento forneceu o diagrama de dissecação completo do protocolo de rede do Minecraft (Versão 1.8.x - Protocolo 47). Ao contrário do projeto C# anterior (`AdvancedBot 2.4.5`), que amontoava a leitura de rede, a descompressão e a lógica de jogo em uma única classe `MPPlayer` e `PacketStream`, a reescrita em Java orienta os desenvolvedores a usarem o **Pipeline Responsável do Netty**.

Ao seguir esse mapa:
1. Nenhuma Thread será bloqueada por causa de rede lenta.
2. Nenhuma String gigante causará `OutOfMemoryError`.
3. Todo pacote inbound virará um `Evento` Java.
4. Todo `ActionIntent` da IA virará um pacote outbound limpo e semanticamente validado.

O próximo passo da arquitetura será desenhar o grafo de dependências e injeções de dependência (`07-Mapa-de-Dependencias-Cruzadas.md`) para garantir que os módulos de IAs encontrem esse barramento de eventos sem gerar acoplamento de código.
