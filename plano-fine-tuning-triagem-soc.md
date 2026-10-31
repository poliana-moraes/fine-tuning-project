# Projeto Didático — Fine-Tuning para Triagem de Alertas em SOC

**Disciplina:** Fine-Tuning de Modelos de Linguagem
**Tema:** Triagem automatizada de alertas de segurança (SOC — Security Operations Center)
**Ecossistema:** Hugging Face (`transformers`, `datasets`, `tokenizers`, `peft`, `trl`, `evaluate`, `huggingface_hub`)
**Formato:** projeto prático, individual ou em dupla, com entregas por fase

---

## 0. Visão geral do projeto

### 0.1 Problema de negócio

Um SOC recebe centenas a milhares de alertas por dia (SIEM, EDR, IDS/IPS, proxy, e-mail). Analistas de nível 1 (L1) gastam a maior parte do tempo **triando** esses alertas: decidindo se é falso positivo, qual a severidade real, e qual ação tomar. Isso gera *alert fatigue* e atraso na resposta a incidentes reais.

O projeto consiste em fazer o **fine-tuning de um modelo de linguagem pequeno** para, a partir da descrição textual de um alerta, produzir uma **triagem estruturada**:

```json
{
  "veredito": "falso_positivo | verdadeiro_positivo",
  "severidade": "baixa | media | alta | critica",
  "categoria_mitre": "ex: T1059 - Command and Scripting Interpreter",
  "acao_recomendada": "ex: isolar host e escalar para L2",
  "justificativa": "explicação curta em linguagem natural"
}
```

Isso transforma a tarefa em **classificação + geração estruturada condicionada**, o que é pedagogicamente rico: cobre tokenização de instrução, templates de chat, LoRA/QLoRA, e métricas combinadas (classificação + qualidade textual).

### 0.2 Por que este tema serve bem à disciplina

- É uma tarefa real de cibersegurança (liga a disciplina de fine-tuning ao eixo de segurança do curso).
- Dados sensíveis/raros → força o aluno a **construir e tratar dados sintéticos**, uma habilidade real de projetos de IA aplicada à segurança.
- Tem uma métrica de negócio clara e assimétrica (falso negativo em "crítico" é muito mais caro que falso positivo), o que obriga discussão de métricas além de acurácia.
- Cabe em hardware de aluno (GPU gratuita de Colab/Kaggle) usando PEFT/LoRA.

### 0.3 Estrutura de pastas recomendada no repositório

```
fine-tuning-project/
├── plano-fine-tuning-triagem-soc.md   <- este arquivo
├── data/
│   ├── raw/          # fontes brutas (MITRE, logs sintéticos, templates)
│   ├── processed/    # dataset limpo e padronizado
│   └── splits/        # train.jsonl, val.jsonl, test.jsonl
├── notebooks/
│   ├── 01_planejamento.ipynb
│   ├── 02_preparacao_dados.ipynb
│   ├── 03_tokenizacao.ipynb
│   ├── 04_treinamento.ipynb
│   ├── 05_configuracao_hparams.ipynb
│   ├── 06_avaliacao.ipynb
│   └── 07_publicacao.ipynb
├── src/
│   ├── data_utils.py
│   ├── metrics.py
│   └── inference.py
└── model_card.md
```

### 0.4 Entregáveis por fase (resumo)

| Fase | Entregável |
|---|---|
| 1. Planejamento | Documento de objetivo + ficha de seleção do modelo |
| 2. Preparação dos dados | Dataset versionado (train/val/test) + relatório de qualidade |
| 3. Tokenização | Notebook de tokenização + análise de distribuição de tokens |
| 4. Execução do treinamento | Script/notebook de treino + checkpoints |
| 5. Estratégia de ajustes | Tabela de hiperparâmetros testados + justificativa da escolha final |
| 6. Avaliação | Relatório de métricas no conjunto de teste + matriz de confusão |
| 7. Publicação | Modelo publicado no Hugging Face Hub + model card + demo (Space) |

---

## Fase 1 — Planejamento do objetivo e do modelo

**Objetivo da fase:** decidir exatamente o que o modelo deve fazer, como medir sucesso, e qual modelo base usar.

### 1.1 Definição da tarefa

- **Entrada:** texto descritivo do alerta (campo livre, estilo "log/alerta" — ex.: origem, host, usuário, processo, hash, IP).
- **Saída:** JSON estruturado com veredito, severidade, categoria MITRE ATT&CK, ação recomendada e justificativa.
- **Tipo de tarefa:** fine-tuning supervisionado tipo *instruction following* (SFT) sobre um LLM decoder-only, com saída em formato restrito.

