# Apache Airflow Conceitos & Componentes

## DAG de Exemplo

Abaixo, temos o código de uma DAG que faz o *download* e o processamento de dados de lançamento de foguetes.

```python
# Extracted from Data Pipelines with Apache Airflow (check "Referências" section)

import json
import pathlib
import requests
import requests.exceptions as requests_exceptions

import airflow
from airflow import DAG
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

dag = DAG(
    dag_id="download_rocket_launches",
    start_date=airflow.utils.dates.days_ago(14),
    schedule_interval=None,
)

download_launches = BashOperator(
    task_id="download_launches",
    bash_command="curl -o /tmp/launches.json -L 'https://ll.thespacedevs.com/2.0.0/launch/upcoming'",
    dag=dag,
)

def _get_pictures():
    # Ensure directory exists
    pathlib.Path("/tmp/images").mkdir(parents=True, exist_ok=True)

    # Download all pictures in launches.json
    with open("/tmp/launches.json") as f:
        launches = json.load(f)
        image_urls = [launch["image"] for launch in launches["results"]]

        for image_url in image_urls:
            try:
                response = requests.get(image_url)
                image_filename = image_url.split("/")[-1]
                target_file = f"/tmp/images/{image_filename}"
                with open(target_file, "wb") as f:
                f.write(response.content)
                print(f"Downloaded {image_url} to {target_file}")
            except requests_exceptions.MissingSchema:
                print(f"{image_url} appears to be an invalid URL.")
            except requests_exceptions.ConnectionError:
                print(f"Could not connect to {image_url}.")Writing your first Airflow DAG

get_pictures = PythonOperator(
    task_id="get_pictures",
    python_callable=_get_pictures,
    dag=dag,
)

notify = BashOperator(
    task_id="notify",
    bash_command='echo "There are now $(ls /tmp/images/ | wc -l) images."',
    dag=dag,
)

download_launches >> get_pictures >> notify
```

Quebrando o código em etapas,

```python
dag = DAG(
    dag_id="download_rocket_launches",
    start_date=airflow.utils.dates.days_ago(14),
    schedule_interval=None,
)
```

Nesta fase, estamos criando um objeto do tipo `DAG`. Este objeto é o ponto de partida para a criação do nosso pipeline e o recurso para o qual o Airflow irá olhar.

Uma DAG tem vários parâmetros, no nosso caso:

- `dag_id` (obrigatório). Identificador da DAG. Deve ser um nome único, sem caracteres especiais (com exceção de *underscores*)
- `start_date` (obrigatório). Data a partir do qual a DAG pode ser executada.
- `schedule_interval`. Intervalo de execução da DAG. Ao atribuirmos `None`, estamos definindo que a DAG em questão só será executada quando a triggarmos manualmente.

Em seguida, definimos nossa primeira tarefa.

```python
download_launches = BashOperator(
    task_id="download_launches",
    bash_command="curl -o /tmp/launches.json -L 'https://ll.thespacedevs.com/2.0.0/launch/upcoming'",
    dag=dag,
)
```

No caso, declaramos um `BashOperator` cuja responsabilidade é executar um *comando Bash*.

Os operadores são os caras responsáveis por executar as tarefas em uma DAG e são independentes entre si. Contudo, ainda podemos definir um fluxo de execução das tarefas $-$ chamamos isso de *dependências*. 

Em nosso código, definimos as dependências no seguinte trecho:

```python
download_launches >> get_pictures >> notify
```

No Airflow, utilizamos o operador binário *"rshift"* (`>>`) para definirmos as dependências entre as tarefas. Assim, a tarefa `get_pictures` só acontecerá após `download_launches`. Afinal, só podemos filtrar as imagens após as coletarmos!

> Outro termo comum para o relacionamento entre tasks é `upstream` e `downstream`.
>
> - `upstream`. Tarefa que é dependência de outra. Por exemplo, `download_launches` é `upstream` de `get_pictures`.
> - `downstream`. A tarefa dependente. Por exemplo, `get_pictures` é `downstream` de `download_launches`.

O `BashOperator` também possui diversos parâmetros.

- `task_id` (obrigatório). Identificador da task.

  > De fato, o argumento `task_id` é um campo obrigatório em todos os operadores.

- `bash_command` (obrigatório). Comando Bash a ser executado.

Na linha `dag=dag`, estamos atrelando a tarefa definida à DAG desejada.

> Na verdade, existem diversas formas de fazermos isso, por exemplo, podemos abstrair esse trecho (`dag=dag`) em particular, se criarmos uma DAG como uma Gerenciadora de Contexto.

Já no seguinte trecho:

