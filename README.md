# Jaguariúna Rodeo Festival — Painel Histórico e Projeção 2027–2029

Este documento explica **cada método estatístico usado no dashboard** (`index.html`): o que ele faz, por que foi escolhido, como foi implementado e quais são os limites. A ideia é que qualquer pessoa — estatística ou não — consiga auditar o raciocínio, não só o resultado.

Tudo roda no próprio navegador, em JavaScript puro, sem nenhuma biblioteca externa de estatística ou de gráficos. Todo o código está nos arquivos `parts/analysis.txt` e `parts/engine.txt`.

**Base de dados:** 37 edições (1989–2026), 134 registros de noite, 152 artistas distintos, 370 escalações no total. É uma base pequena — isso importa e volta várias vezes abaixo.

---

## 1. Contagem de recorrência

**O que é:** para cada artista, uma contagem simples de em quantas *noites* ele tocou e em quantas *edições* (anos) distintas ele apareceu.

**Por quê:** é a pergunta mais direta que dá pra fazer da base — "quem mais voltou?" — e é a matéria-prima de quase toda análise depois dela (o ranking do Capítulo 4, o score de projeção, a matriz de recorrência).

**Como:** um `Object` conta ocorrências por nome ao iterar sobre `FESTIVAL_DATA.nights`. "Aparições" conta noites (um artista que toca 2 vezes na mesma edição conta 2 ali, mas 1 em "edições").

**Limite:** dois artistas com grafias diferentes do mesmo nome inflam a contagem se não forem unificados. Fiz essa limpeza manualmente (ex.: "Bruno & Marrone" vs. "Bruno e Marrone"), mas é um processo humano, não uma verificação automática.

---

## 2. Retenção ano a ano (a parte mais importante para a projeção)

**O que é:** de todo artista que tocou na edição do ano Y, qual fração também tocou na edição do ano seguinte? E isso muda dependendo de o artista já vir de uma sequência, estar voltando de uma pausa, ou ser estreante?

**Por quê:** a suposição inicial do projeto era simples demais (rodízio cego entre os candidatos do ranking). Quando testei contra os dados, descobri que **quem volta depende do histórico recente do artista**, não é uniforme. Essa análise é o que corrigiu a projeção — inclusive o caso do Panda, que tinha aparecido como candidato pra 2027 por ter sido o único nome do papel "Emergente" naquela noite, mesmo com uma única aparição.

**Como (`computeRetention`):**
- Cada artista de uma edição Y é classificado em 1 de 3 grupos, usando **só informação disponível até o ano Y** (nunca "olho pro futuro" pra classificar o presente):
  - **sequência** — também tocou na edição imediatamente anterior;
  - **pausa** — já tinha tocado antes, mas não na edição anterior;
  - **estreante** — primeira aparição na base naquele ano.
- Para cada grupo, calculo a fração que reaparece na edição Y+1.
- Faço o mesmo teste pra um "ciclo de 2 anos": de quem tocou em Y e pulou Y+1, quantos voltam em Y+2?

**Resultado (2011–2025, calibra a projeção sem entrar na fórmula do score):**

| Grupo | Taxa de retorno no ano seguinte |
|---|---|
| Sequência | 64% |
| Pausa | 43% |
| Estreante | 33% |
| **Geral** | **49%** |
| Voltar 2 anos depois de pular 1 | 14% (10 de 69 casos) |

**Limite:** com 69–73 casos por grupo, a margem de erro é grande — uma edição atípica muda a porcentagem visivelmente. Trato essas taxas como leitura de tendência, não como probabilidade fina.

---

## 3. Score de projeção (o motor da Seção 6)

**O que é:** um ranking, por regras — não é machine learning, não há treino nem otimização de parâmetro.

```
score = 3 × (aparições do artista NAQUELA noite específica)
      + 1 × (aparições totais do artista no festival)
      + bônus de recência
```

Bônus de recência: **2,5** se a última aparição foi 2023 ou depois; **1,2** se foi 2019–2022; **0,3** antes disso.

**Por quê esses pesos:** peso 3 pro slot específico porque "já tocou nessa mesma noite (sexta do 1º fim de semana, por exemplo) antes" é o sinal mais forte de afinidade com aquele horário. Peso 1 pra frequência geral captura relevância mesmo sem repetição no mesmo slot exato. O bônus de recência existe pra não projetar um artista que sumiu há 10 anos só porque ele tocou muitas vezes no passado. **Os pesos foram escolhidos por julgamento, não ajustados estatisticamente** — não há dados suficientes pra "treinar" isso com confiança, e isso está declarado explicitamente no painel.