### 1.2 Critério de sucesso (como saber se funcionou)

Definir **antes** de treinar, para evitar mover a meta depois:

- Acurácia de veredito (falso positivo / verdadeiro positivo) ≥ 85% no conjunto de teste.
- **Recall de "crítico" ≥ 90%** (métrica de negócio prioritária — perder um alerta crítico é o pior erro possível em um SOC).
- JSON válido (parseável) em ≥ 98% das saídas.
- Justificativa coerente com o veredito em avaliação qualitativa (checklist humano ou LLM-as-judge) em amostra de 30 casos.

> **Ponto de discussão em aula:** por que recall de "crítico" é mais importante que acurácia geral? Ligar ao conceito de custo assimétrico de erro em segurança (um falso negativo pode significar um ataque não detectado).

### 1.3 Escolha do modelo base

Critérios de seleção e exemplos de modelos candidatos no Hugging Face Hub:

| Critério | Pergunta a responder |
|---|---|
| Tarefa | O modelo é instruction-tuned (aceita chat template) ou é base cru? |
| Licença | Permite uso educacional/comercial? Exige gated access? |
| Hardware | Cabe em GPU gratuita (T4/16GB, Colab/Kaggle) com QLoRA 4-bit? |
| Idioma | Tem bom desempenho em português (alertas podem estar em PT-BR)? |
| Tamanho | Compatível com tempo de treino de uma aula/sprint (≤ 1-3B parâmetros)? |

Candidatos recomendados (ordem de preferência para este projeto):

1. **`Qwen/Qwen2.5-1.5B-Instruct`** — licença Apache 2.0, sem *gating*, bom suporte multilíngue, ótimo custo/benefício para QLoRA em GPU única.
2. **`HuggingFaceTB/SmolLM2-1.7B-Instruct`** — Apache 2.0, feito explicitamente para fine-tuning educacional, leve.
3. **`microsoft/Phi-3.5-mini-instruct`** — licença MIT, forte em seguir formato estruturado (JSON), mais pesado (3.8B).
4. **`meta-llama/Llama-3.2-3B-Instruct`** — ótima qualidade, porém *gated* (exige aprovação de licença na Meta/HF), considerar apenas se o aluno já tem acesso aprovado.

> **Decisão sugerida para a turma:** usar `Qwen2.5-1.5B-Instruct` como padrão (evita bloqueios de licença gated no meio da aula), permitindo que grupos avançados testem alternativas.

### 1.4 Checklist de saída da Fase 1

- [ ] Documento de 1 página com: problema, entrada/saída, critério de sucesso quantitativo.
- [ ] Ficha de seleção do modelo preenchida (tabela de critérios acima) com o modelo escolhido.
- [ ] Verificação de licença do modelo escolhido (link para a licença no model card do Hub).
- [ ] Verificação de hardware disponível (Colab/Kaggle GPU, ou GPU local) e decisão: full fine-tuning, LoRA ou QLoRA.

---

## Fase 2 — Preparação dos dados

**Objetivo da fase:** reunir exemplos representativos, limpar, padronizar e dividir em treino/validação/teste.

### 2.1 Fontes de dados

Como não existe (nem deveria existir publicamente) um dataset massivo de alertas reais de SOC rotulados, o projeto usa uma combinação de fontes:

1. **Taxonomia MITRE ATT&CK** (`attack.mitre.org` ou dataset `mitre-attack` no Hub) — para gerar categorias e vocabulário técnico realista de técnicas de ataque.
2. **Templates sintéticos de alertas** — o aluno escreve/gera templates parametrizados por tipo de fonte (EDR, SIEM, proxy, e-mail/phishing, IDS de rede), variando host, usuário, processo, IP, hash etc.
3. **Geração assistida por LLM** — usar um modelo generalista (ex.: via API ou localmente) para expandir os templates em centenas de variações realistas, **sempre com revisão humana** antes de usar como rótulo verdadeiro.
4. **Datasets públicos de apoio** (opcional, para enriquecer vocabulário, não necessariamente para rótulos finais): buscar no Hugging Face Hub por termos como `security alerts`, `phishing`, `log anomaly`, `cyber threat intelligence`.

