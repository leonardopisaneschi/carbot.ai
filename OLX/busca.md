# OLX - Mapeamento Completo de Busca e Scraping

> Documento técnico do projeto CarBot
> Última atualização: 2026-04-10
> Autores: Leonardo Pisaneschi, Claude (AI)

---

## 1. Visão Geral

O OLX utiliza Next.js como framework frontend. Todos os dados dos anúncios e filtros são disponibilizados via JSON embutido no HTML da página, no elemento `<script id="__NEXT_DATA__">`. Isso permite extração estruturada sem necessidade de parsing de HTML.

### 1.1 Estrutura da URL Base

```
https://www.olx.com.br/{categoria}/{subcategoria}/{localização}?{filtros}
```

**Exemplo completo:**
```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&o=2
```

### 1.2 Categorias Relevantes (Autos)

| Segmento URL | Categoria | ID |
|---|---|---|
| `autos-e-pecas/carros-vans-e-utilitarios` | Carros, vans e utilitários | 2020 |
| `autos-e-pecas` | Autos e Peças (pai) | 2000 |

### 1.3 Localização via Path

| Formato | Exemplo | Funciona com filtros? |
|---|---|---|
| `/estado-{uf}` | `/estado-sp` | **SIM** |
| `/` (sem localização) | Nacional | **SIM** |
| `/{região}` | `/grande-abc`, `/sao-paulo-e-regiao` | **NÃO** (ignora filtros, retorna nacional) |
| `/{marca}/estado-{uf}` | `/toyota/estado-sp` | **SIM** (filtra marca + estado) |

> **IMPORTANTE:** Filtros por região sub-estadual via path NÃO funcionam corretamente quando combinados com query strings de filtro. O resultado é sempre nacional. Usar apenas `estado-{uf}` e filtrar por cidade/DDD programaticamente.

### 1.4 Paginação

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `o` | Número da página (começa em 1) | `o=2` |
| - | pageSize padrão | 50 anúncios por requisição |
| - | Anúncios retornados no JSON | ~57 (inclui ads patrocinados) |
| - | `totalOfAds` | Total de resultados disponíveis |
| - | Limite máximo de páginas | ~100 (5000 anúncios) |

### 1.5 Ordenação

| Parâmetro | Valores | Descrição |
|---|---|---|
| `sp` | `relevance` | Mais Relevantes (padrão) |
| `sp` | `date` | Mais Recentes |
| `sp` | `price` | Menor Preço |
| `sp` | `biggest_price` | Maior Preço |

---

## 2. Filtros de URL (Query Strings)

### 2.1 Tipo de Veículo (`ctp`)

| Valor | Tipo |
|---|---|
| `2` | Conversível |
| `3` | **Pick-up** |
| `5` | SUV |
| `6` | Buggy |
| `7` | Van/Utilitário |
| `8` | Sedã |
| `9` | Hatch |
| `10` | Caminhão Leve |
| `11` | Coupé |
| `12` | Perua |

**Multi-select:** `ctp=3&ctp=5` (Pick-up + SUV)

### 2.2 Combustível (`fu`)

| Valor | Combustível |
|---|---|
| `1` | Gasolina |
| `2` | Álcool |
| `3` | Flex |
| `5` | **Diesel** |
| `6` | Híbrido |
| `7` | Elétrico |

**Multi-select:** `fu=5&fu=3` (Diesel + Flex)

### 2.3 Câmbio (`gb`)

| Valor | Câmbio |
|---|---|
| `1` | Manual |
| `2` | **Automático** |
| `3` | Semi-Automático |
| `4` | Automatizado |

**Multi-select:** `gb=2&gb=3` (Automático + Semi-Automático)

### 2.4 Preço (`ps` / `pe`)

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `ps` | Preço mínimo (em reais, sem pontos) | `ps=30000` |
| `pe` | Preço máximo | `pe=100000` |

**Ambos opcionais.** Se omitido, não há limite.

