---
name: olx-search
description: >
  Guia técnico completo para buscar e extrair dados de veículos no OLX via web scraping.
  Use SEMPRE que precisar pesquisar carros, picapes, SUVs ou qualquer veículo no OLX,
  montar URLs de busca, extrair dados de anúncios, filtrar resultados, ou analisar
  o mercado automotivo usando dados do OLX. Também use quando o usuário mencionar
  OLX, scraping de veículos, pesquisa de carros online, comparação de preços de carros,
  ou qualquer tarefa que envolva coletar ou analisar dados de anúncios automotivos.
  Mesmo que o usuário não mencione "OLX" diretamente, se a tarefa envolver buscar
  veículos seminovos online, consulte esta skill primeiro.
---

# OLX Vehicle Search - Guia Operacional de Scraping

Esta skill contém o mapeamento completo da estrutura de busca do OLX para veículos. Use-a como referência para montar URLs, extrair dados e filtrar resultados.

## Como o OLX funciona por baixo

O OLX usa Next.js. Todos os dados dos anúncios vêm embutidos no HTML dentro de `<script id="__NEXT_DATA__">` como JSON. Isso significa que não precisamos parsear HTML — basta extrair o JSON e trabalhar com dados estruturados.

## Montando a URL de busca

A URL base segue este padrão:

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/{localização}?{filtros}
```

### Localização (via path)

A localização funciona APENAS no nível estadual quando combinada com filtros:

- `/estado-sp` — São Paulo (FUNCIONA com filtros)
- `/estado-rj` — Rio de Janeiro (FUNCIONA com filtros)
- `/` — Nacional, sem filtro de local (FUNCIONA)
- `/grande-abc`, `/sao-paulo-e-regiao` — NÃO FUNCIONA com filtros (retorna nacional)

Para filtrar por cidade/região, use `estado-{uf}` e depois filtre programaticamente pelo campo `locationDetails.municipality` ou `locationDetails.ddd` dos anúncios.

### Filtros de URL - Referência Completa

#### Tipo de veículo (`ctp`)

| Valor | Tipo |
|---|---|
| 3 | **Pick-up** |
| 5 | SUV |
| 8 | Sedã |
| 9 | Hatch |
| 2 | Conversível |
| 7 | Van/Utilitário |
| 10 | Caminhão Leve |
| 11 | Coupé |
| 12 | Perua |
| 6 | Buggy |

#### Combustível (`fu`)

| Valor | Combustível |
|---|---|
| 5 | **Diesel** |
| 3 | Flex |
| 1 | Gasolina |
| 2 | Álcool |
| 6 | Híbrido |
| 7 | Elétrico |

#### Câmbio (`gb`)

| Valor | Câmbio |
|---|---|
| 2 | **Automático** |
| 1 | Manual |
| 3 | Semi-Automático |
| 4 | Automatizado |

#### Preço

| Param | Descrição | Exemplo |
|---|---|---|
| `ps` | Mínimo | `ps=30000` |
| `pe` | Máximo | `pe=100000` |

#### Quilometragem

| Param | Descrição | Exemplo |
|---|---|---|
| `ms` | KM mínimo | `ms=0` |
| `me` | KM máximo | `me=200000` |

#### Ano

| Param | Descrição | Exemplo |
|---|---|---|
| `rs` | Ano mínimo | `rs=2015` |
| `re` | Ano máximo | `re=2023` |

#### Motor (`motp`)

1=1.0, 2=1.2, 3=1.3, 4=1.4, 5=1.5, 6=1.6, 7=1.7, 8=1.8, 9=1.9, 10=2.0-2.9, 11=3.0-3.9, 12=4.0+

#### Direção (`ics`)

1=Hidráulica, 2=Elétrica, 3=Mecânica, 5=Eletro-hidráulica

#### Cor (`cac`)

1=Preto, 2=Branco, 3=Prata, 4=Vermelho, 5=Cinza, 6=Azul, 7=Amarelo, 8=Verde, 9=Laranja, 10=Outra

#### Portas (`cad`)

1=2 portas, 2=4 portas, 3=3 portas

#### Final de placa (`et`)

1=0, 2=1, 3=2, 4=3, 5=4, 6=5, 7=6, 8=7, 9=8, 10=9

#### Aceita trocas (`exc`)

1=Sim, 2=Não

#### Kit GNV (`hgnv`)

1=Sim, 0=Não

#### Estado financeiro (`fncs`)

1=Quitado, 2=Financiado

#### Documentação (`doc`)

1=IPVA Pago, 2=Com multas, 3=De leilão

#### Conservação (`cnwa`)

1=Único dono, 2=Com manual, 3=Chave reserva, 4=Revisões concessionária, 5=Com garantia

#### Opcionais (`cf`)

1=Ar condicionado, 4=Trava elétrica, 5=Air bag, 6=Alarme, 7=Som, 8=Sensor de ré, 9=Câmera de ré, 10=Blindado, 11=Bancos couro, 12=Computador de bordo, 13=Conexão USB, 15=Bluetooth, 16=GPS, 17=Cruise control, 18=Rodas liga leve, 19=Teto solar, 20=Tração 4x4

#### Tipo de anunciante (`f`)

0=Particular, 1=Profissional (loja)

#### Toggles

| Param | Descrição |
|---|---|
| `vtvh=1` | Com histórico veicular |
| `crss=1` | Vistoriado |
| `fpdll=1` | Abaixo da FIPE |

#### Marca/Modelo/Versão

| Param | Descrição |
|---|---|
| `vb` | Marca (ex: Toyota) - dinâmico |
| `vm` | Modelo (ex: Hilux) - dinâmico |
| `vv` | Versão - dinâmico |

#### Busca texto e paginação

| Param | Descrição |
|---|---|
| `q` | Busca por texto livre |
| `o` | Página (começa em 1) |
| `sp` | Ordenação: `relevance`, `date`, `price`, `biggest_price` |

### Multi-select

Para selecionar múltiplos valores, repetir o parâmetro:
```
fu=5&fu=3     → Diesel + Flex
ctp=3&ctp=5   → Pick-up + SUV
gb=2&gb=3     → Automático + Semi-Automático
```

## Extraindo dados dos anúncios

### Acesso ao JSON

```javascript
var nd = JSON.parse(document.getElementById('__NEXT_DATA__').textContent);
var pp = nd.props.pageProps;
var ads = pp.ads;           // Array de anúncios
var total = pp.totalOfAds;  // Total de resultados
var page = pp.pageIndex;    // Página atual (começa em 1)
var pageSize = pp.pageSize; // Tamanho da página (padrão 50, retorna ~57 com patrocinados)
```

### Validação de filtros

Sempre verificar se os filtros foram aplicados corretamente:
```javascript
var activeFilters = pp.filters;
// Exemplo: {"fuel":["5"],"cartype":["3"],"gearbox":["2"],"price":{"max":"100000"}}
```

Se um filtro não aparece no objeto `filters`, significa que o OLX ignorou o parâmetro.

### Parsing de um anúncio

```javascript
ads.forEach(function(ad) {
  // Properties (36 campos possíveis)
  var props = {};
  ad.properties.forEach(function(p) { props[p.name] = p.value; });

  // Dados básicos
  var titulo = ad.subject;
  var preco = parseInt((ad.price || '0').replace(/\D/g, '')) || 0;
  var km = parseInt(String(props.mileage || '0').replace(/\D/g, '')) || 0;

  // Localização (usar locationDetails, não location)
  var loc = ad.locationDetails || {};
  var cidade = loc.municipality;   // "São Bernardo do Campo"
  var bairro = loc.neighbourhood;  // "Anchieta"
  var ddd = loc.ddd;               // "11"
  var uf = loc.uf;                 // "SP"

  // Veículo
  var marca = props.vehicle_brand;    // "Toyota"
  var modelo = props.vehicle_model;   // "Hilux CD SR 4X4 2.8 TDI Diesel Aut."
  var tipo = props.cartype;           // "Pick-up"
  var ano = props.regdate;            // "2020"
  var combustivel = props.fuel;       // "Diesel"
  var cambio = props.gearbox;         // "Automático"
  var motor = props.motorpower;       // "2.0 - 2.9"
  var cor = props.carcolor;           // "Branco"
  var portas = props.doors;           // "4 portas"
  var direcao = props.car_steering;   // "Hidráulica"

  // Situação
  var leilao = props.has_auction;         // "Sim" / "Não"
  var unicoDono = props.owner;            // "Sim" / "Não"
  var quitado = props.is_settled;         // "Sim" / "Não"
  var financiado = props.is_funded;       // "Sim" / "Não"
  var ipvaPago = props.has_paid_ipva;     // "Sim" / "Não"
  var comMultas = props.has_with_fine;    // "Sim" / "Não"
  var kitGnv = props.has_gnv_kit;         // "Sim" / "Não"
  var aceitaTroca = props.exchange;       // "Sim" / "Não"

  // Opcionais (CSV)
  var opcionais = props.car_features;     // "Air bag, Som, Câmera de ré"

  // Conservação
  var manual = props.owner_manual;        // "Sim" / "Não"
  var chaveReserva = props.extra_key;     // "Sim" / "Não"

  // Vistoria
  var vistoriaStatus = props.condition_report_status; // "UNAVAILABLE", "APPROVED", "REJECTED"
  var vistoriadora = props.condition_report_inspector; // "Visão Total"

  // Vendedor
  var profissional = ad.professionalAd;   // true/false
  var vendedorOnline = ad.accountActivityStatus.isOnline;
  var googleRating = ad.userGoogleRating; // 4

  // Mídia
  var fotos = ad.imageCount;              // 16
  var videos = ad.videoCount;             // 0
  var url360 = ad.view360.url;            // null ou URL

  // Imagens
  var primeiraFoto = ad.images[0].original; // URL jpg
  var primeiraWebp = ad.images[0].originalWebp; // URL webp

  // Serviços OLX
  var olxPay = ad.olxPay.enabled;         // true/false
  var olxDelivery = ad.olxDelivery.enabled; // true/false

  // Timestamps
  var dataPublicacao = ad.date;           // Unix timestamp
  var dataOriginal = ad.origListTime;     // Unix timestamp
  var ultimoBump = ad.lastBumpAgeSecs;    // "0"

  // Tags visuais
  var tags = ad.tags;    // [{label:"Aceita troca"}, {label:"IPVA pago"}]
  var pills = ad.vehiclePills; // idem
});
```

## Cuidados e limitações

### Dados inconsistentes (~12%)
Cerca de 12% dos anúncios têm properties vazias. Sempre tratar campos como opcionais.

### Preço "a combinar"
Preço pode ser "R$ 0" ou string vazia. Filtrar `preco > 0` antes de calcular médias.

### Mileage zero
KM = 0 pode significar "não informado" ou "0 km real". Tratar separadamente nas métricas.

### Rate limiting
Manter ~1 req/segundo é seguro. Para scraping em escala (>1000 req/hora), usar proxy rotation.

### Máximo de páginas
OLX limita a ~100 páginas (5000 anúncios). Para buscas com mais resultados, subdividir por filtros.

### BLOCKED: Cookie/query string data
Se o JavaScript retornar esse erro, evitar incluir URLs ou query strings no output do JS. Extrair dados sem montar strings que contenham URLs.

## Exemplos de URLs prontas

```
# Pick-up diesel automática até R$100k em SP
?ctp=3&fu=5&gb=2&pe=100000&me=200000

