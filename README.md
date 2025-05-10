# Compreendendo o Básico de Redes e Anti-Cheat no MTA:SA

## 🚨 Nota Importante
**Apenas educacional** – esse post é para fins de aprendizado e compartilhamento de conceitos, não para vazar dados sensíveis ou incentivar trapaças em servidores oficiais.

**Objetivo Principal** – para ajudar os entusiastas a aprimorar suas habilidades em rede e segurança de jogos.

**Sem Responsabilidade** – Não sou responsável por banimentos, erros ou incidentes do tipo "seu computador pegou fogo". Os jogos têm regras e penalidades rigorosas.

**Seja gentil e doce. ❤️** – use esse conhecimento para caçar bugs, crescer pessoalmente e ajudar a comunidade—não para estragar servidores ou queimar sua própria reputação.

## 1. O Componente Principal: netc.dll no Lado do Cliente
netc.dll é o coração da rede do MTA:SA e do sistema anti-trapaça. Ele conecta os seguintes componentes:

_**RakNet:**_
• Inicializa sessões UDP/TCP
• ACK/NACK, ordenação de pacotes
• Perfuração e compressão NAT

_**SharedUtil:**_
• SString/WString para ASCII e Unicode
• BitStream para serialização e deserialização de dados
• …e mais de [mta-blue on GitHub]

_**Anti-Cheat:**_
• Hooks em modo kernel + verificações de integridade em modo usuário
• Geração de serial e prevenção de spoof

## 2. Anti-Cheat Externo (Modo Kernel)
Funciona lá nas profundezas escuras do kernel do sistema operacional através de um driver assinado:

Injeção de DLL e Hooking

Interceptação de API: CreateProcess, OpenProcess, ZwProtectVirtualMemory, etc.

Escaneamento de Seções de Código: encontra blocos de código não autorizados.

Análise de Call-Stack: fica de olho em cadeias de chamadas suspeitas.

Os Hooks de Callback do IRP: interceptam a entrada/saída de arquivos/dispositivos pra pegar ferramentas de trapaça externas.

Objetivos:

Impedir ferramentas de trapaça em nível de kernel de forma eficaz.

Impedir soluções alternativas em user-mode.

Impedir tentativas de engenharia reversa.

- Por que os trapaceiros não vão para o kernel? Porque eles não conseguem lidar com a raiz de todo o mal! 🌱

## 3. Anti-Cheat Interno (User Mode)
Vive dentro do binário do jogo, garantindo a integridade da lógica:

Validação de Integridade

Validação de checksum em regiões críticas de memória.

ValidatePacket & ValidateRPC para checar os dados que entram e saem.

Ponteiros Encriptados

XOR-obfuscação de ponteiros de chave para dificultar leituras diretas da memória.

Escaneamento de Memória Dinâmica

Caça por assinaturas de cheat codes conhecidas em heaps/módulos dinâmicos.

Sandboxing

Executa rotinas sensíveis em "gaiolas" de memória isoladas.

Rastreamento de Serial

Protege o serial do jogador baseado em hardware contra adulteração.

Proteção contra manipulação de pacotes

Garante que os eventos Lua, chamadas RPC e pacotes do jogo não sejam falsificados.

## 4. Conectando via RakNet e a interface CNet
MTA:SA usa um RakNet bem modificado por trás das câmeras—então não espere nenhum hack de RakNet “original” aqui!

4.1. Inicialização: CNet::StartNetwork

https://prnt.sc/t1FlddTFB_qZ

https://prnt.sc/gf155L9B4xGu

https://prnt.sc/h4LOnzaGAE1P

Converte o endereço do servidor em uma string usando inet_addr

Chama RakPeer::Connect(ip, port) e guarda o identificador

Se der falha ou desconectar, o CNet::StopNetwork desliga tudo de boa.

4.2. Enviando dados: CNet::SendPacket

https://prnt.sc/FCtp2dF8sAGu

Usa BitStream (little-endian) para serializar

Executa verificações de integridade e antimanipulação

Criptografa o payload e, em seguida, chama RakPeer::Send

4.3. Recebendo Dados: CNet::ReceivePacket

Descriptografa, valida e em seguida despacha para a lógica do jogo.

4.4. Desligar: CNet::StopNetwork

Desliga de boa, libera os recursos

4.5. Principais Métodos da RakPeerInterface

Send(...): manipula números de sequência, confiabilidade, cabeçalhos UDP/TCP

