---
name: carbot-project
description: "Contexto completo do projeto CarBot — agente inteligente para compra de veículos seminovos. Use esta skill SEMPRE que o usuário mencionar CarBot, compra de veículos, seminovos, agente automotivo, ou quiser trabalhar no projeto de compra de carros. Também acione quando mencionar: APIs de plataformas (OLX, Webmotors, iCarros), web scraping automotivo, vistoria veicular, tabela FIPE, desvalorização de veículos, TCO automotivo, fraudes em compra de carros, ou qualquer tarefa de desenvolvimento, documentação ou planejamento relacionada ao CarBot."
---

# CarBot — Agente Inteligente para Compra de Veículos Seminovos

## Visão Geral

O CarBot é o primeiro agente de compra de veículos seminovos **100% orientado ao comprador** no Brasil. Enquanto todas as plataformas existentes (OLX, Webmotors, iCarros, Kavak) são monetizadas pelo vendedor, o CarBot inverte essa lógica: entende o perfil do comprador, aplica inteligência técnica e financeira, agrega anúncios de múltiplas plataformas, filtra riscos e fraudes, e acompanha o comprador desde a descoberta até o pós-compra.

**Formato:** Plataforma web (React) + app mobile, com chat conversacional por texto e voz.
**Público-alvo:** Pessoa física compradora de veículos seminovos.
**Nível de dados:** Premium (análise de mercado regional, tendências, melhor época para comprar).

## Equipe

Três sócios: **Leonardo (Léo)**, **Eric** e **Caio**.
Repositório: `https://github.com/leonardopisaneschi/carbot.ai.git`

## Documento de Referência

O documento completo de modelo de negócio e requisitos está em `CarBot_Modelo_de_Negocio_e_Requisitos.docx` na pasta do projeto ("Comprador veiculos"). Contém: Business Model Canvas, 20 user stories, regras de negócio, arquitetura, stack, integrações, roadmap 18 meses, KPIs e análise competitiva.

---

## As 17 Dores do Comprador

Estas são as dores centrais mapeadas a partir de pesquisa qualitativa. Toda decisão de produto deve resolver uma ou mais destas dores.

1. **Assimetria de informação técnica** — O comprador não entende diesel vs flex, automático vs manual, motor 1.0 turbo vs 2.0 aspirado. Também não sabe comparar adequação funcional: espaço de carga, porta-malas, número de lugares reais.

2. **Labirinto de versões** — Dentro do mesmo modelo (ex: Hilux) existem SRX, SRV, SR, STD, cada uma com pacotes diferentes. O comprador não sabe qual faz sentido pro perfil dele e acaba comprando a primeira que aparece.

3. **Desvalorização invisível** — O comprador não tem visão das curvas de depreciação. Um carro mais barato na compra pode ser péssimo negócio se desvaloriza rápido. Depende do horizonte de uso (1, 2, 5 anos).

4. **Quilometragem como número vazio** — Cada faixa de km tem significado diferente: 20k (peças originais), 80-100k (desgaste natural, trocas preventivas), 200k+ (problemas estruturais, borrachas, coxins, motor). O comprador não traduz km em custo futuro.

5. **Problemas crônicos ocultos** — Defeitos recorrentes por modelo/ano/versão (ex: Jeep Compass com câmbio). Informações públicas em fóruns e recalls, mas o comprador não consulta.

6. **Falta de vistoria digital** — Multas, IPVA, restrições, alienação, sinistro, roubo, leilão — tudo consultável por placa/chassi, mas o comprador desconhece.

7. **Plataformas orientadas ao vendedor** — OLX, Webmotors, iCarros monetizam pelo vendedor. Quem paga mais aparece no topo. Loja física tem conflito de interesse. Ninguém defende o comprador.

8. **Custo operacional invisível (TCO)** — IPVA, seguro (varia drasticamente entre modelos), manutenção, custo de peças por montadora, índice de furto. Dois carros com mesmo preço podem ter TCO muito diferente.

9. **Falta de viabilidade financeira personalizada** — Sem cruzamento automático de tabela FIPE com orçamento real, desvalorização e TCO para identificar o melhor custo-benefício.

10. **Risco de fraude digital** — Preço 20-40% abaixo da FIPE, engenharia social (urgência, histórias emocionais), intermediário fantasma (golpista entre comprador e vendedor real, desvia pagamento).

11. **Risco de fraude física** — Chassi remarcado, motor trocado, vidros não originais, estrutura comprometida por batida. Só vistoria cautelar presencial detecta.