**Papéis:** os candidatos de cada noite são divididos em 4 papéis (headliner, segundo nome, crossover, emergente) usando uma classificação manual de ~55 artistas (`ARTIST_PROFILE`). Quem não foi classificado entra como "segundo nome" por padrão.

**Regra de elegibilidade (adicionada depois do caso do Panda):** só vira **nome projetado** quem já tocou em **2 ou mais edições distintas**. Um artista de aparição única, mesmo com o score mais alto do papel, não é elegível — porque a análise de retenção (Seção 2) mostra que estreante só volta em 33% dos casos. Quando não sobra ninguém elegível numa noite/papel, o card mostra uma **vaga aberta**, com a taxa histórica de estreante daquele slot, em vez de forçar um nome.

**Rodízio 2028/2029:** para cada papel, ordeno as 4 noites pelo candidato mais forte primeiro (evita que o mesmo nome "vença" duas noites ao mesmo tempo) e cada ano projetado avança pro próximo candidato elegível da lista. **Isso é uma regra de desenho para diversificar a projeção — não é uma descoberta estatística**, e contraria o dado da Seção 2 (quem vem de sequência tem 64% de chance de voltar, não deveria "perder a vez" automaticamente). O painel marca 2028 e 2029 como cenários mais frágeis que 2027 por causa disso.

**O que a barra mostra:** um score *relativo* dentro do próprio papel e noite (1º colocado = 100). **Não é probabilidade de contratação** — o texto do painel repete isso de propósito, porque contratação depende de agenda, cachê e estratégia comercial, que não estão nos dados.

---

## 4. Regras de associação (co-booking)

**O que é:** para cada par de artistas, quantas noites diferentes eles dividiram o mesmo palco.

**Por quê:** é a versão mais simples de "análise de cesta de compras" (a mesma lógica por trás de "quem comprou X também comprou Y") aplicada a lineups. Acusa parcerias de fato (ex.: Bruno & Marrone + Zezé Di Camargo & Luciano, 5 vezes) sem nenhuma suposição prévia de quem "combina" com quem.

**Como:** para cada noite, gero todos os pares possíveis de artistas daquela noite e conto ocorrências num mapa `par → contagem`. Só mostro pares com 2 ou mais ocorrências.

**A rede de coedição** (Capítulo 5) é a mesma lógica, mas por **edição** em vez de por noite, entre os 12 artistas mais presentes desde 2008 — e desenhada como grafo, com a espessura da linha proporcional ao número de edições em comum.

---

## 5. Clustering — k-means feito do zero (k = 4)

**O que é:** agrupar os artistas automaticamente por dois números — frequência total e ano da última aparição — **sem usar a classificação manual de papéis**. É uma forma de checar se os "arquétipos" que a classificação manual sugere (pilar atual, lenda do passado, ascensão recente, pontual) realmente aparecem nos dados por conta própria.

**Por quê k-means:** é o algoritmo de agrupamento mais simples que existe, fácil de implementar sem biblioteca e fácil de auditar linha por linha. Com só 2 variáveis e ~60 pontos (artistas com 2+ aparições), não precisa de nada mais sofisticado.

**Como (`computeArtistClusters`):**
1. Normalizo cada artista pra dois eixos 0–1: recência (ano da última aparição, min-max) e frequência (aparições totais, em **log**, porque a distribuição é bem assimétrica — poucos artistas com muitas aparições, muitos com poucas).
2. **Inicialização determinística, sem `Math.random`:** ordeno os pontos e pego 4 igualmente espaçados como centróides iniciais. Isso importa porque o k-means clássico é sensível à inicialização aleatória — rodar duas vezes poderia dar resultados diferentes, e o projeto tem uma regra de não usar aleatoriedade em lugar nenhum.
3. Itero (atribuir ponto ao centróide mais próximo → recalcular centróide como média do grupo) até estabilizar ou 25 iterações.
4. Rotulo cada cluster pelo quadrante do centróide final (recência alta/baixa × frequência alta/baixa) — esses **rótulos** ("Pilares atuais", "Lendas históricas" etc.) são interpretação minha, não algo que o algoritmo "sabe".

**Limite:** com só 2 dimensões, é um agrupamento simples — não captura estilo musical, papel na noite ou dia da semana. E k=4 foi escolhido pra combinar com os 4 papéis existentes, não por um critério estatístico de número ideal de clusters (cotovelo, silhueta etc.).

---

## 6. Série temporal — estilos musicais por era

