---
name: carbot-architecture
description: >
  Arquitetura completa de coleta, cache e distribuicao de dados do CarBot.
  Use SEMPRE que precisar implementar, debugar ou discutir o scraper de veiculos,
  o sistema de cache (Redis), o banco de dados historico (PostgreSQL), a API de busca,
  o ranking inteligente, ou qualquer decisao de infraestrutura do CarBot. Tambem
  acione quando o usuario mencionar: latencia de busca, rate limiting, proxy rotation,
  normalizacao de dados de anuncios, verificacao de disponibilidade de link,
  re-scraping sob demanda, ou custos de infra. Se a tarefa envolver como os dados
  fluem desde a coleta nas plataformas ate a entrega ao comprador, esta skill tem
  a resposta. Use em conjunto com a skill carbot-project (contexto de negocio)
  e olx-search (detalhes tecnicos do OLX).
---

# CarBot - Arquitetura de Coleta, Cache e Distribuicao de Dados

## Contexto

O CarBot precisa acessar anuncios de multiplas plataformas (OLX, Webmotors, iCarros, Mercado Livre) e entregar resultados ao comprador com latencia < 200ms. Testes reais realizados em abril/2026 mostraram que o scraping via browser (Claude in Chrome, Selenium, Playwright) e lento (5-10s/pagina), consome muita RAM (200-500MB/instancia), e trunca dados. O OLX entrega JSON puro no HTML via `<script id="__NEXT_DATA__">`, tornando HTTP direto a abordagem ideal.

## Arquitetura em 3 Camadas

### Camada 1 - Coleta (Scraper HTTP)

Servico Python que faz requisicoes HTTP diretas (sem browser), simulando headers de navegador.

**Como funciona:**
1. Job agendado (Celery + Redis broker) dispara a cada 2 horas
2. Para cada combinacao de filtros configurada, faz GET na URL parametrizada
3. Extrai `__NEXT_DATA__` do HTML via regex ou BeautifulSoup
4. Normaliza dados (padroniza marcas, modelos, versoes, precos)
5. Persiste no Redis (cache) e PostgreSQL (historico)

**Performance medida:**
- HTTP direto: ~1-2s por pagina, ~15-20s para 432 anuncios (9 paginas)
- Browser headless: ~5-10s por pagina, dados truncados
- Memoria: ~10-20MB (HTTP) vs ~200-500MB (browser)

**Rate limiting seguro:**
- ~1 requisicao por segundo e seguro (testado abril/2026)
- Headers obrigatorios: User-Agent de browser real, Accept-Language, Accept-Encoding
- Retry com backoff exponencial: 30s, 60s, 120s em caso de 403/429
- Proxy rotation so e necessario em escala (>1000 req/hora, R$500-1000/mes)

**Stack do scraper:**
- Python + httpx (async HTTP) + BeautifulSoup (parsing)
- Celery + Redis (agendamento e filas)
- Regex para extrair __NEXT_DATA__: `re.search(r'<script id="__NEXT_DATA__"[^>]*>(.*?)</script>', html)`

### Camada 2 - Armazenamento (Cache + Historico)

**Redis (cache quente):**
- Armazena anuncios mais recentes para consulta instantanea (<50ms)
- Chave: `olx:sp:pickup:diesel:auto:p1` (plataforma:estado:tipo:combustivel:cambio:pagina)
- Valor: JSON compacto com array de anuncios normalizados
- TTL: 7200 segundos (2 horas)
- Memoria estimada: ~50-100MB para todo o catalogo SP

**PostgreSQL (historico):**
- Persiste TODOS os anuncios ja vistos com timestamps
- Permite analises que sao a vantagem competitiva do CarBot

**Modelo de dados PostgreSQL:**

```
ads (anuncio unico, dedup por plataforma+id_externo)
├── id (PK)
├── plataforma (olx, ml, webmotors, icarros)
├── id_externo (ID na plataforma original)
├── url
├── marca, modelo, versao (normalizados)
├── tipo_veiculo, combustivel, cambio
├── first_seen_at, last_seen_at
└── status (ativo, removido)

ad_snapshots (foto do anuncio a cada coleta)
├── id (PK)
├── ad_id (FK -> ads)
├── preco, km, titulo
├── cidade, estado, bairro
├── profissional (boolean)
├── leilao, unico_dono, quitado, ipva_pago
├── opcionais (JSON)
├── fotos_count
└── captured_at (timestamp)

search_configs (combinacoes de filtros a coletar)
├── id (PK)
├── plataforma
├── filtros (JSON: tipo, combustivel, cambio, preco_max, etc)
├── frequencia_minutos
├── ativo (boolean)
└── ultima_coleta_at

market_stats (agregados diarios)
├── id (PK)
├── modelo, regiao, mes
├── preco_medio, preco_mediana
├── volume_anuncios
├── tempo_medio_venda_dias
└── calculated_at
```

**Analises possiveis com historico (nenhuma plataforma oferece isso):**
- "Este anuncio esta no ar ha 45 dias" - vendedor flexivel para negociar
- "O preco caiu R$5.000 na ultima semana" - oportunidade
- "Este modelo costuma ser vendido em 8 dias" - urgencia legitima
- Sazonalidade de precos por mes, regiao e modelo
- Deteccao de fraude: anuncio que reaparece com preco diferente, conta nova com veiculos caros

