!!! info "Work in progress"

    # Introduction to Apache Airflow

    ## Introduction to Pipelines

    ## Apache Airflow for Pipelines

    ### When to use Airflow

    ## Apache Airflow Architecture

    O Airflow é composto, essencialmente, por 4 componentes.

    - **Webserver (UI).**
      - Interface Gráfica do Airflow
      - Aplicação Flask executada sobre gunicorn que fornece uma interface gráfica para interação com o banco de dados de metadados e arquivos de log.
    - **Scheduler.**
      - Processo responsável pelo escalonamento das tarefas que compõem as DAGs.
      - Essencialmente, um processo Python multi-thread que define, através do banco de dados de metadados, quais tarefas devem ser executadas, quando e onde **(*)**.
    - **Executor.**
      - Componente através do qual as tarefas são, de fato, executadas.
      - O Airflow contém uma grande diversidade de executos, cada um com suas vantagens e desvantagens.
    - **Metadata Database.**
      - Banco de dados que armazena todas as informações, metadados e status das DAGs e tarefas.
      - É o componente através do qual os demais componentes interagem.

    ## Fluxo de Execução das Tarefas

    A partir do momento em que o **scheduler** é iniciado:

    1. O **Scheduler** "lê" o diretório de DAGs e instancia todos os objetos DAG no banco de dados de metadados.

       > **Nota**
       >
       > This means all top level code (i.e. anything that isn't defining the DAG) in a DAG file will get run each scheduler heartbeat. Try to avoid top level code to    your DAG file unless absolutely necessary.

    2. São criadas (através do processo de instanciação) todas as Dag Runs**(\*\*)** necessárias com base nos parâmetros de escalonammento das tarefas de cada DAG.

    3. Em seguida, `TaskInstances` são instanciadas para cada tarefa que precisa ser executada e marcadas como `Scheduled` no banco de dados, além de outros metadados.

    4. O **Scheduler** então, consulta todas as tasks marcadas como `Scheduled` e as envia para o **Executor**, com o status alterado para `Queued`.

    5. O **Executor** puxa as tarefas da fila, aloca *workers* para executar as tarefas e altera o status de cada tarefa em execuçao de `Queued` para `Running`.

       - O comportamento de como as tarefas são puxadas da fila e executadas varia de **executor** para **executor**.

    6. A tarefa, ao ser encerrada, tem seu status alterado pelo *worker* para seu estado final (e.g., `Finised`, `Failed`) que, por sua vez, informa ao **Scheduler** que     então reflete essa mudança no banco de dados de metadados.

    A Figura abaixo ilustra o processo especificado.

    ![img](https://assets2.astronomer.io/main/guides/airflow_component_relationship_fixed.png)

    _**Fonte.** astronomer.io_

    ## Controle de Interações Entre os Componentes

    A forma como os componentes se comportam e/ou interagem entre si pode ser definida através do arquivo de configuração `airflow.cfg`.

    ### Executors

    As opções de Executores padrão para o Airflow são:

    - **`SequentialExecutor`**. Executa tarefas sequencialmente, sem paralelismo.
      - Útil para um ambiente de teste ou para debugar bugs mais profundos do Airflow e aplicações.
    - **`LocalExecutor`**. Executa tarefas com suporte à paralelismo e hyperthreading.
      - Uma boa opção para cenários onde o Airflow está em uma máquina local ou em um único nó.
    - **`CeleryExecutor`**. Opção para execução do Airflow em Clusters Distribuídos.
      - Redis, RabbitMq ou outro sistema de fila de mensagens para coordenar as tarefas entre os *workers*.
    - **`KubernetesExecutor`**. Outra opção para execução do Airflow em Clusters Kubernetes.
      - Executa tarefas através da criação de um pod temporário para cada tarefa a ser executada, permitindo que os usuários passem configurações personalizadas para cada    uma de suas tarefas e usem os recursos de maneira eficiente.

    ### Paralelismo

    Através dos parâmetros `parallelism`, `dag_concurrency` e `max_active_runs_per_dag` do arquivo `airflow.cfg` podemos configurar o Airflow para determinar quantas     tarefas podem ser executadas de uma vez.

    - **`parallelism`**. Número máximo de instâncias de tarefas que podem ser executadas simultaneamente.

      > Note que o número máximo vale para todas as DAGs. Ou seja, o número máximo de instâncias de tarefas considerando todas as instâncias de tarefas de todas as DAGs.

    - **`dag_concurrency`**. Número máximo de instâncias de tarefas que podem ser executadas simultaneamente para uma DAG específica.

    - **`worker_concurrency`** (**`CeleryExecutor`**).  Número máximo de  tarefas que um único *worker* pode processar.

      > Portanto, se você tiver 4 workers em execução em uma simultaneidade de trabalho de 16, poderá processar até 64 tarefas de uma vez.

    - **`max_active_runs_per_dag`**. Número máximo de DagRuns ao longo do tempo podem ser escalonadas para cada DAG específica.

      > Esse número deve depender de quanto tempo as DAGs levam para executar, seu intervalo de escalonamento e desempenho do **scheduler**.
