# LexBank — Estrutura para migração a banco de dados + API

Este documento mapeia o que existe **hoje** no `index.html` (dados fixos em
JavaScript + estado salvo no `localStorage` do navegador) para um schema
relacional e uma API REST, servindo de base para evoluir o LexBank de
arquivo único client-side para uma aplicação com backend real.

**O app continua funcionando exatamente como está** — nada neste documento
foi aplicado ao código. É só o mapa pra quem for construir a próxima versão.

---

## 1. Visão geral da migração

| Hoje (index.html) | Depois |
|---|---|
| Dados fixos em `const` (JS) | Tabelas de referência (seed data, somente leitura pela aplicação) |
| Estado do usuário em `localStorage` | Tabelas por usuário/empresa, servidas por API |
| "Login" cosmético (grava e-mail no localStorage) | Autenticação real (tabela `users` + sessão/token) |
| Funções JS que leem/gravam `localStorage` | Endpoints REST que o front-end passa a chamar |

---

## 2. Tabelas de referência (hoje hardcoded, viram seed data)

Essas tabelas **não mudam por usuário** — são o conhecimento bancário que o
LexBank já tem cadastrado. Hoje vivem como `const` no topo do `<script>`.

### 2.1 `banks`
Origem: `const banks={...}` (linha ~590).

| coluna | tipo | obs |
|---|---|---|
| `codigo` | varchar(3) PK | "001", "237", "341"... (código FEBRABAN) |
| `nome` | varchar | "Banco do Brasil", "Bradesco"... |

8 linhas hoje: 001 BB, 033 Santander, 104 Caixa, 237 Bradesco, 341 Itaú, 422 Safra, 748 Sicredi, 756 Sicoob.

### 2.2 `bank_field_templates`
Origem: `LB_DEFAULT_BANK_TEMPLATES` + `LB_BANK_VARIANTS` + `LB_TEMPLATE_CONFIDENCE`.
Template do **Campo Livre** (25 posições) da linha digitável/código de barras de cada banco.

| coluna | tipo | obs |
|---|---|---|
| `id` | serial PK | |
| `banco_codigo` | varchar(3) FK → banks.codigo | |
| `variante` | varchar, nullable | `null` = template padrão; ou `"convenio_7"`, `"convenio_6_nn17"`, `"especial_15"` etc. (BB e Itaú têm mais de um layout, escolhido automaticamente pelo conteúdo da própria linha) |
| `regra_selecao_variante` | text, nullable | descrição de como decidir qual variante usar (ex.: "convênio 7 posições se zeros nas posições 1–6") — hoje é lógica em `getBankTemplate()`, teria que virar regra configurável ou continuar em código |
| `template` | varchar(25) | string de tokens, ex.: `"CCCCCC" + "NNNNN" + "AAAA" + "VVVVVVVV" + "KK"` — cada letra é um campo (ver seção 4) |
| `confianca_nivel` | varchar | `"verified"` ou `"partial"` |
| `confianca_nota` | text | fonte documental exata (manual oficial, cópia espelhada, lib transcrita etc.) |

### 2.3 `cnab_specs`
Origem: `LB_HOM_SPECS_RAW` (Homologador de Remessa / CNAB Explorer).
Um registro por combinação banco+formato (ex.: `"001_240"`).

| coluna | tipo | obs |
|---|---|---|
| `id` | serial PK | |
| `banco_codigo` | varchar(3) FK → banks.codigo | |
| `formato` | int | 240 ou 400 |
| `nome_layout` | varchar | ex.: "Cobrança CNAB 240" |
| `tamanho_registro` | int | 240 ou 400 |
| `ocorrencias_remessa` | text, nullable | observações sobre a remessa |
| `fonte_tipo` | varchar | `"manual_oficial"` ou `"transcricao"` |
| `fonte_origem` | text | ex.: "eduardokum/laravel-boleto, src/.../Bb.php" ou citação do manual oficial |
| `fonte_commit` | varchar, nullable | hash do commit, quando a fonte é uma lib open-source |
| `fonte_licenca` | varchar, nullable | ex.: "MIT" |

### 2.4 `cnab_spec_fields`
Origem: o objeto `registros` dentro de cada entrada de `LB_HOM_SPECS_RAW`,
e também `LB_CNAB240_FIELD_MAPS` (mapa próprio e mais detalhado do Bradesco
240, tirado do manual oficial — mesma estrutura, pode cair na mesma tabela).