**O que é:** conto quantas atrações de cada estilo (sertanejo, eletrônico, piseiro/forró etc.) aconteceram em 4 janelas de ~10 anos (1989–1999, 2000–2009, 2010–2019, 2020–2026), e mostro a evolução dos 5 estilos mais frequentes.

**Por quê janelas de década em vez de ano a ano:** com 134 noites espalhadas por 37 anos, um gráfico ano a ano teria barras minúsculas e ruidosas demais pra enxergar tendência. Agregar em eras suaviza o ruído e deixa a mudança de composição (sertanejo perdendo espaço pro eletrônico e pro piseiro, por exemplo) visível.

**Limite:** o estilo de cada artista é uma classificação manual minha (`ARTIST_GENRE`), não uma fonte por artista. Um artista que muda de estilo ao longo da carreira é classificado só de um jeito.

---

## 7. Estreantes por edição

**O que é:** para cada artista, marco o primeiro ano em que ele aparece na base como sua "estreia", e conto quantas estreias aconteceram em cada edição.

**Por quê:** foi a pergunta que levantou o problema do Panda — "todo ano tem estreante?" A resposta, nos dados, é sim: 16 de 16 edições desde 2010 tiveram pelo menos 1 estreante (média de 4). Isso virou parte da explicação da "vaga aberta" na projeção.

**Limite importante:** como a base de 1989–2007 tem lacunas, uma "estreia" antiga pode na verdade ser um retorno de alguém que já tinha tocado num ano sem registro. A partir de 2010 (quando a cobertura fica mais completa) essa leitura é bem mais confiável — por isso a maior parte das análises de estreante do painel é filtrada para 2010 em diante.

---

## 8. Mapa de qualidade dos dados

**O que é:** não é uma análise estatística, é uma auditoria da própria base — cada edição é classificada como "linha do tempo completa" (line-up por data), "datas parciais", "só lista de artistas do ano" ou "cancelada".

**Por quê:** um painel de dados devia mostrar onde ele é forte e onde é fraco, principalmente quando parte de uma projeção depende dessa base. Isso é o que fundamenta a frase "trate 2028–2029 com mais cautela" — a incerteza cresce em cascata com dado mais fraco.

---

## Princípios que valem para tudo acima

1. **Nenhum score novo foi inventado depois do desenho original** — a fórmula da Seção 3 nunca mudou; o que mudou foi a *elegibilidade* de quem pode ser nome projetado e o *rótulo* de quem entra em cada seção.
2. **Nada de aleatoriedade** — nem no k-means, nem no rodízio da projeção. Rodar o painel duas vezes dá exatamente o mesmo resultado.
3. **Classificação manual é sinalizada, não escondida** — papel do artista (headliner/segundo/crossover/emergente) e estilo musical são interpretação minha; isso está declarado toda vez que uma análise depende disso.
4. **Inconsistência não é corrigida em silêncio** — quando duas fontes discordam (ex.: o período exato de 2001–2002, ou o recorde de público), o painel mostra os dois números e explica o conflito, em vez de escolher um e seguir em frente.
5. **Amostra pequena, sempre.** 134 registros e 152 artistas não dão margem pra afirmações fortes. Todo percentual no painel é acompanhado do "n" (quantos casos) por perto — é assim que dá pra julgar se uma taxa é robusta ou é 2 casos em 6.

---

## Onde cada coisa está no código

| Análise | Função | Arquivo |
|---|---|---|
| Recorrência, presença | `buildPresence`, `PRESENCE` | `parts/analysis.txt` |
| Retenção (sequência/pausa/estreante) | `computeRetention` | `parts/analysis.txt` |
| Score e projeção | `buildCandidatePools`, `poolForRole`, `buildProjection` | `parts/engine.txt` |
| Elegibilidade (2+ edições) | `isEligibleName` | `parts/engine.txt` |
| Co-booking / pares | `computeCoBookingPairs` | `parts/analysis.txt` |
| Rede de coedição | `computeCoEditionNetwork` | `parts/analysis.txt` |
| Clustering (k-means) | `kmeans`, `computeArtistClusters` | `parts/analysis.txt` |
| Dossiê por artista | `computeArtistDossier` | `parts/analysis.txt` |
| Estilos por era | `computeGenreByEra` | `parts/engine.txt` |
| Estreantes | `computeDebutantsByYear` | `parts/analysis.txt` |
| Qualidade dos dados | `computeDataQuality` | `parts/analysis.txt` |

A metodologia completa, em linguagem menos técnica, também está dentro do próprio painel (Capítulo 8 — Metodologia).
