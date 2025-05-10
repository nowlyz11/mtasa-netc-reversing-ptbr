# Compreendendo o Básico de Redes e Anti-Cheat no MTA:SA

## 🚨 Important Note
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

## 6. FAQs & Cautionary Tales
Q1: Can I just change my hardware info to spoof the serial?
A: Technically yes—edit registry, MachineGuid, and some information like network, etc., but you’ll break your PC, maybe you’ll get probed with other games and risk corrupting Windows, and still get banned by global anti-cheat.

Q2: Can I use someone else’s serial?
A: MTA:SA servers generally blacklist duplicates. Some private servers allow it, but that’s up to server admins to tamper with mtaserver.conf.

Q3: Can I bypass a global ban?
A: Extremely difficult. You’d need to block every outbound check to official servers, wipe cached files/traces/crashs, spoof network fingerprints… and still risk blacklisting.

Q4: no more questions...?

## 7. Bypassing the Anti-Cheat Mechanisms
As I mentioned at the beginning of the article, MTA:SA's security system operates in two layers: one in Kernel Mode and the other in User Mode. Deciphering the entire code requires an effort similar to trying to understand a philosophy text at 3 a.m.

However, for purely educational purposes, we will discuss theoretically how to bypass some of the protections, without mentioning implementation methods, of course:

🧱 First: Suppressing the Reports
All reports that the game sends to the central servers must be suppressed.

This is often done by intercepting communications or modifying transmission points, preventing the detection of your cheating.

These reports may include sensitive information such as memory activity, suspicious processes, and tamper logs.

🕵️ ♂️ Second: Blocking Internal Tracking
The game uses "spy" functions that:

Read RAM.

Process Scanning.

It might even read the contents of your CD-ROM or your personal folders (I swear, even if it's spyware).

The game is illegal. What's stopping it from spying? xd

🔧 Third: Hooking & Patch Points
Many security functions rely on one or more central functions.

If you can track these functions, they can be hooked or even temporarily disabled.

But beware! A simple change to these functions could result in:

Direct expulsion from the server.

Or even a permanent ban (blacklisted faster than your ex blocks you).

🧠 Important Note:
Don't be fooled! Security isn't naive. There are hidden layers, advanced encryption, and behavioral monitoring, not just memory monitoring.

A successful attack requires a very deep understanding of the game's structure and how it works. "It's not a copy-paste crack and that's it." 😒

⚠️ Final Warning:
This section is for awareness purposes only. We do not encourage cheating or manipulation in games. Respecting the rules of the game and the community is always better than being banned and blacklisted.

"Be an ethical hacker, not a cartoon hacker." 😎

-----

Discord Server: https://discord.gg/rntpdB4aqJ
