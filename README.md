# Projeto: Fine-Tuning para Triagem de Alertas em SOC (Security Operations Center)

### Contexto

Num SOC real, cada alerta passa por estas etapas, em ordem, até chegar à resposta:

| Etapa | O que acontece | É fine-tuning? |
|---|---|---|
| 1. Detecção e coleta | Ferramentas de segurança detectam atividade suspeita e geram eventos. O SIEM recebe os eventos e as regras de correlação disparam o alerta. | **Não.** A detecção usa regras de correlação e assinaturas, que já são previsíveis e auditáveis. |
| 2. Normalização | O SIEM converte o alerta para um formato padrão (host, usuário, processo, IP), não importa a ferramenta de origem. | **Não.** É uma conversão de formato com regras fixas (*parsers*), sem julgamento envolvido. |
| 3. Enriquecimento | O alerta ganha contexto consultado automaticamente:<br>• **CMDB:** o que é o ativo e quão crítico ele é<br>• **IAM/AD:** quem é o usuário e quais privilégios tem<br>• **Threat Intel:** reputação de IPs, domínios e hashes<br>• **Histórico do SIEM:** quantas vezes o alerta já disparou | **Não.** São fatos que mudam com frequência (ativos, usuários, reputação). |
| 4. Triagem | Decidir veredito, severidade e justificativa seguindo a política de severidade da organização. **Feito manualmente pelo analista L1.** | **Sim, se um prompt não for suficiente** A política é própria da organização e precisa ser aplicada sempre do mesmo jeito. |
| 5. Resposta | O playbook é executado pelo SOAR, por exemplo isolar o host, bloquear um IOC ou abrir um ticket. | **Não.** As ações de contenção mexem no ambiente e precisam ser previsíveis e auditáveis. |

### Problema

Um SOC recebe de centenas a milhares de alertas por dia. Mesmo com coleta, normalização e enriquecimento automatizados, a etapa de **triagem** continua manual: o L1 precisa ler alerta por alerta e aplicar a política de severidade da organização para decidir se é falso positivo, qual a severidade real e por quê. Essa análise repetitiva consome a maior parte do tempo do L1. Além disso, o resultado varia entre analistas e turnos, o que gera triagens inconsistentes e atraso na resposta a incidentes reais.

### Solução 

Ajustar um modelo para realizar a triagem dos alertas.

A tabela abaixo apresenta cada fase do processo de fine-tuning de forma sequencial. Cada fase é desenvolvida a partir de um notebook que deve ser aberto a partir de um Colab com GPU T4. 
Para você executar o projeto, basta abrir os notebooks de forma sequencial no Google Colab.

| Fase | Objetivo | Notebook |
|---|---|---|
| 1. Planejamento do objetivo e do modelo | Definir a tarefa do modelo, definir as metas de sucesso, executar o teste de necessidade e selecionar o modelo base | [fase_01_planejamento](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_01_planejamento.ipynb) |
| 2. Preparação dos dados | Auditar o dataset fornecido, limpar, padronizar e dividir em treino, validação e teste | [fase_02_prep_dados](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_02_prep_dados.ipynb) |
| 3. Tokenização | Converter os exemplos em tokens com o tokenizador do próprio modelo, no formato do seu chat template | [fase_03_tokenizacao](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_03_tokenizacao.ipynb) |
| 4. Estratégia de ajustes | Definir como o modelo será ajustado e com quais hiperparâmetros | [fase_04_ajustes](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_04_ajustes.ipynb) |
| 5. Execução do treinamento | O modelo é treinado e ajusta os pesos | [fase_05_treinamento](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_05_treinamento.ipynb) |
| 6. Avaliação | Medir no conjunto de **teste** e comparar com o baseline | [fase_06_avaliacao](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_06_avaliacao.ipynb) |
| 7. Demo do assistente | Carregar o modelo base e o adaptador através de uma demo no Gradio, onde se cola um alerta e se recebe a triagem em JSON | [fase_07_demo_gradio](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_07_demo_gradio.ipynb) |

### Ambiente e ferramentas

| Ferramenta | Função | 
|---|---|
| **[Hugging Face Hub](https://huggingface.co/)** | Modelo base e dataset do projeto |
| **[Gradio](https://www.gradio.app/)** | Interface web da demo do assistente de triagem |
| **[Google Colab](https://colab.research.google.com/)** | Ambiente de execução do código com GPU T4 |
| **[GitHub](https://github.com/poliana-moraes/fine-tuning-project/)** | Publicação da versão oficial do material do projeto | 

### Versões das bibliotecas

Os notebooks usam a **versão 2026.10 do runtime do Colab**, com Python 3.13.18. 
O PyTorch e o CUDA são os que já vêm instalados nesse runtime. 
As demais bibliotecas são instaladas com versão fixa em cada notebook:

| Biblioteca | Versão | Usada nas fases |
|---|---|---|
| `transformers` | 5.22.0 | 1, 3, 4, 5, 6, 7 |
| `datasets` | 4.8.5 | 1 a 7 |
| `huggingface_hub` | 1.33.0 | 1 a 7 |
| `accelerate` | 1.15.0 | 1, 3, 4, 5, 6, 7 |
| `trl` | 1.17.1 | 3, 4, 5 |
| `peft` | 0.21.2 | 4, 5, 6, 7 |
| `scikit-learn` | 1.6.1 | 2, 6 |
| `tensorboard` | 2.21.0 | 5 |
| `gradio` | 6.31.0 | 7 |
