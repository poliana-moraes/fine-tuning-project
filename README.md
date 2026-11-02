# Projeto — Fine-Tuning para Triagem de Alertas em SOC (Security Operations Center)

### Contexto

Num SOC real, cada alerta passa por estas etapas, em ordem, até chegar à resposta:

| Etapa | O que acontece | É fine-tuning? |
|---|---|---|
| 1. Detecção e coleta | Ferramentas de segurança detectam atividade suspeita e geram eventos. O SIEM recebe os eventos e as regras de correlação disparam o alerta. | **Não.** A detecção usa regras de correlação e assinaturas, que já são previsíveis e auditáveis. |
| 2. Normalização | O SIEM converte o alerta para um formato padrão (host, usuário, processo, IP), não importa a ferramenta de origem. | **Não.** É uma conversão de formato com regras fixas (*parsers*), sem julgamento envolvido. |
| 3. Enriquecimento | O alerta ganha contexto consultado automaticamente:<br>• **CMDB:** o que é o ativo e quão crítico ele é<br>• **IAM/AD:** quem é o usuário e quais privilégios tem<br>• **Threat Intel:** reputação de IPs, domínios e hashes<br>• **Histórico do SIEM:** quantas vezes o alerta já disparou | **Não.** São fatos que mudam o tempo todo (ativos, usuários, reputação). |
| 4. Triagem | Decidir veredito, severidade e justificativa seguindo a política de severidade da organização. **Hoje é feita manualmente pelo analista L1.** | **Sim, se um prompt não for suficiente** A política é própria da organização e precisa ser aplicada sempre do mesmo jeito. |
| 5. Resposta | O playbook é executado pelo SOAR, por exemplo isolar o host, bloquear um IOC ou abrir um ticket. | **Não.** As ações de contenção mexem no ambiente e precisam ser previsíveis e auditáveis. |

### Problema

Um SOC recebe de centenas a milhares de alertas por dia. Mesmo com coleta, normalização e enriquecimento automatizados, a etapa de **triagem** continua manual: o L1 precisa ler alerta por alerta e aplicar a política de severidade da organização para decidir se é falso positivo, qual a severidade real e por quê. Essa análise repetitiva consome a maior parte do tempo do L1. Além disso, o resultado varia entre analistas e turnos, o que gera triagens inconsistentes e atraso na resposta a incidentes reais.

### Quando o fine-tuning se justifica para triagem de alertas:

- **Privacidade:** os alertas contêm hosts, usuários e IPs internos e não podem sair do ambiente. Um modelo pequeno ajustado roda on-premises.
- **Volume e custo:** com milhares de alertas por dia, um modelo pequeno ajustado é muito mais barato e rápido por alerta do que um modelo grande via API.
- **Consistência:** o modelo aprende a aplicar sempre a **política de severidade da organização** e a devolver sempre o mesmo formato JSON.


### Ambiente e ferramentas

| Ferramenta | Função | 
|---|---|
| **[Google Colab](https://colab.research.google.com/)** | Ambiente de execução do código com GPU T4 |
| **[Hugging Face Hub](https://huggingface.co/)** | Modelo base, dataset do projeto, publicação do adaptador e do model card |
| **[GitHub](https://github.com/poliana-moraes/fine-tuning-project/)** | Publicação da versão oficial do material do projeto | 

### Entregas por fase do processo de fine-tuning

A tabela abaixo apresenta cada fase do processo de fine-tuning de forma sequencial. Cada fase é desenvolvida a partir de um notebook deve ser aberto a partir de um Colab com GPU T4. Para você executar o projeto, basta abrir os notebooks de forma sequencial no Google Colab.

| Fase | Entrega | Notebook |
|---|---|---|
| 1. Planejamento do objetivo e do modelo | Ficha de objetivo e modelo + resultado do teste de necessidade | [fase_01_planejamento](https://github.com/poliana-moraes/fine-tuning-project/blob/main/notebooks/fase_01_planejamento.ipynb) |
| 2. Preparação dos dados | Relatório de auditoria + splits treino/validação/teste | [fase_02_prep_dados](https://github.com/poliana-moraes/fine-tuning-project/) |
| 3. Tokenização | Histograma de tokens + verificação da máscara de loss | [fase_03_tokenizacao](https://github.com/poliana-moraes/fine-tuning-project/) |
| 4. Estratégia de ajustes | Configuração justificada (estratégia + hiperparâmetros) | [fase_04_ajustes](https://github.com/poliana-moraes/fine-tuning-project/) |
| 5. Execução do treinamento | Curvas de loss com diagnóstico escrito + adaptador LoRA | [fase_05_treinamento](https://github.com/poliana-moraes/fine-tuning-project/) |
| 6. Avaliação | Tabela baseline × fine-tuned + matriz de confusão | [fase_06_avaliacao](https://github.com/poliana-moraes/fine-tuning-project/) |
| 7. Publicação | Modelo privado no Hub + model card + notebook de inferência | [fase_07_publicacao](https://github.com/poliana-moraes/fine-tuning-project/) |