```python
get_pictures = PythonOperator(
    task_id="get_pictures",
    python_callable=_get_pictures,
    dag=dag,
)
```

Estamos utilizando o operador `PythonOperator` $-$ cuja responsabilidade é executar uma **função** Python (ou então, um método)  $-$ para executar a tarefa de filtragem das fotos coletadas definida na função `_get_pictures`.

### Tasks vs Operators

Operadores e tasks são um conceito confuso e podem parecer a mesma coisa, mas não são!

- **Operadores.** Componentes especializados em executar um único e específico trabalho dentro do workflow. Por exemplo, temos:

  - `BashOperator`. Responsável por executar um comando ou script Bash.
  - `PythonOperator`. Responsável por executar uma função Python.
  - `SimpleHTTPOperator.` Responsável por fazer uma chamada HTTP à um endpoint.

  De certa forma, podemos pensar que uma DAG simplesmente orquestra a execução de uma coleção de operadores. Ainda, como são os operadores que, conceitualmente, executam as tarefas em si, acabamos usando ambos os termos de forma intercambiável.

- **Tasks.** Componentes que podem ser vistos como "gerenciadores" de operadores. De fato, embora do ponto de vista do usuário tarefas e operadores sejam equivalente, no Airflow ainda temos o componente `task`.

  Tal componente é responsável por gerenciar o estado de operação dos operadores.

  <p style="text-align: center;"><img src="https://raw.githubusercontent.com/ahayasic/apache-airflow-in-a-nutshell/main/docs/assets/tasks_vs_operators.png" alt="tasks_vs_operators" style="border-radius: 1rem"/></p>
  <p style="text-align: center; font-size: 0.75rem; margin-bottom: 1.5rem;">
    <b>Fonte:</b> <a target="_blank" href="https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902">Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter</a>
  </p>

### Triggando a DAG Manualmente

Agora que temos o código da nossa DAG pronto, podemos executá-la.

Uma boa prática, é testar o código a fim de encontrar algum problema de sintaxe, por exemplo. Logo, seja o nosso arquivo `download_and_process_rocket_data.py` a DAG. Basta executarmos

```python
$ python download_and_process_rocket_data.py
```

Caso o interpretador não encontre qualquer problema de importação ou sintaxe, basta acessarmos a UI do Airflow e triggarmos a DAG manualmente!

### Lidando com Tasks que Falharam

É muito comum $-$ principalmente durante a etapa de desenvolvimento $-$ algumas tarefas falharem. Os motivos da falha podem ser diversos, desde erros de sintaxe até questões mais complexas como problemas de conectividade.

A primeira ação a ser tomada neste caso é consultar os *logs* da tarefa que falhou para identificarmos os motivos. 

Uma vez que os motivos são identificados e os problemas corrigidos, podemos retriggar a DAG a partir da tarefa que falhou! Desse modo, não é necessário reexecutarmos tarefas posteriores que foram bem-sucedidas. Basta acessarmos a **tarefa que falhou** através da UI e acionarmos a opção `[Clear]`. Com isso, o Airflow irá resetar o estado atual da tarefa para um estado "escalonável" e reexecutá-la.

### Definindo Intervalos de Execução Regulares

Como visto anteriormente, o Airflow nos fornece um argumento `schedule_interval` onde podemos especificar o intervalo de execução da DAG.

Ao atribuirmos `None`, dizemos ao Airflow que a DAG não possui qualquer intervalo de execução e, portanto, ela só será executada quando a triggarmos manualmente. 

Alternativamente, podemos utilizar a mesma sintaxe da ferramenta *Cronjob* para definirmos intervalos de execução. Por exemplo, ao atribuirmos `@daily` ao argumento `schedule_interval`, estamos dizendo ao Airflow que a DAG em questão deve ser executada diariamente (por padrão, o Airflow irá executá-la às 00hrs).

Além disso, o Airflow conta com um comportamento chamado *Backfill*. Isso significa que o Apache Airflow irá definir todos os horários em que a DAG em questão deveria ter sido executada a partir da sua data de início (i.e. `start_date`) e, nos intervalos onde a DAG não tiver sido executada, o Airflow irá reexecutá-la.

## Apache Airflow Principais Conceitos

Em resumo:

- **DAG.** Coleção de tarefas a serem executadas e organizadas de uma forma cuja suas relações e dependências são expressas.
- **DAG Run.** Instância de um DAG para uma data e hora específicas.
- **Task.** Define um trabalho a ser executado (através de um operador) escrito em Python.
- **Task Instance.** An instance of a task - that has been assigned to a DAG and has a state associated with a specific DAG run (i.e for a specific execution_date).
- **Operator.** Modelo para a realização de alguma tarefa.

