# Análise de acurácia de anotações INCEpTION

Este repositório contém um pacote de comparação de anotações de entidades exportadas do INCEpTION.

O conteúdo versionado está concentrado em:

```text
Extração Candidatos/entrega_csv_comparativo/
```

## O que há no repositório

```text
Extração Candidatos/entrega_csv_comparativo/
├── codigo/
│   └── analise_acuracia.py
├── csv/
│   ├── detalhes.csv
│   ├── resumo.csv
│   ├── ignorados.csv
│   ├── contagem_rotulos.csv
│   └── contas.csv
├── documentacao_metodologia.md
└── documentacao_metodologia.docx
```

## O que o código faz

O código em Python pega anotações exportadas do INCEpTION, extrai tokens e rótulos de entidade, alinha cada anotação de candidato com uma anotação de referência e calcula valores comparativos de acurácia.

Em resumo:

```text
anotações exportadas -> tokens e rótulos -> comparação referência/candidato -> CSVs comparativos
```

A comparação considera apenas a camada de entidade. Pronunciamentos, relações e outras camadas não entram no cálculo.

## Lógica da comparação

O código trabalha com três grupos de informação:

- anotações de referência;
- anotações dos candidatos;
- arquivo de contas, usado para relacionar usuário técnico, nome da pessoa candidata, horário e textos atribuídos.

Antes de comparar, os nomes dos documentos são normalizados. Por exemplo, um nome como `rce-39-teste` é tratado como `rce-39.txt`, para evitar que pequenas variações no nome impeçam o cruzamento correto.

Depois disso, o código localiza anotações no padrão:

```text
annotation/{documento}/{usuario}.tsv
```

De cada TSV, ele extrai:

```text
token textual
rótulo de entidade
```

Ausências de entidade, marcadas como `_` ou `*`, são convertidas para `O`. Sufixos técnicos de indexação, como `[1]`, são removidos para que o cálculo compare o tipo da entidade, não o identificador interno do arquivo.

## Validação antes do cálculo

Antes de comparar rótulos, o código verifica se candidato e referência possuem a mesma sequência de tokens.

Ele confere:

- se a quantidade de tokens é igual;
- se cada token aparece na mesma posição nos dois arquivos.

Se houver divergência, aquela comparação é descartada e o motivo aparece em `csv/ignorados.csv`.

Essa validação é importante porque a comparação é feita por posição. Sem o alinhamento token a token, um rótulo poderia ser comparado com o trecho errado do texto.

## Métricas calculadas

### Acurácia geral

Mede a proporção de tokens em que candidato e referência têm exatamente o mesmo rótulo.

```text
tokens corretos / total de tokens
```

Essa métrica inclui também tokens em que os dois lados marcaram `O`.

### Acurácia em entidades

Mede o acerto apenas nos tokens em que a referência marcou alguma entidade.

```text
acertos nos tokens com entidade na referência / total de tokens com entidade na referência
```

Essa métrica reduz o peso dos tokens sem entidade.

### Acurácia balanceada

Mede o acerto nos tokens em que a referência ou o candidato marcou alguma entidade.

```text
acertos onde referência ou candidato marcou entidade / tokens onde referência ou candidato marcou entidade
```

Ela ignora apenas os casos `O/O` na comparação entre uma referência e um candidato.

## CSVs gerados

### `csv/detalhes.csv`

Arquivo mais granular. Cada linha representa uma comparação entre:

```text
documento + referência + candidato
```

Inclui totais de tokens, acertos, acurácia geral, acurácia em entidades, acurácia balanceada e quantidade de tokens `O/O` ignorados na métrica balanceada.

### `csv/resumo.csv`

Consolida os valores por:

```text
candidato + documento
```

Quando há mais de uma referência disponível, o arquivo registra a média das comparações contra essas referências.

### `csv/ignorados.csv`

Lista comparações descartadas e o motivo do descarte.

Exemplos de motivos:

- usuário técnico ou de sistema;
- documento sem referência;
- TSV sem tokens anotáveis;
- divergência na quantidade de tokens;
- divergência no texto dos tokens.

### `csv/contagem_rotulos.csv`

Registra a quantidade de ocorrências de cada rótulo usado por candidato e documento.

### `csv/contas.csv`

Preserva a ligação entre usuário técnico, nome da pessoa candidata, horário e documentos normalizados.

## Documentação metodológica

A explicação completa do rastreio está disponível em dois formatos:

```text
Extração Candidatos/entrega_csv_comparativo/documentacao_metodologia.md
Extração Candidatos/entrega_csv_comparativo/documentacao_metodologia.docx
```

Esses arquivos explicam como os dados são extraídos, transformados, comparados e registrados nos CSVs.

## Observação sobre reexecução

O repositório contém o código de comparação e os CSVs resultantes.

Os arquivos brutos de origem não fazem parte do conteúdo versionado. Para reexecutar a análise, é necessário disponibilizar localmente as entradas esperadas pelo código nos caminhos definidos em `codigo/analise_acuracia.py`.
