# Migração do bot de WhatsApp Kubo Loko para um servidor 24/7

Guia completo para tirar o bot (n8n + WAHA) do PC local e pôr a correr num VPS (servidor virtual) sempre ligado, para que deixe de parar quando o computador está desligado ou em suspensão.

Escrito para quem está a aprender Linux/Docker — todos os comandos são para copiar/colar.

---

## OPÇÃO A: Oracle Cloud Always Free (100% Grátis - Recomendado)

### Porquê Oracle Cloud?

- **100% grátis para sempre** (não expira, não é trial)
- **2 OCPU + 12GB RAM** (recursos muito generosos)
- **200GB storage**
- **10TB/mês egress**
- **Regions EU disponíveis** (Frankfurt, Amsterdam, Zurich)
- **Suficiente para n8n + WAHA + Postgres + Redis + Caddy**

### Passo 1: Criar conta Oracle Cloud

1. Aceder a: https://signup.oraclecloud.com/
2. Preencher dados pessoais e de contacto
3. **Requer cartão de crédito para verificação** (não cobram nada, é só validação)
4. Aguardar aprovação (pode demorar algumas horas a 1-2 dias)

### Passo 2: Criar VM Instance (Always Free)

1. No dashboard Oracle Cloud, clicar em **"Create Instance"**
2. Configurar:
   - **Compartment in:** root compartment (default)
   - **Name:** `kubo-loko-whatsapp-bot`
   - **Region:** Frankfurt (eu-frankfurt-1) ou Amsterdam (eu-amsterdam-1)
   - **Availability Domain:** qualquer uma disponível
   - **Image and Shape:** clicar em "Change Image"
     - **Image:** Ubuntu 22.04 LTS (ou 24.04 se disponível)
   - **Shape:** clicar em "Change Shape"
     - **Series:** Ampere (ARM)
     - **Shape:** VM.Standard.A1.Flex
     - **OCPUs:** 2
     - **Memory:** 12 GB
   - **Networking:**
     - Virtual cloud network compartment: root compartment
     - Subnet: qualquer uma disponível
     - **Assign a public IPv4 address:** ✅ (checked)
   - **Add SSH key:**
     - Escolher "Generate a key pair for me" OU "Choose an existing public key"
     - Se gerares nova: fazer download do ficheiro `.key` e guardar em segurança
     - Se usares chave existente: colar o conteúdo de `id_ed25519.pub`
   - **Boot volume:**
     - Check: "Custom size"
     - Size: **200 GB** (máximo do free tier)
3. Clicar em **"Create"**

### Passo 3: Configurar Security List (PORTAS)

**MUITO IMPORTANTE:** Oracle bloqueia todas as portas por defeito. Tens de abrir manualmente:

1. No dashboard da instância, clicar em **Subnet** (link azul)
2. Clicar em **Security List** (link azul, ex. "Default Security List")
3. Clicar em **Add Ingress Rules**
4. Adicionar 3 regras:

   **Regra 1 - SSH:**
   - Source Type: CIDR
   - Source CIDR: `0.0.0.0/0`
   - Destination Port Range: `22`
   - Description: `SSH`

   **Regra 2 - HTTP:**
   - Source Type: CIDR
   - Source CIDR: `0.0.0.0/0`
   - Destination Port Range: `80`
   - Description: `HTTP`

   **Regra 3 - HTTPS:**
   - Source Type: CIDR
   - Source CIDR: `0.0.0.0/0`
   - Destination Port Range: `443`
   - Description: `HTTPS`

5. Clicar em **Add Ingress Rules**

### Passo 4: SSH para a VM

**Se usaste chave SSH Oracle:**

```powershell
# No Windows PowerShell
ssh -i C:\caminho\para\a\chave.key ubuntu@SEU_IP_PUBLICO
```

**Se usaste a tua chave existente:**

```powershell
ssh ubuntu@SEU_IP_PUBLICO
```

Substitui `SEU_IP_PUBLICO` pelo IP que aparece no dashboard Oracle (copia-o de lá).

### Passo 5: Continuar deployment

Agora segue o resto deste guia a partir do **Passo 4** (atualizar sistema, criar utilizador, instalar Docker, etc.).

**NOTA IMPORTANTE:** Oracle usa ARM64 (Ampere). n8n e WAHA funcionam perfeitamente em ARM64 — não precisas de mudar nada.

---

## Troubleshooting Oracle Cloud

### "Out of capacity" ao criar VM

**Problema:** Oracle não tem capacidade disponível na region escolhida.