| coluna | tipo | obs |
|---|---|---|
| `id` | serial PK | |
| `spec_id` | int FK → cnab_specs.id | |
| `registro_nome` | varchar | `header_arquivo`, `header_lote`, `segmento_p`, `segmento_q`, `segmento_r`, `trailer_lote`, `trailer_arquivo` etc. |
| `nome_campo` | varchar | ex.: `"convenio"`, `"nosso_numero"`, `"fixo_42"` |
| `posicao_inicio` | int | |
| `posicao_fim` | int | |
| `tipo` | varchar(1) | `N` (numérico), `A` (alfanumérico), `D` (data), `V2` (valor com 2 decimais) |
| `valor_fixo` | varchar, nullable | quando a posição é um literal fixo exigido pelo layout |
| `papel` | varchar, nullable | nome semântico pro motor de diagnóstico (`convenio`, `carteira`, `nosso_numero`...) — usado pra rotular o campo em português na tela |
| `casas_decimais` | int, nullable | só quando `tipo = 'V2'` ou similar |
| `formato_data` | varchar, nullable | quando `tipo = 'D'` |

### 2.5 `engine_rules_reference`
Origem: `const rules=[...]` (linha ~600) — tabela de referência da aba "Motor de Diagnóstico".

| coluna | tipo |
|---|---|
| `id` | serial PK |
| `banco_codigo` | varchar(3) FK → banks.codigo |
| `produto` | varchar |
| `formato` | varchar |
| `observacao` | text |
| `fonte` | varchar |

### 2.6 `reference_docs`
Origem: `const docs=[...]` (linha ~610) — links de referência mostrados na aba "Ajuda".

| coluna | tipo |
|---|---|
| `id` | serial PK |
| `entidade` | varchar | ex.: "Bradesco", "Sankhya" |
| `titulo` | varchar |
| `url` | text |

---

## 3. Tabelas por usuário/empresa (hoje em `localStorage`, nunca saem do navegador)

Essas tabelas **mudam por usuário** (ou por empresa, se o uso for multi-tenant
— decisão de produto a tomar). Hoje cada consultor tem o próprio
`localStorage`, sem compartilhamento entre máquinas nem backup.

### 3.1 `users`
Hoje não existe de verdade — é só `localStorage.lexbank_auth_email`, sem
senha nem validação server-side (`lbEmailAllowed()` só confere o domínio
`@sankhya.com.br` por regex, no próprio navegador).

| coluna | tipo |
|---|---|
| `id` | serial/uuid PK |
| `email` | varchar unique |
| `nome` | varchar, nullable (hoje derivado do e-mail via `lbFirstNameFromEmail()`) |
| `criado_em` | timestamp |
| `tema_preferido` | varchar, nullable | hoje é `localStorage.lexbank_theme` ("light"/"dark") |

### 3.2 `cases`
Origem: `localStorage.lexbank_cases` — histórico da aba "Casos", já
alimentado hoje por 3 fontes: Laudo Automático, Homologador de Remessa e
CNAB Explorer (`lbSaveCase()`).

| coluna | tipo | obs |
|---|---|---|
| `id` | serial PK | |
| `user_id` | FK → users.id | |
| `criado_em` | timestamp | hoje é string formatada `toLocaleString("pt-BR")` — vira timestamp de verdade |
| `origem` | varchar | `"Laudo Automático"`, `"Homologador de Remessa"`, `"CNAB Explorer"`, `"Diagnóstico guiado"` |
| `banco` | varchar | nome do banco |
| `referencia` | varchar, nullable | nº do chamado / layout |
| `status` | text | hoje já vem com o ícone/HTML embutido — na migração, separar em `status_code` (enum: ok/warn/bad) + `status_texto` |
| `campo` | varchar, nullable | nome do campo que divergiu (usado hoje pra achar "Casos parecidos") |

### 3.3 `custom_field_templates`
Origem: `localStorage.lexbank_templates` — templates de Campo Livre que o
usuário editou manualmente (sobrescreve o default da tabela 2.2 pra aquele
usuário/empresa).

| coluna | tipo |
|---|---|
| `id` | serial PK |
| `user_id` (ou `empresa_id`) | FK |
| `banco_codigo` | FK → banks.codigo |
| `template` | varchar(25) |
| `atualizado_em` | timestamp |

### 3.4 `custom_engine_rules`
Origem: `localStorage.lexbank_engine_rules` — regras que o usuário cadastra
manualmente na aba "Motor de Diagnóstico" (`saveRule()`).

| coluna | tipo |
|---|---|
| `id` | serial PK |
| `user_id` (ou `empresa_id`) | FK |
| `banco_codigo` | FK → banks.codigo |
| `produto` | varchar |
| `formato` | varchar |
| `modalidade` | varchar |
| `versao` | varchar |
| `fonte` | varchar |
| `regra` | text |
| `criado_em` | timestamp |

