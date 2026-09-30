# Checklist de migração — bot WhatsApp Kubo Loko (local → VPS)

Segue por ordem. Cada item tem o comando/local exato. Ver `WHATSAPP_BOT_DEPLOYMENT.md` para o guia completo de cada passo de setup do VPS.

## Antes de começar

- [ ] **Backup do workflow n8n (export JSON)**
  No n8n local (http://localhost:5678), abrir o workflow "Kubo Loko - Atendimento WhatsApp (IA)" → menu (⋯) → **Download** (exporta o JSON). Guardar em `C:\Kubo\Docker\backups\` com a data, ex. `workflow-kubo-whatsapp-2026-09-30.json`.

- [ ] **Backup das credenciais do n8n (anotar, não exportar)**
  As credenciais (Google Gemini, etc.) não saem no export por segurança. Anotar à parte, em local seguro, quais credenciais existem e onde obter os valores de novo (ex. API key do Gemini em https://aistudio.google.com/apikey).

- [ ] **Backup da configuração WAHA**
  Anotar: nome da sessão (`default`), número de negócio (351927789544), webhook configurado (`/webhook/kubo-wa`, evento `message`), API key, credenciais do dashboard/swagger (já documentadas no `CLAUDE.md`).

- [ ] **Backup da base de dados Postgres (opcional mas recomendado)**
  ```powershell
  docker exec kubo-postgres pg_dump -U kubo n8n > C:\Kubo\Docker\backups\postgres-backup-2026-09-30.sql
  ```

## Setup do VPS

- [ ] Criar conta Hetzner Cloud e o servidor CX23 (`WHATSAPP_BOT_DEPLOYMENT.md`, secção 3, passos 1–2)
- [ ] Configurar acesso SSH por chave e criar utilizador `kubo` (passos 3–4)
- [ ] Instalar Docker + Docker Compose (passo 5)
- [ ] Configurar UFW, fail2ban, atualizações automáticas (passos 6–8)
- [ ] Apontar DNS (`n8n.` e `waha.` subdomínios) para o IP do servidor (passo 9)
- [ ] Copiar `docker-compose.vps.yml`, `Caddyfile` e `.env` (a partir de `.env.vps.example`) para o servidor (passos 10–11)

## Deploy e testes no VPS

- [ ] `docker compose up -d` no servidor, confirmar todos os containers "Up" (`docker compose ps`)
- [ ] Confirmar HTTPS a funcionar em `https://n8n.kuboloko.pt` e `https://waha.kuboloko.pt`
- [ ] Criar conta de owner do n8n no VPS
- [ ] Importar o JSON do workflow exportado (Import from File)
- [ ] Recriar a credencial do Google Gemini no n8n do VPS
- [ ] Criar sessão WAHA `default` e ler QR code com o WhatsApp de negócio (351927789544)
- [ ] Configurar o webhook da sessão WAHA para `http://n8n:5678/webhook/kubo-wa`, evento `message`
- [ ] Ativar o workflow no n8n do VPS (toggle "Active")
- [ ] **Testar ponta a ponta**: enviar mensagem de outro número, confirmar execução em Executions e resposta recebida no WhatsApp
- [ ] Testar o fluxo de alerta ao dono (pedir para falar com humano, ou mencionar orçamento) e confirmar que 351931313362 recebe o alerta

## Corte (cutover)

- [ ] **Monitorizar 24–48h** com o VPS e o Docker local a correr **ambos**, mas só o WAHA do VPS com a sessão WhatsApp ativa (nunca ter duas sessões WAHA ligadas ao mesmo número ao mesmo tempo — desligar a sessão local antes de ligar a do VPS)
  - Confirmar que não há mensagens duplicadas nem perdidas
  - Rever logs diariamente: `docker logs kubo-waha -f` e `docker logs kubo-n8n -f` no VPS
- [ ] Atualizar qualquer referência a `localhost:5678` ou `localhost:3000` usada noutros sítios (ex. bookmarks, notas) para os novos domínios do VPS
- [ ] **Desligar os containers locais** (não apagar, para já, por segurança):
  ```powershell
  cd C:\Kubo\Docker
  docker compose stop
  ```
- [ ] Confirmar que o bot continua a responder normalmente só com o VPS, durante pelo menos mais 48h
- [ ] Só depois de confirmado — opcional — remover os containers e volumes locais:
  ```powershell
  docker compose down -v
  ```

## Pós-migração

- [ ] Atualizar o `CLAUDE.md` do projeto com os novos URLs (n8n e WAHA do VPS) em vez de `localhost`
- [ ] Guardar as credenciais do VPS (IP, chave SSH, passwords do `.env`) num local seguro (gestor de passwords)
- [ ] Agendar lembrete mensal para verificar espaço em disco e logs do VPS (`df -h`, `docker system prune -a`)
- [ ] Confirmar que o backup do `.env` do VPS está guardado fora do servidor (ex. gestor de passwords), já que não vai para o GitHub