**Solução:**
- Tentar outra region: Frankfurt → Amsterdam → Zurich
- Tentar em horários diferentes (madrugada PT tem mais disponibilidade)
- Tentar shape de fallback: VM.Standard.E2.1.Micro (1GB RAM, também grátis, mas menos potente)

### SSH não funciona

## 1. Recomendação de fornecedor VPS

Comparação entre os 4 fornecedores pedidos (preços de setembro de 2026):

| Fornecedor | Plano | vCPU | RAM | SSD | Preço/mês | Localização EU |
|---|---|---|---|---|---|---|
| **Hetzner** ⭐ | CX23 | 2 | 4 GB | 40 GB NVMe | ~€5,49 | Alemanha / Finlândia |
| Contabo | Cloud VPS 10 | 4 | 8 GB | 75 GB NVMe | ~$4,50 (~€4,20) | Alemanha |
| DigitalOcean | Basic (1 vCPU/1GB) | 1 | 1 GB | 25 GB | $6 | Amesterdão |
| Linode (Akamai) | Nanode 1GB | 1 | 1 GB | 25 GB | $5 | Londres / Frankfurt |

### Recomendação: **Hetzner Cloud (CX23)**

**Porquê:**
- Melhor relação preço/desempenho dos quatro — 2 vCPU + 4 GB RAM é mais do que suficiente para n8n + WAHA + Postgres + Redis com folga.
- Data centers na Alemanha e Finlândia — baixa latência para Portugal e conformidade RGPD/UE sem complicações.
- Painel de controlo (Hetzner Cloud Console) muito simples, documentação excelente em inglês, snapshots e backups automáticos com um clique.
- Rede e IPv4 incluídos sem custos escondidos; firewall de rede gratuito integrado no painel (além do UFW dentro do servidor).
- Comunidade grande — qualquer erro que aparecer já foi resolvido em fórum ou Stack Overflow.

**Contras:**
- Suporte só por email/ticket (sem chat ao vivo) — mas raramente é necessário.
- Preços subiram ligeiramente em 2026 (o CX23 substituiu o CX22 mais barato).

**Alternativa mais barata:** Contabo (Cloud VPS 10) dá mais RAM/CPU por menos dinheiro, mas tem reputação de rede mais lenta e suporte mais fraco — só compensa se o orçamento for mesmo apertado.

**Link para criar conta:** https://www.hetzner.com/cloud/

**Especificações a escolher ao criar o servidor:**
- Localização: Falkenstein ou Nuremberg (Alemanha) ou Helsinki (Finlândia)
- Imagem: **Ubuntu 24.04 LTS**
- Tipo: **CX23** (2 vCPU, 4 GB RAM, 40 GB SSD)
- Autenticação: **SSH key** (ver secção 2)

---

## 2. Estimativa de custos e tempo

### Custos mensais
| Item | Custo |
|---|---|
| VPS Hetzner CX23 | ~€5,49/mês |
| Domínio (opcional, ex. `.pt` ou subdomínio de `kuboloko.pt` já existente) | €0 se usares subdomínio do domínio que já tens |
| Backups automáticos Hetzner (opcional, +20%) | ~€1,10/mês |
| **Total estimado** | **~€5,50–€6,60/mês** |

### Tempo de setup
- Criar conta + servidor: 15 min
- Ligar por SSH + instalar Docker: 20 min
- Subir n8n + WAHA, configurar domínio/HTTPS: 30–45 min
- Importar workflow do n8n + reler QR code do WhatsApp: 15 min
- Testar ponta a ponta: 15–20 min
- **Total: ~2 a 2,5 horas** na primeira vez

### Manutenção contínua
- Atualizações de segurança automáticas (unattended-upgrades) — 0h/mês de esforço manual
- Verificação ocasional de logs/espaço em disco — ~15–30 min/mês
- Backup manual do export do workflow n8n — 5 min/mês (recomendado após alterações importantes)

---

## 3. Guia de instalação passo a passo

### Passo 1 — Criar o servidor

1. Cria conta em https://www.hetzner.com/cloud/
2. No painel, **New Project** → **Add Server**.
3. Escolhe: localização (Nuremberg ou Falkenstein), imagem **Ubuntu 24.04**, tipo **CX23**.
4. Em "SSH keys", clica em **Add SSH key** (ver Passo 2 antes de continuares se ainda não tens uma).
5. Dá um nome ao servidor, ex. `kubo-whatsapp-bot`, e clica **Create & Buy now**.
6. Anota o **IP público** do servidor (aparece no painel em poucos segundos).

