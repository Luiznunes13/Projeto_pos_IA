# 🏭 Sistema Baseado em Conhecimento — Monitoramento de Impressão 3D

**Disciplina:** Sistemas Baseados em Conhecimento — 3° Módulo  
**Notebook principal:** `Aula_6_Integracao_Outputs_IA_SBC_Laboratório_atualizado.ipynb`

---

## 📌 Visão Geral

Sistema especialista completo para monitoramento de qualidade em impressão 3D.
Integra **Visão Computacional (Azure Custom Vision)**, **Classificação de Vibração (SVM)**,
**Detecção de Anomalia (PCA)** e **Contexto Operacional (CSV)** em um **Motor de Inferência
baseado em regras** com cadeia de encadeamento para frente (*forward-chaining*).

O sistema avalia cada peça impressa e decide entre 5 ações:
**GO · MONITORAR · AJUSTAR · INVESTIGAR · NOGO**

---

## 🗺️ Pipeline Completo

```
 [Imagem da peça]  ──▶  Azure Custom Vision API  ──▶  { tag, probabilidade }
                                                                │
 [Dados de vibração] ──▶  SVM (.joblib)           ──▶  { classe: defected/no_defected }
                                                                │
 [Dados de vibração] ──▶  PCA (Aula 8)            ──▶  { anomalia: True/False, MSE }
                                                                │
 [Contexto CSV]      ──▶  Criticidade + Prazo     ──▶  { criticidade, prazo_horas }
                                                                │
                                                     grounding()  ← números → símbolos
                                                                │
                                                  [ fatos simbólicos na WM ]
                                                                │
                                                   Motor de Inferência (21 regras)
                                                                │
                                                  decisao(GO / MONITORAR / AJUSTAR /
                                                          INVESTIGAR / NOGO)
                                                        + por_que() explicável
                                                                │
                                                   Score de Risco (pesos ponderados)
                                                                │
                                                   Tabela comparativa Motor × Score
```

---

## 🧩 Módulos Implementados

### 1. Azure Custom Vision API (Visão Computacional)

- **Endpoint:** Azure Custom Vision v3.0 — Iteration `Aula3`
- **Classes treinadas (5):**
  | Classe | Descrição |
  |---|---|
  | `leg_broken` | Fratura estrutural (perna/suporte quebrado) |
  | `bed_no_stick` | Falha de adesão à mesa |
  | `no_support` | Suporte de impressão ausente |
  | `no_bottom` | Base/fundo inexistente |
  | `no_defected` | Peça sem defeito |
- Recebe imagem binária via HTTP POST, retorna tag + probabilidade

### 2. SVM de Vibração

- **Modelo:** `svm_model.joblib` + `scaler.joblib`
- **Entrada:** 7 features estatísticas extraídas de `vibration_features.csv`
- **Saída:** `defected` ou `no_defected`
- Simula leitura de sensor sorteando uma linha do CSV

### 3. PCA — Detecção de Drift (Aula 8)

- **Treinamento inline** a partir de `vibration_features.csv` (7 amostras normais)
- **Componentes:** 5 (limitado a `max_comp - 2` para evitar reconstrução perfeita)
- **Threshold (MSE):** 0.305605 (baseado em `mean + 2*std` das amostras normais)
- **Lógica:** Se o erro de reconstrução > threshold → drift detectado (🔴)
- **Artefatos exportados:** `modelos_exportados/reducao_dimensionalidade/`
  - `pca_reference.joblib` — modelo PCA treinado
  - `scaler_features.joblib` — StandardScaler
  - `reducao_config.joblib` — configuração (threshold, n_components)

### 4. Contexto Operacional (CSV)

- **Arquivo:** `contexto_pecas - contexto_pecas.csv`
- **17 peças** com IDs correspondentes aos nomes das imagens
- **Campos:** `peca_id`, `criticidade` (alta/media/baixa), `prazo_horas`

### 5. Grounding (Números → Símbolos)

Três funções de grounding convertem outputs numéricos em fatos simbólicos:

| Função | Entrada | Saída (exemplos) |
|---|---|---|
| `grounding_azure()` | tag + prob | `evento_visao(leg_broken)`, `confianca(alta)` |
| `grounding_vibracao()` | classe SVM | `vibracao(normal)` ou `vibracao(erro)` |
| `grounding_pca()` | anomalia + MSE | `anomalia_pca(detectada)` ou `anomalia_pca(normal)` |

