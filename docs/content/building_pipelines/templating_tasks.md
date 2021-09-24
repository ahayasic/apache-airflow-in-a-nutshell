# Jinja Templates

## Introdução

O *templating* é uma funcionalidade muito útil e poderosa para o compartilhamento de informações à tasks em tempo de execução.

Através dela podemos passar $-$ para cada operador $-$ informações como datas de execução, nomes de tabela, valores, etc.

!!! info "Jinja"
    Para habilitar o templating, o Airflow utiliza a/o Jinja, uma engine de templating que substitui variáveis e/ou expressões em strings modelos em tempo de execução.

Seu uso é bem simples, através de strings modelo (ou, macros). Essencialmente, um macro é uma string da forma `{{ valor }}` onde valor é a informação que queremos renderizar, seja um objeto ou uma string qualquer.

Logo, basta passarmos um macro para qualquer parâmetro (de um operador) que aceite templating e cujo `<valor>` seja default (i.e. fornecido pelo Airflow através do Contexto).

!!! example "Exemplo"
    O macro `{{ ds }}` é um macro padrão do Airflow que renderiza um objeto do tipo `pendulum`.

    ```python
    BashOperator(
        task_id="print_execution_date",
        bash_command="echo Executing DAG on {{ ds }}",
    )
    ```

## Macros Padrões

O Airflow fornece uma lista padrão de macros que são renderizados em objetos. Você pode consultar a [lista completa na documentação](https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html#default-variables).

Note ainda que cada macro é, na verdade, um componente do dicionário de contextos da tarefa.

Além disso, como estes macros são renderizados em objetos, podemos manipulá-los como tal, incluindo o uso da notação ponto para acessar atributos e métodos.

!!! example "Exemplo"
    ```python
    BashOperator(
        task_id="print_execution_date",
        bash_command="echo Executing DAG on {{ ds.format('dddd') }}",
    )
    ```

## Visualizando os Templates

Podemos visualizar as saídas geradas em cada macro (i.e. renderização) tanto pela UI quanto CLI. No caso da CLI, podemos visualizá-las sem que seja preciso executar qualquer tasks.

Para isso, basta executarmos o comando `airflow tasks render` passando como parâmetro a `dag_id`, `task_id` e uma `execution_date`.

```bash
$ airflow tasks render dag run_template 2021-01-01

# ----------------------------------------------------------
# property: bash_command
# ----------------------------------------------------------
echo "Today is Friday"

# ----------------------------------------------------------
# property: env
# ----------------------------------------------------------
None
```

Já pela UI, basta acessarmos as opções da task (clicando na mesma) e acionando o botão ++"Rendered"++.

## Argumentos e Scripts Templateable

Nem todos os argumentos de um operador podem receber templates. Apenas argumentos presentes no atributo `template_fields` da classe `BaseOperator` podem ser renderizados. Ainda, dado que alguns parâmetros (de algunso operadores), como `bash_command` (do `BashOperator`), podem receber scripts, também há o atributo `template_ext` que define quais arquivos podem ser renderizados. Dessa forma, podemos incluir macro nestes arquivos.

Logo:

- `template_fields`. Quais argumentos do operador são "templateable"
- `template_ext`. Quais arquivos do operador são "templateable"

!!! example "Exemplo"
    No caso do `BashOperator`, os argumentos "templateable" são `bash_command` e `env`. Já os arquivos são `.sh` e `.bash`.

    Com isso, podemos passar macros tanto em strings para o argumento `bash_command`.

    ```python
    BashOperator(
        task_id="print_execution_date",
        bash_command="echo Executing DAG on {{ ds }}",
    )
    ```

    Como também para arquivos `.sh` e `.bash`

    === "dag.py"
        ```python
            BashOperator(
            task_id="print_execution_date",
            bash_command="script.sh",
        )
        ```

    === "script.sh"
        ```bash
        echo Executing DAG on {{ ds }}
        ```

## Macros Customizados

!!! failure "TODO"

## Renderizando Objetos Python Nativos

!!! failure "TODO"
