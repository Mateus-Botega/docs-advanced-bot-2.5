# Estratégia de testes

Não foram encontrados projetos ou arquivos de testes automatizados. A primeira entrega da migração deve transformar comportamento observável em testes.

1. Unitários: VarInt, buffers, NBT, AABB, física, inventário, parser de comandos/scripts e pathfinding.
2. Contrato: fixtures de pacotes por versão, incluindo compressão/AES, login, chunk, inventário, chat e teleporte.
3. Integração: servidor Minecraft de teste compatível, proxy simulado e endpoints de autenticação falsos.
4. Concorrência: vários bots, cancelamento, reconexão, falhas de rede e fechamento limpo.
5. E2E: criação de sessão autorizada, telemetria e auditoria de comandos.

Critério de paridade: para uma fixture/cenário aprovado, o estado canônico e os pacotes emitidos devem ser equivalentes, sem exigir reproduzir threads, UI ou bugs do legado.
