---
name: inteligencia-antibot
description: >
  Guia completo de inteligência anti-bot para web scraping em qualquer plataforma.
  Use SEMPRE que precisar implementar, configurar ou debugar um scraper que precisa
  evitar detecção por sistemas anti-bot. Acione quando o usuário mencionar: anti-bot,
  rate limiting, bloqueio 403, erro 429, fingerprint, proxy rotation, User-Agent,
  cookies de sessão, Cloudflare, DataDome, captcha, headers HTTP para scraping,
  jitter, backoff, detecção de bot, TLS fingerprint, JA3, stealth, ou qualquer
  problema relacionado a ser bloqueado por uma plataforma durante coleta de dados.
  Também use quando for adicionar uma nova plataforma de busca ao CarBot ou a
  qualquer outro sistema de scraping. Esta skill é genérica e independente de
  plataforma — funciona para OLX, Webmotors, iCarros, Mercado Livre, ou qualquer
  outro site.
---

# Inteligência Anti-Bot para Web Scraping

Guia operacional para construir scrapers que se comportam de forma indistinguível de um usuário humano real navegando. Aplicável a qualquer plataforma web.

## Por que scrapers são detectados

Sistemas anti-bot não procuram "um bot" — eles procuram anomalias estatísticas. Um humano real tem comportamento irregular, lento, com pausas, erros e inconsistências. Um bot tem comportamento regular, rápido, previsível e perfeito. A defesa é introduzir irregularidade em cada camada.

Os principais sinais que entregam um bot:

1. **Timing uniforme** — requests a cada exatos 1.000ms é assinatura de máquina
2. **Headers incompletos ou ausentes** — browsers reais enviam 10-15 headers por request
3. **Ausência de cookies** — toda sessão real começa com cookies da homepage
4. **Volume concentrado** — 100 páginas do mesmo filtro em sequência não é navegação humana
5. **TLS fingerprint** — bibliotecas Python geram fingerprints diferentes dos browsers
6. **Ausência de Referer** — humanos chegam de algum lugar, não surgem do nada
7. **Padrão de paginação** — humano não navega da página 1 à 100 sem parar

## Camada 1 — Headers HTTP

Toda requisição deve incluir o conjunto completo de headers que um browser real envia. Headers faltantes são o primeiro sinal de bot.

### Headers obrigatórios

```
User-Agent: (rotacionar — ver pool abaixo)
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
DNT: 1
```

### Headers que variam por contexto

| Situação | Sec-Fetch-Site | Referer |
|---|---|---|
| Primeira página (entrada direta) | `none` | (omitir ou Google) |
| Navegação dentro do site | `same-origin` | URL da página anterior |
| Clique em link externo | `cross-site` | URL do site de origem |

Manter consistência: se `Sec-Fetch-Site` é `same-origin`, o `Referer` deve ser do mesmo domínio.

### Pool de User-Agents

Manter 5-10 User-Agent strings de versões recentes do Chrome e Firefox. Atualizar a cada 6 meses porque versões antigas são suspeitas.

```python
UA_POOL = [
    # Chrome Windows
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    # Chrome Mac
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    # Chrome Linux
    "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36",
    # Firefox Windows
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:125.0) Gecko/20100101 Firefox/125.0",
    # Firefox Mac
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10.15; rv:125.0) Gecko/20100101 Firefox/125.0",
    # Edge Windows
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36 Edg/124.0.0.0",
]
```

**Regra de rotação:** rotacionar por sessão, não por request. Um humano real não troca de browser a cada página. Escolher um UA no início do ciclo de coleta e manter durante toda a sessão (~20-40 requests). Trocar na próxima sessão.

## Camada 2 — Gerenciamento de Sessão e Cookies

Requests sem cookies são flag imediata de bot. Todo site seta cookies na primeira visita e espera recebê-los de volta.

### Fluxo correto

```python
import httpx

async with httpx.AsyncClient(
    follow_redirects=True,
    timeout=30.0,
    headers=DEFAULT_HEADERS,
) as client:
    # 1. Iniciar sessão na homepage (pega cookies)
    await client.get("https://www.plataforma.com.br/")
    
    # 2. Agora fazer as buscas (cookies vão automaticamente)
    response = await client.get("https://www.plataforma.com.br/busca?filtro=valor")
    
    # 3. O client mantém cookies entre requests
```