> **Ponto de atenção ético/técnico:** dados sintéticos devem ser claramente documentados como sintéticos no model card final. Nunca usar dados reais de incidentes de empresas sem anonimização e autorização.

### 2.2 Limpeza e padronização

- Remover duplicatas exatas e quase-duplicatas (ex.: `datasets.Dataset.unique()` ou hashing do texto normalizado).
- Padronizar codificação (UTF-8), remover caracteres de controle, normalizar espaços.
- Padronizar o **schema de saída** (sempre as mesmas chaves JSON, mesmos valores categóricos possíveis — isso é crítico para a Fase 3 e Fase 6).
- Balancear classes minimamente: garantir que "crítico" e "falso positivo" tenham exemplos suficientes (evitar dataset 95% "baixa severidade").
- Validar que todo exemplo tem JSON de saída **sintaticamente válido** (`json.loads` sem erro).

### 2.3 Formato final do exemplo (estilo instrução)

```json
{
  "instruction": "Você é um analista de SOC. Analise o alerta abaixo e produza a triagem em JSON.",
  "input": "EDR ALERT | Host: WIN-FIN03 | Usuário: j.silva | Processo: powershell.exe -enc JAB... | Pai: outlook.exe | Hash: 3f2a...",
  "output": "{\"veredito\": \"verdadeiro_positivo\", \"severidade\": \"alta\", \"categoria_mitre\": \"T1059.001 - PowerShell\", \"acao_recomendada\": \"isolar host e coletar memória\", \"justificativa\": \"PowerShell codificado iniciado por processo de e-mail é padrão típico de macro malicioso.\"}"
}
```

Usar `datasets.Dataset.from_list(...)` para carregar e manipular com a biblioteca `datasets` do Hugging Face.

### 2.4 Divisão treino/validação/teste

- Proporção sugerida: **70% treino / 15% validação / 15% teste**.
- Estratificar a divisão por `severidade` e `veredito` (usar `train_test_split` com `stratify=` manual, já que `datasets` puro não estratifica — pode-se usar `scikit-learn` só para gerar os índices).
- Garantir que **nenhum template-base** apareça simultaneamente em treino e teste (evita vazamento de dados / *data leakage* por memorização de template).

### 2.5 Checklist de saída da Fase 2

- [ ] Dataset consolidado em `data/processed/` com schema único e documentado.
- [ ] Relatório de qualidade: nº de exemplos, distribuição de classes, nº de duplicatas removidas.
- [ ] Arquivos `train.jsonl`, `val.jsonl`, `test.jsonl` salvos em `data/splits/`.
- [ ] (Opcional, recomendado) Dataset publicado como **privado** no Hugging Face Hub via `datasets.push_to_hub()`.

---

## Fase 3 — Tokenização

**Objetivo da fase:** converter cada exemplo em tokens usando o tokenizador do modelo escolhido, aplicando o template de chat correto.

### 3.1 Carregar o tokenizador correto

```python
from transformers import AutoTokenizer

model_id = "Qwen/Qwen2.5-1.5B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)
```

> **Regra de ouro:** sempre usar o tokenizador do **mesmo modelo** que será treinado. Tokenizadores diferentes têm vocabulários diferentes — usar o errado corrompe o treinamento silenciosamente.

### 3.2 Aplicar o chat template do modelo

Modelos instruction-tuned têm um formato de conversa específico (system/user/assistant) embutido no tokenizador:

```python
messages = [
    {"role": "system", "content": "Você é um analista de SOC. Analise o alerta e produza a triagem em JSON."},
    {"role": "user", "content": exemplo["input"]},
    {"role": "assistant", "content": exemplo["output"]},
]

texto_formatado = tokenizer.apply_chat_template(messages, tokenize=False)
```

### 3.3 Tokenização e truncamento

- Definir `max_length` com base na análise de distribuição real (ver 3.4), não em um número arbitrário.
- Usar `padding="max_length"` ou *dynamic padding* via `DataCollatorForLanguageModeling` / `DataCollatorForSeq2Seq` conforme a abordagem de treino escolhida na Fase 5.
- **Mascarar o loss do prompt**: durante SFT, o modelo só deve ser penalizado por errar a *saída* (assistant), não por "prever" o prompt do usuário. Isso é feito atribuindo `-100` aos `labels` correspondentes ao trecho de instrução/input (o `SFTTrainer` da biblioteca `trl` faz isso automaticamente quando configurado corretamente).

### 3.4 Análise de distribuição de tokens (obrigatória)

