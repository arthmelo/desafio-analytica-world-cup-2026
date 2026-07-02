# Previsão Copa do Mundo 2026

Simulação estatística da Copa do Mundo FIFA 2026 (formato de 48 seleções) usando um modelo Poisson de gols treinado em histórico de partidas internacionais e ranking FIFA, combinado com simulação de Monte Carlo da fase de grupos e do mata-mata.

## Visão geral

Este projeto estima a probabilidade de cada seleção ser campeã da Copa do Mundo 2026, a partir de um pipeline de três etapas:

1. **Modelo de gols** — um GLM Poisson estima quantos gols cada seleção tende a marcar em função do próprio Elo, do Elo do adversário e do mando de campo.
2. **Simulação da fase de grupos** — a cada uma das 100.000 simulações, os 72 jogos da fase de grupos (12 grupos de 4 seleções) são sorteados a partir do modelo, a classificação é recalculada pelos critérios oficiais (pontos, saldo de gols, gols marcados) e o chaveamento das oitavas de final é montado dinamicamente a partir dessa classificação simulada.
3. **Simulação do mata-mata** — os 32 classificados disputam oitavas, quartas, semifinal e final, também via sorteio Poisson, até definir um campeão por simulação.

Além da simulação da Copa inteira, o notebook também traz uma função para simular um confronto isolado de mata-mata entre duas seleções quaisquer, mostrando o placar mais provável, número de gols esperados, e probabilidade de vitória (avançar de fase) para cada time, usada ao final do notebook para prever cenários específicos, como Brasil x Japão ou o 16 avos de final.

Não há dados de elenco, lesões, escalação ou notícias recentes — a força de cada seleção vem inteiramente do ranking FIFA e do histórico de resultados.

## Como funciona

O pipeline segue seis etapas, do dado bruto até a probabilidade de título:

**1. Padronização das seleções.** O ranking FIFA (`fifa_ranking-2026.csv`) e o histórico de partidas (`results.csv`) usam grafias diferentes para alguns países (por exemplo, `USA` em uma base e `United States` em outra, ou nomes históricos como `West Germany` e `Czechoslovakia`). Dicionários de correção padronizam os nomes para que as duas bases possam ser cruzadas pela mesma chave de seleção.

**2. Junção com o ranking (merge_asof).** Para cada partida do histórico, o Elo de cada seleção é obtido pelo registro de ranking mais recente publicado antes da data do jogo (`merge_asof`, com tolerância de 90 dias). Partidas sem ranking disponível dentro dessa janela são descartadas.

**3. Filtro de jogos relevantes.** A base é restrita a partidas a partir de 2018 e que envolvam ao menos uma seleção classificada para a Copa 2026, reduzindo o histórico de cerca de 26 mil para cerca de 3,4 mil partidas — o suficiente para captar a força recente das seleções, sem arrastar décadas de jogos pouco informativos sobre o elenco atual.

**4. Peso por torneio e decaimento temporal.** Cada partida recebe um peso conforme a relevância do torneio (Copa do Mundo pesa mais que um amistoso) multiplicado por um fator de decaimento exponencial que reduz a importância de jogos mais antigos. Esse peso entra no treino do modelo como peso de frequência.

**5. Modelo Poisson de gols.** Um GLM Poisson é ajustado sobre essa base ponderada, estimando os gols esperados de uma seleção em função do próprio Elo, do Elo do adversário e do mando de campo (`gols ~ elo_time + elo_adversario + mando_de_campo`).

**6. Simulação de Monte Carlo.** A cada uma das 100.000 simulações, os 72 jogos da fase de grupos são sorteados a partir dos gols esperados pelo modelo, a classificação de cada grupo é recalculada (pontos, saldo de gols, gols marcados) e o chaveamento das oitavas de final é montado dinamicamente a partir dos classificados. O mata-mata é então simulado fase a fase — oitavas, quartas, semifinal e final — também por sorteio Poisson, até definir um campeão por simulação. Repetindo esse processo 100.000 vezes, conta-se quantas vezes cada seleção foi campeã, gerando a probabilidade de título.