### Regras de cookies

- Sempre iniciar sessão na homepage antes de qualquer busca
- Aceitar todos os Set-Cookie recebidos durante a navegação
- Se o site tiver banner de cookies (LGPD), fazer request de aceitação antes de continuar
- Manter o mesmo cookie jar durante toda a sessão
- Se receber 403 após muitas requests, criar nova sessão (novo client, novos cookies)
- Não misturar cookies de sessões diferentes

## Camada 3 — Controle de Timing (Jitter)

O timing entre requests é o sinal mais forte para detecção. Intervalo fixo = bot. Intervalo aleatório uniforme (random entre 1 e 3) ainda é detectável porque distribuição uniforme não é natural.

### Distribuição log-normal (recomendada)

Humanos têm distribuição assimétrica: maioria das ações são rápidas (1-2s), mas eventualmente pausam mais (3-5s). A distribuição log-normal modela isso bem.

```python
import random
import asyncio

def next_delay():
    """Gera delay entre requests com distribuição log-normal.
    Média efetiva: ~1.5s. Range: 0.8s a 5.0s.
    ~70% dos delays ficam entre 1.0s e 2.0s.
    ~20% ficam entre 2.0s e 3.5s.
    ~10% ficam entre 3.5s e 5.0s.
    """
    base = random.lognormvariate(0.3, 0.4)
    return max(0.8, min(base, 5.0))

async def wait_between_requests():
    delay = next_delay()
    await asyncio.sleep(delay)
```

### Pausas longas (simulação de leitura)

Além do jitter por request, inserir pausas maiores que simulam "humano analisando resultados":

```python
import random

request_count = 0

async def maybe_long_pause():
    """A cada 20-40 requests, pausa longa de 10-30s."""
    global request_count
    request_count += 1
    
    # Intervalo aleatório (não a cada exatos 30 requests)
    if request_count >= random.randint(20, 40):
        pause = random.uniform(10, 30)
        await asyncio.sleep(pause)
        request_count = 0
```

### Taxa efetiva

Com jitter log-normal (~1.5s médio) + pausas longas: média efetiva de ~2.400 req/hora. Suficiente para coleta incremental do CarBot (~375-750 req/ciclo em ~10 min).

## Camada 4 — Padrão de Navegação

### Intercalação de segmentos

Nunca fazer mais de 30 requisições seguidas para o mesmo path ou segmento de busca. Um humano real não navega 100 páginas do mesmo filtro sequencialmente.

```python
from itertools import cycle

segments = [
    {"name": "pickup_0_80k",  "params": "ctp=3&pe=80000"},
    {"name": "suv_0_50k",     "params": "ctp=5&pe=50000"},
    {"name": "sedan_0_30k",   "params": "ctp=8&pe=30000"},
    {"name": "hatch_0_25k",   "params": "ctp=9&pe=25000"},
]

MAX_CONSECUTIVE = 3  # máximo de páginas seguidas por segmento

async def collect_interleaved(segments, pages_per_segment):
    """Coleta intercalada entre segmentos."""
    progress = {s["name"]: 1 for s in segments}  # página atual
    
    while any(progress[s["name"]] <= pages_per_segment for s in segments):
        for segment in segments:
            name = segment["name"]
            if progress[name] > pages_per_segment:
                continue
            
            # Fazer até MAX_CONSECUTIVE páginas deste segmento
            for _ in range(MAX_CONSECUTIVE):
                if progress[name] > pages_per_segment:
                    break
                await fetch_page(segment, progress[name])
                progress[name] += 1
                await wait_between_requests()
            
            await maybe_long_pause()
```

### Requests decorativos

Alguns sistemas rastreiam se o usuário "navega normalmente" ou só acessa páginas de dados. Inserir requests decorativos ocasionais:

```python
DECORATIVE_URLS = [
    "/",                          # homepage
    "/autos-e-pecas",             # página de categoria
    "/autos-e-pecas/carros-vans-e-utilitarios",  # subcategoria
]

async def maybe_decorative_request(client, base_url):
    """~5% de chance de fazer um request decorativo."""
    if random.random() < 0.05:
        url = base_url + random.choice(DECORATIVE_URLS)
        await client.get(url)
        await asyncio.sleep(random.uniform(2, 5))
```

### Referer chain realista

Manter cadeia de Referer que faça sentido para navegação humana:

```
Request 1: GET /                          Referer: (nenhum ou google.com)
Request 2: GET /autos-e-pecas/...?filtro  Referer: homepage
Request 3: GET /autos-e-pecas/...?o=2     Referer: página 1
Request 4: GET /autos-e-pecas/...?o=3     Referer: página 2
```

Nunca acessar página 50 com Referer da homepage — isso não faz sentido para um humano.

## Camada 5 — Tratamento de Erros e Rate Limiting

### Tabela de ações por status HTTP

| Status | Significado | Ação imediata | Ação de follow-up |
|---|---|---|---|
| 200 | OK | Processar normalmente | — |
| 301/302 | Redirect | Seguir redirect | Verificar se não é redirect para captcha |
| 403 | Bloqueado | Parar segmento | Backoff 5 min, trocar UA, nova sessão |
| 429 | Rate Limited | Parar TUDO | Backoff 10 min, reduzir velocidade 50% |
| 503 | Server Overload | Retry 30s | Máximo 3 tentativas, depois parar |
| 5xx | Erro servidor | Retry backoff | 30s → 60s → 120s, depois parar |
| Timeout | Sem resposta | Retry 1x | Se repetir, tratar como 503 |

### Implementação de backoff

```python
class RateLimitHandler:
    def __init__(self):
        self.global_pause_until = 0
        self.segment_pauses = {}  # segment_name -> pause_until timestamp
        self.speed_multiplier = 1.0  # aumenta após 429
        self.consecutive_errors = 0
    
    async def handle_response(self, status, segment_name):
        if status == 200:
            self.consecutive_errors = 0
            return True  # continuar
        
        if status == 403:
            # Pausar só este segmento por 5 minutos
            self.segment_pauses[segment_name] = time.time() + 300
            self.consecutive_errors += 1
            log_error(status, segment_name)
            
            if self.consecutive_errors >= 3:
                # 3 erros seguidos: abortar ciclo inteiro
                return False
            return True  # continuar outros segmentos
        
        if status == 429:
            # Pausar TUDO por 10 minutos e reduzir velocidade
            self.global_pause_until = time.time() + 600
            self.speed_multiplier *= 2.0  # delay 2x maior
            log_error(status, segment_name)
            return True  # retomar após pausa
        
        return True
    
    def is_segment_paused(self, segment_name):
        return time.time() < self.segment_pauses.get(segment_name, 0)
    
    def is_globally_paused(self):
        return time.time() < self.global_pause_until
```

### Regras de ouro para erros

- **Nunca retentar imediatamente** — sempre esperar pelo menos 30 segundos
- **Nunca ignorar 429** — é o aviso antes do ban permanente de IP
- **Logar tudo** — timestamp, URL, status, User-Agent, cookies presentes
- **Parar cedo** — melhor perder um ciclo de coleta do que ser banido permanentemente
- **Distribuir retries** — se um segmento falhou, continuar outros e voltar depois

## Camada 6 — Fingerprinting Avançado

### TLS Fingerprint (JA3/JA4)

Bibliotecas Python (requests, httpx, aiohttp) geram fingerprints TLS diferentes dos browsers reais. Plataformas como Cloudflare e DataDome verificam isso.

**Nível 1 — httpx (padrão, funciona na maioria dos sites):**
```python
import httpx
# httpx usa h11/h2 que tem fingerprint Python padrão
# Funciona para OLX, Mercado Livre, maioria dos sites brasileiros
```

**Nível 2 — curl_cffi (emula TLS de browser real):**
```python
# pip install curl_cffi
from curl_cffi.requests import AsyncSession

async with AsyncSession(impersonate="chrome124") as session:
    response = await session.get("https://site-protegido.com")
    # TLS fingerprint idêntico ao Chrome 124
```