### 2.5 Quilometragem (`ms` / `me`)

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `ms` | KM mínimo | `ms=0` |
| `me` | KM máximo | `me=200000` |

**Valores aceitos na interface:** 0, 5.000, 10.000, 20.000, 30.000, 40.000, 60.000, 80.000, 100.000, 120.000, 140.000, 160.000, 180.000, 200.000, 250.000, 300.000, 400.000, 500.000

> Nota: Na URL aceita qualquer valor numérico inteiro (ex: `me=150000`), mas a interface mostra apenas os valores acima.

### 2.6 Ano (`rs` / `re`)

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `rs` | Ano mínimo | `rs=2015` |
| `re` | Ano máximo | `re=2023` |

**Range:** 1950 até o ano atual.

### 2.7 Potência do Motor (`motp`)

| Valor | Motor |
|---|---|
| `1` | 1.0 |
| `2` | 1.2 |
| `3` | 1.3 |
| `4` | 1.4 |
| `5` | 1.5 |
| `6` | 1.6 |
| `7` | 1.7 |
| `8` | 1.8 |
| `9` | 1.9 |
| `10` | 2.0 - 2.9 |
| `11` | 3.0 - 3.9 |
| `12` | 4.0 ou mais |

### 2.8 Direção (`ics`)

| Valor | Tipo |
|---|---|
| `1` | Hidráulica |
| `2` | Elétrica |
| `3` | Mecânica |
| `5` | Eletro-hidráulica |

### 2.9 Cor (`cac`)

| Valor | Cor |
|---|---|
| `1` | Preto |
| `2` | Branco |
| `3` | Prata |
| `4` | Vermelho |
| `5` | Cinza |
| `6` | Azul |
| `7` | Amarelo |
| `8` | Verde |
| `9` | Laranja |
| `10` | Outra |

### 2.10 Portas (`cad`)

| Valor | Portas |
|---|---|
| `1` | 2 portas |
| `2` | 4 portas |
| `3` | 3 portas |

### 2.11 Final de Placa (`et`)

| Valor | Final |
|---|---|
| `1` | 0 |
| `2` | 1 |
| `3` | 2 |
| `4` | 3 |
| `5` | 4 |
| `6` | 5 |
| `7` | 6 |
| `8` | 7 |
| `9` | 8 |
| `10` | 9 |

### 2.12 Aceita Trocas (`exc`)

| Valor | Descrição |
|---|---|
| `1` | Sim |
| `2` | Não |

### 2.13 Kit GNV (`hgnv`)

| Valor | Descrição |
|---|---|
| `1` | Sim |
| `0` | Não |

### 2.14 Estado Financeiro (`fncs`)

| Valor | Status |
|---|---|
| `1` | Quitado |
| `2` | Financiado |

### 2.15 Documentação e Regularização (`doc`)

| Valor | Descrição |
|---|---|
| `1` | IPVA Pago |
| `2` | Com multas |
| `3` | **De leilão** |

**Multi-select:** `doc=1&doc=3` (IPVA Pago + De leilão)

### 2.16 Conservação e Garantia (`cnwa`)

| Valor | Descrição |
|---|---|
| `1` | Único dono |
| `2` | Com manual |
| `3` | Chave reserva |
| `4` | Revisões feitas em concessionária |
| `5` | Com garantia |

### 2.17 Opcionais do Veículo (`cf`)

| Valor | Opcional |
|---|---|
| `1` | Ar condicionado |
| `4` | Trava elétrica |
| `5` | Air bag |
| `6` | Alarme |
| `7` | Som |
| `8` | Sensor de ré |
| `9` | Câmera de ré |
| `10` | Blindado |
| `11` | Bancos de couro |
| `12` | Computador de bordo |
| `13` | Conexão USB |
| `15` | Interface bluetooth |
| `16` | Navegador GPS |
| `17` | Controle automático de velocidade |
| `18` | Rodas de liga leve |
| `19` | Teto solar |
| `20` | Tração 4x4 |