### Passo 2 — Gerar e configurar a chave SSH (antes ou depois de criar o servidor)

No teu PC Windows, no PowerShell:

```powershell
ssh-keygen -t ed25519 -C "kuboloko-vps"
```

Prime Enter em todas as perguntas (fica em `C:\Users\<o-teu-user>\.ssh\id_ed25519`). Depois mostra a chave pública:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub
```

Copia o resultado e cola-o no painel Hetzner (New Server → SSH keys → Add SSH key), **antes** de criares o servidor. Se já criaste o servidor sem chave, podes adicioná-la depois em Hetzner Console → Security → SSH Keys, e reconstruir o servidor, ou seguir o Passo 3 com password e configurar a chave manualmente.

### Passo 3 — Ligar por SSH

```powershell
ssh root@SEU_IP_DO_SERVIDOR
```

Substitui `SEU_IP_DO_SERVIDOR` pelo IP anotado no Passo 1. Na primeira ligação, escreve `yes` para aceitar o fingerprint.

### Passo 4 — Atualizar o sistema e criar um utilizador não-root

Por segurança, não vamos usar o `root` no dia a dia.

```bash
apt update && apt upgrade -y
adduser kubo
usermod -aG sudo kubo
rsync --archive --chown=kubo:kubo ~/.ssh /home/kubo
```

Testa numa nova janela de terminal (sem fechar a atual) que consegues entrar como `kubo`:

```powershell
ssh kubo@SEU_IP_DO_SERVIDOR
```

Se entrar sem pedir password, está tudo certo. Continua os próximos passos autenticado como `kubo` (usa `sudo` quando precisares de privilégios).

### Passo 5 — Instalar Docker e Docker Compose

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker kubo
```

Sai da sessão SSH (`exit`) e volta a entrar para o grupo `docker` fazer efeito:

```powershell
ssh kubo@SEU_IP_DO_SERVIDOR
```

Confirma a instalação:

```bash
docker --version
docker compose version
```

### Passo 6 — Configurar a firewall (UFW)

Só vamos deixar passar SSH, HTTP e HTTPS (o n8n e o WAHA ficam atrás do Caddy, que trata do HTTPS — não expomos as portas 5678/3000 diretamente).

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

Confirma com `sudo ufw status` — deve mostrar as 3 regras como "ALLOW".

### Passo 7 — Instalar o fail2ban (proteção contra ataques de força bruta ao SSH)

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
```

### Passo 8 — Ativar atualizações automáticas de segurança

```bash
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

Escolhe "Yes" quando perguntar se queres atualizações automáticas.

### Passo 9 — Apontar o domínio para o servidor

Se já tens o domínio `kuboloko.pt` (ou outro), cria dois registos **A** no teu fornecedor de DNS (ex. onde o domínio está registado):

| Tipo | Nome | Valor |
|---|---|---|
| A | `n8n` | SEU_IP_DO_SERVIDOR |
| A | `waha` | SEU_IP_DO_SERVIDOR |

Isto cria `n8n.kuboloko.pt` e `waha.kuboloko.pt`. Espera 5–30 min para propagar (podes confirmar com `ping n8n.kuboloko.pt` no teu PC).