**Nível 3 — Playwright com stealth (último recurso):**
```python
# pip install playwright playwright-stealth
from playwright.async_api import async_playwright
from playwright_stealth import stealth_async

async with async_playwright() as pw:
    browser = await pw.chromium.launch(headless=True)
    page = await browser.new_page()
    await stealth_async(page)  # remove sinais de automação
    await page.goto("https://site-muito-protegido.com")
```

**Quando escalar de nível:**
- Nível 1 (httpx) → funciona? Manter.
- Recebendo 403 mesmo com headers/cookies corretos? → Testar Nível 2 (curl_cffi)
- Ainda bloqueado? → Verificar se há JavaScript challenge → Nível 3 (Playwright)

### JavaScript Challenges

Alguns sites (Cloudflare, DataDome, PerimeterX) servem uma página de challenge JavaScript antes do conteúdo real. HTTP direto não consegue resolver.

**Detecção:** se o HTML retornado contém `<title>Just a moment...</title>` (Cloudflare) ou scripts de verificação ao invés do conteúdo esperado, há JS challenge ativo.

**Solução:** usar Playwright com stealth plugin. Manter pool de browsers abertos para reutilizar sessões já autenticadas. Custo: ~200-500MB RAM por instância.

### Browser Fingerprinting (Canvas, WebGL, Fonts)

Plataformas avançadas podem verificar fingerprint de canvas, WebGL e fontes instaladas. Só se aplica a scraping via browser (Playwright/Selenium).

**Defesa com Playwright:**
```python
# playwright-stealth já cuida de:
# - navigator.webdriver = false
# - Chrome runtime objects presentes
# - Consistent canvas fingerprint
# - WebGL vendor/renderer realistas
# - Plugins e mimeTypes populados
```

Se mesmo com stealth o site detecta, considerar usar browser real via CDP (Chrome DevTools Protocol) com perfil persistente.

## Camada 7 — Proxy e Rotação de IP

### Quando usar proxy

| Cenário | Proxy necessário? |
|---|---|
| MVP (<750 req/hora por plataforma) | Não |
| Escala (750-3.000 req/hora) | Sim, pool pequeno (5-10 IPs) |
| Grande escala (>3.000 req/hora) | Sim, pool grande (50+ IPs) |
| IP residencial banido | Sim, imediatamente |
| Múltiplas plataformas simultâneas | Depende do volume total |

### Tipos de proxy

| Tipo | Prós | Contras | Custo |
|---|---|---|---|
| Datacenter | Barato, rápido | Facilmente detectado | R$50-200/mês |
| Residencial | Difícil de detectar | Mais lento, mais caro | R$500-2.000/mês |
| Mobile | Quase indetectável | Caro, lento | R$1.000-5.000/mês |
| ISP (estático residencial) | Bom equilíbrio | Disponibilidade limitada | R$300-1.000/mês |

**Recomendação para o CarBot:** proxies residenciais brasileiros. Datacenter IPs são detectados por geolocalização e ASN (Autonomous System Number) — plataformas mantêm listas de ASNs de datacenters conhecidos.

### Rotação de proxy

```python
class ProxyRotator:
    def __init__(self, proxies):
        self.proxies = proxies
        self.current_index = 0
        self.requests_per_proxy = 0
        self.max_per_proxy = random.randint(20, 40)  # trocar após 20-40 req
    
    def get_proxy(self):
        if self.requests_per_proxy >= self.max_per_proxy:
            self.current_index = (self.current_index + 1) % len(self.proxies)
            self.requests_per_proxy = 0
            self.max_per_proxy = random.randint(20, 40)
        
        self.requests_per_proxy += 1
        return self.proxies[self.current_index]
```

**Regra:** rotacionar por sessão (20-40 requests), não por request. Mesmo IP durante uma "sessão de navegação" é mais natural.

### Serviços de proxy recomendados

- **BrightData (ex-Luminati):** maior pool residencial, API robusta, ~$15/GB
- **ScraperAPI:** proxy + anti-bot integrado, mais simples, ~$49/mês (100k req)
- **Oxylabs:** bom para Brasil, pool residencial grande, ~$15/GB
- **SmartProxy:** bom custo-benefício para volume médio, ~$12.5/GB

