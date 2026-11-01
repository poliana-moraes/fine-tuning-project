# Projeto — Fine-Tuning para Triagem de Alertas em SOC (Security Operations Center)

## Visão geral do projeto

### Problema

Um SOC recebe centenas a milhares de alertas por dia (SIEM, EDR, IDS/IPS, proxy, e-mail). Mesmo quando as etapas de coleta, normalização e enriquecimento (detalhadas abaixo) já entregam cada alerta com contexto (criticidade do ativo, perfil do usuário, reputação de IOC, histórico), o analista de nível 1 (L1) ainda precisa ler alerta por alerta e aplicar a política de severidade da organização para decidir se é falso positivo, qual a severidade real e por quê. Essa decisão repetitiva consome a maior parte do tempo do L1. Ela também varia entre analistas e turnos, o que gera triagens inconsistentes e atraso na resposta a incidentes reais.

### Tarefa do modelo

A partir de um **alerta com sinal técnico e contexto**, produzir:

```json
{
  "veredito": "falso_positivo | verdadeiro_positivo",
  "severidade": "baixa | media | alta | critica",
  "justificativa": "explicação curta citando o sinal técnico e o contexto"
}
```

### Entregas por fase

| Fase | Entregável |
|---|---|
| 1. Planejamento do objetivo e do modelo | Ficha de objetivo e modelo + resultado do teste de necessidade (baseline) |
| 2. Preparação dos dados | Relatório de auditoria + splits treino/validação/teste |
| 3. Tokenização | Histograma de tokens + verificação da máscara de loss |
| 4. Execução do treinamento | Curvas de loss com diagnóstico escrito + adaptador LoRA |
| 5. Estratégia de ajustes | Configuração justificada (estratégia + hiperparâmetros) |
| 6. Avaliação | Tabela baseline × fine-tuned + matriz de confusão + uma iteração |
| 7. Publicação | Modelo privado no Hub + model card + notebook de inferência |

### Ambiente e ferramentas

| Ferramenta | Papel | Quem usa |
|---|---|---|
| **Google Colab** | Onde o código roda: um notebook por fase, com GPU T4 | Alunos e professor |
| **Hugging Face Hub** | Modelo base, dataset do projeto (privado), splits de cada aluno e publicação do adaptador e do model card | Alunos e professor |
| **GitHub** | Versão oficial do material (plano, política de severidade, notebooks). O Colab abre notebooks direto do GitHub e salva cópias de volta | Professor; alunos para entregar |
| **VS Code** | Edição do plano e dos documentos e trabalho com git | Só o professor |

**Configuração do Colab:** em *Ambiente de execução → Alterar o tipo de ambiente de execução*, selecionar **GPU T4**.

**Primeira célula de todo notebook:**

```python
!pip install -q transformers datasets peft trl evaluate scikit-learn

from google.colab import userdata
from huggingface_hub import login

login(token=userdata.get("HF_TOKEN"))
```

> O professor fixa na célula acima as versões testadas das bibliotecas (ex.: `transformers==X.Y.Z`) antes do início do projeto, para que todos rodem o mesmo ambiente.

**Cuidados:**

- **Token do Hugging Face só nos *Secrets* do Colab** (ícone de chave, nome `HF_TOKEN`). Nunca colar o token numa célula nem commitá-lo no GitHub. Um token vazado num notebook versionado é um incidente de segurança real.
- **Uma fonte por artefato.** O dataset fica só no Hugging Face, e notebooks e documentos ficam só no GitHub. Assim ninguém trabalha com uma versão desatualizada.
- **O runtime do Colab é temporário.** Tudo que é salvo no disco do runtime some quando ele desconecta. Por isso os splits (Fase 2) e o adaptador (Fase 4) são enviados ao Hub assim que gerados.
- **Acesso:** o dataset fica numa *organization* da turma no Hugging Face (alunos como membros), e o repositório do GitHub dá acesso aos alunos como colaboradores ou via repositório da turma.

---