Antes de treinar, plotar o histograma do número de tokens por exemplo (prompt + resposta):

```python
lens = [len(tokenizer(ex)["input_ids"]) for ex in textos_formatados]
```

- Decidir `max_length` cobrindo ~ 95-99º percentil, sem desperdiçar memória com um valor excessivo.
- Identificar outliers (exemplos anormalmente longos) — podem indicar erro de geração de dados na Fase 2.

### 3.5 Checklist de saída da Fase 3

- [ ] Notebook de tokenização com `apply_chat_template` aplicado corretamente.
- [ ] Histograma de distribuição de tokens e `max_length` justificado.
- [ ] Confirmação visual (decodificar um exemplo tokenizado de volta para texto) de que o template está correto.
- [ ] Dataset tokenizado salvo (`.map()` com `datasets`, cacheado em disco).

---

## Fase 5 (configuração) antes da Fase 4 (execução) — nota pedagógica

> Na prática de código, a **estratégia e configuração (Fase 5)** é decidida **antes** de rodar o treino (Fase 4) — é ela que determina o script de treino. O enunciado do projeto numera "Execução" como fase 4 e "Estratégia" como fase 5 porque são apresentadas nessa ordem didaticamente (primeiro "o que acontece durante o treino", depois "como configurar isso"). Ao executar o projeto, trate as duas como um ciclo: **configure (Fase 5) → execute (Fase 4) → avalie (Fase 6) → ajuste a configuração e repita**.

---

## Fase 4 — Execução do treinamento

**Objetivo da fase:** o modelo vê os exemplos de treino repetidamente e ajusta pesos para errar menos, enquanto se monitora a curva de *loss*.

### 4.1 Setup do treino com PEFT/LoRA + TRL

```python
from transformers import AutoModelForCausalLM, BitsAndBytesConfig
from peft import LoraConfig
from trl import SFTTrainer, SFTConfig

bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_quant_type="nf4",
    bnb_4bit_compute_dtype="bfloat16",
)

model = AutoModelForCausalLM.from_pretrained(model_id, quantization_config=bnb_config)

lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    task_type="CAUSAL_LM",
)

trainer = SFTTrainer(
    model=model,
    train_dataset=dataset_train,
    eval_dataset=dataset_val,
    peft_config=lora_config,
    args=SFTConfig(...),  # ver Fase 5 para os hiperparâmetros
)

trainer.train()
```

### 4.2 Monitoramento da curva de loss (o coração da fase)

Acompanhar **treino vs. validação** a cada *eval_steps* (usar `report_to="tensorboard"` ou `wandb`):

| Padrão observado | Diagnóstico | Ação |
|---|---|---|
| Loss de treino ↓ e loss de validação ↓ | Aprendizado saudável | Continuar / considerar mais épocas |
| Loss de treino ↓ e loss de validação ↑ (ou estagna) | **Overfitting** — o modelo decorou exemplos específicos (ex.: hosts, hashes fixos dos templates) em vez do padrão geral | Reduzir épocas, aumentar `lora_dropout`, aumentar variedade de dados (voltar à Fase 2), reduzir `r` do LoRA |
| Ambas estagnadas em valor alto | Underfitting | Aumentar `r`/`lora_alpha`, aumentar taxa de aprendizado, verificar se o template de chat/máscara de loss está correto (erro comum na Fase 3) |
| Loss oscilando/explodindo (`NaN`) | Taxa de aprendizado alta demais, ou `bf16/fp16` mal configurado | Reduzir learning rate, verificar `gradient_clipping` |

> **Atividade em sala:** cada aluno deve printar e anexar ao relatório o gráfico das duas curvas e escrever 2-3 frases de diagnóstico — essa é a evidência de que entendeu o conceito, não apenas rodou o código.

### 4.3 Checklist de saída da Fase 4

- [ ] Gráfico de loss treino vs. validação, com diagnóstico escrito.
- [ ] Checkpoints do adaptador LoRA salvos (`trainer.save_model()`).
- [ ] Log de treino (tempo total, GPU usada, VRAM de pico) documentado — relevante para a Fase 1 (viabilidade de hardware).

---

## Fase 5 — Estratégia de ajustes e configuração

**Objetivo da fase:** definir **como** o modelo será ajustado (full fine-tuning vs. PEFT) e com quais hiperparâmetros.

### 5.1 Escolha da estratégia de ajuste

