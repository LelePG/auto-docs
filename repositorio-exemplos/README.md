## Action para um repositório de exemplos

## Formato esperado

O formato esperado para este tipo de repositório é um repositório com várias subpastas, onde cada subpasta possui um arquivo README.md, podendo também ter uma pasta img. A ideia é que todos os arquivos READMEs e pastas img associadas aos exemplos sejam transferidas para o repositório de documentação. Cada pasta deve ser nomeada com dois números e um traço seguido do nome. Um exemplo de árvore pode ser vista abaixo:

Exemplo:

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
        git config --global user.name "Action README"
        git config --global user.email " " # Usar o link do repositório de destino na linha abaixo.
        git submodule add https://x-access-token:${{ secrets.TOKEN }}@github.com/LelePG/action-destino.git action-destino
```

- Check for README.md and img directory (aqui é o script que faz a mágina acontecer em si.)

```bash
      - name: Check for README.md and img directory
        run: |
            while IFS= read -r -d '' file; do
            dir=$(dirname "$file")
            dir=${dir#./} # remove leading ./


            if [ ! -f "$dir/README.md" ]; then # se não tiver readme na pasta....
                echo "No README.md found"
                continue
            fi

            if git diff --quiet HEAD~1 HEAD "$dir/README.md"; then
                echo "No changes to $dir/README.md"
                continue
            fi

            if [["$dir" != "." && "$dir" != ".."]]; then
                cd ./action-destino/
                # Usar o nome onde os arquivos do projeto devem ser usados na documentação na próxima linha
                repo_name="Franzininho C0 - Exemplos Arduino"
                # O fatiamento dir:3 é para retirar os 3 primeiros caracteres, então "00-exemplo" se torna apenas "exemplo".
                rm -rf "./content/$repo_name/${dir:3}" # Deletar o conteúdo se ele já existir
                hugo new content "$repo_name/${dir:3}/\_index.md" # Criar o conteúdo
                cat "../$file" >> "./content/$repo_name/${dir:3}/\_index.md" #copiar o conteúdo do README para o _index.md criado

                if [[ -d "../$dir/img" ]]; then # se a pasta tiver uma pasta imagens, copiar a pasta imagens.
                    cp -r "../$dir/img" "./content/$repo_name/${dir:3}/img"
                fi
                cd ..
            fi
            done < <(find . -path './.git' -prune -o -path './action-destino' -prune -o -iname 'readme.md' -print0)
            cd ./action-destino/
            git add .
            git diff --exit-code --quiet || git commit -m "Documentação automática" # Evita um erro gerado pela action quando acontece um commit vazio
            # Usar o link do repositório de destino na linha abaixo.
            git push https://${{ secrets.TOKEN }}@github.com/LelePG/action-destino.git

```