**Multi-select:** `cf=20&cf=9` (Tração 4x4 + Câmera de ré)

### 2.18 Tipo de Anunciante (`f`)

| Valor | Tipo |
|---|---|
| `0` | Particular |
| `1` | Profissional |
| _(omitido)_ | Ambos |

### 2.19 Toggles (Booleanos)

| Parâmetro | Descrição | Valor para ativar |
|---|---|---|
| `vtvh` | Histórico Veicular disponível | `1` |
| `crss` | Vistoriado | `1` |
| `fpdll` | Ofertas abaixo da FIPE | `1` |

### 2.20 Marca / Modelo / Versão

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `vb` | Marca (vehicle brand) | `vb=Toyota` |
| `vm` | Modelo (vehicle model) | `vm=Hilux` |
| `vv` | Versão (vehicle version) | `vv=SRV+4x4` |

> **Nota:** Os valores de marca/modelo/versão são carregados dinamicamente (datasource remoto). Os valores exatos dependem de qual marca/modelo está selecionado. Alternativa mais confiável: usar o path da marca (ex: `/toyota/estado-sp`) + query string `q=hilux`.

### 2.21 Busca por Texto (`q`)

| Parâmetro | Descrição | Exemplo |
|---|---|---|
| `q` | Texto livre de busca | `q=hilux+diesel` |

> Busca no título e descrição do anúncio. Pode ser combinado com todos os outros filtros.

---

## 3. Estrutura do Anúncio (__NEXT_DATA__)

### 3.1 Campos Principais do Ad

| Campo | Tipo | Descrição | Exemplo |
|---|---|---|---|
| `subject` | string | Título do anúncio | `"Toyota Hilux CD SR 4X4 2.8 TDI Diesel Aut. 2020"` |
| `title` | string | Igual ao subject | idem |
| `price` | string | Preço formatado | `"R$ 40.000"` |
| `priceValue` | string | Igual ao price | idem |
| `oldPrice` | object/null | Preço anterior (se houve redução) | `null` |
| `listId` | number | ID único do anúncio | `1493121735` |
| `friendlyUrl` | string | URL completa do anúncio | `"https://sp.olx.com.br/..."` |
| `url` | string | URL do anúncio | similar |
| `date` | number | Unix timestamp da publicação | `1775859643` |
| `origListTime` | number | Timestamp original da publicação | `1774979897` |
| `lastBumpAgeSecs` | string | Segundos desde o último bump/destaque | `"0"` |
| `professionalAd` | boolean | Se é anunciante profissional (loja) | `true` |
| `isFeatured` | boolean | Se é anúncio destacado/pago | `false` |
| `fixedOnTop` | boolean | Se está fixo no topo | `false` |
| `isFavorited` | boolean | Se foi favoritado pelo usuário | `false` |
| `isChatEnabled` | boolean | Se chat está habilitado | `true` |
| `imageCount` | number | Quantidade de fotos | `16` |
| `videoCount` | number | Quantidade de vídeos | `0` |
| `position` | number | Posição na lista de resultados | `0` |
| `category` | string | Nome da categoria | `"Carros, vans e utilitários"` |
| `categoryName` | string | idem | idem |
| `listingCategoryId` | number | ID da categoria | `2020` |
| `searchCategoryLevelZero` | number | Categoria pai | `2000` |
| `searchCategoryLevelOne` | number | Subcategoria | `2020` |
| `adReply` | string | Resposta do vendedor | `"0"` |
| `priceReductionBadge` | boolean | Se tem badge de redução de preço | `false` |

### 3.2 Localização

**Campo `location`** (string formatada):
```
"São Bernardo do Campo, Anchieta - DDD 11"
```
Formato: `"{município}, {bairro} - DDD {ddd}"`

**Campo `locationDetails`** (objeto estruturado):