### Camada 3 - Distribuicao (API CarBot)

API REST FastAPI que o frontend consome.

**Fluxo de uma busca:**
1. Frontend envia perfil: tipo, orcamento, regiao, preferencias
2. API consulta Redis: <50ms
3. Aplica ranking inteligente: aderencia ao perfil (40%), custo-beneficio com TCO (30%), saude do anuncio (20%), proximidade (10%)
4. Enriquece com historico: tempo no ar, variacao de preco, velocidade de venda
5. Retorna JSON: ranking com score, dados tecnicos, alertas de risco
6. Tempo total: <200ms end-to-end

**Verificacao de disponibilidade no clique:**
Quando o comprador clica em um anuncio, a API faz HEAD request (~100ms) na URL original. Se 404, mostra mensagem amigavel e sugere proximos do ranking.

**Re-scraping sob demanda:**
Quando um comprador faz uma busca, o backend agenda re-scraping daquela combinacao de filtros para 5-10 minutos em background. Quando o comprador voltar, dados ja estao frescos.

## Trade-offs Documentados

### Delay de 2h vs tempo real
- Veiculo seminovo fica ~15-30 dias anunciado
- Na janela de 2h, ~0,1-0,5% dos anuncios saem do ar
- Comprador clica em 5-10 anuncios - chance de link morto e minima
- Mitigado por verificacao no clique + re-scraping sob demanda
- O ganho: resposta instantanea + historico que ninguem oferece

### HTTP direto vs browser headless
- HTTP: 1-2s, 10-20MB RAM, alta robustez
- Browser: 5-10s, 200-500MB RAM, media robustez
- OLX/Webmotors/iCarros usam SSR - HTTP direto funciona
- Browser headless fica como fallback para plataformas client-side only

### Legalidade do scraping
1. MVP (0-3 meses): scraping com cache agressivo, volume baixo (<500 req/h). Risco juridico minimo.
2. Escala (3-6 meses): negociar parceria comercial. CarBot gera trafego qualificado - incentivo mutuo.
3. Maturidade (6+): APIs oficiais via contrato. Scraping apenas como fallback.

## Infraestrutura MVP

| Componente | Tecnologia | Custo/mes |
|---|---|---|
| Scraper | Python + httpx + BS4 | - |
| Agendador | Celery + Redis | - |
| Cache | Redis (128MB) | Incluido |
| Banco | PostgreSQL | Incluido |
| API | FastAPI | - |
| VPS | 2 vCPU, 4GB RAM, 80GB SSD | R$50-100 |
| Dominio + SSL | carbot.ai | ~R$15 |
| **TOTAL** | | **R$65-115/mes** |

Em escala: adicionar proxy rotation (R$500-1000/mes) e separar Redis/PostgreSQL em servicos gerenciados.

## Metricas de Sucesso

| Metrica | Alvo MVP | Alvo Escala |
|---|---|---|
| Latencia busca (API) | <200ms (p95) | <100ms (p95) |
| Freshness cache | <=2 horas | <=30 minutos |
| Taxa link indisponivel | <2% | <0,5% |
| Cobertura plataformas | OLX + ML | OLX + ML + Webmotors + iCarros |
| Uptime scraper | >95% | >99,5% |
| Volume indexado | >50.000 (SP) | >500.000 (Brasil) |

## Proteção Anti-Bot (genérico para qualquer plataforma)

Estas práticas se aplicam a qualquer plataforma que o CarBot for coletar dados (OLX, Webmotors, iCarros, Mercado Livre, etc). O objetivo é que o scraper se comporte indistintamente de um usuário real navegando.

### Princípios

O scraper deve parecer um humano navegando em velocidade normal. Sistemas anti-bot detectam anomalias estatísticas: intervalos regulares demais, volume alto demais, headers ausentes, falta de cookies, padrões sequenciais. A defesa é eliminar cada uma dessas anomalias.

### Headers HTTP obrigatórios

Toda requisição deve incluir o conjunto completo de headers que um browser real envia:

```
User-Agent: (rotacionar entre 5-10 strings reais — ver pool abaixo)
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: pt-BR,pt;q=0.9,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Referer: (URL da página anterior ou homepage da plataforma)
```

**Pool de User-Agents (atualizar semestralmente):**
Manter 5-10 User-Agent strings de versões recentes do Chrome e Firefox em Windows/Mac/Linux. Rotacionar por sessão (não por request — um humano real não troca de browser a cada página).

### Gerenciamento de Sessão e Cookies

1. Antes de iniciar qualquer coleta, fazer GET na homepage da plataforma para obter cookies de sessão
2. Manter cookie jar por sessão usando `httpx.AsyncClient` com cookies persistentes
3. Requests sem cookies são flag imediata de bot — nunca fazer request "limpa"
4. Se receber Set-Cookie durante a coleta, aceitar e reenviar nos requests seguintes

### Controle de Timing (Jitter)

