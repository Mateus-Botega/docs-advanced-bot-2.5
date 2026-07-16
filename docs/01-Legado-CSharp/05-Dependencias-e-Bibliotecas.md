# 05 — Dependências e Bibliotecas

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-05                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Dependências                 |
| **Escopo**         | Todas as dependências externas e internas        |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-01, LEG-06, LEG-08                 |

---

## Plataforma

| Item | Valor |
|------|-------|
| **Runtime** | .NET Framework 4.6.2 |
| **Plataforma** | Windows x86 (32 bits) |
| **Linguagem** | C# 7.3 |
| **IDE** | Visual Studio 2022 |

---

## Dependências Externas

### Newtonsoft.Json

| Campo | Valor |
|-------|-------|
| **Arquivo** | `Newtonsoft.Json.dll` |
| **Tamanho** | 701.992 bytes |
| **Documentação** | `Newtonsoft.Json.xml` (710.224 bytes) |
| **Origem** | NuGet / bundled |

**Finalidade no projeto:**

- Parsing do JSON de chat do Minecraft (protocolo 1.7+).
- Parsing das respostas da API Mojang (autenticação Yggdrasil, join session).
- Parsing da resposta de ping do servidor.
- Leitura do cache de login (`LoginCache`).

**Uso Principal:**

- `ChatParser.cs` — `JObject.Parse()` para interpretar o JSON do chat.
- `SessionUtils.cs` — `JsonConvert.DeserializeObject()` para autenticação.

**Equivalente Java:**

- `com.fasterxml.jackson.databind:jackson-databind` (recomendado).
- Alternativa: `com.google.code.gson:gson`.

**Observação:** A estrutura de parsing do chat Minecraft é complexa. O Java deverá implementar a mesma lógica de `ChatParser.ParseText()`.

---

### Jint

| Campo | Valor |
|-------|-------|
| **Arquivo** | `Jint.dll` |
| **Tamanho** | 326.656 bytes |
| **Debug** | `Jint.pdb` (974.336 bytes) |
| **Origem** | NuGet / compilado localmente |

**Finalidade no projeto:**

- Motor de execução JavaScript integrado ao sistema de scripts.
- Usado pela classe `JsScriptContext.cs` para permitir que macros sejam escritas em JavaScript.

**Uso Principal:**

- `JsScriptContext.cs` — cria engine Jint, expõe API do bot para JavaScript.

**Equivalente Java:**

- `org.graalvm.sdk:graal-sdk` com `org.graalvm.js:js` (GraalVM JavaScript).
- Alternativa mais leve: `org.mozilla:rhino` (Rhino JavaScript Engine).
- Alternativa moderna: `org.openjdk.nashorn:nashorn-core`.

**Observação:** A API exposta para os scripts precisa ser reimplementada no equivalente Java.

---

### Transitions

| Campo | Valor |
|-------|-------|
| **Arquivo** | `Transitions.dll` |
| **Tamanho** | 17.408 bytes |
| **Origem** | Bundled |

**Finalidade no projeto:**

- Biblioteca de animações de propriedades para Windows Forms.
- Usada possivelmente para animações de UI no formulário principal.

**Equivalente Java:**

- Não aplicável. A interface gráfica será completamente redesenhada em Java.

---

### MaxMind.MaxMindDb

| Campo | Valor |
|-------|-------|
| **Localização** | Pasta `MaxMind.MaxMindDb/` (código-fonte incluído) |
| **Origem** | Código-fonte incluído diretamente |

**Finalidade no projeto:**

- Não utilizado diretamente no core. Possivelmente usado em verificação de proxies ou geolocalização de IPs.

**Equivalente Java:**

- `com.maxmind.geoip2:geoip2` (biblioteca oficial MaxMind para Java).

---

### FastColoredTextBoxNS

| Campo | Valor |
|-------|-------|
| **Localização** | Pasta `FastColoredTextBoxNS/` (código-fonte incluído) |
| **Origem** | Código-fonte incluído diretamente |

**Finalidade no projeto:**

- Editor de texto com highlight de sintaxe usado no `MacroEditor.cs`.
- Fornece o componente visual de edição de scripts com colorização.

**Equivalente Java:**

- Não aplicável. A interface gráfica será completamente redesenhada.
- Se necessário: `RSyntaxTextArea` (componente Swing para edição de código).

---

### Ionic.Zlib / Ionic.Crc

| Campo | Valor |
|-------|-------|
| **Localização** | Pastas `Ionic.Zlib/` e `Ionic.Crc/` (código-fonte incluído) |
| **Origem** | Código-fonte incluído (derivado de DotNetZip) |

**Finalidade no projeto:**

- Compressão e descompressão zlib usada na `PacketStream.cs`.
- Protocolo Minecraft 1.8+ usa compressão zlib para pacotes acima do threshold.

**Equivalente Java:**

- `java.util.zip.Deflater` e `java.util.zip.Inflater` (JDK nativo).
- Nenhuma dependência externa necessária.

**Observação:** A implementação Java pode usar as classes nativas do JDK sem dependências adicionais.

---

## Dependências do .NET Framework

### System.Core

**Finalidade:** LINQ, delegates, expressões lambda.

**Equivalente Java:** Streams API do Java 8+ (`java.util.stream`).

---

### System.Windows.Forms

**Finalidade:** Interface gráfica (formulários, controles, timer, FileSystemWatcher).