| Subcampo | Tipo | Descrição | Exemplo |
|---|---|---|---|
| `municipality` | string | Nome da cidade | `"São Bernardo do Campo"` |
| `neighbourhood` | string | Nome do bairro | `"Anchieta"` |
| `ddd` | string | Código DDD | `"11"` |
| `uf` | string | Sigla do estado | `"SP"` |

### 3.3 Imagens (`images[]`)

Array de objetos. Cada imagem:

| Subcampo | Tipo | Descrição |
|---|---|---|
| `original` | string | URL da imagem JPG |
| `originalWebp` | string | URL da imagem WebP |

**Padrão de URL:** `https://img.olx.com.br/images/{hash}.jpg`

### 3.4 Usuário/Vendedor (`user`)

| Subcampo | Tipo | Descrição |
|---|---|---|
| `configs.proAccount` | boolean | Se é conta profissional |
| `configs.chatEnabled` | boolean | Se chat está ativo |
| `configs.showFullAddress` | boolean | Se mostra endereço completo |

### 3.5 Status de Atividade (`accountActivityStatus`)

| Subcampo | Tipo | Descrição |
|---|---|---|
| `isOnline` | boolean | Se o vendedor está online |

### 3.6 Google Reviews

| Campo | Tipo | Descrição |
|---|---|---|
| `userGoogleReviewsVisible` | boolean | Se avaliações Google são visíveis |
| `userGoogleRating` | number | Nota média no Google |

### 3.7 Tags e Pills (`tags[]`, `vehiclePills[]`)

Array de objetos com `label`. Exemplos de labels:
- `"Aceita troca"`
- `"IPVA pago"`
- `"Único dono"`

### 3.8 Veículo - Report e Inspeção

**`vehicleReport`:**

| Subcampo | Tipo | Descrição |
|---|---|---|
| `enabled` | boolean | Se relatório está disponível |
| `title` | string | Título do relatório |
| `reportLink` | string | Link para o relatório |

**`carSpecificData`:**

| Subcampo | Tipo | Descrição |
|---|---|---|
| `hasAutosInspectionReport` | boolean | Se tem laudo de inspeção |

**`vehicleHasInspectionReport`:** boolean direto no ad.

### 3.9 Serviços OLX

**`olxPay`:**

| Subcampo | Tipo | Descrição |
|---|---|---|
| `enabled` | boolean | Se OLX Pay está ativo |
| `installments` | array | Opções de parcelamento |

**`olxDelivery`:**

| Subcampo | Tipo | Descrição |
|---|---|---|
| `enabled` | boolean | Se entrega OLX está ativa |

**`view360`:**

| Subcampo | Tipo | Descrição |
|---|---|---|
| `url` | string/null | URL do tour 360° |

---

## 4. Properties do Anúncio (36 campos)

As properties ficam em `ad.properties[]`, cada uma com `name`, `value` e `label`.

### 4.1 Dados Básicos do Veículo

| name | label | Valores exemplo |
|---|---|---|
| `category` | Categoria | `"Carros, vans e utilitários"` |
| `vehicle_brand` | Marca | `"Toyota"`, `"Ford"`, `"Volkswagen"` |
| `vehicle_model` | Modelo | `"Hilux CD SR 4X4 2.8 TDI Diesel Aut."` |
| `cartype` | Tipo de veículo | `"Pick-up"`, `"Hatch"`, `"Sedã"`, `"SUV"` |
| `regdate` | Ano | `"2020"`, `"2015"` |
| `mileage` | Quilometragem | `"90000"` (numérico como string) |
| `motorpower` | Potência do motor | `"1.0"`, `"2.0 - 2.9"`, `"3.0 - 3.9"` |
| `fuel` | Combustível | `"Diesel"`, `"Flex"`, `"Gasolina"` |
| `gearbox` | Câmbio | `"Automático"`, `"Manual"`, `"Semi-Automático"` |
| `car_steering` | Direção | `"Hidráulica"`, `"Elétrica"` |
| `carcolor` | Cor | `"Prata"`, `"Branco"`, `"Preto"` |
| `doors` | Portas | `"2 portas"`, `"4 portas"` |
| `end_tag` | Final de placa | `"6"` (número 0-9) |

