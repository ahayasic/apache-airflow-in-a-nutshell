# Dependências entre Tarefas

## Introdução

As dependências entre tarefas tem como propósito informar ao Airflow a ordem de execução das tarefas de uma DAG. Assim, uma tarefa A só pode ser executada quando suas dependências (upstream dependences) tiverem sido executadas (no geral, com sucesso).

Como já vimos em [Conceitos & Componentes Essenciais](../../introduction_to_apache_airflow/essential_concepts_and_components), usamos o operador `>>` (*right bitshift*) para definir a dependência entre as tarefas. Assim:

- Uma tarefa A só será executada após suas dependências (upstream tasks)
- Erros  de uma tarefa A são propagados para as tarefas posteriores que a possuem como dependência (downstream tasks).

### Dependências Lineares

A dependência linear é uma cadeia linear de tasks onde uma task A deve ser completada antes que a task B possa ser executada, uma vez que o resultado da task A é utilizado como entrada para a task B.

!!! example "Exemplo"
    Exemplo de DAG formada uma única cadeia de dependências lineares

    ```python
    task_a >> task_b >> task_c
    ```

### Dependências Fan-in/out

Uma dependência do tipo *fan-in/out* é aquela cujo uma tarefa A é upstream de múltiplas tarefas ou possuem muitas tarefas como dependências.

Mais precisamente, quando uma tarefa A possui múltiplas dependências então usamos *"fan-in"*. Já quando uma tarefa A é dependência de diversas tarefas, então usamos "fan-out".

No caso, podemos usar a sintaxe `[<upstream tasks>] >> <downstream task>` para definir dependências do tipo *fan-in/out*

!!! example "Exemplo"
    Exemplo de dependências *fan-in* e *fan-out*

    === "Fan-in (muitos-para-um)"

        ```python
        [task_x, task_y, task_z] >> task_a
        ```

    === "Fan-out (um-para-muitos)"

        ```python
        task_a >> [task_x, task_y, task_z]
        ```

## Branching

Branching é uma funcionalidade do Airflow que nos permite percorrer um caminho específico da DAG com base em algum critério condicional. Para tal, usamos o `BranchPythonOperator`.

O `BranchPythonOperator`, assim como o `PythonOperator` recebe um objeto Python executável. A diferença é que para o `BranchPythonOperator`, o executável deve retornar a *task_id* (ou uma lista de *task_id*) das tarefas que devem ser executadas, enquanto todas as outras tarefas não serão ignoradas (e seus status serão marcados como `skipped`).

!!! example "Exemplo"
    TODO: Adicionar figura de exemplo

Perceba que se usarmos `depends_on_past=True` nos possíveis caminhos da DAG, a partir da primeira execução todos os outros caminhos não executados não serão escalonados em qualquer próximo agendamento. Afinal, como estas tarefas não terão status=`Succeed` será impossível executá-las.

Ainda, caso diferentes caminhos se encaminhem para uma única tarefa (fan-in), precisamos alterar o parâmetro `trigger_rule`. Regras de disparo (trigger rules) são critérios que definem quando uma tarefa pode ser executada. Por exemplo, o padrão do argumento `trigger_rule` é `all_success`, o que significa que uma tarefa só será executada quando **todas as suas dependências** forem executadas com sucesso.

Porém, este é um caso impossível de acontecer quando lidamos com branching. Consequentemente, a tarefa com dependência *fan-in* ficará em estado de espera eternamente.

Uma forma de corrigir este problema é alterar a regra de disparo para `none_failed` que define que uma tarefa deve rodar assim que todas as dependências forem executadas sem falhar.

!!! example "Exemplo"
    TODO: Adicionar figura de exemplo

### Branching Customizado

Se quisermos implementar nosso próprio operador de branching, basta criarmos uma classe que herda de `BaseBranchOperator` e re-implementarmos o método `choose_branch`, que é o método chamado no procedimento de branching.

!!! note "Nota"
    O `choose_branch` deve retornar a task_id (ou lista de task_id) que devem ser executadas.

## Tarefas Condicionais

### `ShortCircuitOperator`

!!! info "Work in progress"

### `LatestOnlyOperator`

!!! info "Work in progress"

## Regras de Disparo

!!! info "Work in progress"

## Referências

- [Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter](https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902)
- [Airflow Guides by Astronomer](https://www.astronomer.io/guides/)
- [Concepts $-$ Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/concepts/index.html)