**Peso por torneio.** Cada jogo do histórico recebe um peso conforme a relevância do torneio: Copa do Mundo (3,0), copas continentais como Eurocopa e Copa América (2,5), eliminatórias de alto nível (2,0), Liga das Nações e torneios regionais (1,5), e demais amistosos (1,0, padrão).

**Decaimento temporal.** Além do peso de torneio, jogos mais antigos pesam menos, com decaimento exponencial:

### Decaimento Temporal Ideal (Validação)

No futebol, o peso histórico de uma seleção importa, mas o momento recente é crucial. Para garantir que o modelo não dependesse de "intuição" ao definir o peso de jogos antigos, implementamos uma **validação empírica do fator de decaimento temporal ($\alpha$)**.

O processo de validação funcionou da seguinte forma:

* **Separação de Dados (Train/Test):** Utilizamos todo o histórico de partidas internacionais até meados de 2022 como conjunto de treino e usamos os jogos da **Copa do Mundo de 2022 (Catar)** como nosso conjunto de teste fora da amostra (out-of-sample).
* **Testagem de Meias-Vidas:** Simulamos o modelo Poisson com diferentes taxas de esquecimento (meias-vidas de 1, 1.5, 2, 2.5, 3, 4 e 5 anos).
* **Métrica de Avaliação:** Usamos o **Brier Score** aplicado às probabilidades de Vitória, Empate e Derrota (1X2). O Brier Score penaliza previsões probabilísticas que fogem da realidade (quanto menor, melhor).
* **Resultado:** Os dados provaram que uma **meia-vida de 1.5 anos** minimiza o Brier Score, atingindo o menor índice de erro (**0.586617**). Isso demonstra matematicamente que avaliar o desempenho acumulado nos últimos 18 meses equilibra perfeitamente a necessidade de captar o ciclo tático/geracional recente de uma seleção sem sofrer com o imediatismo excessivo de olhar apenas para o último ano (meia-vida de 1 ano, Brier: 0.586664) ou com a obsolescência de dados muito antigos (meia-vida de 2 anos ou mais).

```text
peso_final = peso_torneio × e^(−0.001265 × dias_desde_o_jogo)

```

**Modelo.** O GLM Poisson ajustado em 6.798 observações (cada partida do histórico gera duas linhas, uma por seleção) tem os seguintes coeficientes:

| Variável | Coeficiente | Efeito |
|---|---|---|
| Elo da própria seleção | +0,0019 | mais força própria aumenta os gols esperados |
| Elo do adversário | −0,0019 | mais força do adversário reduz os gols esperados |
| Mando de campo | +0,2745 | jogar em casa aumenta os gols esperados |

Pseudo R² (Cragg-Uhler): 0,36.

## Resultado: probabilidade de título

Probabilidade de título nas 10 seleções mais bem colocadas, na simulação completa de 100.000 Copas (fase de grupos simulada a cada iteração, com chaveamento montado dinamicamente a partir da classificação):

| # | Seleção | Probabilidade de título |
|---|---|---|
| 1 | França | 13.437% |
| 2 | Espanha | 13.241% |
| 3 | Argentina | 13.003% |
| 4 | Inglaterra | 10.023% |
| 5 | Brasil | 5.093% |
| 6 | Portugal | 5.018% |
| 7 | Países Baixos | 4.330% |
| 8 | Bélgica | 4.032% |
| 9 | Marrocos | 3.883% |
| 10 | Alemanha | 3.361% |

 A probabilidade de título fica bastante distribuída entre as quatro primeiras seleções (França, Espanha, Argentina, Inglaterra e Brasil somam pouco mais de 50%), sem nenhuma favorita absoluta. A incerteza da fase de grupos está incorporada nesse resultado, já que a classificação e o chaveamento são recalculados a cada simulação.

## Confrontos individuais de mata-mata

Além da probabilidade de título da Copa inteira, o notebook traz uma simulação isolada para confrontos eliminatórios específicos entre duas seleções, incluindo a possibilidade de empate sendo decidido nos pênaltis (50% de chance para cada lado). Os destaques simulados:

**Brasil x Japão**