### 4.2 Situação do Veículo

| name | label | Valores |
|---|---|---|
| `owner` | Único dono | `"Sim"` / `"Não"` |
| `exchange` | Aceita trocas | `"Sim"` / `"Não"` |
| `financial` | Estado financeiro | `"IPVA Pago"` |
| `has_gnv_kit` | Possui Kit GNV | `"Sim"` / `"Não"` |
| `has_auction` | **De leilão** | `"Sim"` / `"Não"` |
| `has_paid_ipva` | IPVA pago | `"Sim"` / `"Não"` |
| `has_with_fine` | Com multas | `"Sim"` / `"Não"` |
| `is_settled` | Quitado | `"Sim"` / `"Não"` |
| `is_funded` | Financiado | `"Sim"` / `"Não"` |

### 4.3 Conservação e Documentação

| name | label | Valores |
|---|---|---|
| `owner_manual` | Com manual | `"Sim"` / `"Não"` |
| `extra_key` | Chave reserva | `"Sim"` / `"Não"` |
| `indexed_financial_status` | Status financeiro | `"Quitado"` / `"Financiado"` |
| `documentation_and_regularization` | Doc. e regularização | `"IPVA Pago"` (CSV múltiplo) |
| `conservation_and_warranty` | Conservação e garantia | `"Único dono, Com manual, Chave reserva"` |
| `indexed_car_steering` | Tipo de direção | `"Hidráulica"`, `"Elétrica"` |
| `has_hydraulic_steering` | Direção hidráulica | `"Sim"` / `"Não"` |

### 4.4 Opcionais

| name | label | Valores |
|---|---|---|
| `car_features` | Opcionais | `"Air bag, Computador de bordo, Conexão USB, Som"` (CSV) |

### 4.5 Vistoria / Inspeção

| name | label | Valores |
|---|---|---|
| `on_autos_fair` | No Feirão de Autos | `"Sim"` / `"Não"` |
| `condition_report_date` | Data do relatório | ISO 8601: `"2026-04-10T22:22:54.863Z"` |
| `condition_report_detail` | Detalhes do relatório | Texto descritivo |
| `condition_report_inspector` | Vistoriadora | `"Visão Total"` |
| `condition_report_status` | Status do relatório | `"UNAVAILABLE"`, `"APPROVED"`, `"REJECTED"` |
| `condition_report_visible` | Exibe relatório | `"true"` / `"false"` |

---

## 5. Metadados da Página

### 5.1 pageProps (nível superior)

| Campo | Tipo | Descrição |
|---|---|---|
| `ads` | array | Lista de anúncios da página atual |
| `totalOfAds` | number | **Total de resultados** para a busca |
| `pageIndex` | number | Página atual (começa em 1) |
| `pageSize` | number | Tamanho da página (padrão 50) |
| `pageTitle` | string | Título da página |
| `fullPageTitle` | string | Título completo com " \| OLX" |
| `adLocation` | string | Localização formatada (ex: `"em SP"`) |
| `listingUrl` | string | URL base da listagem |
| `main_category` | string | Categoria principal (`"Autos"`) |
| `main_category_id` | string | ID da categoria principal (`"2000"`) |
| `sub_category` | string | Subcategoria (`"Carros, vans e utilitários"`) |
| `sub_category_id` | string | ID da subcategoria (`"2020"`) |
| `selectedCategoryCode` | number | Código da categoria selecionada |
| `pageType` | string | Tipo de página (`"listing"`) |
| `filters` | object | **Filtros ativos na busca atual** |
| `filtersTemplate` | object | Template com todos os filtros disponíveis |
| `locations` | array | Localizações disponíveis na hierarquia |
| `searchBoxProps` | object | Props da barra de busca |
| `categoriesList` | array | Lista de categorias disponíveis (153) |
| `metaKeywords` | string | Meta keywords da página |
| `metaDescription` | string | Meta description da página |
| `canonicalUrl` | string | URL canônica |
| `isCategoryEligible` | boolean | Se categoria é elegível para features |
| `hasAutosFairBanner` | boolean | Se mostra banner de feirão |
| `adListExtendedFeatures` | object | Features de layout |
| `abTestGroups` | object | Grupos de teste A/B ativos |