Intervalo fixo entre requests é o padrão mais óbvio de bot. Usar jitter aleatório:

```python
import random

def next_delay():
    # Distribuição log-normal: maioria entre 1-2s, alguns até 4s
    base = random.lognormvariate(0.3, 0.4)
    return max(0.8, min(base, 5.0))  # clamp entre 0.8s e 5.0s
```

Média efetiva: ~1.5 req/segundo. Máximo sustentável: ~2.400 req/hora.

Além do jitter por request, inserir pausas maiores (10-30 segundos) a cada 20-40 requests para simular "humano lendo resultados".

### Intercalação de Segmentos

Nunca fazer mais de 30 requisições seguidas para o mesmo path/segmento de busca. Intercalar entre segmentos diferentes. Um humano real não navega 100 páginas seguidas do mesmo filtro.

```
Padrão BOM:  3pp SUV → 3pp Sedã → 3pp Hatch → 3pp Pick-up → (pausa) → repete
Padrão RUIM: 100pp Pick-up → 100pp SUV → 100pp Sedã (sequencial por segmento)
```

### Tratamento de Erros e Rate Limiting

| HTTP Status | Significado | Ação |
|---|---|---|
| 200 | OK | Processar normalmente |
| 403 | Bloqueado/Forbidden | Parar segmento, backoff 5 min, trocar User-Agent |
| 429 | Rate Limited | Parar TUDO, backoff 10 min, reduzir velocidade 50% |
| 503 | Server Overloaded | Retry em 30s, máximo 3 tentativas |
| 5xx | Erro do servidor | Retry com backoff exponencial: 30s, 60s, 120s |

**Regras de backoff:**
- Após 403: pausar aquele segmento por 5 minutos, continuar outros segmentos
- Após 429: pausar TODA a coleta por 10 minutos, depois retomar com delay 2x maior
- Após 3 erros 403 no mesmo ciclo: abortar ciclo, tentar no próximo
- Nunca retentar imediatamente — sempre esperar pelo menos 30 segundos

**Logging:** registrar todo 403/429 com timestamp, URL, e User-Agent usado. Permite identificar se há padrão (ex: sempre bloqueia após página 50, ou bloqueia em horário X).

### Limite de Volume por Plataforma

| Plataforma | Req/hora seguro (MVP) | Req/hora escala (com proxy) |
|---|---|---|
| OLX | 500-750 | 3.000-5.000 |
| Webmotors | 300-500 (estimar) | 2.000-3.000 |
| iCarros | 300-500 (estimar) | 2.000-3.000 |
| Mercado Livre | Via API oficial | Via API oficial |

Estes são limites conservadores. Ajustar com base no monitoramento de erros 403/429.

### Proxy Rotation (escala)

Desnecessário no MVP (<750 req/hora). Quando necessário:

- Usar pool de proxies residenciais (não datacenter — datacenter IPs são detectados facilmente)
- Serviços recomendados: BrightData, ScraperAPI, Oxylabs (~R$500-1000/mês)
- Rotacionar por sessão (não por request) — manter mesmo IP durante 20-40 requests
- Priorizar proxies brasileiros (mesmo país do site alvo)

### Checklist de Detecção Anti-Bot

Antes de cada deploy do scraper, verificar:

```
[ ] User-Agent rotacionado entre 5-10 strings reais de browsers atuais
[ ] Headers completos (Accept, Accept-Language, Referer, Sec-Fetch-*)
[ ] Cookie jar ativo com sessão iniciada na homepage
[ ] Jitter entre requests (0.8-5.0s, distribuição log-normal)
[ ] Intercalação entre segmentos (máx 30 req seguidas no mesmo path)
[ ] Pausas longas (10-30s) a cada 20-40 requests
[ ] Backoff implementado para 403/429/5xx
[ ] Limite de volume respeitado por plataforma
[ ] Logging de todos os erros HTTP com timestamp e URL
[ ] Proxy rotation configurado (se volume > 750 req/hora)
```

### Fingerprint e Detecção Avançada

Plataformas maiores podem usar fingerprinting mais sofisticado:
- **TLS fingerprint (JA3):** httpx e requests geram fingerprints diferentes dos browsers. Se bloqueado mesmo com headers corretos, considerar usar `curl_cffi` ou `tls-client` que emulam TLS de browsers reais.
- **Behavioral analysis:** Se a plataforma rastreia padrões de navegação (ex: sempre entra direto na página 1 sem passar pela home), adicionar requests "decorativos" ocasionais (home, página de categoria).
- **JavaScript challenges:** Se a plataforma passar a exigir execução de JS (Cloudflare, DataDome), HTTP direto não funciona. Nesse caso, fallback para Playwright com stealth plugin.

## Documentos Relacionados

- `CarBot_Modelo_de_Negocio_e_Requisitos.docx` - modelo de negocio, user stories, roadmap
- `OLX_busca.md` - mapeamento tecnico completo do OLX (filtros, JSON, 742 linhas)
- Skill `carbot-project` - contexto de negocio (17 dores, 8 fases, receita)
- Skill `olx-search` - guia operacional de scraping OLX