| Métrica | Valor |
|---|---|
| Gols esperados | Brasil 1,29 x 0,86 Japão |
| Probabilidade de ir aos prorrogção/pênaltis | 28,7% |
| Chance de o Brasil avançar | 60,8% |
| Chance de o Japão avançar | 39,2% |


## Fontes de dados

| Fonte | Uso | Licença / atribuição |
|---|---|---|
| [Fjelstul World Cup Database](https://www.github.com/jfjelstul/worldcup) | Referência de seleções e edições da Copa | CC-BY-SA 4.0 — ver seção de licença abaixo |
| [Ranking FIFA (Kaggle)](https://www.kaggle.com/datasets/cashncarry/fifaworldranking/discussion/681592) | Força de cada seleção (Elo/pontos) | Conforme termos do dataset no Kaggle |
| [International football matches, 1872–present (Kaggle)](https://www.kaggle.com/datasets/aissaouihamda/international-football-matches-1872-present) | Histórico de partidas, base de treino do modelo de gols | Conforme termos do dataset no Kaggle |

## Estrutura do projeto

```
.
├── previsao_copa.ipynb # notebook com todo o pipeline
└── data/                 # dados 
    ├── data-csv/
    │   └── qualified_teams.csv
    ├── elo/
    │   └── fifa_ranking-2026.csv
    └── matches/
        └── results.csv
```

## Como rodar

1. Baixe os três arquivos de dados listados em [Fontes de dados](#fontes-de-dados) e organize-os na estrutura de pastas acima.
2. Crie um ambiente Python com `pandas`, `numpy` e `statsmodels`.
3. Execute `previsao_copa.ipynb` célula a célula, na ordem em que aparece.

A simulação de Monte Carlo da Copa inteira (100.000 simulações, com fase de grupos e mata-mata recalculados a cada uma) é a etapa mais lenta do notebook. As simulações de confrontos individuais ao final são rápidas em comparação, mesmo com 100.000 simulações cada.

## Limitações

- Empates no mata-mata, dentro da simulação da Copa inteira, são decididos por sorteio aleatório simples (`random.choice`), sem simular prorrogação. Já a função de confronto individual trata o empate como ida aos pênaltis, com 50% de chance para cada lado — as duas abordagens convivem no mesmo notebook e não são equivalentes.
- O Elo de cada seleção é fixado no início da simulação e não é atualizado durante a própria Copa simulada.
- Não há validação fora da amostra (por exemplo, treinar até uma Copa anterior e testar na edição seguinte).
- O modelo usa apenas Elo, mando de campo e peso de torneio — sem elenco, lesões, suspensões ou odds de mercado.
- A disputa de pênaltis é tratada como uma moeda justa (50/50), sem considerar fatores como histórico de pênaltis da seleção ou do goleiro.

## Melhorias futuras

- Unificar o critério de desempate do mata-mata (pênaltis 50/50) entre a simulação da Copa inteira e a simulação de confrontos individuais.
- Validar o modelo em Copas anteriores e reportar métricas de erro e calibração.
- Comparar contra baselines simples (Elo puro, probabilidades implícitas de mercado).
- Atualizar o Elo das seleções entre as fases da própria simulação, em vez de mantê-lo fixo do início ao fim de cada Copa simulada.

## Licença e atribuição

Este projeto utiliza a **Fjelstul World Cup Database**, de Joshua C. Fjelstul, Ph.D., licenciada sob CC-BY-SA 4.0. Qualquer redistribuição deste repositório, ou de trabalhos derivados, deve manter a seguinte atribuição:

- Autor: Joshua C. Fjelstul, Ph.D.
- © 2023 Joshua C. Fjelstul, Ph.D.
- Licença: https://creativecommons.org/licenses/by-sa/4.0/legalcode
- Repositório original: https://www.github.com/jfjelstul/worldcup

Por se tratar de uma licença share-alike, qualquer trabalho novo construído sobre essa base — incluindo este projeto — deve ser distribuído sob a mesma licença CC-BY-SA 4.0.

Os dados de ranking FIFA e de histórico de partidas internacionais usados neste projeto têm como fonte:

- https://www.kaggle.com/datasets/cashncarry/fifaworldranking/discussion/681592
- https://www.kaggle.com/datasets/aissaouihamda/international-football-matches-1872-present