> Não tens domínio? Podes usar um serviço gratuito como DuckDNS (https://www.duckdns.org) para obter um subdomínio tipo `kuboloko.duckdns.org` apontado para o IP do servidor, e usar esse nas configurações abaixo.

### Passo 10 — Copiar os ficheiros do projeto para o servidor

No teu PC, dentro da pasta `C:\Kubo\Docker`:

```powershell
scp docker-compose.vps.yml kubo@SEU_IP_DO_SERVIDOR:~/docker-compose.yml
scp Caddyfile kubo@SEU_IP_DO_SERVIDOR:~/Caddyfile
scp .env.vps.example kubo@SEU_IP_DO_SERVIDOR:~/.env
```

### Passo 11 — Preencher as credenciais reais no servidor

Já ligado por SSH ao servidor:

```bash
nano ~/.env
```

Edita todos os valores `troca-esta-...` para passwords fortes e únicas (usa por exemplo `openssl rand -base64 24` para gerar cada uma), e ajusta `N8N_HOST` e `WAHA_HOST` para os domínios que configuraste no Passo 9. Guarda com `Ctrl+O`, Enter, depois `Ctrl+X` para sair.

Edita também o `Caddyfile` com os mesmos domínios:

```bash
nano ~/Caddyfile
```

### Passo 12 — Arrancar os containers

```bash
cd ~
docker compose up -d
```

Confirma que tudo está a correr:

```bash
docker compose ps
```

Deves ver `kubo-postgres`, `kubo-redis`, `kubo-n8n`, `kubo-waha` e `kubo-caddy`, todos com estado "Up" (o Caddy demora ~30-60s a emitir os certificados HTTPS na primeira vez).

### Passo 13 — Aceder ao n8n e reconfigurar o bot

1. Abre `https://n8n.kuboloko.pt` no browser. Faz login com `N8N_BASIC_AUTH_USER` / `N8N_BASIC_AUTH_PASSWORD` do `.env`.
2. Cria a tua conta de owner do n8n (primeira vez).
3. Importa o workflow exportado do n8n local (ver `MIGRATION_CHECKLIST.md`, passo de backup).
4. Reconfigura a credencial do Google Gemini (as credenciais não se exportam por segurança — tens de as recriar).
5. Abre `https://waha.kuboloko.pt/dashboard/` com `WAHA_DASHBOARD_USERNAME` / `WAHA_DASHBOARD_PASSWORD`, cria a sessão `default`, e lê o QR code com o WhatsApp do número de negócio (351927789544) — Definições → Dispositivos ligados → Ligar dispositivo.
6. Configura o webhook da sessão WAHA para `http://n8n:5678/webhook/kubo-wa` (nome de serviço interno Docker, tal como no ambiente local — não muda).
7. Ativa o workflow no n8n (toggle "Active").

### Passo 14 — Testar ponta a ponta

Envia uma mensagem de outro número para o WhatsApp de negócio e confirma:
- Aparece uma execução nova em `https://n8n.kuboloko.pt` → Executions.
- Recebes a resposta automática no WhatsApp.
- Se pedires para falar com humano, o número pessoal do dono (351931313362) recebe o alerta.

---

## 4. Segurança — resumo do que já está aplicado

| Medida | Onde |
|---|---|
| Login SSH só por chave (sem password) | Passo 4 — usa `PasswordAuthentication no` em `/etc/ssh/sshd_config` se quiseres reforçar ainda mais, depois `sudo systemctl restart ssh` |
| Firewall UFW (só 22, 80, 443 abertos) | Passo 6 |
| fail2ban (bloqueia IPs com tentativas falhadas de SSH) | Passo 7 |
| Atualizações automáticas de segurança | Passo 8 |
| HTTPS automático (Let's Encrypt via Caddy) | Passo 12 |
| n8n protegido por basic auth + HTTPS | `.env` / Caddyfile |
| WAHA protegido por API key + basic auth no dashboard | `.env` |
| Portas do n8n/WAHA não expostas diretamente à internet (só via Caddy) | `docker-compose.vps.yml` |

Para reforçar o SSH (opcional, fazer só depois de confirmar que a chave funciona bem):

```bash
sudo nano /etc/ssh/sshd_config
# alterar: PasswordAuthentication no
sudo systemctl restart ssh
```

---

## 5. Resolução de problemas (troubleshooting)

**`docker compose up -d` falha com erro de porta ocupada**
→ Confirma que nada mais está a usar as portas 80/443: `sudo ss -tulpn | grep -E ':80|:443'`

**Caddy não consegue emitir certificado HTTPS**
→ Confirma que o DNS já propagou: `ping n8n.kuboloko.pt` deve devolver o IP do servidor. Vê os logs: `docker logs kubo-caddy --tail 50`

**WAHA perde a sessão do WhatsApp depois de reiniciar**
→ Confirma que os volumes `waha_sessions` e `waha_media` existem: `docker volume ls`. Nunca corras `docker compose down -v` (o `-v` apaga os volumes).

**n8n não liga ao Postgres**
→ Vê logs: `docker logs kubo-n8n --tail 50`. Confirma que `POSTGRES_PASSWORD` é igual em ambos os serviços (vem do mesmo `.env`).

**Quero ver os logs de um serviço em tempo real**
```bash
docker logs kubo-waha -f
docker logs kubo-n8n -f
```

**O servidor ficou sem espaço em disco**
```bash
df -h
docker system prune -a
```
(`docker system prune -a` remove imagens Docker não usadas — não apaga os volumes de dados.)

**Preciso de reiniciar tudo**
```bash
cd ~
docker compose restart
```

**Preciso de atualizar as imagens (n8n, WAHA) para a versão mais recente**
```bash
cd ~
docker compose pull
docker compose up -d
```