**Equivalente Java:** Não aplicável. Interface gráfica não será migrada para Java.

**Observação:** Toda a camada GUI (Windows Forms) será descartada na migração Java. O Java terá interface headless ou nova GUI.

---

### System.Windows.Forms.DataVisualization

**Finalidade:** Componente de gráficos para Windows Forms (usado em estatísticas).

**Equivalente Java:** Não aplicável.

---

### System.Net

**Finalidade:** `TcpClient`, `WebRequest`, `HttpWebRequest`, `IPAddress`, `Dns`.

**Equivalente Java:**
- `java.net.Socket`, `java.net.InetAddress` — sockets TCP.
- `java.net.http.HttpClient` (Java 11+) — requisições HTTP.
- `org.apache.httpcomponents.client5:httpclient5` — alternativa externa.

---

### System.Security.Cryptography

**Finalidade:** RSA (`RSACryptoServiceProvider`), AES, SHA-1 para o protocolo Minecraft.

**Equivalente Java:**
- `javax.crypto.Cipher` — AES-CFB8.
- `java.security.KeyFactory` — RSA.
- `java.security.MessageDigest` — SHA-1.

**Observação Crítica:** O algoritmo SHA-1 do server hash do Minecraft é **não-standard** (permite valores negativos). A implementação Java deve usar `BigInteger` para reproduzir o comportamento exato.

---

### System.Threading

**Finalidade:** `Thread`, `ThreadPool`, `Monitor`, `Interlocked`, `Task`.

**Equivalente Java:**
- `java.util.concurrent.ExecutorService` — thread pool.
- `java.util.concurrent.CompletableFuture` — operações assíncronas.
- `java.util.concurrent.ScheduledExecutorService` — tick timer.

---

### System.Management

**Finalidade:** WMI para leitura de informações de hardware (HWID).

**Equivalente Java:**
- `com.github.oshi:oshi-core` — hardware info para Java (multi-plataforma).
- Alternativa: `java.lang.management.ManagementFactory` + `com.sun.management`.

---

### System.Runtime.InteropServices

**Finalidade:** P/Invoke para chamar funções nativas do Windows (`uxtheme.dll`, `winmm.dll`, OpenGL via `opengl32.dll`).

**Equivalente Java:** Não aplicável. O renderizador 3D não será migrado.

---

### System.Xml

**Finalidade:** Parsing de XML para configurações ou dados legados.

**Equivalente Java:** `javax.xml.parsers` (JDK nativo) ou `org.dom4j:dom4j`.

---

## Dependências Implícitas (P/Invoke)

### OpenGL (opengl32.dll / gdi32.dll)

**Localização:** `AdvancedBot.Viewer/GL.cs`, `WGL.cs`

**Finalidade:** Renderização 3D via OpenGL para o visualizador de mundo.

**Equivalente Java:** Não aplicável. O renderizador 3D será descartado ou completamente reescrito com tecnologia adequada (JavaFX, LWJGL, etc.).

---

### uxtheme.dll

**Localização:** `Program.cs`

**Finalidade:** `SetWindowTheme()` para aplicar tema Explorer às `ListView`.

**Equivalente Java:** Não aplicável.

---

### winmm.dll

**Localização:** `Utils.cs` (presumido para `GetTickCount64`)

**Finalidade:** `GetTickCount64()` — contador de ticks em milissegundos desde o boot do Windows.

**Equivalente Java:** `System.currentTimeMillis()` ou `System.nanoTime()`.

---

## Resumo de Equivalências para Java

| Dependência C# | Finalidade | Equivalente Java | Criticidade |
|---------------|------------|------------------|-------------|
| `Newtonsoft.Json` | JSON parsing | `jackson-databind` | ALTO |
| `Jint` | JavaScript engine | GraalVM JS ou Rhino | MÉDIO |
| `Ionic.Zlib` | Compressão zlib | `java.util.zip` (nativo) | CRÍTICO |
| `System.Security.Cryptography (AES)` | AES-CFB8 | `javax.crypto` | CRÍTICO |
| `System.Security.Cryptography (RSA)` | RSA | `java.security` | CRÍTICO |
| `System.Net.Sockets.TcpClient` | TCP | `java.net.Socket` | CRÍTICO |
| `System.Threading.ThreadPool` | Thread pool | `ExecutorService` | ALTO |
| `System.Threading.Task` | Async/await | `CompletableFuture` | ALTO |
| `System.Management` | HWID | `oshi-core` | BAIXO |
| `MaxMind.MaxMindDb` | Geolocalização | `geoip2` | BAIXO |
| `Transitions` | Animações UI | N/A | N/A |
| `FastColoredTextBoxNS` | Editor de código | RSyntaxTextArea | N/A |
| `System.Windows.Forms` | GUI | N/A (descartado) | N/A |
| P/Invoke OpenGL | Renderização 3D | N/A (descartado) | N/A |

---

## Observações

- A ausência de um arquivo `packages.config` ou referência de NuGet explícita sugere que as DLLs foram incluídas manualmente no repositório.
- O projeto usa **código-fonte incluído diretamente** para `Ionic.Zlib`, `FastColoredTextBoxNS` e `MaxMind.MaxMindDb`, em vez de referências de pacote.
- A versão exata do `Jint` não está declarada explicitamente. É necessário identificar pela inspeção do binário.
- A dependência de `unsafe blocks` sugere que existem trechos de código com manipulação direta de memória que precisarão de atenção especial na migração.