**Limiares de confiança:**
- `prob ≥ 0.80` → `confianca(alta)`
- `prob < 0.80` → `confianca(baixa)`

### 6. Motor de Inferência (*Forward-Chaining*)

- **Classe `Rule`:** dataclass com `nome`, `condicoes`, `acao`, `prioridade`, `justificativa`
- **Classe `WorkingMemory`:** set de fatos com `add()` e `contains()`
- **Classe `InferenceEngine`:** ciclos de match → resolve → fire até agenda vazia
- **Explainability:** método `por_que(fato)` retorna árvore JSON de raciocínio

### 7. Score de Risco

Soma ponderada de fatos presentes na WM → 5 faixas de decisão:

| Faixa | Score | Ação |
|---|---|---|
| 🟢 | < 30 | GO |
| 🟡 | 30–49 | MONITORAR |
| 🟠 | 50–69 | AJUSTAR |
| 🔴 | 70–84 | INVESTIGAR |
| ⛔ | ≥ 85 | NOGO |

**Pesos configurados:**

| Fato | Peso |
|---|---|
| `evento_visao(leg_broken)` | 45 |
| `evento_visao(bed_no_stick)` | 30 |
| `evento_visao(no_support)` | 35 |
| `evento_visao(no_bottom)` | 40 |
| `vibracao(erro)` | 25 |
| `confianca(alta)` | 15 |
| `criticidade(alta)` | 20 |
| `criticidade(media)` | 10 |
| `prazo(curto)` | 10 |
| `anomalia_pca(detectada)` | 30 |

---

## 📏 Base de Regras — 21 Regras

### Camada 1 — Interpretação dos sensores
| Regra | Condição | Ação |
|---|---|---|
| R01 | `vibracao(erro)` | `risco_mecanico(alto)` |
| R02 | `confianca(alta)` | `sinal_visual_confiavel(sim)` |
| R03 | `vibracao(normal)` | `risco_mecanico(baixo)` |
| R19 | `anomalia_pca(detectada)` | `drift_detectado(sim)` |

### Camada 2 — Decisões base
| Regra | Condição | Ação |
|---|---|---|
| R04 | `risco_mecanico(alto)` + `criticidade(alta)` | `decisao(NOGO)` |
| R05 | `risco_mecanico(alto)` + `criticidade(media)` | `decisao(INVESTIGAR)` |
| R06 | `evento_visao(no_defected)` + `vibracao(normal)` | `decisao(GO)` |

### Classe A — leg_broken
| Regra | Condição | Ação |
|---|---|---|
| R07 | `evento_visao(leg_broken)` + `sinal_visual_confiavel(sim)` | `suporte_quebrado(confirmado)` |
| R08 | `suporte_quebrado(confirmado)` + `criticidade(alta)` | `decisao(NOGO)` |
| R09 | `evento_visao(leg_broken)` + `confianca(baixa)` + `vibracao(erro)` | `decisao(INVESTIGAR)` |

### Classe B — bed_no_stick
| Regra | Condição | Ação |
|---|---|---|
| R10 | `evento_visao(bed_no_stick)` + `sinal_visual_confiavel(sim)` | `sem_fixacao(confirmado)` |
| R11 | `sem_fixacao(confirmado)` + `criticidade(alta)` | `decisao(NOGO)` |
| R12 | `sem_fixacao(confirmado)` + `criticidade(media)` + `prazo(curto)` | `decisao(AJUSTAR)` |

### Classe C — no_support
| Regra | Condição | Ação |
|---|---|---|
| R13 | `evento_visao(no_support)` + `sinal_visual_confiavel(sim)` | `suporte_ausente(confirmado)` |
| R14 | `suporte_ausente(confirmado)` + `criticidade(alta)` | `decisao(NOGO)` |
| R15 | `suporte_ausente(confirmado)` + `criticidade(media)` + `prazo(longo)` | `decisao(INVESTIGAR)` |

