# Jinja Templates

## Introdução

O Templating é uma funcionalidade muito útil e poderosa para o compartilhamento de informações à tasks em tempo de execução.

Através dela (funcionalidade), podemos passar $-$ para cada operador $-$ informações, como, datas de execução, nomes de tabela, valores, etc.

!!! info "Jinja"
    Para habilitar o templating, o Airflow utiliza a/o Jinja, uma engine de templating que substitui variáveis e/ou expressões em strings modelos em tempo de execução.

Seu uso é bem simples. Essencialmente, um string modelo (aka macro) é uma string da forma `{{ valor }}`. Logo, basta passarmos uma string modelo para qualquer parâmetro (de um operador) que aceite templating e cujo `<valor>` seja default (i.e. fornecido pelo Airflow através do Contexto) ou informado.

!!! example "Exemplo"
    O macro `{{ ds }}` é um macro padrão do Airflow que renderiza um objeto do tipo `pendulum`.

    ```python
    BashOperator(
        task_id="print_execution_date",
        bash_command="echo Executing DAG on {{ ds }}",
    )
    ```

### Macros

O Airflow fornece uma lista padrão de macros que são renderizados em objetos. Você pode consultar a [lista completa na documentação](https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html#default-variables).

Note ainda que cada macro é, na verdade, um componente do dicionário de contextos da tarefa.

Além disso, como os macros são renderizados em objetos, podemos manipulá-los como tal, incluindo o uso da notação ponto para acessar atributos e métodos.

!!! example "Exemplo"
    ```python
    BashOperator(
        task_id="print_execution_date",
        bash_command="echo Executing DAG on {{ ds.format('dddd') }}",
    )
    ```