## Fase 1 — Planejamento do objetivo e do modelo

**Objetivo da fase:** definir o que o modelo deve fazer, como saber se funcionou, provar que o fine-tuning é necessário e escolher o modelo base.

### 1.1 Definição da tarefa

- **Entrada:** alerta com sinal técnico + contexto.
- **Saída:** JSON com `veredito`, `severidade` e `justificativa`.
- **Tipo:** fine-tuning supervisionado (SFT) de um LLM pequeno, com saída em formato restrito.

### 1.2 Política de severidade da organização (o que o modelo deve aprender)

A severidade **não** depende só do sinal técnico. Depende da combinação com o contexto. Exemplo resumido:

| Sinal técnico | Ativo crítico (DC, servidor financeiro) | Ativo comum | Ativo de laboratório |
|---|---|---|---|
| Forte (ex.: PowerShell codificado filho do Outlook, IOC com reputação maliciosa) | crítica | alta | media |
| Moderado (ex.: login fora do horário, ferramenta administrativa incomum) | alta | media | baixa |
| Fraco e recorrente (ex.: mesmo alerta > 50 vezes em 7 dias sem incidente) | media | baixa | baixa (falso positivo provável) |

> **Ponto de discussão:** o mesmo PowerShell é crítico num controlador de domínio e médio numa máquina de laboratório. É essa política, aplicada de forma consistente, que justifica ajustar o modelo. Um modelo genérico não conhece o critério da sua organização.

### 1.3 Critério de sucesso

| Métrica | Meta |
|---|---|
| JSON válido com as 3 chaves | ≥ 98% |
| Acurácia de veredito | ≥ 85% |
| **Recall de severidade "crítica"** | **≥ 90%** (métrica de negócio: perder um crítico é o pior erro) |
| Justificativa coerente (checklist humano, 10 casos) | ≥ 8 de 10 |
| **Resultado esperado** | O fine-tuned precisa superar o baseline nas métricas acima |

### 1.4 Teste de necessidade

Antes de decidir ajustar, prove que o prompt sozinho não basta:

1. Rodar o modelo base escolhido com um **prompt few-shot** (política de severidade + 3 exemplos) sobre a amostra de baseline: `load_dataset("org-da-turma/alertas-soc", data_files="amostra_baseline.jsonl")`.
2. Contar: quantas saídas são JSON válido? Quantas severidades seguem a política?
3. Registrar o resultado. Ele será comparado formalmente na Fase 6.

> Se o baseline já atingir as metas, a resposta correta é **não fazer fine-tuning**. Isso precisa ser dito no relatório. Para fins didáticos o projeto segue adiante, mas o aluno deve saber reconhecer esse caso.

### 1.5 Escolha do modelo base

| Critério | Pergunta |
|---|---|
| Tarefa | É instruction-tuned (tem chat template)? |
| Licença | Permite uso educacional e comercial? Exige acesso *gated*? |
| Hardware | Treina numa T4 dentro do tempo disponível? |
| Idioma | Lida bem com português? |
| Implantação | É pequeno o bastante para rodar on-premises em volume? |

Decisão para o projeto:

- **Padrão: `Qwen/Qwen2.5-0.5B-Instruct`.** Licença Apache 2.0, sem *gating*, multilíngue, treina em cerca de 15 min numa T4.
- **Alternativa (GPU melhor ou tempo extra): `Qwen/Qwen2.5-1.5B-Instruct`.** Mesma família e licença, mais capacidade para a justificativa.

### 1.6 Checklist de saída

- [ ] Ficha com tarefa, métricas-alvo e modelo escolhido (com link da licença).
- [ ] Resultado do baseline registrado (JSON válido e acertos de severidade em 20 casos).

---

## Fase 2 — Preparação dos dados

**Objetivo da fase:** auditar o dataset fornecido, limpar, padronizar e dividir em treino, validação e teste.

### 2.1 Formato do exemplo

