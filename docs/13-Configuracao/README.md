# Configuração e persistência

`Program.Config` é uma árvore NBT gravada em `conf.dat`. Armazena preferências de interface e cliente (knockback, auto-reconnect, viewer, FPS, chat), além de histórico como servidor/usuários e comandos de login. `Properties/Settings` é suporte .NET. Arquivos auxiliares incluem listas de proxy, plugins, scripts e logs em `errlogs`.

## Java

Configuração de implantação: `application.yml`, variáveis de ambiente e secret manager. Preferências por usuário/sessão: banco relacional ou documento versionado. Oferecer importador somente-leitura de NBT e migração explícita; não guardar senhas, tokens ou proxies sensíveis em arquivo plano.