## Checklist de Deploy do Scraper

Antes de cada deploy ou nova plataforma, verificar todos os itens:

```
HEADERS E IDENTIDADE
[ ] User-Agent rotacionado entre 5-10 strings de browsers atuais
[ ] Headers completos: Accept, Accept-Language, Accept-Encoding
[ ] Sec-Fetch-* headers presentes e consistentes
[ ] Referer chain realista (homepage → categoria → busca → paginação)
[ ] DNT: 1 presente

SESSÃO E COOKIES
[ ] Cookie jar ativo com sessão iniciada na homepage
[ ] Set-Cookie aceito e reenviado em requests seguintes
[ ] Nova sessão criada após erros 403 persistentes
[ ] Cookies não misturados entre sessões diferentes

TIMING E VOLUME
[ ] Jitter entre requests com distribuição log-normal (0.8-5.0s)
[ ] Pausas longas (10-30s) a cada 20-40 requests
[ ] Intercalação entre segmentos (máx 30 req seguidas no mesmo path)
[ ] Requests decorativos (~5% das requests)
[ ] Limite de volume respeitado por plataforma

TRATAMENTO DE ERROS
[ ] Backoff implementado para 403 (5 min por segmento)
[ ] Backoff implementado para 429 (10 min global + redução de velocidade)
[ ] Retry com backoff exponencial para 5xx
[ ] Abort após 3 erros 403 consecutivos no mesmo ciclo
[ ] Nunca retry imediato — mínimo 30 segundos

LOGGING E MONITORAMENTO
[ ] Log de todos os erros HTTP com timestamp, URL, status, UA
[ ] Alerta quando taxa de erro > 5% no ciclo
[ ] Métricas de requests/hora por plataforma
[ ] Histórico de bloqueios para identificar padrões

INFRA (se escala)
[ ] Proxy rotation com proxies residenciais brasileiros
[ ] Rotação por sessão (20-40 req por IP), não por request
[ ] curl_cffi ou Playwright stealth se httpx for bloqueado
[ ] Pool de browsers se JS challenge detectado
```

## Escalando a Proteção: Árvore de Decisão

Quando um scraper começa a ser bloqueado, seguir esta árvore:

```
Bloqueado?
├── Headers estão completos? → NÃO → Adicionar headers faltantes
├── Cookies estão sendo enviados? → NÃO → Implementar cookie jar + sessão
├── Timing é irregular? → NÃO → Implementar jitter log-normal
├── Volume está alto demais? → SIM → Reduzir para 50% e testar
├── Tudo acima OK, ainda bloqueado?
│   ├── TLS fingerprint? → Testar curl_cffi (impersonate Chrome)
│   ├── JS challenge? → Implementar Playwright + stealth
│   └── IP banido? → Implementar proxy rotation residencial
└── Nada funciona? → Plataforma tem proteção enterprise
    → Considerar parceria comercial / API oficial
```

## Monitoramento Contínuo

Anti-bot é uma corrida armamentista. Plataformas atualizam proteções regularmente. Monitorar:

- **Taxa de sucesso por plataforma** — se cair abaixo de 95%, investigar
- **Tempo de resposta** — aumento súbito pode indicar throttling antes do bloqueio
- **Conteúdo das respostas** — verificar se HTML retornado contém dados esperados (não página de erro/captcha)
- **Versões de User-Agent** — atualizar pool a cada 6 meses
- **Mudanças de estrutura** — se `__NEXT_DATA__` sumir ou mudar de formato, o scraper quebra silenciosamente

```python
async def health_check(response, expected_content_key):
    """Verificar se a resposta contém dados reais, não página de bloqueio."""
    if response.status_code != 200:
        return False
    
    html = response.text
    
    # Detectar JS challenges
    if "Just a moment" in html or "Checking your browser" in html:
        log_warning("JS challenge detectado")
        return False
    
    # Verificar se tem o conteúdo esperado
    if expected_content_key not in html:
        log_warning(f"Conteúdo esperado '{expected_content_key}' não encontrado")
        return False
    
    return True
```