### Classe D — no_bottom
| Regra | Condição | Ação |
|---|---|---|
| R16 | `evento_visao(no_bottom)` + `sinal_visual_confiavel(sim)` | `base_inexistente(confirmado)` |
| R17 | `base_inexistente(confirmado)` + `criticidade(alta)` | `decisao(NOGO)` |
| R18 | `base_inexistente(confirmado)` + `criticidade(baixa)` + `prazo(longo)` | `decisao(MONITORAR)` |

### PCA — Detecção de Drift
| Regra | Condição | Ação |
|---|---|---|
| R19 | `anomalia_pca(detectada)` | `drift_detectado(sim)` |
| R20 | `drift_detectado(sim)` + `vibracao(normal)` + `criticidade(alta)` | `decisao(INVESTIGAR)` |
| R21 | `drift_detectado(sim)` + `vibracao(normal)` + `criticidade(baixa)` | `decisao(MONITORAR)` |

> **R20 e R21** capturam alerta precoce: PCA detecta degradação gradual enquanto SVM ainda classifica como normal.

### Cobertura de Ações
| Ação | Regras |
|---|---|
| **GO** | R06 |
| **MONITORAR** | R18, R21 |
| **AJUSTAR** | R12 |
| **INVESTIGAR** | R05, R09, R15, R20 |
| **NOGO** | R04, R08, R11, R14, R17 |

---

## 🧪 Testes Realizados — 17 Peças

Todas as 17 peças do CSV foram processadas pelo pipeline completo (Azure API + SVM + PCA + Contexto):

| ID | Azure (tag) | Prob | Conf | SVM | PCA | Crit | Prazo |
|---|---|---|---|---|---|---|---|
| bed_not_stick_0 | bed_no_stick | 0.88 | alta | no_defected | 🔴 drift | alta | 38h |
| bed_not_stick_1 | bed_no_stick | 0.44 | baixa | no_defected | 🔴 drift | alta | 52h |
| bed_not_stick_10 | bed_no_stick | 0.37 | baixa | no_defected | 🔴 drift | baixa | 24h |
| leg_broken_118 | no_bottom | 0.61 | baixa | no_defected | 🔴 drift | baixa | 68h |
| leg_broken_119 | no_bottom | 0.51 | baixa | no_defected | 🔴 drift | alta | 55h |
| leg_broken_120 | leg_broken | 0.35 | baixa | no_defected | 🔴 drift | media | 29h |
| no_bottom_4 | no_bottom | 0.56 | baixa | no_defected | 🔴 drift | baixa | 33h |
| no_bottom_5 | no_bottom | 0.44 | baixa | no_defected | 🔴 drift | media | 18h |
| no_bottom_6 | no_bottom | 0.42 | baixa | no_defected | 🔴 drift | baixa | 24h |
| no_support_234 | no_support | 0.97 | alta | no_defected | 🟢 ok | baixa | 22h |
| no_support_235 | no_support | 0.98 | alta | no_defected | 🔴 drift | media | 15h |
| no_support_236 | no_support | 0.99 | alta | no_defected | 🔴 drift | alta | 27h |
| no_support_34 | no_support | 1.00 | alta | no_defected | 🔴 drift | media | 63h |
| scratch_3_4 | no_defected | 0.90 | alta | no_defected | 🔴 drift | baixa | 40h |
| scratch_3_5 | no_defected | 0.75 | baixa | no_defected | 🔴 drift | media | 30h |
| scratch_4_3 | no_defected | 0.98 | alta | no_defected | 🔴 drift | alta | 45h |
| scratch_4_4 | no_defected | 0.95 | alta | no_defected | 🔴 drift | media | 20h |

### Exemplo de Raciocínio — `bed_not_stick_0`

```
Ciclo 1: FIRE R19 → drift_detectado(sim)
Ciclo 2: FIRE R20 → decisao(INVESTIGAR)
Ciclo 3: FIRE R02 → sinal_visual_confiavel(sim)
Ciclo 4: FIRE R10 → sem_fixacao(confirmado)
Ciclo 5: FIRE R11 → decisao(NOGO)
Ciclo 6: FIRE R03 → risco_mecanico(baixo)
Ciclo 7: agenda vazia — motor para.
```

