# Explicação do código: extração e comparação das anotações

Este documento explica o que o código em Python faz para transformar os arquivos de origem em valores comparativos de acurácia.

A saída considerada aqui para rastreio é o CSV comparativo, especialmente:

```text
csv/detalhes.csv
csv/resumo.csv
```

## Ideia geral

O código em Python pega anotações exportadas do INCEpTION, extrai os tokens e os rótulos de entidade de cada pessoa, alinha candidato e referência pelo texto token a token, compara os rótulos e grava os valores comparativos em CSV.

Em termos simples:

```text
arquivos de anotação -> tokens e rótulos -> comparação referência/candidato -> métricas em CSV
```

## Código em Python: entrada dos dados

O código pega três tipos de origem.

Primeiro, pega o conjunto de referência. Esse conjunto contém as anotações usadas como base de comparação. Nele, as pessoas consideradas referência são `jacques` e `lauana`.

Segundo, pega o conjunto dos candidatos. Esse conjunto contém as anotações feitas pelas pessoas avaliadas.

Terceiro, pega o arquivo de contas. Esse arquivo serve para relacionar usuário, nome da pessoa candidata, horário e textos atribuídos.

A lógica por trás disso é separar três papéis:

- quem é referência;
- quem é candidato;
- como o usuário técnico do sistema vira um nome legível de pessoa candidata.

## Código em Python: normalização dos nomes

Antes de comparar, o código padroniza nomes de documentos e usuários.

Ele transforma nomes como:

```text
rce-39-teste
```

em:

```text
rce-39.txt
```

Também adiciona `.txt` quando a extensão não aparece e remove pequenas variações que atrapalhariam o cruzamento entre os arquivos.

A lógica por trás é evitar que o mesmo texto seja tratado como textos diferentes só por causa de uma diferença pequena no nome.

Também existe uma correção manual:

```text
costajoyce -> Maria Luiza Moreira Macedo
```

Essa correção não muda a anotação. Ela muda apenas o nome exibido nos resultados.

## Código em Python: localização das anotações dentro dos arquivos

Cada arquivo compactado contém várias anotações em TSV.

O código procura anotações no formato:

```text
annotation/{documento}/{usuario}.tsv
```

Quando encontra esse padrão, ele entende:

```text
este usuário anotou este documento
```

A lógica por trás é montar um mapa de disponibilidade:

```text
documento -> usuários que possuem anotação
```

Esse mapa é criado para o conjunto de referência e para o conjunto dos candidatos.

## Código em Python: extração dos tokens e rótulos

Depois de localizar um TSV, o código abre o arquivo e lê linha por linha.

Ele ignora linhas vazias, comentários e linhas que não representam tokens anotáveis.

De cada linha válida, ele extrai duas coisas:

```text
token textual
rótulo de entidade
```

O token textual vem da coluna do token.

O rótulo avaliado vem da coluna de entidade.

Na prática, o código usa apenas a camada de entidade. Outras camadas do TSV não entram na comparação.

A lógica por trás é: este teste avaliou a marcação de entidades, então a comparação precisa olhar somente para a coluna de entidade.

## Código em Python: transformação dos rótulos

O TSV pode trazer ausência de entidade como `_` ou `*`.

O código transforma esses casos em:

```text
O
```

Aqui, `O` significa “sem entidade”.

Se o rótulo aparece com sufixo de indexação, como:

```text
PessoaFisica[1]
```

o código mantém apenas:

```text
PessoaFisica
```

A lógica por trás é deixar os rótulos comparáveis. O que importa para este cálculo é o tipo da entidade, não o marcador técnico de indexação.

## Código em Python: escolha das comparações

Para cada documento anotado por candidatos, o código verifica se o mesmo documento existe no conjunto de referência.

Se o documento não existir no conjunto de referência, ele não é comparado.

Se existir, o código procura as referências `jacques` e `lauana`.

Cada candidato é comparado separadamente contra cada referência disponível.

Isso gera pares como:

```text
candidato X documento X jacques
candidato X documento X lauana
```

A lógica por trás é permitir que a nota do candidato considere mais de uma referência humana, quando as duas existem.

## Código em Python: filtros antes da comparação

O código ignora usuários técnicos, como:

```text
INITIAL_CAS
admin
usuarioteste
```

Esses usuários não representam candidatos reais.

O código também ignora TSVs sem tokens anotáveis.

A lógica por trás é impedir que arquivos técnicos ou vazios entrem no cálculo e distorçam os valores.

## Código em Python: validação do alinhamento

Antes de comparar os rótulos, o código confirma se candidato e referência têm a mesma sequência de tokens.

Primeiro, confere a quantidade:

```text
quantidade de tokens da referência == quantidade de tokens do candidato
```

Depois, confere o texto de cada token na mesma posição:

```text
token 1 da referência == token 1 do candidato
token 2 da referência == token 2 do candidato
...
```

Se houver divergência, a comparação é descartada.

A lógica por trás é fundamental: a comparação é feita por posição. Se os tokens não forem exatamente os mesmos, comparar a posição 15 da referência com a posição 15 do candidato poderia comparar trechos diferentes do texto.

## Código em Python: comparação dos rótulos

Depois que os tokens estão alinhados, o código compara os rótulos posição por posição.