### DAG (Directed Acyclic Graph)

Uma DAG *(Directed Acyclic Graph)* é uma coleção de tarefas a serem executadas e organizadas de uma forma cuja suas relações e dependências são expressas.

DAGs são:

- Definidas através de um script Python.
- Podem ser utilizadas na forma de um Gerenciador de Contexto.

Os parâmetros mais importantes na declaração de uma DAG são:

- `dag_id`. Identificador do DAG.

- `start_date`. *Timestamp* a partir do qual o _scheduler_ deve escalonar a DAG

  > Ou seja, o parâmetro `start_date` é responsável por definir a data à partir da qual a DAG está disponível para execução.

- `schedule_interval`. Intervalo de execução da DAG.

- `default_args`. Dicionário com parâmetros padrões que devem ser passados à todas as tasks

  > Portanto, quando todos os operadores possuírem um conjunto de parâmetros em comum, o uso do `defaults_args` é a forma mais inteligente de evitar duplicidade de código.

Outros parâmetros interessantes de se ter conhecimento são:

- `params`. Dicionário de parâmetros à nível DAG que são disponibilizados em *templates* e *namespaces* sob parâmetros.
- `concurrency`. Número de instâncias de tarefas que podem ser executadas paralelamente.
- `on_failure_callback`. Uma função a ser chamada quando a DAG (mais precisamente, a `DAG Run` referente à DAG) falhar.
- `on_success_callback`. Mesmo comportamento do parâmetro `on_falilure_callback`, porém quando a DAG roda com sucesso.

>  *Note que o Airflow carregará apenas objetos `DAG` globais!*

No Airflow 2.x, através da *decorator* DAG é possível gerar DAGs através de funções. Assim, qualquer função decorada com `@dag` retorna um objeto DAG.

### DAG Run

Uma `DAG Run` é uma instância de uma DAG que, por sua vez, contém instâncias das tarefas que são executadas para uma `execution_date` específica.

> A `execution_date` é a data e hora que a `DAG Run` e as `TaskInstance` estão sendo (ou foram) executadas.

<p><img src="https://raw.githubusercontent.com/ahayasic/apache-airflow-in-a-nutshell/main/docs/assets/task_lifecycle_diagram.png" alt="tasks_lifecycle" style="text-align: center; border-radius: 1rem"/></p>
<p style="text-align: center; font-size: 0.75rem; margin-bottom: 1.5rem;">
  <b>Fonte:</b> <a target="_blank" href="https://airflow.apache.org/docs/apache-airflow/stable/concepts/tasks.html#task-instances">Task Instances - Apache Airflow Documentation</a>
</p>

Fazendo uma analogia com a Programação Orientada à Objetos, podemos pensar nas **DAGs** como **classes** e nas **DAG Runs** como **objetos da classe DAG** (ou seja, uma instância da classe DAG).

### Argumentos Padrão (`default_args`)

Os argumentos padrões são úteis para aplicar um parâmetro comum a muitos operadores (ou tarefas) através de um dicionário passado pelo argumento `default_args`.

```python
default_args = {
    'owner': 'airflow',
    'depends_on_past': False,
    'email': ['airflow@example.com'],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': timedelta(minutes=5),
    # 'queue': 'bash_queue', ##
    # 'pool': 'backfill', ##
    # 'priority_weight': 10, ##
    # 'wait_for_downstream': False,
    # 'sla': timedelta(hours=2),
    # 'execution_timeout': timedelta(seconds=300),
    # 'on_failure_callback': some_function,
    # 'on_success_callback': some_other_function,
    # 'on_retry_callback': another_function,
    # 'sla_miss_callback': yet_another_function,
    # 'trigger_rule': 'all_success'
}
```

Os parâmetros mais importantes de serem compartilhados entre as tarefas são:

- `owner`. Administrador da tarefa (ou *task*).

- `depends_on_past`. Define se as *tasks* da DAG dependem do sucesso de *tasks* anteriores para execução.

- `retries`. Quantidade de vezes que uma *task* deve tentar ser reexecutada, se falhar.

- `retry_delay`. Tempo de espera até uma próxima tentativa de re-execução.

- `on_failure_callback`. Uma função a ser chamada quando a tarefa (mais precisamente, a `TaskInstance` referente à *task*) falhar.

  > No caso, um Dicionário de Contexto é passado como um único parâmetro para esta função. O contexto contém referências a objetos relacionados à instância da tarefa e está documentado na seção de macros da API.