### 3.5 `custom_field_maps`
Origem: `localStorage.lexbank_maps` — "Mapas Oficiais" cadastrados manualmente
(`abiAddMap()`), hoje agrupados por banco + `"formato / segmento"`.

| coluna | tipo |
|---|---|
| `id` | serial PK |
| `user_id` (ou `empresa_id`) | FK |
| `banco_codigo` | FK → banks.codigo |
| `formato` | varchar |
| `segmento` | varchar |
| `posicao` | varchar | hoje é texto livre tipo `"33-52"` — considerar separar em `posicao_inicio`/`posicao_fim` inteiros |
| `descricao` | varchar |
| `tipo` | varchar, nullable |
| `valor_default` | varchar, nullable |
| `regra` | text, nullable |
| `fonte` | text, nullable |

---

## 4. Tokens do template de Campo Livre (dicionário usado pela tabela 2.2)

A coluna `bank_field_templates.template` usa uma letra por posição,
decodificada hoje em `lbTokenLabel()`:

| token | significado |
|---|---|
| `N` | Nosso Número |
| `D` | DV do Nosso Número |
| `C` | Convênio |
| `A` | Agência |
| `V` | Conta |
| `K` | Carteira |
| `G` | DV Agência/Conta |
| `Z` | DV do Campo Livre (mód. 10) |
| `O` | DV do Campo Livre (mód. 11) |
| `B` | Beneficiário / Código do Cedente |
| `E` | DV do Beneficiário |
| `S` | Seu Número |
| `X` | DAC do Campo Livre |
| `M` | Modalidade |
| `P` | Parcela |
| `T` | Tipo de Cobrança |
| `U` | Posto |
| `I` | Indicador de Valor |
| dígito literal (`0`-`9`) | posição fixa do layout |

Na migração, essa tabela de tokens também pode virar uma tabela de apoio
(`field_template_tokens`) se o time quiser editar os rótulos sem mexer em código.

---

## 5. Endpoints sugeridos (REST)

Mapeamento direto: cada função JS que hoje lê/grava `localStorage` vira uma
chamada HTTP. Agrupado por recurso:

```
# Referência (somente leitura pela aplicação; escrita é administrativa)
GET  /api/banks
GET  /api/banks/:codigo/field-template          # bank_field_templates (já resolve a variante certa)
GET  /api/cnab-specs?banco=&formato=             # cnab_specs + cnab_spec_fields
GET  /api/engine-rules/reference?banco=          # engine_rules_reference
GET  /api/reference-docs

# Autenticação
POST /api/auth/login                             # substitui lbEmailAllowed() client-side
GET  /api/me

# Histórico de casos (hoje: lbSaveCase, renderHistory, clearHistory)
GET    /api/cases
POST   /api/cases
DELETE /api/cases                                 # "Apagar casos locais"
GET    /api/cases/similar?banco=&campo=           # hoje é lbFindSimilarCases() rodando sobre o próprio localStorage

# Templates customizados (hoje: getBankTemplates, setBankTemplate)
GET  /api/custom-templates
PUT  /api/custom-templates/:banco_codigo

# Regras do motor (hoje: getEngineRules, saveRule, renderEngineList)
GET    /api/custom-engine-rules
POST   /api/custom-engine-rules

# Mapas oficiais (hoje: getMaps, abiAddMap, abiRestoreMap, renderMaps)
GET    /api/custom-field-maps
POST   /api/custom-field-maps
POST   /api/custom-field-maps/restore-bradesco-default
```

---

## 6. O que fica igual (não precisa de banco)

Toda a **lógica de cálculo** continua fazendo sentido no front-end (ou pode
virar uma camada de serviço no backend, se quiser validar no servidor
também) — não é dado, é comportamento:

- Motor FEBRABAN: `lbMod10`, `lbMod11General`, `lbMod11Campo`, `lbMod11Arrecadacao`, `lbDateToFactor`/`lbFactorToDates`
- Parsers: `lbParseCobranca`, `lbParseBarcode`, `lbParseArrecadacao`, `lbCnabParseLine`, `lbCnabParseLineGenerico`
- Comparador campo a campo: `lbCampoLivreFields`, `lbCompareAllFields`, `lbClassifyDiffs`
- Geração de HTML/relatório, código de barras (SVG), exportação de laudo

Se o objetivo da evolução é também permitir **chamadas às APIs dos bancos**
(como você mencionou antes, pra emissão/validação de boleto), isso é a peça
que realmente exige um backend novo — o cálculo acima roda local, mas uma
chamada de API de banco não pode ser feita direto do navegador do consultor
(CORS + seria preciso guardar segredo de API em servidor, nunca no front-end).
Esse ponto fica à parte deste documento de estrutura de dados.