12. **Falta de assessoria completa** — Checklist de inspeção presencial, verificação de originalidade (rodas, blindagem com delaminação, acessórios de série, som original), análise de fotos do anúncio por IA.

13. **Financiamento às cegas** — Taxas abusivas, diferença SAC vs Price, custo total real do financiamento invisível. Lojas empurram financiamentos ruins por comissão.

14. **Solidão no pós-compra** — Sem orientação sobre transferência, prazos, despachante, primeira revisão, oficina confiável.

15. **Histórico de manutenção opaco** — "Sempre revisado em concessionária" sem comprovação. Algumas montadoras têm registro digital por chassi.

16. **Melhor momento de comprar** — Sazonalidade: início do ano (IPVA), lançamento de modelo novo, final de mês (meta de loja).

17. **Teste drive às cegas** — Comprador não sabe ouvir motor frio, testar câmbio em subida, sentir frenagem, verificar alinhamento, ouvir suspensão.

---

## As 8 Fases da Solução

### Fase 1 — Descoberta do Perfil
Chat conversacional (texto ou voz), 5-10 perguntas adaptativas. Mapeia: orçamento, tipo de uso, necessidades específicas, preferências, horizonte de uso, tolerância a risco e quilometragem. Aprofunda se houver dúvidas.
**Saída:** Perfil detalhado do comprador.

### Fase 2 — Inteligência Interna
Consulta base de conhecimento automotivo e identifica modelos/versões/anos que atendem ao perfil. Aplica: adequação funcional, problemas crônicos, desvalorização, TCO.
**Saída:** Lista qualificada de veículos ideais com justificativa técnica.

### Fase 3 — Busca no Mercado
Conecta via API/scraping a todas as plataformas (OLX, Webmotors, iCarros, Kavak, Mercado Livre) + parceiros físicos.
**Saída:** Base de anúncios reais de múltiplas fontes.

### Fase 4 — Filtro Inteligente
Elimina lixo: fraudes (preço muito abaixo FIPE, contas novas), km excessiva, problemas crônicos graves. Ranqueia por relevância do COMPRADOR (não do vendedor): aderência ao perfil (40%), custo-benefício real com TCO e desvalorização (30%), saúde do anúncio (20%), proximidade geográfica (10%).
**Saída:** Ranking orientado ao comprador.

### Fase 5 — Apresentação dos Resultados
Ranking visual com score/estrelas. Ao clicar: link do anúncio, reputação do vendedor (tempo na plataforma, histórico), comparativo preço vs FIPE, gráfico de desvalorização, TCO anual, problemas crônicos, análise de originalidade por fotos (IA), detalhamento da versão.
**Saída:** Interface rica com dados acionáveis.

### Fase 6 — Consultoria Pré-Compra
Vistoria digital rápida (multas, IPVA, restrições), checklist personalizado por modelo, roteiro de test drive, orientações de segurança, direcionamento para vistoria cautelar (parceiro), cotação online de seguro (corretora parceira), simulação de custos de transferência. Exibe rede de parceiros integrados.
**Saída:** Comprador totalmente preparado.

### Fase 7 — Estratégia de Negociação e Financiamento
Calcula valor justo: tabela FIPE - multas - IPVA atrasado - manutenções imediatas - itens faltantes. Sugere valor de abertura e faixa de fechamento com argumentos baseados em dados. Simulador financeiro compara taxas de múltiplas instituições, custo total (SAC vs Price), pré-aprovação via fintechs/bancos parceiros.
**Saída:** Estratégia de negociação + financiamento simulado.

### Fase 8 — Acompanhamento Pós-Compra
Checklist de transferência, despachante parceiro, ativação de seguro, calendário de manutenção programada, sugestão de oficinas por região, canal de dúvidas. Avaliação do veículo pela comunidade.
**Saída:** Comprador acompanhado + fidelização.

---

## Modelo de Receita

| Canal | Descrição | Prioridade |
|-------|-----------|------------|
| Freemium | Busca básica grátis, relatórios premium pagos | Alta |
| Assinatura | Acesso ilimitado mensal/anual | Alta |
| Comissão de Seguros | Via corretoras parceiras, fechamento online na plataforma | Alta |
| Comissão de Financiamento | Via fintechs/bancos parceiros, contratação na plataforma | Alta |
| Comissão de Vistorias | Percentual sobre vistorias cautelares via parceiros | Média |
| Comissão de Despachante | Serviços de transferência integrados | Média |
| Dados e Insights B2B | Relatórios para concessionárias e seguradoras (futuro) | Baixa |

---

## Integrações e APIs — Estado Atual da Pesquisa

