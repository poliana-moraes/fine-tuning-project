# Fine-Tuning para Triagem de Alertas em SOC

Projeto prático da disciplina de Fine-Tuning. Você vai ajustar um modelo de linguagem pequeno para triar alertas de segurança de um SOC (Security Operations Center), seguindo a política de severidade da organização. O projeto passa por todas as fases do fine-tuning, do planejamento à publicação no Hugging Face.

O enunciado completo está em [reports/plano-fine-tuning-triagem-soc.md](reports/plano-fine-tuning-triagem-soc.md).

## Atividades

| Fase | Notebook |
|---|---|
| 1. Planejamento do objetivo e do modelo | [01_planejamento_baseline.ipynb](notebooks/01_planejamento_baseline.ipynb) · [Abrir no Colab](https://colab.research.google.com/github/poliana-moraes/fine-tuning-project/blob/main/notebooks/01_planejamento_baseline.ipynb) |

Os notebooks das próximas fases serão publicados aqui ao longo do projeto.

## Antes de começar

1. **Conta no Hugging Face:** crie uma conta em [huggingface.co](https://huggingface.co) e aceite o convite para a *organization* da turma. O dataset do projeto é privado e só pode ser lido por membros dela.
2. **Token de acesso:** em *Settings → Access Tokens*, crie um token do tipo **Read**. Nas fases de publicação você vai precisar de um token **Write**.
3. **Abrir o notebook no Colab:** use o link "Abrir no Colab" da tabela acima e, em seguida, *Arquivo → Salvar uma cópia no Drive*. Sem a cópia, suas alterações se perdem.
4. **GPU:** *Ambiente de execução → Alterar o tipo de ambiente de execução → GPU T4*.
5. **Secrets:** no ícone de chave do Colab, crie o segredo `HF_TOKEN` com o seu token e libere o acesso do notebook.

## Regras de segurança

- O token do Hugging Face fica **só** nos *Secrets* do Colab. Nunca cole o token numa célula, num arquivo ou num commit.
- Modelos e datasets que você publicar no Hugging Face devem ser **privados**.
- Todos os dados do projeto são **sintéticos**. Nenhum dado real de incidente deve ser usado.

## Entrega

Para cada fase, entregue o notebook com as células executadas e as respostas preenchidas, junto com o entregável descrito no enunciado (na Fase 1, a ficha do Passo 11), conforme combinado em aula.
