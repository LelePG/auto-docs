## Action para um repositório de exemplos

## Formato esperado

O formato esperado para este tipo de repositório é um repositório com várias subpastas, onde cada subpasta possui um arquivo README.md, podendo também ter uma pasta img. A ideia é que todos os arquivos READMEs e pastas img associadas aos exemplos sejam transferidas para o repositório de documentação. Cada pasta deve ser nomeada com dois números e um traço seguido do nome. Um exemplo de árvore pode ser vista abaixo:

```
-- Pasta Principal
    -- 01-exemplo1
        -- README.md
        -- img/
            -- foto1.jpeg
            -- foto2.jpeg
    -- 02-exemplo1
        -- README.md
    -- 03-exemplo3
        -- README.md
    -- 04-exemplo4
        -- README.md
        -- img/
            -- foto1.jpeg
            -- foto3.jpeg
```

Para atingir esse objetivo, uso uma action e a criação de um archetype específico do Hugo no arquivo de destino para fazer formatações e organizações com base no nome do arquivo. O archetype pode ser acessado [clicando aqui](./exemplo.md).

### Explicação do código:

- Nome da Action e quando ela será executada (no caso, quando houver um push):

```bash
name: Check README

on: push
```

- Jobs (ações que serão executadas ).
- Check_readme: nome da ação
- runs-on: onde essa action vai rodar, que máquina vai ser montada pra rodar isso
- steps: passo pra execução da action, que vai usar a segunda versão das actions

```bash
jobs:
    check_readme:
        runs-on: ubuntu-latest
```

- Checkout code: pega para a execução dessa action o commit atual, mas também o último

```bash
- name: Checkout code
    uses: actions/checkout@v2
    with:
    fetch-depth: 2
```

- Install Hugo: instalação do Hugo dentro do repositório

```bash
- name: Install Hugo
    run: |
        wget https://github.com/gohugoio/hugo/releases/download/v0.123.3/hugo_0.123.3_linux-amd64.deb
        sudo dpkg -i hugo_0.123.3_linux-amd64.deb
```

- Config Git: Etapas de configuração do git e do clone do repositório de destino, cuja URL precisa estar especificada dentro do submódulo.

```bash
- name: Config git
    run: |
        git config --global user.name "Auto Docs"
        git config --global user.email "auto-docs@mail.com"
        # Usar o link do repositório de destino na linha abaixo.
        git submodule add https://x-access-token:${{ secrets.TOKEN }}@github.com/LelePG/action-destino.git action-destino
```

- Check for README.md and img directory (aqui é o script que faz a mágina acontecer em si.)

```bash
- name: Checar por README.md e pasta de imagens
    run: |
    caminho_destino="NOME-FINAL-DO-REPOSITORIO"
    while IFS= read -r -d '' file; do
        dir=$(dirname "$file")
        dir=${dir#./}  # remove leading ./

        if [ ! -f "$dir/README.md" ]; then # Se não for encontrado um README nessa pasta, pula a pasta
            echo "No README.md found"
            continue
        fi

        if git diff --quiet HEAD~1 HEAD "$dir/README.md"; then # Se não existirem mudanças no README
            echo "$dir/README.md não foi alterado"
            if [[ -d "./action-destino/content/$caminho_destino/$dir" ]]; then # E já existir uma pasta desse tipo no repositório de destino
                    continue # Pula a pasta
            fi
        fi

        if [[ "$dir" != "." && "$dir" != ".." ]]; then
        cd ./action-destino/
        # Usar o nome onde os arquivos do projeto devem ser usados na documentação
        rm -rf "./content/$caminho_destino/$dir"
        hugo new content -k exercicio "$caminho_destino/$dir/_index.md" # Cria um conteúdo do tipo exercício que precisa ser definido nos modelos do Hugo
        cat "../$file" >> "./content/$caminho_destino/$dir/_index.md"

        echo "Copiando $dir"

        if [[ -d "../$dir/img" ]]; then # Se tiver pasta imagens, copia a pasta imagens
            cp -r "../$dir/img" "./content/$caminho_destino/$dir/img"
        fi
        cd ..
        fi
    done < <(find . -path './.git' -prune -o -path './action-destino' -prune -o -iname 'readme.md' -print0)
    cd ./action-destino/

    if ! git diff --quiet HEAD~1 HEAD "../README.md"; then # Se teve modificação no readme principal do repositório, copiamos esse readme
        hugo new content "$caminho_destino/_index.md" --force
        cat ../README.md >> "./content/$caminho_destino/_index.md"
    fi

    {
        git add . #Adiciona tudo que foi alterado e prepara o commit
        git commit -m "Documentação automática ($caminho_destino $(date))" &&
        git push https://${{ secrets.TOKEN }}@github.com/LelePG/action-destino.git
    } || {
        echo "Sem alterações significativas para committar"
    }
```