### 5.2 Objeto `filters` (filtros ativos)

Quando filtros são aplicados via URL, o objeto `filters` reflete o estado:

```json
{
  "pageIndex": 1,
  "selectedCategoryCode": "2020",
  "fuel": ["5"],
  "cartype": ["3"],
  "state": "1",
  "price": { "max": "100000" },
  "mileage": { "max": "200000" },
  "gearbox": ["2"],
  "sort": "relevance"
}
```

> **Validação:** Sempre verificar se os filtros aplicados via URL estão refletidos no objeto `filters` da resposta. Se não estiverem, significa que o OLX ignorou o parâmetro.

---

## 6. Estratégia de Scraping para o CarBot

### 6.1 URL Otimizada

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-{uf}?ctp={tipo}&fu={combustivel}&gb={cambio}&ps={preco_min}&pe={preco_max}&ms={km_min}&me={km_max}&rs={ano_min}&re={ano_max}&o={pagina}&sp={ordenacao}
```

### 6.2 Extração de Dados (JavaScript)

```javascript
var nd = JSON.parse(document.getElementById('__NEXT_DATA__').textContent);
var pp = nd.props.pageProps;
var ads = pp.ads;          // Array de anúncios
var total = pp.totalOfAds; // Total de resultados
var page = pp.pageIndex;   // Página atual
```

### 6.3 Parsing de Preço

O preço vem como string formatada. Para converter:

```javascript
var priceStr = ad.price || ad.priceValue || "0";
var priceNum = parseInt(priceStr.replace(/\D/g, '')) || 0;
// "R$ 40.000" → 40000
```

### 6.4 Parsing de Properties

```javascript
var props = {};
ad.properties.forEach(function(p) {
  props[p.name] = p.value;
});
// props.fuel → "Diesel"
// props.gearbox → "Automático"
// props.mileage → "90000"
// props.has_auction → "Não"
```

### 6.5 Parsing de Localização

```javascript
// Opção 1: locationDetails (estruturado)
var loc = ad.locationDetails;
// loc.municipality → "São Bernardo do Campo"
// loc.neighbourhood → "Anchieta"
// loc.ddd → "11"
// loc.uf → "SP"