**`por_que(decisao(NOGO))`:**
```
decisao(NOGO)
  └─ R11: Falha de adesão em peça crítica: interromper produção.
       ├─ sem_fixacao(confirmado)
       │    └─ R10: Azure detectou falha de adesão com confiança alta.
       │         ├─ evento_visao(bed_no_stick) ← fato inicial (sensor)
       │         └─ sinal_visual_confiavel(sim)
       │              └─ R02: Confiança alta: predição do Azure é confiável.
       │                   └─ confianca(alta) ← fato inicial (sensor)
       └─ criticidade(alta) ← fato inicial (contexto)
```

---

## 📂 Estrutura de Arquivos

```
3° Modulo sistemas baseados em conhecimento/
│
├── Aula_6_Integracao_Outputs_IA_SBC_Laboratório_atualizado.ipynb  ← Notebook principal
├── Escopo_Projeto_Final.ipynb                                      ← Escopo/guia do projeto
│
├── vibration_features.csv              ← Dataset de vibração (features extraídas)
├── contexto_pecas - contexto_pecas.csv ← Contexto operacional (17 peças)
├── svm_model.joblib                    ← Modelo SVM treinado
├── scaler.joblib                       ← Scaler para SVM
│
├── archive/                            ← Imagens das peças
│   ├── defected/                       ← Peças com defeito
│   └── no_defected/                    ← Peças sem defeito
│
├── modelos_exportados/
│   └── reducao_dimensionalidade/       ← Artefatos PCA exportados
│       ├── pca_reference.joblib
│       ├── scaler_features.joblib
│       └── reducao_config.joblib
│
└── README.md                           ← Este arquivo
```

---

## ⚙️ Dependências

```
pandas
numpy
scikit-learn
joblib
requests
Pillow
IPython
```

---

## 🚀 Como Executar

1. Abrir `Aula_6_Integracao_Outputs_IA_SBC_Laboratório_atualizado.ipynb` no VS Code ou Jupyter
2. Executar as células **na ordem** (de cima para baixo):
   - **Célula 3:** Importações
   - **Célula 5:** Configuração Azure + função `chamar_azure()`
   - **Célula 7:** Classes do Motor de Inferência (Rule, WorkingMemory, InferenceEngine)
   - **Célula 10:** Upload vibração CSV
   - **Célula 11:** Upload modelos SVM
   - **Célula 12:** Upload contexto CSV
   - **Célula 13:** Funções de vibração (`sortear_leitura_vibracao`, `chamar_svm_vibracao`)
   - **Célula 15:** Módulo PCA (treinamento + exportação)
   - **Célula 17:** Funções de Grounding (Azure + SVM + PCA + `construir_fatos`)
   - **Célula 19:** Mapeamento de imagens (17 peças)
   - **Célula 20:** Geração de fatos reais (chama Azure API para todas as imagens)
   - **Célula 22:** BASE_REGRAS (21 regras)
   - **Célula 24:** Teste do motor — caso 1
   - **Célula 25:** Teste do motor — caso 2
   - **Célula 27:** Demo do Score de Risco
   - **Célula 29:** Tabela comparativa Motor × Score

> ⚠️ A célula 20 faz chamadas reais à API do Azure — pode levar ~20s para 17 imagens.

---

## 🔧 Problemas Resolvidos Durante o Desenvolvimento

| Problema | Causa | Solução |
|---|---|---|
| SVM nunca produzia `vibracao(erro)` | Classes do SVM são `defected`/`no_defected`, não `yes`/`no` | Grounding adaptado com `_CLASSES_DEFEITO = {'yes', 'defected'}` |
| PCA marcava 100% como drift | 7 amostras + 7 componentes = reconstrução perfeita (threshold=0) | Limitado a `max(1, max_comp - 2)` = 5 componentes; threshold = 0.305605 |
| `NameError: sortear_leitura_vibracao` | Células executadas fora de ordem | Documentação da ordem correta de execução |
| Acento `média` no CSV | Inconsistência entre CSV e código | Normalizado para `media` (sem acento) |

---

## 📊 Resultados

- **17 peças** testadas com dados reais (imagens + vibração + contexto)
- **4 módulos de IA** integrados ao motor simbólico
- **21 regras** cobrindo todas as 5 classes do Azure + PCA
- **5 ações** possíveis com explicabilidade via `por_que()`
- **Score de risco** ponderado com 10 fatores
- **Tabela comparativa** Motor × Score demonstrando divergências e complementaridade