## Explicação do archetype exemplo.md

O Archetype Exemplo possui as priedades `title`, `date`, `draft` e `weight`. A propriedade date é criada a partir do valor da variável de página `Date` do Hugo, enquanto a propriedade `draft` é setada para falso para que o documento não seja entendido como rascunho. Já o processo de geração da propriedade `title` e `weight` explicarei abaixo do código do achertype.

```
+++
title = '{{ (delimit (after 1 (split .File.ContentBaseName "-")) " ") | title }}'
date = {{ .Date }}
draft = false
weight = {{ index (split (strings.TrimPrefix "0" .File.ContentBaseName) "-") 0 | int }}
+++
```

As propriedades title e weight são geradas a partir do nome da pasta, ou seja, uma pasta cujo nome é `01-exemplo-teste` deverá ter como título `Exemplo Teste` e como peso `1`, para isso, são realizadas uma série de manipilações a partir do nome da pasta para chegar a esse resultado. O nome da pasta está na variável `.File.ContentBaseName` e a partir daí temos as manipulações:

#### Para o título

- `{{ split .File.ContentBaseName "-" }}`: irá "quebrar" a string em `.File.ContentBaseName` ao encontrar um caractere `-`, o que transforma `01-exemplo-teste` em algo como `["01", "exemplo", "teste"]`
- `{{ (after 1 (split .File.ContentBaseName "-")) }}`: faz um "fatiamento" na lista começando a partir do primeiro elemento, ou seja, `["01", "exemplo", "teste"]` vira `[ "exemplo", "teste"]`
- `{{ (delimit (after 1 (split .File.ContentBaseName "-")) " ") }}`: une as partes do array gerado no passo anterior com um caractere de espaço, ou seja, `[ "exemplo", "teste"]` vira `exemplo teste`
- `{{ (delimit (after 1 (split .File.ContentBaseName "-")) " ") | title }}`: passa o texto `exemplo teste` para a função title do Hugo que capitalizará as palavras, chegando ao resultado final `Exemplo Teste`.

#### Para o peso

- `{{ strings.TrimPrefix "0" .File.ContentBaseName }}`: usa a função `strings.TrimPrefix` para retirar um prefixo de `.File.ContentBaseName`. O prefixo que quero remover é o zero, o que é necessário para facilitar a conversão final e também para evitar problemas com pastas nomeadas com `08`, que acaba sendo entendido como um valor em base octal. Sendo assim, `01-exemplo-teste` se torna `1-exemplo-teste`.
- `{{ (split (strings.TrimPrefix "0" .File.ContentBaseName) "-") }}` : faz o split da string resultante do passo anterior onde encontrar um caractere `-`, sendo assim , `1-exemplo-teste` se torna `["1","exemplo","teste]`
- `{{ index (split (strings.TrimPrefix "0" .File.ContentBaseName) "-") 0  }}`: pega o elemento de índice zero do array, que é referente ao número no nome da pasta, ou seja, o valor que desejo utilizar para o peso da página. Então de `["1","exemplo","teste]` resta `"1"`.
- `{{ index (split (strings.TrimPrefix "0" .File.ContentBaseName) "-") 0 | int }}`: por fim, se converte o valor `"1"` para um valor inteiro usando a função int.