// Opção 2: location (string)
// "São Bernardo do Campo, Anchieta - DDD 11"
```

### 6.6 Cálculo de Páginas

```javascript
var totalPages = Math.ceil(totalOfAds / 50);
// Limitar a 100 páginas (limite do OLX)
totalPages = Math.min(totalPages, 100);
```

### 6.7 Multi-Select de Filtros

Para selecionar múltiplos valores no mesmo filtro, repetir o parâmetro:
```
fu=5&fu=3       → Diesel + Flex
ctp=3&ctp=5     → Pick-up + SUV
gb=2&gb=3       → Automático + Semi-Automático
cf=20&cf=9&cf=1 → Tração 4x4 + Câmera de ré + Ar condicionado
```

---

## 7. Limitações e Cuidados

### 7.1 Anti-Bot

- Rate limiting: ~1 requisição por segundo é seguro
- User-Agent: necessário simular browser
- Cookies: podem ser necessários para evitar bloqueio
- Proxy rotation: necessário para scraping em escala (>1000 req/hora)

### 7.2 Dados Inconsistentes

- ~12% dos anúncios não possuem `properties` preenchidas (campos vazios)
- Preço pode ser "R$ 0" ou vazio em anúncios "a combinar"
- Mileage pode ser "0" quando vendedor não informou
- `cartype` pode estar vazio mesmo sendo Pick-up
- `location` vs `locationDetails`: preferir `locationDetails` quando disponível

### 7.3 Filtros de URL

- Região sub-estadual via path **NÃO funciona** com query strings de filtro
- Marca via path (`/toyota/estado-sp`) funciona mas limita a busca
- `vb`, `vm`, `vv` (marca/modelo/versão) têm valores dinâmicos — usar com cautela
- Sempre validar `filters` no JSON para confirmar que o filtro foi aplicado

### 7.4 Volumes de Teste (SP, Abril 2026)

| Filtro | Total |
|---|---|
| Sem filtro (todos carros SP) | 174.979 |
| Pick-up + Diesel + Automático + até R$100k + até 200k km | 293 |
| Pick-up + Diesel+Flex + Automático + até R$100k + até 200k km | 904 |

---

## 8. Exemplos de URLs Prontas para o CarBot

### 8.1 Busca Padrão do Léo (Pick-up Diesel Automática)

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000
```
→ 293 resultados

### 8.2 Com filtro de Ano (2015+)

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&rs=2015
```

### 8.3 Apenas Quitados, sem Leilão

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&fncs=1
```

### 8.4 Com Opcionais (4x4 + Câmera de Ré)

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&cf=20&cf=9
```

### 8.5 Mais Recentes, Página 3

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&sp=date&o=3
```

### 8.6 Profissionais (Lojas) apenas

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios/estado-sp?ctp=3&fu=5&gb=2&pe=100000&me=200000&f=1
```

### 8.7 Busca Nacional sem restrição de tipo

```
https://www.olx.com.br/autos-e-pecas/carros-vans-e-utilitarios?fu=5&gb=2&pe=100000&me=200000
```

---

## 9. Referência Rápida - Todos os Query Strings

| QS | Filtro | Tipo | Valores |
|---|---|---|---|
| `q` | Busca texto | string | texto livre |
| `ctp` | Tipo veículo | multi | 2,3,5,6,7,8,9,10,11,12 |
| `fu` | Combustível | multi | 1,2,3,5,6,7 |
| `gb` | Câmbio | multi | 1,2,3,4 |
| `ps` | Preço mínimo | number | livre |
| `pe` | Preço máximo | number | livre |
| `ms` | KM mínimo | number | livre |
| `me` | KM máximo | number | livre |
| `rs` | Ano mínimo | number | 1950-atual |
| `re` | Ano máximo | number | 1950-atual |
| `motp` | Motor | single | 1-12 |
| `ics` | Direção | multi | 1,2,3,5 |
| `cac` | Cor | multi | 1-10 |
| `cad` | Portas | multi | 1,2,3 |
| `et` | Final placa | multi | 1-10 |
| `exc` | Aceita trocas | single | 1,2 |
| `hgnv` | Kit GNV | single | 0,1 |
| `fncs` | Financeiro | single | 1,2 |
| `doc` | Documentação | multi | 1,2,3 |
| `cnwa` | Conservação | multi | 1,2,3,4,5 |
| `cf` | Opcionais | multi | 1,4,5,6,7,8,9,10,11,12,13,15,16,17,18,19,20 |
| `f` | Anunciante | single | 0,1 |
| `vtvh` | Hist. veicular | toggle | 1 |
| `crss` | Vistoriado | toggle | 1 |
| `fpdll` | Abaixo FIPE | toggle | 1 |
| `vb` | Marca | string | dinâmico |
| `vm` | Modelo | string | dinâmico |
| `vv` | Versão | string | dinâmico |
| `sp` | Ordenação | string | relevance, date, price, biggest_price |
| `o` | Página | number | 1-100 |
