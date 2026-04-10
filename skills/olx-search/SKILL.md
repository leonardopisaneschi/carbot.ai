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