- `on_sucess_callback`. Mesmo comportamento do parâmetro `on_falilure_callback`, porém executado  quando a *task* DAG roda com sucesso.

- `on_retry_callback` Mesmo comportamento do parâmetro `on_falilure_callback`, porém executado .quando ocorre uma tentativa de re-execução da tarefa.

- `trigger_rule`. Define a regra pela qual as dependências são aplicadas para que a tarefa seja acionada. As opções são:

  - `{ all_success | all_failed | all_done | one_success | one_failed | none_failed | none_failed_or_skipped | none_skipped | dummy}`
  - Padrão é `all_success`. 

  > As opções podem ser definidas como *string* ou usando as constantes definidas na classe estática `airflow.utils.TriggerRule`

Outros parâmetros interessantes de se ter conhecimento são:

- `queue`.
- `pool`.
- `priority_weight`.
- `wait_for_downstream`.
- `execution_timeout`.
- `sla`.
- `sla_miss_callback`.

### Tarefas

Uma **Task** define (através de Operadores) uma unidade de trabalho dentro de uma DAG e são representadas como um **nó** na DAG.

#### Relacionamento Entre Tarefas

Para definir o relacionamento (i.e. dependências) entre tarefas, utilizamos os operadores `>>` e `<<`. Por exemplo, considere a seguinte DAG com duas *tasks*:

```python
with DAG('my_dag', start_date=datetime(2016, 1, 1)) as dag:
    task_1 = DummyOperator('task_1')
    task_2 = DummyOperator('task_2')
    task_1 >> task_2 # Define dependencies
```

Assim,

- `task_1` começará a ser executado enquanto a `task_2` espera que `task_1` seja concluída com sucesso para então iniciar.

  > Podemos dizer que `task_1` está à *upstream* da `task_2` e, inversamente, `task_2` está à downstream da `task_1`.

### Task Instances

Assim como *DAGs* são instanciadas em *DAG Runs*, Tasks são instanciadas em *Tasks Instances*. Dentre seus diversos papéis, as *Tasks Instances* são as responsáveis por representar o estado de uma tarefa $-$ ou seja, em qual etapa do seu ciclo de vida a tarefa se encontra.

Os possíveis estados são:

- `none`. A task ainda não foi adicionada à fila de execução (i.e. escalonada), uma vez que suas dependências ainda não foram supridas.
- `scheduled`. As dependências da task foram supridas e ela pode ser executada.
- `queued`. A task foi atrelada à um *worker* pelo Executor e está esperando a disponibilidade do *worker* para ser executada.
- `running`. A task está em execução (i.e. sendo executada por um *worker*).
- `success`. A task foi executada sem erros (logo, com sucesso).
- `failed`. A task encontrou erros durante a execução e, portanto, falhou.
- `skipped`. A task foi "pulada" devido à algum mecanismo de "branching" ou similares.
- `upstream_failed`. As dependências da task falharam e, portanto, ela não pode ser executada.
- `up_for_retry`. A task falhou, mas possui mecanismos de "tentativas" definidos e portanto pode ser re-escalonada.
- `up_for_reeschedule`.  A task é um **Sensor** (mais informações em documentos mais avançados) e está no modo `scheduled`.
- `sensing`. A task é um **Smart Sensor** (mais informações em documentos mais avançados).
- `removed`. A task foi removida da DAG desde sua última execução.

### Operadores

Operadores definem uma única e exclusiva tarefa em um *workflow*.

Os operadores são geralmente (mas nem sempre) atômicos, o que significa que *podem ser autônomos e não precisam compartilhar recursos com quaisquer outros operadores*.

- Se dois operadores precisam compartilhar informações entre si (e.g, nome de arquivo ou uma pequena quantidade de dados), você deve considerar combiná-los em um único operador.
  - Se isso não for possível (ou puder ser evitado de forma alguma), o Airflow tem uma *feature*  para a comunicação cruzada entre operadores chamada **XCom** descrita na seção [XComs](https://airflow.apache.org/docs/apache-airflow/stable/concepts.html#concepts-xcom).
- Operadores **não precisam ser atribuídos as DAGs imediatamente** (anteriormente, a `dag` era um argumento obrigatório).
  - No entanto, uma vez que um operador é atribuído a um DAG, **ele não pode ser transferido ou não atribuído**

Alguns exemplos de operadores populares são:

- [`BashOperator`](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/operators/bash/index.html#airflow.operators.bash.BashOperator). Executa um comando bash.
- [`PythonOperator`](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/operators/python/index.html#airflow.operators.python.PythonOperator). Chama uma função Python qualquer.

## Referências

- [Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter](https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902)
- [Concepts $-$ Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/concepts/index.html)