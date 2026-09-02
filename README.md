# LexBank

Plataforma de diagnóstico bancário para análise de boletos, linha digitável, código de barras e arquivos CNAB.

## Identidade
Nome da plataforma: **LexBank**

## Módulos
- Diagnóstico (cadastro + linha digitável embutida)
- Comparador de linha digitável (nomeia o campo exato da divergência)
- CNAB Explorer
- Regras Bancárias
- Biblioteca Oficial
- Motor de Diagnóstico (regras versionadas por banco/produto/manual)
- Gerador / Validador de linha digitável
- Laudo Automático de Divergência
- CNAB Inteligente
- Mapas Oficiais CNAB
- Teste da Plataforma (suíte de autoteste do motor)
- Diagnóstico Guiado
- Histórico de Casos

## O que mudou nesta versão (v9)

A v9 substitui o protótipo de identidade/branding (v8) por um motor de diagnóstico
funcional de verdade, aplicando as fórmulas públicas FEBRABAN de linha digitável e
código de barras — o mesmo raciocínio usado para diagnosticar manualmente uma
divergência de linha digitável:

1. **Valida cada linha isoladamente** (dígitos verificadores dos 3 campos, DV geral do
   código de barras, decodifica valor e data de vencimento a partir do fator — incluindo
   a regra vigente desde 22/02/2025, quando o fator de vencimento reiniciou em 1000).
2. **Compara duas linhas posição a posição** e nomeia o campo exato onde a divergência
   ocorre (Banco, Moeda, Campo Livre, DV de cada bloco, DV Geral, Fator de Vencimento
   ou Valor) — em vez de só listar índices numéricos.
3. **Aponta a causa provável**: se a divergência está inteiramente dentro do Campo Livre
   e os DVs da linha do Sankhya conferem, o problema é de composição/cadastro, não de
   fórmula. Se um template de Campo Livre estiver cadastrado para o banco, o laudo indica
   exatamente qual dado (Nosso Número, DV do Nosso Número etc.) corresponde à posição
   divergente.
4. **Cruza os dados do cadastro com a linha**: no Diagnóstico Guiado, o valor e o
   vencimento informados no formulário são comparados com os decodificados da linha,
   sinalizando divergência mesmo quando os dígitos verificadores fecham.
5. Também dá suporte a código de barras isolado (44 dígitos) e a linhas de
   arrecadação/tributos (48 dígitos, padrão FEBRABAN de blocos com módulo 10/11).

Todos os botões da interface original (Gerador/Validador, Laudo Automático, Mapas
Oficiais, Motor de Diagnóstico, CNAB Inteligente, Teste da Plataforma) — que na v8
existiam apenas visualmente, sem função associada — agora estão implementados e
funcionais, com persistência local (localStorage) de casos, regras, templates de
Campo Livre e mapas de CNAB.

### Limitações conscientes (por design)

A plataforma não presume uma regra bancária específica (ex.: qual posição do Campo
Livre é a Carteira de um banco) sem que ela esteja cadastrada — assim como um analista
não deveria "chutar" a composição de um banco sem consultar o manual oficial. Onde não
há template/regra cadastrada, o laudo diz isso explicitamente e orienta cadastrar em
Motor de Diagnóstico / Gerador, em vez de arriscar uma conclusão incorreta.