```json
{
  "template_id": "edr_ps_encoded_07",
  "alerta": "EDR | Host: SRV-FIN01 | Usuário: j.silva | Processo: powershell.exe -enc JAB... | Pai: outlook.exe",
  "contexto": "Ativo: servidor financeiro (criticidade alta) | Usuário: analista financeiro, sem privilégio admin | Reputação do hash: desconhecida | Ocorrências em 7 dias: 1",
  "triagem": {"veredito": "verdadeiro_positivo", "severidade": "critica", "justificativa": "PowerShell codificado iniciado pelo Outlook em servidor financeiro de alta criticidade, padrão típico de macro maliciosa."}
}
```

### 2.2 Auditoria

Carregar do Hub com `datasets.load_dataset("org-da-turma/alertas-soc", data_files="alertas_soc.jsonl")` e verificar:

- [ ] **Duplicatas** exatas no par `alerta` + `contexto`.
- [ ] **Triagem inválida**: chaves faltando ou valores fora das categorias permitidas.
- [ ] **Rótulos inconsistentes com a política** da Fase 1. Ex.: sinal forte em ativo crítico rotulado como "baixa". Corrigir ou remover, e registrar quantos.
- [ ] **Distribuição de classes**: há exemplos suficientes de "crítica" e de "falso_positivo"?

> **Ponto de discussão:** um rótulo errado no treino ensina o modelo a errar. Em segurança, um único padrão "crítico rotulado como baixa" repetido vira um ponto cego do modelo.

### 2.3 Divisão treino/validação/teste

- Proporção: **70% / 15% / 15%**.
- **Dividir por `template_id`** (todos os exemplos de um template ficam no mesmo split). Isso evita *data leakage*: se variações do mesmo template caem no treino e no teste, o modelo "acerta" o teste por memorização e a métrica fica inflada.
- Conferir se a distribuição de severidade ficou parecida nos três splits.

```python
from sklearn.model_selection import GroupShuffleSplit
from datasets import DatasetDict

splits = DatasetDict({"train": train, "validation": val, "test": test})
splits.push_to_hub("seu-usuario/alertas-soc-splits", private=True)
```

Os notebooks das fases seguintes carregam os splits com `load_dataset("seu-usuario/alertas-soc-splits")`.

### 2.4 Checklist de saída

- [ ] Relatório curto: nº de exemplos antes e depois, duplicatas removidas, rótulos corrigidos, distribuição de classes por split.
- [ ] Splits treino/validação/teste publicados como privados no Hub.

---

## Fase 3 — Tokenização

**Objetivo da fase:** converter os exemplos em tokens com o tokenizador do próprio modelo, no formato do seu chat template.

### 3.1 Formato conversacional prompt/completion

Converter cada exemplo para o formato que o `SFTTrainer` (TRL) entende:

```python
SYSTEM = "Você é um analista de SOC. Aplique a política de severidade da organização e responda apenas com o JSON da triagem."

def to_conversation(ex):
    return {
        "prompt": [
            {"role": "system", "content": SYSTEM},
            {"role": "user", "content": f"{ex['alerta']}\n{ex['contexto']}"},
        ],
        "completion": [
            {"role": "assistant", "content": json.dumps(ex["triagem"], ensure_ascii=False)},
        ],
    }
```

### 3.2 Tokenizador e chat template

```python
from transformers import AutoTokenizer

model_id = "Qwen/Qwen2.5-0.5B-Instruct"
tokenizer = AutoTokenizer.from_pretrained(model_id)

texto = tokenizer.apply_chat_template(ex["prompt"] + ex["completion"], tokenize=False)
print(texto)
```

> **Regra de ouro:** sempre o tokenizador do **mesmo modelo** que será treinado, e o **mesmo template e prompt de sistema** no treino e na inferência (Fase 7). Divergência aqui corrompe o modelo de forma silenciosa.

### 3.3 Máscara de loss

O modelo deve ser penalizado apenas por errar a **triagem**, não por "prever" o alerta. No formato prompt/completion, o `SFTTrainer` calcula o loss só na completion. **Verifique isso em vez de confiar:**

