# Bootstrap e classe `Program`

Fonte primária: [`AdvancedBot/Program.cs`](../../AdvancedBot/Program.cs). Esta página descreve o comportamento efetivamente presente no legado; propostas para Java não alteram as regras observadas.

## Responsabilidade

`AdvancedBot.Program` é uma classe estática que concentra cinco papéis que, no executável .NET, são inseparáveis:

1. ponto de entrada WinForms e dono do ciclo de vida do processo;
2. registro de tratamento global de exceções e escrita de arquivos de erro;
3. registro global de configuração NBT (`conf.dat`);
4. pontos globais de acesso ao formulário principal e ao gerenciador de plugins;
5. utilitários heterogêneos de strings, HTTP, aparência de `ListView` e uma rotina privada de demonstração do viewer.

Ela é compilada como executável x86 .NET Framework 4.6.2 (`AdvancedBot_Crack.csproj`), com `Main` marcado `[STAThread]`. Portanto, a thread inicial também é a thread STA exigida por COM/WinForms e pelo loop de mensagens da interface.

## Estado interno e dependências

| Membro | Estado / semântica | Quem observa ou altera |
|---|---|---|
| `AppVersion` | constante `"v2.4.5.2-f"`; é o rótulo lógico da aplicação, independente da data de build. | consumidores que exibem/versionam o cliente. |
| `FrmMain` | referência estática ao único formulário `Main` construído no bootstrap. | UI, fluxos de sessão e código que precisa alcançar a janela principal. |
| `Config` | raiz mutável (`CompoundTag`) da árvore NBT em memória. Só passa a existir após `LoadConf()`. | getters/setters de configuração, `SaveConf()`, UI e subsistemas de cliente/viewer. |
| `pluginManager` | único `PluginManager`, construído antes da configuração e inicializado imediatamente. | plugins e comandos extensíveis. |
| `DEBUG` | flag pública inicializada como `false`; nesta classe não é consultada nem alterada. | código externo potencial. |
| `appDate` | cache da data de build; `Ticks == 0` significa “ainda não calculada”. | `GetBuildDate()` e cabeçalho de log. |
| `errLogLock` | objeto de exclusão mútua para serializar `CreateErrLog`. | handlers globais e qualquer chamador público. |

Dependências diretas: `System.Windows.Forms`, `System.Threading`, `System.Diagnostics`, `System.IO`, `System.Net`, reflexão, interop `uxtheme.dll`, NBT (`CompoundTag`, subclasses de `Tag`, `NbtIO`), cliente Minecraft, mundo/chunks, fila e pacotes, plugins e viewer OpenGL. A classe não depende de eventos de domínio Minecraft, mas registra eventos globais de exceção do runtime/UI.

## Fluxo de execução do processo

```mermaid
sequenceDiagram
    participant CLR as CLR / thread STA
    participant P as Program.Main
    participant UI as Main (WinForms)
    participant PM as PluginManager
    participant NBT as conf.dat
    participant Loop as Application.Run

    CLR->>P: invoca Main()
    P->>P: ThreadPool.SetMaxThreads(16,16)
    P->>P: assina exceções AppDomain e WinForms
    P->>P: calcula máscara de afinidade
    P->>P: define ProcessorAffinity
    P->>UI: cria Main e guarda FrmMain
    P->>PM: cria, guarda e chama Init()
    P->>NBT: LoadConf()
    P->>UI: CheckKey()
    P->>Loop: Application.Run(FrmMain)
    Loop-->>P: exceção não tratada no Run
    P->>P: CreateErrLog("crash") e Environment.Exit(0)
```

### `Main()` — privado, ponto de entrada

1. Limita o pool de threads do processo a 16 workers e 16 I/O completion threads. O efeito é global: qualquer subsistema que use `ThreadPool` passa a concorrer dentro desse teto.
2. Assina `AppDomain.CurrentDomain.UnhandledException`. O delegado converte `ExceptionObject` para `Exception` sem verificação de tipo e chama `CreateErrLog`; o tipo de arquivo é `crash` se `IsTerminating`, caso contrário `err`.
3. Assina `Application.ThreadException`. Exceções da thread de UI entregues por WinForms vão para `CreateErrLog` com tipo `threadexception`.
4. Cria uma máscara de afinidade com os bits de `0` até `max(1, ProcessorCount - 1) - 1`. Assim, em uma máquina com `N > 1` processadores lógicos, reserva os `N-1` primeiros; em máquina de um processador, reserva o bit 0. A máscara é atribuída a `Process.GetCurrentProcess().ProcessorAffinity`; não há `try/catch` nessa operação.
5. Habilita estilos visuais e desliga o modo de renderização de texto compatível antigo do WinForms.
6. Instancia `Main`, atribui a `FrmMain`, instancia `PluginManager`, atribui a `pluginManager` e chama `Init()`. Portanto, plugins são descobertos/inicializados antes de `conf.dat` ser lido e antes de `CheckKey()`.
7. Executa `LoadConf()`, depois `FrmMain.CheckKey()`.
8. Inicia o loop modal da janela com `Application.Run(FrmMain)`. Somente a exceção que escapar dessa chamada é capturada localmente: é gravada como `crash` e o processo termina com código 0, inclusive no caso de falha.