| Estratégia | Quando usar | Custo |
|---|---|---|
| Full fine-tuning | Dataset grande, GPU com muita VRAM, necessidade de máxima qualidade | Alto (atualiza todos os pesos) |
| **LoRA** | Caso padrão deste projeto — bom equilíbrio qualidade/custo | Baixo (treina ~0.1-1% dos parâmetros) |
| QLoRA (LoRA + quantização 4-bit) | GPU gratuita limitada (Colab T4 16GB) | Muito baixo — **recomendado para este projeto** |

### 5.2 Hiperparâmetros a configurar e faixa inicial sugerida

| Hiperparâmetro | O que controla | Valor inicial sugerido | Sinal de ajuste |
|---|---|---|---|
| `learning_rate` | Tamanho do passo de atualização dos pesos | `2e-4` (LoRA) | Loss oscila/explode → diminuir; loss não desce → aumentar |
| `per_device_train_batch_size` | Nº de exemplos por passo, limitado pela VRAM | `4` (com `gradient_accumulation_steps=4`) | OOM (falta de memória) → diminuir e compensar com acumulação |
| `num_train_epochs` | Quantas vezes o modelo vê o dataset completo | `3` | Validação piora antes do fim → reduzir (ou usar *early stopping*) |
| `lora_r` / `lora_alpha` | Capacidade do adaptador LoRA | `r=16`, `alpha=32` | Underfitting → aumentar `r`; overfitting → diminuir |
| `warmup_ratio` | Aquecimento da taxa de aprendizado | `0.03` | Instabilidade no início do treino → aumentar |
| `weight_decay` | Regularização | `0.01` | Overfitting persistente → aumentar |

### 5.3 Metodologia de busca de hiperparâmetros

Para fins didáticos (sem gastar GPU excessivamente), rodar uma **busca manual em grade pequena**, não uma busca automática cara:

1. Fixar `num_train_epochs=1` e testar 3 valores de `learning_rate` (ex.: `1e-4`, `2e-4`, `5e-4`) observando só a tendência inicial da curva de validação.
2. Com o melhor `learning_rate`, treinar as 3 épocas completas.
3. Documentar **todas** as combinações testadas em uma tabela — mesmo as que falharam (overfitting, instabilidade) — pois isso é parte da avaliação (processo, não só resultado final).

### 5.4 Checklist de saída da Fase 5

- [ ] Tabela de experimentos (hiperparâmetro × loss final treino/val × observação).
- [ ] Justificativa escrita da configuração final escolhida.
- [ ] Arquivo de configuração versionado (`SFTConfig`/`LoraConfig` salvos como JSON/YAML junto ao checkpoint).

---

## Fase 6 — Avaliar o desempenho

**Objetivo da fase:** medir o desempenho real no conjunto de **teste** (nunca visto durante treino ou ajuste de hiperparâmetros).

### 6.1 Métricas quantitativas

- **Validade de formato:** % de saídas que são JSON parseável com todas as chaves esperadas.
- **Classificação (veredito, severidade, categoria MITRE):**
  - Acurácia geral.
  - Precision/Recall/F1 **por classe**, usando `evaluate` (`evaluate.load("f1")`, `evaluate.load("recall")`) ou `sklearn.metrics.classification_report`.
  - **Matriz de confusão de severidade** — item obrigatório do relatório, destacando especificamente quantos casos "crítico" reais foram classificados como severidade menor (falso negativo caro).
- **Geração (ação recomendada, justificativa):**
  - Similaridade textual (ROUGE-L ou BLEU) contra a resposta de referência — métrica fraca, mas informativa como baseline.
  - **LLM-as-judge**: usar um modelo maior (ou o próprio professor/colegas) para avaliar em escala 1-5 se a justificativa é coerente com o veredito, em amostra de 30-50 casos do teste.

### 6.2 Métrica de negócio (a mais importante deste projeto)

Construir explicitamente a **matriz de custo de erro**:

| Real → Previsto | Falso positivo previsto | Verdadeiro positivo previsto |
|---|---|---|
| Real: falso positivo | ✅ OK | Custo baixo (analista perde tempo revisando) |
| Real: verdadeiro positivo (crítico) | ❌ **Custo altíssimo** (ataque real ignorado) | ✅ OK |

Reportar explicitamente o **recall da classe "crítico"** comparado à meta definida na Fase 1 (≥ 90%). Se não atingida, isso direciona o ciclo de iteração.

### 6.3 Iterar

Com base nos erros de avaliação, decidir **onde** voltar:

| Sintoma na avaliação | Provável causa | Voltar para |
|---|---|---|
| JSON inválido com frequência | Template de saída inconsistente nos dados | Fase 2 (padronizar schema) |
| Baixo recall em "crítico" | Poucos exemplos críticos no treino, ou classes desbalanceadas | Fase 2 (mais dados/balanceamento) |
| Boa performance no treino, ruim no teste | Overfitting | Fase 5 (regularização, menos épocas) |
| Erro sistemático em uma categoria MITRE específica | Dados insuficientes para aquela técnica | Fase 2 |
| Justificativas incoerentes mas veredito correto | Modelo pequeno demais para raciocínio textual | Fase 1 (reconsiderar modelo base) |

### 6.4 Checklist de saída da Fase 6

- [ ] Relatório de métricas completo no conjunto de teste (nunca usado antes).
- [ ] Matriz de confusão de severidade com destaque para falsos negativos críticos.
- [ ] Decisão documentada: "resultado satisfatório, seguir para publicação" OU "iterar, retornando à fase X pelo motivo Y".

---

## Fase 7 — Publicação

**Objetivo da fase:** disponibilizar o modelo para uso e acompanhar seu comportamento em produção.

### 7.1 Publicar no Hugging Face Hub

```python
model.push_to_hub("seu-usuario/soc-triage-qwen2.5-1.5b-lora")
tokenizer.push_to_hub("seu-usuario/soc-triage-qwen2.5-1.5b-lora")
```

- Se usou LoRA, decidir entre publicar **apenas o adaptador** (leve, requer carregar o modelo base + `peft` na inferência) ou fazer `merge_and_unload()` e publicar os pesos completos (mais simples de consumir, maior tamanho).

### 7.2 Model Card (obrigatório)

Criar `README.md`/`model_card.md` do repositório no Hub contendo, no mínimo:

- Descrição da tarefa e limitações (ex.: "treinado com dados **sintéticos**, não validado em ambiente de produção real").
- Licença herdada do modelo base.
- Métricas da Fase 6 (incluindo a métrica de negócio — recall de crítico).
- **Aviso explícito de uso responsável:** este modelo é um **assistente de triagem**, não deve tomar decisão final sem revisão humana (*human-in-the-loop* obrigatório dado o risco de falso negativo em segurança).

### 7.3 Demonstração (recomendado)

Publicar um **Hugging Face Space** com Gradio para demo interativa:

```python
import gradio as gr

def triar(alerta):
    # carrega modelo + tokenizer, aplica chat template, gera e faz parse do JSON
    ...

gr.Interface(fn=triar, inputs="text", outputs="json").launch()
```

### 7.4 Acompanhamento em produção (discussão conceitual para a disciplina)

Mesmo em escopo de projeto didático, discutir e documentar como seria o monitoramento real:

- **Logging de previsões** (entrada, saída, timestamp) para auditoria — cuidado com dados sensíveis reais.
- **Feedback loop**: analistas humanos marcam se a triagem do modelo estava certa → vira dado para retraining futuro (volta à Fase 2).
- **Monitoramento de drift**: novos tipos de ataque/ferramentas (ex.: nova técnica MITRE) podem não estar no vocabulário de treino — acompanhar taxa de "incerteza"/JSON inválido como sinal de alerta.
- **Métricas de produção**: tempo médio de triagem economizado, taxa de concordância analista-modelo, nº de críticos escalados corretamente.

### 7.5 Checklist de saída da Fase 7

- [ ] Modelo (ou adaptador LoRA) publicado no Hugging Face Hub.
- [ ] Model card completo com métricas, limitações e aviso de uso responsável.
- [ ] (Recomendado) Space com demo funcional.
- [ ] Seção de "plano de monitoramento em produção" documentada mesmo que não implementada de fato.

---

## Rubrica geral de avaliação do projeto

| Critério | Peso |
|---|---|
| Clareza do objetivo e adequação da escolha do modelo (Fase 1) | 10% |
| Qualidade e documentação do dataset (Fase 2) | 20% |
| Correção técnica da tokenização/template (Fase 3) | 10% |
| Execução do treino com monitoramento e diagnóstico correto da curva de loss (Fase 4) | 15% |
| Rigor na busca e justificativa de hiperparâmetros (Fase 5) | 15% |
| Qualidade da avaliação, incluindo métrica de negócio (Fase 6) | 20% |
| Publicação, model card e discussão de uso responsável (Fase 7) | 10% |
