# Infraestrutura, dependências e segurança

Projeto: `AdvancedBot_Crack.csproj`, executável WinForms x86 em .NET 4.6.2. Referências diretas: Newtonsoft.Json, System.Management/XML/DataVisualization; binários incluem Jint, Ionic.Zlib, MaxMind e Transitions. Há fontes vendorizadas de Jint, Ionic, FastColoredTextBox e MaxMind — não são domínio do bot e não devem ser migradas linha a linha.

Serviços legados: HTTP, DNS SRV, TCP/proxy, autenticação Mojang, filesystem, WGL/OpenGL e verificação HWID. `Protection/HWIDChecker` e `Crypto/AesEncryption` tratam proteção/licença/ABP, não a regra de jogo.

## Java

Java 21+, Spring Boot 3.x, Netty, Jackson, Micrometer/OpenTelemetry, Testcontainers e banco PostgreSQL são uma base sugerida. Fixar versões, gerar SBOM, usar TLS quando aplicável, secrets externos, timeouts/retries limitados, logs sem credenciais e rate limits por usuário/conta/servidor.
