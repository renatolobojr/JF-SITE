# 🤝 AjudaJF — Central de Solidariedade

Plataforma de resposta a emergências para famílias atingidas pelas chuvas em **Juiz de Fora, MG**. Conecta pessoas que precisam de ajuda com voluntários, pontos de doação e serviços técnicos, tudo sem servidor — 100% Google Sheets + static hosting.

---

## Índice

- [Visão geral](#visão-geral)
- [Estrutura de arquivos](#estrutura-de-arquivos)
- [Arquitetura técnica](#arquitetura-técnica)
- [Configuração do Google Sheets](#configuração-do-google-sheets)
- [Configuração do Apps Script](#configuração-do-apps-script)
- [Deploy no GitHub Pages](#deploy-no-github-pages)
- [API — Endpoints](#api--endpoints)
- [Páginas — Referência detalhada](#páginas--referência-detalhada)
- [Funcionalidades técnicas](#funcionalidades-técnicas)
- [Solução de problemas](#solução-de-problemas)
- [Manutenção da planilha](#manutenção-da-planilha)

---

## Visão geral

```
Usuário → GitHub Pages (HTML estático)
                ↕ JSONP / form POST
         Google Apps Script
                ↕
         Google Sheets (banco de dados)
```

Sem Node.js, sem banco de dados, sem mensalidade. O Apps Script atua como API serverless; o frontend é HTML/CSS/JS puro hospedado gratuitamente.

---

## Estrutura de arquivos

```
/
├── index.html          ← Página inicial (hub/linktree) com splash screen
├── pedidos.html        ← Pedidos de ajuda + pontos de doação + mapa
├── voluntarios.html    ← Cadastro e mapa de voluntários
├── vistoria.html       ← Solicitações de vistoria técnica de imóveis
├── riscos.html         ← Mapa de risco geológico/hidrológico (embed Google Maps)
├── Code.gs             ← Backend Google Apps Script (API)
└── README.md           ← Este arquivo
```

---

## Arquitetura técnica

### Por que JSONP?

O Apps Script retorna um redirect 302 → `script.googleusercontent.com`, o que impede `fetch()` normal por CORS. A solução é JSONP: o frontend injeta uma `<script src="URL&callback=fn">` e o Apps Script retorna `fn({...dados...})`. Para gravações (POST), o formulário é submetido para um `<iframe hidden>`, contornando o bloqueio de CORS sem proxy.

### Fluxo de leitura

```
renderAll()
  └─ sessionStorage hit? (TTL 5 min) → usa cache → renderiza
  └─ miss → jsonp(API) → salva cache → renderiza
```

### Fluxo de gravação

```
form.submit() → POST → Apps Script → appendRow() na planilha
iframe.onload() → confirma → sessionStorage.removeItem() → refreshAll(forceNetwork=true)
```

### Prefetch da index.html

Ao abrir a `index.html`, as 4 APIs são chamadas em `Promise.all` e salvas no `sessionStorage`. Quando o usuário navega para qualquer subpágina, os dados já estão prontos — zero lag.

---

## Configuração do Google Sheets

Crie uma planilha com **4 abas** com os nomes e colunas exatos abaixo. Os nomes são case-sensitive e sem espaços.

### Aba `requests`

| Coluna | Descrição |
|---|---|
| `id` | UUID gerado pelo frontend |
| `created_at` | ISO 8601 |
| `updated_at` | ISO 8601 |
| `name` | Nome do solicitante (opcional) |
| `phone` | Somente dígitos |
| `needs` | Necessidades separadas por vírgula |
| `details` | Detalhes livres |
| `delivery_needed` | `"true"` ou `"false"` |
| `neighborhood` | Bairro |
| `address` | Endereço completo |
| `lat` | Latitude (pode estar vazio) |
| `lng` | Longitude (pode estar vazio) |
| `location_precision` | `gps`, `geocoded` ou `manual` |
| `status` | `open` ou `closed` |

### Aba `donation_points`

| Coluna | Descrição |
|---|---|
| `id` | UUID |
| `created_at` | ISO 8601 |
| `updated_at` | ISO 8601 |
| `name` | Nome do ponto |
| `address` | Endereço |
| `neighborhood` | Bairro |
| `lat` | Latitude (pode estar vazio — geocodificado pelo frontend) |
| `lng` | Longitude (pode estar vazio) |
| `items` | Itens aceitos separados por vírgula |
| `hours` | Horário de funcionamento |
| `contact` | Telefone/contato |
| `notes` | Observações |
| `status` | `active` ou `inactive` |

> **Pontos sem lat/lng** são geocodificados automaticamente via Nominatim e aparecem no mapa progressivamente (1 por segundo). O resultado é cacheado no `localStorage` do navegador.

### Aba `volunteers`

| Coluna | Descrição |
|---|---|
| `id` | UUID |
| `created_at` | ISO 8601 |
| `updated_at` | ISO 8601 |
| `name` | Nome |
| `phone` | Somente dígitos |
| `skills` | Habilidades separadas por vírgula |
| `availability` | `disponivel`, `fins_de_semana` ou `sob_aviso` |
| `neighborhood` | Bairro |
| `address` | Endereço |
| `lat` | Latitude |
| `lng` | Longitude |
| `location_precision` | `gps`, `geocoded` ou `manual` |
| `notes` | Observações |
| `status` | `active` ou `inactive` |

### Aba `vistorias`

| Coluna | Descrição |
|---|---|
| `id` | UUID |
| `created_at` | ISO 8601 |
| `updated_at` | ISO 8601 |
| `name` | Nome do responsável |
| `phone` | Somente dígitos |
| `damage_types` | Tipos de dano separados por vírgula |
| `urgency` | `alta`, `media` ou `baixa` |
| `property_type` | `residencial`, `comercial`, `muro_talude` ou `outro` |
| `neighborhood` | Bairro |
| `address` | Endereço |
| `lat` | Latitude |
| `lng` | Longitude |
| `location_precision` | `gps`, `geocoded` ou `manual` |
| `description` | Descrição do problema |
| `status` | `pending` ou `visited` |

---

## Configuração do Apps Script

### 1. Criar o projeto

1. Abra a planilha → menu **Extensões → Apps Script**
2. Apague o código padrão e cole o conteúdo de `Code.gs`
3. Substitua os dois valores no topo:

```javascript
const SPREADSHEET_ID = "SEU_ID_AQUI";  // ID da planilha (trecho entre /d/ e /edit na URL)
const ADMIN_CODE     = "CODIGO_FORTE"; // Código secreto — não exponha publicamente
```

### 2. Publicar como Web App

1. Clique em **Implantar → Nova implantação**
2. Tipo: **Web App**
3. Executar como: **Eu (minha conta)**
4. Quem tem acesso: **Qualquer pessoa**
5. Clique em **Implantar** e copie a URL gerada

> ⚠️ **Atenção crítica:** toda vez que alterar o `Code.gs`, você precisa criar uma **nova implantação** em "Implantar → Gerenciar implantações → Nova versão". Simplesmente salvar o arquivo **não atualiza** a versão em produção.

### 3. Colar a URL nos HTMLs

Substitua `WEB_APP_URL_AQUI` pela URL copiada nos 4 arquivos:

```javascript
const WEB_APP_URL = "https://script.google.com/macros/s/SEU_ID/exec";
```

Arquivos que precisam: `index.html`, `pedidos.html`, `voluntarios.html`, `vistoria.html`.

---

## Deploy no GitHub Pages

1. Suba todos os arquivos para um repositório público no GitHub
2. Acesse **Settings → Pages → Branch: main → / (root) → Save**
3. Aguarde ~1 minuto e acesse `https://seu-usuario.github.io/nome-do-repo/`

O GitHub Pages serve o `index.html` automaticamente como página raiz.

---

## API — Endpoints

Base URL: `https://script.google.com/macros/s/SEU_ID/exec`

### GET (JSONP)

| `?action=` | Parâmetros extras | Descrição |
|---|---|---|
| `listRequests` | — | Pedidos ordenados: `open` primeiro |
| `listPoints` | — | Pontos de doação com `status=active` |
| `listVolunteers` | — | Voluntários com `status=active` |
| `listVistorias` | — | Vistorias ordenadas por urgência |
| `publicUpdateRequest` | `id`, `status=done` | Marca pedido como atendido |
| `publicUpdateVistoria` | `id`, `status=visited` | Marca vistoria como realizada |
| `ping` | — | Diagnóstico: retorna `pong` com timestamp |

Todos aceitam `&callback=nome_funcao` para JSONP.

**Teste rápido no navegador:**
```
SUA_URL?action=ping&callback=teste
→ teste({"ok":true,"msg":"pong","ts":"2026-..."});
```

### POST (via form + iframe oculto)

| `action=` | Campos obrigatórios | Descrição |
|---|---|---|
| `createRequest` | `id`, `phone`, `needs`, `address` ou `neighborhood` | Novo pedido de ajuda |
| `createPoint` | `id`, `name`, `address` | Novo ponto de doação |
| `createVolunteer` | `id`, `name`, `phone`, `skills` | Novo voluntário |
| `createVistoria` | `id`, `name`, `phone`, `damage_types`, `address` | Nova solicitação de vistoria |
| `adminUpdate` | `admin_code`, `target`, `id`, `status` | Atualização administrativa |

---

## Páginas — Referência detalhada

### `index.html` — Hub / Linktree

Página de entrada com **splash screen** de carregamento. Ao abrir, dispara `Promise.all` com as 4 APIs simultaneamente e salva no `sessionStorage`. A splash some quando todos os dados chegam ou após **5 segundos** (timeout de segurança).

Contém: contadores em tempo real, cards para as 4 plataformas, links para sites parceiros (Interdições de Vias JF, SOS Minas Gerais, Prefeitura JF) e números de emergência.

---

### `pedidos.html` — Pedidos de Ajuda

**Formulário de pedido:** nome (opcional), telefone, necessidades em chips multi-seleção, detalhes, endereço com opção de GPS, indicação de entrega necessária.

**Formulário de ponto de doação:** nome do local, endereço, itens aceitos, horário, contato, observações.

**Mapa Leaflet:**
- Pedidos em terracota (abertos) e verde (atendidos)
- Pontos de doação em azul
- Pontos **sem lat/lng** geocodificados automaticamente em background via Nominatim

**Marcar como atendido:** via JSONP com optimistic UI. Reverte se o servidor retornar erro. Após sucesso, invalida cache e força refresh da rede.

---

### `voluntarios.html` — Voluntários

**Formulário:** nome, telefone, 12 habilidades em chips, disponibilidade, bairro, endereço com GPS, observações.

**Habilidades:** transporte, doação de alimentos, água, roupas, higiene, abrigo, saúde, apoio psicológico, construção/reparo, pets, apoio financeiro, outros.

**Lista e mapa:** filtro por habilidade, cards com botões de WhatsApp, copiar telefone e rota.

---

### `vistoria.html` — Vistoria Técnica

Repositório para moradores solicitarem inspeção de imóveis danificados.

**Formulário:** nome, telefone, tipos de dano em chips (rachaduras, deslizamento, infiltração, fundação, teto, muro, alagamento, risco elétrico, outro), urgência, tipo de imóvel, endereço com GPS, descrição livre.

**Mapa:** marcadores vermelhos (alta urgência), âmbar (média), verdes (vistoriado). Botão "Marcar vistoriado" via JSONP.

---

### `riscos.html` — Áreas de Risco Geológico

Mapa em tela cheia com mapeamento oficial de risco geológico e hidrológico de Juiz de Fora, incorporado via `<iframe>` do Google My Maps.

---

## Funcionalidades técnicas

### Splash screen

- Entrada do logo com animação elástica (`cubic-bezier(.34,1.56,.64,1)`) e flutuação contínua
- Barra de progresso cresce automaticamente até 80% e avança 25% a cada API carregada
- Status textual por etapa: "Pedidos carregados ✓", "Pontos de doação carregados ✓", etc.
- Timeout de **5 segundos**: a splash some mesmo se uma API falhar

### Cache em camadas

| Camada | Escopo | TTL | Dados |
|---|---|---|---|
| `sessionStorage` | Sessão da aba | 5 minutos | requests, points, volunteers, vistorias |
| `localStorage` | Permanente | Ilimitado | Geocodificação de endereços |

Após qualquer gravação, o cache relevante é invalidado antes do próximo refresh.

### Geocodificação automática de pontos

1. Backend retorna `lat: null` para células vazias (proteção contra `Number("") === 0`)
2. Frontend separa pontos com e sem coordenadas
3. Pontos sem coords entram em fila processada por `geocodePointsBatch()`
4. Rate-limit de 1.1s entre requisições (Nominatim ToS)
5. Flag `geoBatchRunning` evita múltiplos batches paralelos
6. Coordenadas salvas no objeto em memória e no `localStorage`

### Proteção contra duplo envio

- `data-sending="1"` bloqueia o botão durante o envio
- Botão reabilitado **somente** no `iframe.onload` (confirmação do servidor)
- Backend rejeita IDs duplicados em `createRequest_` (idempotência)
- `updateStatusById_` atualiza **todas** as linhas com o mesmo ID (cobre duplicatas existentes)

### Proteção anti-spam

- Campo honeypot oculto `name="website"` — bots preenchem, humanos não
- `safeText_()` bloqueia URLs em todos os campos de texto
- Validação de telefone: somente dígitos, 10–13 caracteres

---

## Solução de problemas

### Pedido volta como "aberto" após marcar como atendido

Há linhas duplicadas com o mesmo UUID na planilha. Execute `deduplicateRequests()` no Apps Script e certifique-se de ter publicado a versão mais recente do `Code.gs`.

### Pontos de doação não aparecem no mapa

- **Coordenadas `0,0`:** a versão antiga do `Code.gs` converte células vazias para `0`. Publique uma nova versão com o fix de `null`.
- **Geocodificação lenta:** pontos aparecem progressivamente (1/s). O `localStorage` acelera nas próximas visitas.
- **Endereço não encontrado:** inclua número, bairro e "Juiz de Fora MG" no endereço da planilha.

### Formulário envia mas não aparece na planilha

- Confirme que o nome da aba é exatamente `requests` (minúsculo, sem espaços)
- Verifique os cabeçalhos da linha 1 (sem espaços extras, sem acentos fora do padrão)
- No Apps Script: **Ver → Registros de execução** para ver erros detalhados

### API retorna HTML em vez de JSON/JS

O Apps Script não está publicado ou a URL está incorreta. Teste:
```
SUA_URL?action=ping&callback=teste
```
Resposta esperada: `teste({"ok":true,"msg":"pong",...});`

---

## Manutenção da planilha

### Remover duplicatas

```
Editor do Apps Script → selecionar função → deduplicateRequests → Executar
```

Para outras abas, execute diretamente no console do Apps Script:
```javascript
deduplicateSheet("donation_points");
deduplicateSheet("volunteers");
deduplicateSheet("vistorias");
```

A função mantém a linha com status `closed`/`visited` sobre `open`/`active` em caso de duplicata.

### Desativar um ponto de doação

Altere `status` para `inactive` na planilha. O backend filtra automaticamente.

### Reabrir um pedido fechado por engano

Altere `status` de `closed` para `open` diretamente na planilha.

---

## Dependências externas

| Dependência | Versão | Uso |
|---|---|---|
| Leaflet.js | 1.9.4 | Mapas interativos |
| OpenStreetMap | — | Tiles do mapa |
| Nominatim (OSM) | — | Geocodificação de endereços |
| Google Fonts | — | Lora + DM Sans |
| Google Maps Embed | — | Mapa de riscos geológicos |
| Google Sheets | — | Banco de dados |
| Google Apps Script | — | API serverless |

Sem npm, sem build step, sem dependências locais.

---

## Desenvolvido por

**Jonathan Coelho** — [LinkedIn](https://www.linkedin.com/in/jonathan-coelho-06a91014b/)

Projeto voluntário de resposta a emergências. Use com responsabilidade.

---

*Emergências: Defesa Civil **199** · Bombeiros **193** · SAMU **192***
