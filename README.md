# LexBank

Plataforma de diagnóstico bancário para análise de boletos, linha digitável, código de barras e arquivos CNAB. Aplicativo estático de página única (`index.html`), sem servidor — tudo roda no navegador, com persistência local (localStorage) de casos, templates e mapas.

## Identidade
Nome da plataforma: **LexBank**

## Módulos

**Serviços principais** (menu de cima):
- Decodificação — decompõe uma linha digitável/código de barras/linha de arrecadação em seus campos de negócio.
- Comparador — compara duas linhas dígito a dígito e nomeia o campo divergente.
- Laudo Automático — diagnóstico completo: consistência matemática, divergência campo a campo, causa provável, casos parecidos, impressão/PDF.
- Gerador / Validador — monta uma linha digitável a partir do cadastro, aplicando as fórmulas públicas FEBRABAN; também decompõe uma linha colada por um template configurável.
- Análise Guiada — resposta rápida a partir de só a linha do Sankhya (banco opcional), sem precisar do laudo formal.

**Demais serviços** (gaveta lateral):
- CNAB Explorer — leitura posicional de arquivos CNAB 240/400 (formato, tipo de registro, segmento, busca por termo/posição) e decodificação campo a campo dos 8 bancos suportados, com resumo de negócio da remessa para Bradesco.
- Homologador de Remessa — simula, sem gravar nada, como o banco reagiria a um arquivo de remessa CNAB 240/400: aceito, aceito com ressalvas, rejeição provável ou rejeitado por título, com o campo e a regra exatos.
- Regras Bancárias / Motor de Diagnóstico — catálogo de regras versionadas por banco/produto (em breve — ainda sem conteúdo cadastrado pra maioria dos bancos).
- Biblioteca Oficial — referências e fontes documentais usadas no LexBank.
- Mapas Oficiais CNAB — mapa de campos por posição, banco e tipo de registro.
- Casos — histórico local dos laudos e diagnósticos já gerados neste navegador.
- Conhecendo o LexBank — guia de uso embutido no app (versão completa em `guia.html`).

## Cobertura por banco

Os 8 bancos suportados (Bradesco, Itaú, Santander, Banco do Brasil, Caixa, Sicoob, Sicredi e Safra) têm o Campo Livre da linha digitável mapeado por completo, a partir de manual oficial ou fonte secundária citada no selo de confiança de cada um (Itaú e BB com múltiplas variantes, escolhidas automaticamente a partir da própria linha). Além disso:

- **Nosso Número impresso** (dígito verificador calculado, que não está na linha/código de barras): Bradesco, Banco do Brasil, Caixa.
- **DV embutido no Campo Livre, autoconferido contra a própria fórmula**: Itaú (DV do Nosso Número, DV Agência/Conta) e Caixa (DV do Beneficiário, DV do Campo Livre). Sicredi e a variante Itaú de 15 posições ficam de fora por não terem vetor de teste oficial confirmado na fonte usada; Santander, Sicoob e Safra não têm fórmula documentada nessa fonte.
- **Homologador de Remessa (CNAB 240/400)**: layout transcrito de biblioteca open-source (eduardokum/laravel-boleto, MIT) cross-validado contra os manuais/fontes já usados no restante do app, cobrindo os 8 bancos (Safra só em CNAB 400 — não existe layout de cobrança 240 publicado). Não inclui código de rejeição do banco, cálculo de DV do Nosso Número nem retorno simulado, nem o "ambiente" compartilhado de duplicidade entre remessas (isso exige estado compartilhado que uma página estática não tem).
- **CNAB Explorer, decodificação campo a campo**: os 8 bancos, em CNAB 240 e/ou 400 conforme o layout disponível — Bradesco 240 a partir do manual oficial (maior fidelidade), os demais com o mesmo layout transcrito usado no Homologador.

## Limitações conscientes (por design)

O LexBank não presume uma regra bancária sem que ela esteja documentada — assim como um analista não deveria "chutar" a composição de um banco sem consultar o manual oficial. Onde não há template, fórmula de DV ou mapa de campo confirmado, o resultado diz isso explicitamente ("não mapeado", "sem vetor de teste oficial", "layout transcrito, confirme no manual") em vez de arriscar uma conclusão incorreta. O mesmo vale para o Homologador de Remessa: é uma simulação local, não substitui o teste no próprio canal do banco antes de produção.
