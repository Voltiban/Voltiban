# Operation Slither — TryHackMe Writeup

**Categoria:** OSINT / Threat Intelligence  
**Dificuldade:** Fácil-Médio  
**Objetivo:** Rastrear os três operadores do grupo hacker "Sneaky Viper" a partir de um único handle encontrado em um fórum.

---

## Contexto

A empresa TryTelecomMe descobriu que seu banco de dados estava sendo vendido online pelo grupo **Sneaky Viper**. O único dado disponível é o handle do primeiro operador: `@v3n0mbyt3_`.

---

## Task 1 — Operador 1: v3n0mbyt3_

### Plataforma alternativa

**Ferramenta:** Sherlock — busca de username em múltiplas plataformas simultaneamente.

```bash
sherlock v3n0mbyt3_
```

Entre os resultados, o perfil no **Threads** se destacou com atividade real.

**Resultado Q1:** `threads`

### Flag — Base64 em reply

Vasculhando as replies do perfil no Threads, um segundo usuário (`_myst1cv1x3n_`) deixou uma string suspeita nos comentários:

```
VEhNe3NsMXRoM3J5X3R3MzN0el80bmRfbDM0a3lfcjNwbDEzcyF9
```

Decodificando via Base64:

```bash
echo "VEhNe3NsMXRoM3J5X3R3MzN0el80bmRfbDM0a3lfcjNwbDEzcyF9" | base64 -d
```

**Flag:** `THM{sl1th3ry_tw33tz_4nd_l34ky_r3pl13s!}`

---

## Task 2 — Operador 2: _myst1cv1x3n_

### Identificação

O handle `_myst1cv1x3n_` foi descoberto na reply do Threads na task anterior.

**Resultado Q1:** `_myst1cv1x3n_`

### Plataforma alternativa e flag

O perfil no Threads linkava diretamente para o **Instagram**. No Instagram, uma das postagens linkava para o **SoundCloud**. Na descrição de uma das músicas no SoundCloud havia uma string Base64:

```bash
echo "[string]" | base64 -d
```

**Flag:** `THM{s0cm1nt_00ps3c_f1ng3r_m1scl1ck}`

---

## Task 3 — Operador 3: sh4d0wF4NG

### Identificação

Rastreando interações entre os perfis anteriores (curtidas, comentários, follows no SoundCloud), o terceiro operador foi identificado.

**Resultado Q1:** `sh4d0wF4NG`

### Plataforma alternativa

```bash
sherlock sh4d0wF4NG
```

Perfil ativo no **GitHub** com três repositórios relevantes, incluindo `red-team-infra` (Terraform/HCL), `evilginx2` (fork de framework de phishing) e `gophish` (toolkit de phishing).

**Resultado Q2:** `github`

### Flag — Commit history

No repositório `red-team-infra`, o histórico de commits continha credenciais e a flag embutida.

**Flag:** `THM{sh4rp_f4ngz_l34k3d_bl00dy_pw}`

---

## Lições aprendidas

- **OpSec ruim** é o maior inimigo de threat actors: vincular contas entre plataformas cria uma cadeia rastreável.
- **Sherlock** automatiza a busca de usernames em dezenas de plataformas — ferramenta essencial para OSINT de pessoas.
- **Base64** é frequentemente usado para ofuscar dados sensíveis em posts públicos — sempre verifique strings aparentemente aleatórias.
- A cadeia completa deste lab: `Threads → Instagram → SoundCloud → Base64 → Flag` — OSINT raramente termina na primeira plataforma.

---

## Ferramentas utilizadas

| Ferramenta | Uso |
|---|---|
| Sherlock | Busca de usernames em múltiplas plataformas |
| Base64 (terminal) | Decodificação de mensagens ofuscadas |
| GitHub | Análise de repositórios e histórico de commits |

```bash
# Instalar Sherlock
pip install sherlock-project --break-system-packages

# Encode/Decode Base64
echo "texto" | base64
echo "dGV4dG8=" | base64 -d
```