- Pegar um batch do `trainer` e decodificar apenas os tokens com `labels != -100`.
- O texto decodificado deve conter **somente** o JSON da triagem.

### 3.4 Distribuição de tokens

```python
lens = [len(tokenizer.apply_chat_template(ex["prompt"] + ex["completion"])) for ex in train]
```

- Plotar o histograma e escolher `max_length` cobrindo cerca do percentil 99.
- Exemplos muito longos fora da curva costumam indicar defeito de dado que escapou na Fase 2.

### 3.5 Checklist de saída

- [ ] Um exemplo impresso com o chat template aplicado.
- [ ] Histograma de tokens e `max_length` justificado.
- [ ] Evidência de que a máscara de loss cobre só o JSON da triagem.

---

## Nota sobre a ordem das Fases 4 e 5

> No código, a **configuração (Fase 5)** é definida **antes** de rodar o treino (Fase 4). As fases são numeradas nesta ordem porque primeiro se entende o que acontece durante o treino e depois como controlá-lo. Na execução, trate como ciclo: **configurar (5) → executar (4) → avaliar (6) → ajustar e repetir**. As duas fases ficam no mesmo notebook (`04_05_treinamento.ipynb`).

---

## Fase 4 — Execução do treinamento

**Objetivo da fase:** o modelo vê os exemplos de treino repetidamente e ajusta os pesos para errar menos, enquanto se acompanham as curvas de loss.

### 4.1 Treino com LoRA + TRL

```python
import torch
from transformers import AutoModelForCausalLM
from peft import LoraConfig
from trl import SFTTrainer, SFTConfig

model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch.float16)

trainer = SFTTrainer(
    model=model,
    train_dataset=train,
    eval_dataset=val,
    peft_config=lora_config,   # Fase 5
    args=sft_config,           # Fase 5
    processing_class=tokenizer,
)
trainer.train()

repo_id = "seu-usuario/soc-triage-qwen2.5-0.5b-lora"
trainer.model.push_to_hub(repo_id, private=True)
```

> Enviar o adaptador ao Hub logo após o treino é a cópia de segurança: se o runtime do Colab desconectar, a avaliação da Fase 6 recarrega o adaptador do Hub sem precisar treinar de novo.

### 4.2 Leitura das curvas de loss

| Padrão observado | Diagnóstico | Ação |
|---|---|---|
| Treino ↓ e validação ↓ | Aprendizado saudável | Manter |
| Treino ↓ e validação ↑ ou estagnada | **Overfitting**: o modelo decorou os exemplos (hosts, usuários, frases do template) em vez de aprender a política | Menos épocas, mais `lora_dropout`, menor `r` |
| Ambas altas e estagnadas | Underfitting | Maior `learning_rate` ou `r`. Conferir máscara e template (Fase 3) |
| Oscilando ou `NaN` | Instabilidade | Menor `learning_rate`, conferir `fp16` |

### 4.3 Checklist de saída

- [ ] Gráfico das duas curvas com 2 a 3 frases de diagnóstico.
- [ ] Adaptador LoRA enviado como privado ao Hub.
- [ ] Tempo de treino e VRAM de pico anotados (confirma a viabilidade on-premises da Fase 1).

---

## Fase 5 — Estratégia de ajustes e configuração

**Tempo estimado:** 15 min
**Objetivo da fase:** definir como o modelo será ajustado e com quais hiperparâmetros.

### 5.1 Estratégia de ajuste

| Estratégia | Quando usar | Neste projeto |
|---|---|---|
| Full fine-tuning | Muitos dados e GPU grande | Não: custo alto e risco de overfitting com 1.500 exemplos |
| **LoRA** | Caso geral: treina menos de 1% dos parâmetros | **Sim, com o modelo 0.5B em `fp16`** |
| QLoRA (LoRA + 4-bit) | Modelo que não cabe na GPU em `fp16` | Só se usar o 1.5B ou maior |