Pesquisa realizada em abril/2026 sobre disponibilidade de APIs das principais plataformas:

### OLX (developers.olx.com.br)
- **API existe**, mas é orientada ao vendedor/integrador
- Funcionalidades: autenticação OAuth, importação de anúncios (inserção/edição/deleção), consulta de status, catálogo de marcas/modelos, webhooks
- **NÃO oferece endpoint de busca pública de anúncios**
- Requer contrato de parceiro aprovado
- Contato: suporteintegrador@olxbr.com
- **Útil pro CarBot:** catálogo de marcas/modelos apenas

### Webmotors (via Cockpit CRM)
- API focada em publicação de anúncios, gestão de estoque e integração de leads
- Requer contratação de produto Webmotors
- Perfil: "Integrador de API" ou "API Site"
- **NÃO oferece busca pública de anúncios**

### iCarros (icarros.com.br/apidocs)
- OAuth 2.0 com client_id/client_secret fornecidos por eles
- Endpoints focados em dealers/revendedores (ex: `/rest/dealerservice/dealer`)
- Acesso restrito a parceiros aprovados (montadoras, integradores)
- **NÃO oferece busca pública de anúncios**

### Facebook Marketplace
- API antiga descontinuada (blog post de 2007)
- **NÃO possui API pública atual** para busca de veículos
- Restrições de privacidade da Meta impedem acesso

### Mercado Livre (developers.mercadolivre.com.br)
- API mais robusta entre todas, com OAuth 2.0
- Permite consultar anúncios publicados (parcial)
- **Melhor opção** para integração na Fase 1

### Web Scraping — Análise de Viabilidade (OLX)

Teste realizado com sucesso: busca de "Hilux em SP" retornou 1.503 resultados com dados ricos (modelo, versão, ano, km, cor, motor, preço, localização, tipo de vendedor, avaliação).

**Prós:**
- Dados ricos e bem estruturados no HTML
- URLs previsíveis e parametrizáveis (filtros por estado, marca, modelo, preço, km, combustível, câmbio)
- Paginação clara (50 resultados/página)

**Contras e riscos de escalabilidade:**
- **Bloqueio/rate limiting:** OLX implementa proteções anti-bot. Volume alto dispara erro 403. Proxies rotativos encarecem (R$ 3.000-8.000/mês de infra)
- **Fragilidade:** Qualquer mudança no HTML quebra o scraper sem aviso
- **Legalidade:** Termos de uso da OLX proíbem scraping. Risco de notificação extrajudicial se crescer
- **Latência:** 2-5 segundos por busca via scraping vs 200ms via API. Experiência do usuário degrada

**Estratégia recomendada:**
1. **MVP (0-3 meses):** Scraping com cache agressivo (atualizar a cada 1-2h, não tempo real) + API do Mercado Livre
2. **Escala (3-6 meses):** Com tração comprovada, negociar parceria comercial direta com OLX, Webmotors e iCarros

---

## Stack Tecnológico Recomendado

| Camada | Tecnologia |
|--------|------------|
| Frontend Web | React/Next.js, TypeScript, Tailwind CSS, PWA |
| Frontend Mobile | React Native ou Flutter |
| Backend | Node.js (NestJS) ou Python (FastAPI) |
| IA Conversacional | LLM (considerar LLM própria na AWS para reduzir custos) + Whisper para voz |
| Visão Computacional | Modelo treinado para detecção de originalidade em fotos |
| Banco de Dados | PostgreSQL + MongoDB + Redis |
| Mensageria | Apache Kafka ou RabbitMQ |
| Infraestrutura | AWS com Kubernetes |
| CI/CD | GitHub Actions + Docker |

**Nota sobre LLM:** A equipe está considerando hospedar uma LLM própria na AWS para evitar custos de API por volume de mensagens. Avaliar trade-off entre custo de GPU vs custo de API (Claude/GPT) conforme volume de usuários.

---

## Roadmap

| Fase | Prazo | Entregas |
|------|-------|----------|
| MVP | 0-3 meses | Chat de perfil, base de 50 modelos, 2 plataformas, ranking básico, vistoria digital |
| Inteligência | 3-6 meses | Todos os modelos, desvalorização/TCO, filtro de fraude, reputação vendedor, voz |
| Parcerias | 6-9 meses | Vistorias cautelares, seguros, financiamento, checklist personalizado |
| Premium | 9-12 meses | IA de imagens, negociação com valor justo, tendências de preço |
| Ecossistema | 12-18 meses | App mobile, pós-compra, comunidade, B2B |
