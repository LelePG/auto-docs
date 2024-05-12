## Action para um repositório de exemplos

## Formato esperado

O formato esperado para este tipo de repositório é um repositório com um único arquivo README localizado na pasta raíz com a possibilidade de uma pasta img associada. A organização dos demais arquivos do projeto acaba sendo irrelevante. Um exemplo de árvore pode ser vista abaixo:

Exemplo:

```
-- Pasta Principal
    -- README.md
    -- img/
        -- foto1.jpeg
        -- foto2.jpeg
    -- Projeto Kicad
        -- Arquivos do projeto
    -- PDF de documentação
    -- Modelos de Componentes
        -- arquivo1
        -- arquivo2
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
        # O nome e o e-mail são genéricos para o projeto
        git config --global user.name "Auto Docs"
        git config --global user.email "auto-docs@mail.com"
        # Usar o link do repositório de destino na linha abaixo.
        git submodule add https://${{ secrets.TOKEN }}@github.com/CAMINHO-PARA/REPOSITORIO-DESTINO.git repositorio-destino
```

- Check for README.md and img directory (aqui é o script que faz a mágina acontecer em si.)

```bash
 - name: Check for README.md and img directory
        run: |
            if [ -f "./README.md" ]; then
              caminho_destino="Franzininho DIY Board"

            # Se o README não for modificado, só sai da action
              if git diff --quiet HEAD~1 HEAD "./README.md"; then
                  echo "No changes to README.md"
                  exit 0
              fi

              cd ./repositorio-destino/
              ## Adicionar o nome que deve aparecer no repositório final na variável abaico
              rm -rf "./content/$caminho_destino/" # remove the directory na pasta de destino
              hugo new content "$caminho_destino/_index.md" ## cria o conteúdo
              cat "../README.md" >> "./content/$caminho_destino/_index.md" ## copia o README pro conteúdo criado

              if [[ -d "../img" ]]; then ## se tiver a pasta imagens
                cp -r "../img" "./content/$caminho_destino/img" #copia a pasta imagens
              fi
              cd ..
            fi
            cd ./repositorio-destino/
            git add .
            git commit -m "Copied README.md files and img directories"
            git push https://${{ secrets.TOKEN }}@github.com/CAMINHO-PARA/REPOSITORIO-DESTINO.git

```