Não existe uma máquina de estado explícita, porém a ordem observável é: `não inicializado → UI criada → plugins inicializados → configuração carregada → chave verificada → loop WinForms ativo → encerrado`. Qualquer código que usa `Program.Config` antes de `LoadConf` encontra `null`.

## Eventos utilizados

| Evento | Registro | Tratamento | Limite importante |
|---|---|---|---|
| `AppDomain.CurrentDomain.UnhandledException` | `Main()` | gera arquivo de erro. | o cast explícito para `Exception` pode falhar se o runtime publicar objeto não-`Exception`. |
| `Application.ThreadException` | `Main()` | gera arquivo de erro. | não força encerramento nem apresenta diálogo nesta classe. |

`LaunchViewerTest()` não usa eventos .NET: cria uma thread manual que faz polling de `vf.IsDisposed` e da fila de saída.

## API pública de utilitários

### Texto, endereço e aparência

| Método | Comportamento exato |
|---|---|
| `ParseIP(string addr, out string ip, ref ushort port)` | Procura o primeiro `:`. Se existir, `ip` recebe tudo antes dele e tenta converter o restante para `ushort`, sobrescrevendo `port` somente no sucesso; retorna `false` se a conversão falhar. Sem `:`, devolve toda a entrada em `ip`, conserva a porta recebida e retorna `true`. Não entende IPv6, whitespace nem vários dois-pontos. |
| `SplitColon(string s, out string a, ref string b)` | Divide no primeiro `:`. Com separador, `a` recebe o prefixo e `b` o sufixo; sem ele, somente `a` é atualizado para a string inteira e `b` preserva o valor anterior. |
| `Lines(string str)` | Executa `Split` sobre `\r` e `\n`, removendo entradas vazias. Uma linha em branco desaparece e CRLF resulta na mesma lista que LF; não preserva delimitadores. |
| `SetExplorerTheme(ListView lv)` | Usa reflexão para definir a propriedade não pública `DoubleBuffered` como `true`, em seguida chama `SetWindowTheme(handle, "explorer", null)` da DLL nativa `uxtheme`. Exige que o handle seja utilizável e depende da implementação WinForms/Windows. |
| `HttpGet(string url)` | Cria `HttpWebRequest`, desliga proxy, aceita gzip/deflate, habilita keep-alive e aplica `Timeout`/`ReadWriteTimeout` de 10 s. Lê toda a resposta como UTF-8 e a devolve. Não captura erros HTTP, não expõe status/cabeçalhos e não permite cancelamento. |
| `GetBuildDate()` | Retorna `appDate` se já calculado. Caso contrário, lê diretamente o cabeçalho PE do módulo executável: offset `e_lfanew + 8`, interpreta-o como segundos desde Unix epoch e converte de UTC para o fuso Windows `E. South America Standard Time`; grava o resultado no cache. Falha se o fuso não existir ou se o formato PE/offset não puder ser lido. |

`HandleMessage(byte[] buffer, int count)` também é privado. Ele apenas escreve `Received ` seguido do hexadecimal dos primeiros `count` bytes em `Console`; não há chamada interna para ele no arquivo, logo não participa do bootstrap normal.

### Persistência e acesso à configuração

`LoadConf()` tenta desserializar o arquivo relativo `conf.dat` por `NbtIO.Read`. Qualquer exceção — inclusive corrupção, ausência, acesso negado ou tipo inválido — é descartada e substituída por `new CompoundTag()`. Depois aplica estas chaves aos estáticos de outros módulos:

| Chave NBT | Destino | Regra de ausência |
|---|---|---|
| `Knockback` | `MinecraftClient.Knockback` | `true` (`GetBoolOrTrue`). |
| `ViewerTextures` | `ViewForm.UseTexture` | `true` (`GetBoolOrTrue`). |
| `ViewerRenderSignText` | `ViewForm.RenderSignText` | `false`. |
| `AutoReconnect` | `MinecraftClient.AutoReconnect` | `false`. |
| `ViewerUseVBO` | `ChunkRenderer.UseVBO` | `false`. |
| `ViewerFpsLimit` | `ViewForm.FpsLimit` | `40`. |
| `ViewerZSorting` | `WorldRenderer.ZSorting` | `true` (`GetBoolOrTrue`). |
| `ChatMaxLines` | `MinecraftClient.MaximumChatLines` | `150`. |

`SaveConf()` primeiro sobrescreve/adiciona essas mesmas oito chaves na árvore em memória, depois chama `NbtIO.Write(Config, "conf.dat")`. Toda exceção dessa escrita é suprimida; assim, o chamador não distingue persistência concluída de falha. Não há flush, lock de arquivo ou escrita atômica nesta camada.

Os getters públicos (`GetConfigInt`, `GetConfigStr`, `GetConfigBool`, `GetConfigIntArr`, `GetConfigDouble`) chamam `GetConfigEntry(path)`. Se a tag final não for `null`, fazem cast direto para a subclasse esperada e retornam `Data`; se não existir, devolvem o padrão indicado. Tipo incompatível causa `InvalidCastException`; um trecho intermediário inexistente pode falhar dentro de `CompoundTag.GetCompound`, pois não é validado aqui.

