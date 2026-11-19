# Fire Intelligence · IA & Machine Learning

Componente inteligente da CitizenTech **Fire Intelligence** (Global Solution FIAP 2026.2).

## Pergunta do modelo

> Quantos focos de calor o satélite de referência do INPE deve detectar em cada célula da área de estudo no próximo mês, se o padrão histórico se repetir?

- **Tipo:** regressão de contagem (Poisson).
- **Unidade:** célula de 0,1° × 0,1° (~11 km) × mês, na Chapada dos Veadeiros e entorno (GO).
- **Alvo:** número de focos do satélite de referência na célula, no mês.
- **Uso na solução:** a tela de diagnóstico mostra a soma das células que cobrem o polígono escolhido, rotulada como **estimativa experimental**, com a margem de erro histórica daquela área.
- **O que o modelo NÃO faz:** não classifica risco, não define limiares e não indica áreas aptas ou seguras para queima.

## Dataset e origem

| Item | Valor |
|---|---|
| Fonte | INPE, Programa Queimadas, servidor de dados abertos: https://dataserver-coids.inpe.br/queimadas/queimadas/focos/csv/ |
| Anos fechados | Arquivos anuais do satélite de referência (pasta `anual/EstadosBr_sat_ref/GO/` ou `anual/Brasil_sat_ref/`) |
| Ano corrente | Arquivos mensais (`mensal/Brasil/`, todos os satélites). O notebook filtra o de referência |
| Satélite usado | Apenas o satélite de referência (AQUA_M-T) |
| Recorte | lat −14,80 a −13,50 / lon −48,30 a −46,90 (Chapada dos Veadeiros e entorno) |
| Arquivo no repositório | `dados/focos_chapada_2012_<ano>.csv` + `dados/ORIGEM.txt` (data do download e arquivos usados) |

### Como os dados chegam

Na primeira vez que o notebook roda com a pasta `dados/` vazia, a célula de download busca os arquivos anuais do INPE, recorta a área e grava um único CSV pequeno em `dados/`. Façam commit desse CSV e do `ORIGEM.txt`.

Quem rodar o notebook depois (inclusive o professor) não baixa nada: o CSV já está no repositório.

O BDQueimadas (portal com mapa) limita cada exportação a 366 dias, por isso ele não é usado aqui.

## Como rodar

```bash
pip install pandas numpy scikit-learn matplotlib shapely joblib jupyter
cd ml
jupyter notebook fire_intelligence_ml.ipynb   # ou abrir no Google Colab
```

Rode todas as células em ordem. O notebook gera em `saida/`:

| Arquivo | Conteúdo |
|---|---|
| `modelo_focos.joblib` | Modelo final treinado com todo o histórico |
| `previsoes_proximo_mes.csv` | Estimativa por célula para o mês seguinte ao último mês completo |
| `teste_previsoes.csv` | Previsões do período de teste, usadas para calcular o erro de cada área |
| `metadados.json` | Fonte, período, satélite, métricas e aviso |

**Contingência:** se o download do INPE falhar, `FI_MODO_SIMULADO=1 jupyter notebook` roda com focos simulados no mesmo formato, só para testar o código. A entrega precisa rodar com os dados reais.

## Decisões

| Decisão | Motivo |
|---|---|
| Só o satélite de referência | Cada satélite detecta quantidades diferentes. Misturar quebra a série |
| Grade de ~11 km | A posição de cada foco tem imprecisão de cerca de 1 km e os dados são esparsos |
| Painel com zeros | Meses sem foco também são informação. Mais de 90% das linhas são zero |
| Regressão de Poisson | Alvo é contagem não negativa, com muitos zeros |
| Divisão temporal (treino até 2022, validação 2023, teste 2024+) | Divisão aleatória deixaria o modelo ver o futuro |
| Comparação com 3 baselines | O modelo só se justifica se errar menos que regras simples |
| Deviance de Poisson como métrica principal, MAE região/mês para comunicar | MAE por célula engana com tantos zeros |
| Variáveis só com informação anterior ao mês previsto | Evitar vazamento de dados |

## Resultados

_Preencher depois de rodar com os dados reais (tabela da seção 4 do notebook)._

| Modelo | Deviance Poisson | MAE região/mês |
|---|---|---|
| Baseline sazonal | | |
| Baseline média histórica | | |
| Modelo escolhido | | |

## Integração na solução

A seção 5 do notebook mostra onde a previsão entra no Fire Intelligence:

1. A função `estimar_area` recebe a área escolhida no mapa (polígono GeoJSON) e devolve o bloco **"Estimativa experimental"** da tela de diagnóstico.
2. O mapa da estimativa por célula mostra a área de estudo com o polígono por cima.

Exemplo de resultado para um polígono em torno de Alto Paraíso de Goiás:

```json
{
  "situacao": "ok",
  "tipo": "estimativa_experimental",
  "indicador": "Focos de calor estimados para o próximo mês",
  "mes_referencia": "2026-11",
  "valor": 5.1,
  "unidade": "focos (satélite de referência)",
  "margem_erro_historica": 1.4,
  "aviso": "Estimativa experimental ... não indica áreas aptas ou seguras para queima ..."
}
```

Na tela de diagnóstico, esse resultado aparece em um bloco separado dos indicadores, com valor, margem de erro, método, fonte e aviso.

_Opcional:_ se o backend do piloto estiver rodando, a mesma função vira uma rota (`GET /areas/{id}/estimativa-focos`) sem mudanças.

## Limitações

- Foco de calor é detecção por satélite, não incêndio nem área queimada.
- A estimativa vale para o satélite de referência, não para o total real de focos.
- Se o satélite de referência mudou no período, a série precisa ser tratada antes do treino.
- Usa só o histórico de focos. Anos atípicos (muito secos ou úmidos) geram erro maior.

## Roteiro do vídeo (até 2 min)

| Tempo | Conteúdo |
|---|---|
| 0:00–0:20 | Problema e pergunta do modelo. Para quem é útil |
| 0:20–0:45 | Dados do INPE, filtro do satélite de referência, gráfico de sazonalidade |
| 0:45–1:10 | Painel célula × mês, variáveis e divisão temporal |
| 1:10–1:35 | Tabela de métricas contra os baselines e gráfico observado × estimado |
| 1:35–2:00 | Estimativa para um polígono (saída da `estimar_area` e mapa) e onde aparece na tela de diagnóstico, com o aviso |
