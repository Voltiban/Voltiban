# Dev Diaries — TryHackMe Writeup

**Categoria:** OSINT / Reconhecimento Passivo  
**Dificuldade:** Fácil  
**Objetivo:** Recuperar informações sobre um desenvolvedor freelancer que desapareceu com o código-fonte do cliente.

---

## Contexto

A empresa Marvenly contratou um desenvolvedor freelancer para criar seu site. O desenvolvedor sumiu sem entregar o código-fonte. A única informação disponível é o domínio principal: `marvenly.com`.

---

## Reconhecimento

### Q1 — Subdomínio de desenvolvimento

O enunciado sugere que rastros do processo de desenvolvimento podem existir online. Uma das formas mais eficazes de encontrar subdomínios sem enviar pacotes para o alvo é consultar o histórico de certificados SSL públicos.

**Ferramenta:** [crt.sh](https://crt.sh)

```
https://crt.sh/?q=marvenly.com
```

**Resultado:** `uat-testing.marvenly.com`

---

### Q2 — GitHub Username do desenvolvedor

Com o subdomínio em mãos, a próxima etapa é buscar por repositórios públicos relacionados ao projeto. Desenvolvedores freelancers frequentemente usam o GitHub para versionar código.

**Busca:**
```
https://github.com/search?q=marvenly
```

**Resultado:** perfil `notvibecoder23` com repositório do projeto Marvenly.

---

### Q3 — Email do desenvolvedor

O Git registra automaticamente o nome e email do autor em cada commit. Mesmo que o código tenha sido deletado, o histórico de commits permanece. Adicionando `.patch` ao final da URL de qualquer commit, os metadados ficam expostos.

**Técnica:**
```
https://github.com/notvibecoder23/[repo]/commit/[hash].patch
```

**Resultado:** `freelancedevbycoder23@gmail.com`

---

### Q4 — Motivo da remoção do código

Encontrado diretamente no histórico de commits do repositório.

**Resultado:** `The project was marked as abandoned due to a payment dispute`

---

### Q5 — Flag escondida

Localizada dentro de um arquivo no repositório (README ou arquivo de configuração).

**Flag:** `THM{g1t_h1st0ry_n3v3r_f0rg3ts}`

---

## Lições aprendidas

- **crt.sh** é uma fonte poderosa de reconhecimento passivo — certificados SSL são registros públicos permanentes.
- O Git **nunca esquece**. Mesmo após deletar arquivos, o histórico de commits preserva metadados do autor (nome, email) acessíveis via `.patch`.
- Desenvolvedores freelancers frequentemente vinculam identidades reais a projetos de clientes sem perceber.

---

## Ferramentas utilizadas

| Ferramenta | Uso |
|---|---|
| crt.sh | Enumeração de subdomínios via certificados SSL |
| GitHub Search | Busca de repositórios relacionados ao alvo |
| Git `.patch` | Extração de metadados de commits |