Os setters públicos criam respectivamente `IntTag`, `StringTag`, `ByteTag` (0/1), `IntArrayTag` ou `DoubleTag` com nome vazio e delegam a `SetConfigEntry`. `GetConfigEntry`, `ConfigEntryExists` e `SetConfigEntry` interpretam caminhos com ponto:

1. iteram cada segmento antes do último com `IndexOf('.', índice)`;
2. no getter/existência, substituem o nó corrente por `GetCompound(segmento)`;
3. no setter, obtêm o composto; se `IsEmpty()`, anexam-no ao pai com `AddCompound`; em seguida o usam como pai;
4. o último segmento é lido com `Get`, testado por `Contains`, ou gravado por `Add`.

Não há escape para `.` em nomes, não há validação de caminho vazio e não há sincronização para leituras/escritas concorrentes de `Config`.

### Falhas globais

`CreateErrLog(Exception ex, string type)` é público e sincroniza todo o método no objeto estático `errLogLock`. Cria `errlogs` relativo se necessário, monta a base `errlogs\\{type}_{dd-MM-yyyy_HH.mm.ss}`, procura o primeiro sufixo numérico livre (`_0.txt`, `_1.txt`, ...), e grava uma linha de cabeçalho com versão fixa e `GetBuildDate`, seguida de `ex.ToString()`. O lock impede colisões entre invocações no mesmo processo, mas não protege contra outro processo. Exceções de I/O dentro da própria rotina não são capturadas.

## Rotina privada de teste: `LaunchViewerTest()`

Este método não é chamado por `Main`. Ele é um ambiente local de teste do viewer, não uma sessão de protocolo:

1. Constrói `MinecraftClient("localhost", 8000, "ViewerTest", "123", null)` e substitui explicitamente mundo e jogador por `World` e `Entity` locais.
2. Gera uma grade 10×10 de chunks, centrada em `(0,0)`, com duas `ChunkSection`; calcula alturas por ruído determinístico baseado em `Utils.XorShift32`, mas a vegetação/árvores usa `Random` sem seed, portanto não é reprodutível integralmente.
3. Povoa bedrock, pedra/terra/grama, flores ocasionais e no máximo duas árvores por chunk; insere cada chunk no mundo.
4. Popula slots 36–44 com maçãs e substitui 36–39 por ferramentas/bloco específicos; define versão 1.8, instala uma `PacketQueue` sem stream e posiciona o jogador em `(0,24,0)`.
5. Cria `ViewForm`, inicia uma thread sem nome/background explicitado e, a cada aproximadamente 50 ms, chama `Player.Tick()`. Ao cair abaixo de Y=-10, reposiciona o jogador.
6. Por reflexão, extrai o campo privado `queue` de `PacketQueue` como `ConcurrentQueue<IPacket>` e o consome. `PacketPlayerDigging` remove o bloco alvo do mundo; `PacketChatMessage` cuja mensagem começa com `block ` coloca sob o jogador o bloco retornado por `Blocks.GetBlockByName`; qualquer outra mensagem é ecoada no chat local com prefixo de cor Minecraft.
7. Desliga globalmente `Control.CheckForIllegalCrossThreadCalls` e inicia um segundo loop WinForms modal com `Application.Run(vf)`.

Os estados implícitos dessa rotina são `mundo construído → viewer ativo/thread de simulação ativa → formulário descartado`, sendo que a thread termina apenas quando observa `IsDisposed`. A dependência de reflexão e o acesso simultâneo a mundo/player/UI tornam esse caminho estritamente diagnóstico.

## Problemas observáveis e fronteiras para Java

- A ordem de inicialização é parte do contrato: plugins precedem `LoadConf` e `CheckKey`; uma migração compatível deve preservar esse efeito ou registrar a alteração deliberadamente.
- O estado estático mistura UI, domínio e configuração. Em Java, mantenha o comportamento com um contexto de aplicação explícito, mas isole uma configuração por processo e sessão por cliente para não reproduzir compartilhamento acidental.
- `conf.dat` é NBT binário e a ausência/corrupção equivale a configuração vazia. O adaptador Java precisa preservar defaults, chaves e tipos exatos durante a fase de compatibilidade.
- Casts de tag, erros silenciosos de `LoadConf`/`SaveConf` e parsing simples de endereço são comportamentos legados relevantes para testes de compatibilidade; validações novas devem ficar numa fronteira opt-in, não mudar esses resultados sem decisão explícita.
- CPU affinity, WinForms, `uxtheme.dll`, cabeçalho PE e `ThreadException` são detalhes de host Windows, não regras de bot. Para Java, mapeie-os para adaptadores de plataforma/observabilidade; não os coloque no núcleo Minecraft.
- O teste do viewer deve ser tratado como fixture ou harness de integração. Não use reflexão sobre filas privadas no novo desenho; exponha uma porta de saída de pacotes de teste que mantenha as mesmas reações a cavar e chat.