# SUV flex automática 2018+ até R$80k em RJ
?ctp=5&fu=3&gb=2&pe=80000&rs=2018

# Quitados, sem leilão, com 4x4
?ctp=3&fu=5&gb=2&pe=100000&fncs=1&cf=20

# Apenas profissionais, mais recentes
?ctp=3&fu=5&gb=2&pe=100000&f=1&sp=date

# Abaixo da FIPE, com histórico veicular
?fpdll=1&vtvh=1&ctp=3&fu=5&gb=2
```

## Volumes de referência (SP, Abril 2026)

| Filtro | Total |
|---|---|
| Todos os carros em SP | 174.979 |
| Pick-up + Diesel + Auto + <R$100k + <200k km | 293 |
| Pick-up + Diesel/Flex + Auto + <R$100k + <200k km | 904 |
| Pick-up + Diesel + Auto + <R$100k + abaixo FIPE | 432 |

---

## Estratégia de Coleta OLX

O OLX tem ~175.000 veículos só em SP, mas limita a 100 páginas por busca (5.000 anúncios). Uma busca única "todos os carros de SP" pega apenas os 5.000 primeiros. A estratégia se divide em duas fases: crawl completo inicial e coletas incrementais.

### Fase 1 — Crawl Completo (primeira execução)

Segmentar por tipo de veículo (`ctp`) + faixa de preço (`ps`/`pe`) para que cada busca fique abaixo de 5.000 resultados:

| Segmento | ctp | Faixas de preço | Volume estimado | Buscas |
|---|---|---|---|---|
| Pick-up | 3 | 0-80k / 80k-150k / 150k+ | ~8.000 | 3 |
| SUV | 5 | 0-50k / 50-80k / 80-120k / 120-200k / 200k+ | ~35.000 | 5 |
| Sedã | 8 | 0-30k / 30-60k / 60-100k / 100-200k / 200k+ | ~45.000 | 5 |
| Hatch | 9 | 0-25k / 25-50k / 50-80k / 80k+ | ~55.000 | 4 |
| Van/Utilitário | 7 | 0-80k / 80k+ | ~8.000 | 2 |
| Conversível+Coupé+Perua+Outros | 2,6,10,11,12 | 0-80k / 80k+ | ~24.000 | 4 |
| **TOTAL** | | | **~175.000** | **~23 buscas** |

Cada busca pode ter até 100 páginas. Total estimado: ~3.500 páginas. A 1 req/segundo com jitter, ~58 minutos para popular o banco inteiro de SP.

**Regra de segmentação:** se uma busca retorna `totalOfAds > 4.500`, subdividir por faixa de preço adicional ou por combustível (`fu`). Verificar sempre o campo `totalOfAds` do `pageProps` para confirmar cobertura.

### Fase 2 — Coleta Incremental (a cada 2 horas)

Ao invés de re-coletar tudo, usar `sp=date` (ordenar por mais recentes) e varrer até encontrar um anúncio já conhecido no banco.

**Lógica:**
1. Para cada segmento configurado, fazer GET com `sp=date&o=1`
2. Parsear anúncios, verificar `listId` contra o banco
3. Se todos os anúncios da página são novos, avançar para `o=2`, `o=3`...
4. Quando encontrar o primeiro `listId` já conhecido, parar aquele segmento
5. Persistir novos anúncios no Redis + PostgreSQL

**Volume estimado por ciclo:**
Um veículo seminovo fica ~15-30 dias anunciado. Com ~175k anúncios ativos, entram ~500-1.500 novos a cada 2 horas (175k / 20 dias média / 12 ciclos por dia). Isso são ~10-30 páginas por segmento, ou ~75-150 requisições totais. A 1 req/segundo, **1 a 3 minutos** ao invés de 58.

### Detecção de Alteração de Preço

Além dos novos, queremos detectar anúncios existentes que mudaram de preço.

**Estratégia:** a cada ciclo, escolher aleatoriamente 10-20% dos segmentos e fazer scan mais profundo (20-30 páginas), comparando `preco` com o último `ad_snapshot` no PostgreSQL. Se houver diferença, criar novo snapshot.

**Sinais valiosos para o comprador:**
- "O preço caiu R$5.000 na última semana" → oportunidade
- "O preço subiu R$3.000" → vendedor testando mercado
- "3 reduções de preço em 30 dias" → vendedor desesperado, boa margem de negociação

**Volume:** ~200-400 requests por ciclo (~3-7 minutos)

### Detecção de Anúncios Removidos

Anúncios podem ser removidos por venda, expiração ou exclusão do vendedor. Precisamos marcar como `status = removido` no banco.

**Estratégia:** a cada ciclo, pegar uma amostra de anúncios ativos no banco que não foram vistos na última coleta incremental. Fazer HEAD request na URL original:
- HTTP 200 → ainda ativo, manter
- HTTP 404/410 → removido, atualizar status e `removed_at`
- HTTP 403/429 → rate limited, tentar no próximo ciclo

**Volume:** ~100-200 requests por ciclo (~2-3 minutos)

**Inteligência derivada:**
- Tempo médio de venda por modelo/região (diferença entre `first_seen_at` e `removed_at`)
- "Este modelo costuma ser vendido em 8 dias nessa faixa" → urgência legítima baseada em dados

### Budget Total de Requests por Ciclo (2h)

| Operação | Requests | Tempo estimado |
|---|---|---|
| Incremental (novos anúncios) | 75-150 | ~2 min |
| Detecção de alteração de preço | 200-400 | ~5 min |
| Verificação de removidos | 100-200 | ~3 min |
| **TOTAL** | **375-750** | **~10 min** |

Sobra 1h50 de folga na janela de 2h. O budget fica muito abaixo do limite de rate limiting (~3.600 req/hora a 1 req/s).

### Intercalação de Segmentos

Nunca fazer mais de 30 requisições seguidas para o mesmo segmento de busca. Intercalar para simular comportamento humano:

```
Ciclo exemplo:
  3 páginas Pick-up 0-80k
  3 páginas SUV 0-50k
  3 páginas Sedã 0-30k
  3 páginas Hatch 0-25k
  3 páginas Pick-up 80k-150k
  ... (rotação)
```

Isso distribui as requisições entre diferentes URLs e evita padrão de acesso sequencial que dispara detecção de bot.

### Campos-chave para Dedup e Tracking

| Campo | Uso |
|---|---|
| `listId` | ID único do anúncio — chave primária para dedup |
| `origListTime` | Timestamp original da publicação — detecta "republicações" (mesmo carro, novo anúncio) |
| `date` | Timestamp da última atualização — detecta bumps/destaques |
| `lastBumpAgeSecs` | Segundos desde o último bump — "0" significa bump recente |
| `price` / `priceValue` | Comparar com último snapshot para detectar alteração |
| `professionalAd` | Loja vs PF — padrões de comportamento diferentes |

### Detecção de Republicação (mesmo carro, novo anúncio)

Vendedores frequentemente deletam e recriam o anúncio para aparecer como "novo". Detectar comparando:
- Mesmo `vehicle_brand` + `vehicle_model` + `regdate` + `mileage` (±5%) + mesma cidade
- `listId` diferente mas perfil muito similar
- Marcar como `republished_from = listId_anterior` no banco
