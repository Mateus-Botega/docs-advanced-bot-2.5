# Documentação de reengenharia — AdvancedBot 2.4.5

Este diretório descreve o comportamento observado no código-fonte C# para orientar uma reescrita em Java com Spring Boot. O sistema atual é um cliente desktop WinForms para Minecraft, com múltiplas sessões, automação, interpretação de protocolo, mundo local, comandos, macros, extensões e visualizador OpenGL.

## Como usar

1. Comece por [Arquitetura Geral](01-Arquitetura-Geral.md) e [Bootstrap](02-Bootstrap.md).
2. Consulte o domínio correspondente à funcionalidade a migrar.
3. Use [Arquitetura Java](16-Arquitetura-Java.md) como destino, não como descrição do legado.
4. O inventário em `anexos/Inventario-de-Codigo.md` é a cobertura de classes próprias analisadas; bibliotecas vendorizadas não fazem parte do domínio.

## Escopo e cautelas

- O projeto atual é .NET Framework 4.6.2, WinForms e x86; não é um servidor Minecraft.
- Os protocolos cobrem 1.5.2, 1.7, 1.7.10, 1.8, 1.9 e 1.12.1. IDs e formatos variam por versão.
- Credenciais, proxies, plugins binários e automações são superfícies sensíveis. A migração deve usar armazenamento seguro, autorização explícita e limites operacionais.
- Não há testes automatizados encontrados. A especificação precisa ser consolidada em testes de contrato antes de substituir o legado.

## Mapa

| Área | Documento |
|---|---|
| Vocabulário | [00-Glossario.md](00-Glossario.md) |
| Inicialização | [02-Bootstrap.md](02-Bootstrap.md) |
| Núcleo | [03-Core/Main.md](03-Core/Main.md) |
| Transporte e autenticação | [04-Rede/README.md](04-Rede/README.md) |
| Codec/protocolo | [05-Protocolo-Minecraft/README.md](05-Protocolo-Minecraft/README.md) |
| Migração alvo | [16-Arquitetura-Java.md](16-Arquitetura-Java.md) |