Receive(): leitura de socket de baixo nível → Pacote*

## 5. Geração de Serial e Seus Mistérios
Na inicialização: GenerateSerial() coleta IDs de hardware via WinAPI (GUIDs de volume, MACs, BIOS, CPU, GPU, etc.).

Consultas do MountPointManager: usos

``
RtlInitUnicodeStringEx("\\?\Volume{GUID}") + NtOpenFile + DeviceIoControl(IOCTL_MOUNTMGR_QUERY_POINTS) para buscar todos os pontos de montagem.
``

Criptografar e Ofuscar: XOR, deslocamentos de bits, transformações dependentes de posição embaralham cada identificador.

Combine: concatena as partes em um serial final determinístico.

Store Locally: Salva na memória local

Rebuild: O jogo executa o mesmo comando para reconfigurar um serial com funções diferentes e o compara ao serial original antes de estabelecer a conexão.

Server-Side Check: Ao se conectar, o servidor recalcula e verifica seu serial, possivelmente bloqueando as incompatibilidades.

## 6. Perguntas Frequentes & Contos de Precaução
P1: Posso simplesmente alterar as informações do meu hardware para falsificar o serial?
R: Tecnicamente sim — editar o registro, MachineGuid e algumas informações como rede, etc., mas você vai quebrar seu PC, talvez você seja investigado por outros jogos e arriscar corromper o Windows, e ainda ser banido pelo anti-cheat global.

P2: Posso usar o serial de outra pessoa?
R: Servidores MTA:SA geralmente colocam duplicatas na lista negra. Alguns servidores privados permitem, mas isso depende dos administradores do servidor alterarem o arquivo mtaserver.conf.

P3: Posso contornar um banimento global?
R: Extremamente difícil. Você precisaria bloquear todas as verificações de saída para servidores oficiais, limpar arquivos em cache/rastros/travamentos, falsificar impressões digitais de rede… e ainda arriscar ser colocado na lista negra.

P4: sem mais perguntas...?

## 7. Contornando os Mecanismos Anti-Cheat
Como mencionei no início do artigo, o sistema de segurança do MTA:SA opera em duas camadas: uma no Modo Kernel e outra no Modo Usuário. Decifrar todo o código requer um esforço semelhante a tentar entender um texto de filosofia às 3 da manhã.

No entanto, para fins puramente educacionais, discutiremos teoricamente como contornar algumas das proteções, sem mencionar métodos de implementação, é claro:

🧱 Primeiro: Suprimindo os Relatórios
Todos os relatórios que o jogo envia para os servidores centrais devem ser suprimidos.

Isso geralmente é feito interceptando comunicações ou modificando pontos de transmissão, impedindo a detecção da sua trapaça.

Esses relatórios podem incluir informações sensíveis como atividade de memória, processos suspeitos e registros de adulteração.

🕵️ ♂️ Segundo: Bloqueando o Rastreamento Interno
O jogo usa funções "espiãs" que:

Leem a RAM.

Escaneiam processos.

Pode até ler o conteúdo do seu CD-ROM ou suas pastas pessoais (juro, mesmo que seja spyware).

O jogo é ilegal. O que o impede de espionar? xd

🔧 Terceiro: Pontos de Hooking & Patching
Muitas funções de segurança dependem de uma ou mais funções centrais.

Se você conseguir rastrear essas funções, elas podem ser "hookadas" ou até mesmo desabilitadas temporariamente.

Mas cuidado! Uma simples alteração nessas funções pode resultar em:

Expulsão direta do servidor.

Ou até mesmo um banimento permanente (colocado na lista negra mais rápido do que sua ex te bloqueia).

🧠 Nota Importante:
Não se engane! A segurança não é ingênua. Existem camadas ocultas, criptografia avançada e monitoramento comportamental, não apenas monitoramento de memória.

Um ataque bem-sucedido requer um entendimento muito profundo da estrutura do jogo e de como ele funciona. "Não é um crack de copiar e colar e pronto." 😒

⚠️ Aviso Final:
Esta seção é apenas para fins de conscientização. Não incentivamos trapaças ou manipulação em jogos. Respeitar as regras do jogo e da comunidade é sempre melhor do que ser banido e colocado na lista negra.

"Seja um hacker ético, não um hacker de desenho animado." 😎

-----

Discord Server: https://discord.gg/rntpdB4aqJ

Creditos: https://github.com/devilmare/mtasa-netc-reversing/