```python
lora_config = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj"],
    task_type="CAUSAL_LM",
)
```

### 5.2 Hiperparâmetros

| Hiperparâmetro | O que controla | Valor inicial | Quando mudar |
|---|---|---|---|
| `learning_rate` | Tamanho do passo de ajuste dos pesos | `2e-4` | Oscila → diminuir. Não desce → aumentar |
| `per_device_train_batch_size` | Exemplos por passo (limitado pela VRAM) | `8` | Falta de memória → reduzir e aumentar `gradient_accumulation_steps` |
| `gradient_accumulation_steps` | Lote efetivo sem gastar mais VRAM | `2` (lote efetivo 16) | Compensar batch menor |
| `num_train_epochs` | Quantas vezes o modelo vê o treino inteiro | `2` | Validação sobe antes do fim → reduzir |
| `max_length` | Tamanho máximo da sequência | Fase 3 (percentil 99) | — |

```python
sft_config = SFTConfig(
    output_dir="outputs",
    learning_rate=2e-4,
    per_device_train_batch_size=8,
    gradient_accumulation_steps=2,
    num_train_epochs=2,
    max_length=MAX_LENGTH,
    eval_strategy="steps", eval_steps=20, logging_steps=10,
    fp16=True,
    report_to="tensorboard",
)
```

### 5.3 Checklist de saída

- [ ] Justificativa curta da estratégia (por que LoRA e não full fine-tuning) e dos valores escolhidos.
- [ ] Configuração salva junto ao adaptador.

---

## Fase 6 — Avaliar o desempenho

**Tempo estimado:** 60 min (inclui uma iteração com novo treino)
**Objetivo da fase:** medir no conjunto de **teste**, comparar com o baseline e iterar uma vez.

### 6.1 Métricas

Gerar a triagem para todo o `test.jsonl` com `do_sample=False`. Rodar o **baseline da Fase 1** (prompt few-shot no modelo base) sobre o mesmo teste e preencher:

| Métrica | Meta | Baseline (prompt) | Fine-tuned |
|---|---|---|---|
| JSON válido | ≥ 98% | | |
| Acurácia de veredito | ≥ 85% | | |
| F1 macro de severidade | — | | |
| **Recall de "crítica"** | **≥ 90%** | | |
| Justificativa coerente (10 casos) | ≥ 8/10 | | |

- Usar `sklearn.metrics.classification_report` e `confusion_matrix`.
- **Matriz de confusão de severidade** obrigatória, destacando os críticos classificados como severidade menor.
- Justificativa: checklist humano em 10 casos. A justificativa cita o sinal técnico? Cita o contexto? É coerente com a severidade?

### 6.2 Custo do erro (métrica de negócio)

| Real \ Previsto | Severidade menor que a real | Severidade maior que a real |
|---|---|---|
| Crítico | ❌ **Custo altíssimo**: ataque real fica na fila | — |
| Baixo ou médio | — | Custo baixo: analista perde alguns minutos |

Por isso o recall de "crítica" pesa mais que a acurácia geral.

### 6.3 Iterar (uma vez)

Escolher **um** sintoma, fazer **uma** mudança, treinar de novo e medir:

| Sintoma | Provável causa | Voltar para |
|---|---|---|
| JSON inválido frequente | Máscara ou template incorretos | Fase 3 |
| Recall de "crítica" baixo | Poucos críticos no treino ou rótulos ruins | Fase 2 |
| Treino bom e teste ruim | Overfitting | Fase 5 (menos épocas, menor `r`) |
| Severidade certa, justificativa fraca | Capacidade do modelo | Fase 1 (testar o 1.5B) |

### 6.4 Conclusão obrigatória

Responder em até 5 linhas: **o fine-tuning valeu a pena em relação ao baseline? Em quais métricas?** Se não superou, dizer isso também é um resultado válido.

### 6.5 Checklist de saída