Para cada token, ele olha:

```text
rótulo da referência
rótulo do candidato
```

Se os dois são iguais, conta como acerto.

Se são diferentes, conta como erro.

Exemplo:

```text
Token: João
Referência: PessoaFisica
Candidato: PessoaFisica
Resultado: acerto
```

Outro exemplo:

```text
Token: OAB
Referência: NumeroOAB
Candidato: O
Resultado: erro
```

A lógica por trás é medir se a pessoa candidata marcou o mesmo tipo de entidade que a referência marcou para aquele token.

## Código em Python: acurácia geral

A acurácia geral responde:

```text
em quantos tokens o candidato marcou exatamente igual à referência?
```

A fórmula é:

```text
tokens corretos / total de tokens
```

Ela conta tudo, inclusive tokens em que os dois deixaram `O`.

A lógica por trás é medir concordância total do texto. O cuidado é que essa métrica pode ficar alta quando há muitos tokens sem entidade.

## Código em Python: acurácia em entidades

A acurácia em entidades responde:

```text
quando a referência marcou entidade, o candidato acertou?
```

A fórmula é:

```text
acertos nos tokens em que a referência marcou entidade / total de tokens em que a referência marcou entidade
```

Ela olha apenas posições em que a referência é diferente de `O`.

A lógica por trás é focar nas entidades esperadas pela referência. Essa métrica reduz o efeito dos tokens sem entidade, mas não mede completamente o excesso de marcações feitas pelo candidato onde a referência deixou `O`.

## Código em Python: acurácia balanceada

A acurácia balanceada responde:

```text
nos pontos em que alguém marcou entidade, candidato e referência concordaram?
```

A fórmula é:

```text
acertos nos tokens em que referência ou candidato marcou entidade / tokens em que referência ou candidato marcou entidade
```

Ela ignora apenas os casos `O/O` de cada comparação entre uma referência e um candidato.

Exemplo de token ignorado na balanceada:

```text
Referência: O
Candidato: O
```

Exemplo de token avaliado na balanceada:

```text
Referência: PessoaFisica
Candidato: O
```

Outro exemplo avaliado:

```text
Referência: O
Candidato: PessoaFisica
```

A lógica por trás é não deixar a nota ser dominada por tokens em que não houve decisão real de entidade.

Limite importante: essa remoção é feita par a par. Ela não verifica se todos os candidatos deixaram `O`. Ela verifica apenas a comparação atual:

```text
referência atual X candidato atual
```

## Código em Python: registro dos valores comparativos

Para cada comparação válida, o código grava uma linha com os valores calculados.

Cada linha representa:

```text
documento + referência + candidato
```

Essa linha contém:

- documento;
- referência usada;
- usuário do candidato;
- nome corrigido do candidato;
- horário;
- total de tokens;
- acertos totais;
- acurácia geral;
- tokens em que a referência marcou entidade;
- acertos nesses tokens de entidade;
- acurácia em entidades;
- tokens avaliados pela balanceada;
- tokens `O/O` ignorados;
- acertos na balanceada;
- acurácia balanceada.

Esse é o rastreio principal do cálculo.

A saída é:

```text
csv/detalhes.csv
```

## Código em Python: resumo por candidato e documento

Depois de criar as comparações individuais, o código consolida os valores por:

```text
candidato + documento
```

Se o candidato foi comparado contra duas referências, o código tira a média entre elas.

Exemplo:

```text
valor médio no documento =
(valor contra jacques + valor contra lauana) / 2
```

A lógica por trás é transformar duas comparações de referência em um valor único por candidato e por texto.

A saída é:

```text
csv/resumo.csv
```

## Código em Python: valores descartados

Quando algo não pode ser comparado, o código não apaga silenciosamente.

Ele registra o motivo em:

```text
csv/ignorados.csv
```

Motivos possíveis:

- documento sem referência;
- documento sem `jacques` ou `lauana`;
- usuário técnico;
- TSV sem tokens anotáveis;
- quantidade de tokens divergente;
- texto dos tokens divergente.

A lógica por trás é permitir auditoria: dá para saber não só o que entrou, mas também o que ficou fora e por quê.

## Resultado final considerado aqui

O resultado final deste processo, para fins de rastreio e explicação, são os CSVs:

```text
csv/detalhes.csv
csv/resumo.csv
csv/ignorados.csv
csv/contagem_rotulos.csv
csv/contas.csv
```

O arquivo `detalhes.csv` mostra o cálculo comparativo mais granular.

O arquivo `resumo.csv` mostra a média por candidato e documento.

O arquivo `ignorados.csv` mostra o que não entrou na comparação.

O arquivo `contagem_rotulos.csv` ajuda a rastrear quais rótulos cada candidato usou.

O arquivo `contas.csv` preserva a ligação entre usuário e pessoa candidata.

## O que não entra neste cálculo

Este cálculo não avalia pronunciamento.

Este cálculo não avalia relações.

Este cálculo não avalia span completo por entidade.

Este cálculo não calcula F1 por entidade.

Este cálculo não faz consenso global entre todos os candidatos.

Ele faz uma comparação token a token entre:

```text
uma referência humana
um candidato
um documento
```

Depois consolida essas comparações em CSV.

