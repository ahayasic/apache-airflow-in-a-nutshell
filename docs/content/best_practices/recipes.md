# Receitas & Códigos Padrões

## Configurando o Agendamento de DAGs

O agendamento de DAGs é simples e dividido em duas etapas:

- Agendamento da DAG
- Escalonamento das Tarefas

Logo, no geral precisamos responder as seguintes perguntas:

- A partir de qual momento a DAG deve ser escalonada?
- Com qual frequência a DAG deve ser executada?

    - Se manual, atribuir `None`.
    - Se possível utilizar [cron](), utilizar cron. Caso contrário, utilizar [datetime]().

- Qual critério condicional para disparar a DAG?
- Em que momento a DAG não deve mais ser executada?
- As tasks dependem do sucesso das execuções passadas? (Ou dependem de tasks passadas, no geral)

    - Se sim, usar `depends_on_past=True`

- Qual o tempo máximo que uma task pode se manter em execução?
- Se uma task falhar:

    - (Enquanto a DAG ainda estiver em execução) Quantas vezes a task pode tentar ser reexecutada?
    - As execuções passadas devem ser refeitas?

        - Se sim, use `catchup=True`. Caso contrário, use `False`

!!! note "Nota"
    As perguntas são apenas para orientação e não necessariamente algo obrigatório.

### Exemplo

- *A partir de qual momento a DAG deve ser escalonada?* A DAG deve ser executada a partir do dia 20 de Setembro de 2021.
- *Com qual frequência a DAG deve ser executada?* A DAG deve ser executada a cada dois dias às 9hrs.
- *Qual critério condicional para disparar a DAG?* Nenhum.
- *Em que momento a DAG não deve mais ser executada?* Nenhum.
- *As tasks dependem do sucesso das execuções passadas?* Não.
- *Qual o tempo máximo que uma task pode se manter em execução?* Padrão.
- *Se uma task falhar, quantas vezes a task pode tentar ser reexecutada?* Padrão.
- *Se uma task falhar, as execuções passadas devem ser refeitas?* Não.

```python
from datetime import datetime
from airflow.models import DAG

dag = DAG(
    start_date=datetime(2021, 9, 20),
    schedule_interval="0 9 */2 * *",
    catchup=False,
    default_args={"depends_on_past": False},
)
```

### Dicas Adicionais

#### Catchup & Backfilling

Ao combinarmos `depends_on_past=True` e `catchup=True`, execuções passadas de uma tarefa que falhou serão reexecutadas, automaticamente, de forma ordenada.

!!! tip "Dica"
    Podemos forçar a reexecução de tarefas passadas através do botão ++"Clear"++.

Se definirmos `depends_on_past=True` e `catchup=False`, execuções passadas não serão refeitas automaticamente. Porém, ainda podemos forçar a reexecução que, no caso acontecerá de forma ordenada. Ao mesmo tempo, note que ao definirmos `depends_on_past=True` para uma tarefa e ela falhar, **execuções posteriores desta mesma tarefa não irão acontecer automaticamente**.

Se definirmos `depends_on_past=False` e `catchup=True`, tasks que falharam serão reexecutadas mas de forma desordenada.

Já se `depends_on_past=False` e `catchup=False`, não haverá qualquer reexecução de tarefa ou dependências.

Portanto, caso a dependência com tasks passadas não seja um comportamento desejado, mas a reexecução de tarefas passadas (e.g. processamento) seja desejada eventualmente, o procedimento a ser seguido é:

1. Alterar os valores de `depends_on_past` e `catchup`.
2. Executar as tarefas.
3. Voltar as configurações para a forma inicial.

## Configurando Sensores

## Configurando o `PythonOperator`