- [ ] Tabela baseline × fine-tuned preenchida (antes e depois da iteração).
- [ ] Matriz de confusão de severidade.
- [ ] Registro da iteração: sintoma, mudança e efeito.
- [ ] Conclusão sobre o valor do fine-tuning.

---

## Fase 7 — Publicação

**Objetivo da fase:** disponibilizar o modelo para uso, garantir que ele funciona fora do notebook e definir como seria acompanhado em produção.

### 7.1 Publicar no Hugging Face Hub (privado)

Na Fase 4 o adaptador já foi enviado como cópia de segurança. Aqui se publica a **versão final** (depois da iteração da Fase 6) junto com o tokenizador:

```python
repo_id = "seu-usuario/soc-triage-qwen2.5-0.5b-lora"
trainer.model.push_to_hub(repo_id, private=True)
tokenizer.push_to_hub(repo_id, private=True)
```

- **Privado** porque é um modelo de segurança treinado com dados sintéticos e não validado em produção.
- Publicar apenas o **adaptador LoRA** (poucos MB), que é carregado sobre o modelo base na inferência.

### 7.2 Model card

`README.md` do repositório no Hub com:

- Tarefa e **recorte**: o modelo tria alertas já enriquecidos e não substitui enriquecimento, SOAR nem analista.
- Dados: **sintéticos**, gerados segundo uma política de severidade fictícia.
- Licença herdada do modelo base (Apache 2.0).
- Métricas da Fase 6, incluindo a comparação com o baseline e o recall de "crítica".
- **Uso responsável:** assistente de triagem com revisão humana obrigatória.

### 7.3 Notebook de inferência em sessão limpa

Notebook separado (`07_inferencia.ipynb`), aberto num **runtime novo** do Colab, sem nada do notebook de treino. Só a célula padrão de setup, o código abaixo e os alertas de teste carregados de `load_dataset("seu-usuario/alertas-soc-splits", split="test")`:

```python
import json
from transformers import AutoModelForCausalLM, AutoTokenizer
from peft import PeftModel

BASE_ID = "Qwen/Qwen2.5-0.5B-Instruct"
REPO_ID = "seu-usuario/soc-triage-qwen2.5-0.5b-lora"
SYSTEM = "Você é um analista de SOC. Aplique a política de severidade da organização e responda apenas com o JSON da triagem."

tokenizer = AutoTokenizer.from_pretrained(REPO_ID)
model = PeftModel.from_pretrained(AutoModelForCausalLM.from_pretrained(BASE_ID, device_map="auto"), REPO_ID)

def triar(alerta: str, contexto: str) -> dict:
    messages = [
        {"role": "system", "content": SYSTEM},
        {"role": "user", "content": f"{alerta}\n{contexto}"},
    ]
    inputs = tokenizer.apply_chat_template(messages, add_generation_prompt=True, return_tensors="pt").to(model.device)
    out = model.generate(inputs, max_new_tokens=200, do_sample=False)
    return json.loads(tokenizer.decode(out[0][inputs.shape[1]:], skip_special_tokens=True))
```

**Critério de aceite:** triar 5 alertas do teste (pelo menos 1 crítico e 1 falso positivo) e imprimir o resultado ao lado do rótulo esperado. Se falhar, o problema costuma ser algo que não foi publicado ou um prompt diferente do usado no treino.

### 7.4 Acompanhamento em produção (discussão escrita)

Em até meia página, descrever:

- **Feedback do analista:** cada triagem validada ou corrigida pelo L1 vira dado novo para o próximo ciclo (Fase 2).
- **Drift:** mudança na política da organização ou novos tipos de alerta. Sinal de alerta: queda na concordância analista-modelo.
- **Métricas de produção:** concordância analista-modelo, recall de críticos confirmados e tempo médio de triagem.

### 7.5 Checklist de saída

- [ ] Adaptador publicado como privado no Hub.
- [ ] Model card completo.
- [ ] `07_inferencia.ipynb` funcionando em sessão limpa.
- [ ] Discussão de acompanhamento em produção.

---
