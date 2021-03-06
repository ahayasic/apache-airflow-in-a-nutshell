# Introdução ao Apache Airflow

O Apache Airflow é uma plataforma para projetar, construir e monitorar fluxos de trabalho (ou *workflows*) através de scripts Python.

O Airflow possui quatro características principais:

- **Dinâmico.** Os pipelines (sinônimo para *workflows*) no Airflow são como configurações definidas em código. Com isso, é possível criar pipelines dinâmicos e adaptativos de maneira simplificada.
- **Extensível.** O Airflow permite ao usuário definir os próprios componentes e/ou estender componentes já existem para alcançar as funcionalidades desejadas.
- **Elegante.** Os pipelines no Airflow possuem uma declaração limpa e explícita.
- **Escalável.** O Airflow possui uma arquitetura modular e usa mensageiros para orquestrar um número indefinido de *workers*, possibilitando uma grande escalabilidade tanto vertical quanto horinzotal.

Ainda, essas características são mais do que características. Na verdade, são princípios que guiam tanto as funcionalidades quanto o desenvolvimento do Aiflow em si.

Por fim, é importante ressaltar que o Airflow não é uma solução de streaming de dados, como é o caso de Spark Streaming ou Storm! O Airflow é uma ferramenta para a orquestração de tarefas na forma de DAGs e, portanto, deve ser utilizado para tal.

## Arquitetura do Apache Airflow

O Airflow é composto arquiteturalmente por 4 componentes:

- **Webserver (UI).** Interface gráfica do Airflow via web.

    Nada mais é que uma aplicação Flask executada sobre gunicorn que nos fornece uma interface gráfica para interação com o banco de metadados e arquivos de log.

- **Scheduler.** Processo responsável pelo escalonamento das tarefas que compõem as DAGs.

    Essencialmente um processo Python multi-thread que define, através do banco de metadados, quais tarefas devem ser executadas, quando e onde.

- **Executor.** Componente através do qual as tarefas são, de fato, executadas. O Airflow contém uma grande diversidade de executos, cada um com suas vantagens e desvantagens.

- **Metadata Database.** Banco de dados (por simplicidade, vamos usar a terminologia "banco de metadados") que armazena todas as informações, metadados e status das DAGs e tarefas.

    É o componente através do qual os demais componentes interagem entre si.

### Fluxo de Execução das Tarefa

Para entendermos melhor como o Airflow funciona e o papel de cada componente da arquitetura na execução de pipelines, precisamos entender o fluxo de execução das DAGs suas respectivas tarefas.

Essencialmente, a partir do momento em que o **scheduler** é iniciado:

1. O **Scheduler** "lê" o diretório de DAGs e instancia todos os objetos DAG no banco de metadados.

    !!! note "Nota"
        Isso significa que todos os códigos *top-level* $-$ mesmo que não definam DAGs $-$ serão lidos pelo scheduler, o que pode causar problemas de desempenho. Portanto, devemos evitar códigos desnecessários no diretório de DAGs.

2. Através do processo de instanciação (citado acima), todas as Dag Runs[^1] necessárias são criadas de acordo com os parâmetros de agendamento (ou escalonamento) das tarefas de cada DAG.
3. Para cada tarefa que precisa ser executada, são instanciadas `TaskInstances`[^2] que são marcadas como `Scheduled` no banco de metadados (além de outros metadados necessários para a execução das tarefas).
4. O **Scheduler** consulta todas as tasks marcadas como `Scheduled` e as envia para o **Executor**, atualizando seu status para `Queued`
5. O **Executor** puxa as tarefas da fila de execução e aloca *workers* para executá-las, alterando seu status de `Queued` para `Running`

    !!! note "Nota"
        O comportamento de como as tarefas são puxadas da fila e executadas difere para cada **Executor** escolhido.

6. A tarefa, ao ser encerrada, tem seu status alterado pelo *worker* para seu estado final (e.g., `Finised`, `Failed`) que então informa ao **Scheduler** que, por sua vez, reflete as mudanças no banco de metadados.

A Figura abaixo ilustra o processo especificado.

<p style="text-align: center;"><img src="https://assets2.astronomer.io/main/guides/airflow_component_relationship_fixed.png" alt="airflow_component_relationship_fixed"style="border-radius: 1rem"/></p>
<p class="post__img_legend">
  <b>Fonte:</b> <a target="_blank" href="https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp1617296902">Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter</a>
</p>


  [^1]: DAG Runs são instancias de uma DAG. É um conceito importante no Airflow e é abordado com mais detalhes na seção [Apache Airflow Conceitos & Componentes](apache-airflow-in-a-nutshell/content/introduction_to_apache_airflow/essential_concepts_and_components/)
  [^2]: `TaskInstances` são instancias de tarefas atreladas a uma DAG. Assim como as DAG Runs, também são um conceito importante no Airflow e, portanto, abordadas com mais detalhes na seção [Apache Airflow Conceitos & Componentes](apache-airflow-in-a-nutshell/content/introduction_to_apache_airflow/essential_concepts_and_components/).

## Controle de Interações Entre os Componente

A forma como os componentes se comportam e/ou interagem entre si pode ser definida através do arquivo de configuração `airflow.cfg`.

### Executor

As opções de Executores padrões para o Airflow são:

- **`SequentialExecutor`**. Executa tarefas sequencialmente, sem paralelismo. Útil em ambientes de teste ou para solução de bugs complexos.
- **`LocalExecutor`**. Executa tarefas com suporte à paralelismo e hyperthreading. Uma boa opção em cenários onde o Airflow está em uma máquina local ou em um único nó.
- **`CeleryExecutor`**. Opção para execução do Airflow em clusters distribuídos que utiliza de Redis, RabbitMq ou outro sistema de fila de mensagens para coordenar o envio de tarefas aos *workers*.
- **`KubernetesExecutor`**. Opção para execução do Airflow em clusters Kubernetes. Executa tarefas através da criação de um pod temporário para cada tarefa a ser executada, permitindo que os usuários passem configurações personalizadas para cada uma de suas tarefas e usem os recursos do cluster de maneira eficiente.

### Paralelismo

Através dos parâmetros `parallelism`, `dag_concurrency` e `max_active_runs_per_dag` do arquivo `airflow.cfg` podemos configurar o Airflow para determinar quantas     tarefas podem ser executadas simultaneamente.

- **`parallelism`**. Número máximo de instâncias de tarefas que podem ser executadas simultaneamente.

    !!! note "Nota"
        O número máximo compreende a soma de tarefas de todas as DAGs. Em outras palavras, o `parallelism` informa ao Airflow quantas tarefas ele pode executar simultaneamente, considerando todas as DAGs ativas.

- **`dag_concurrency`**. Número máximo de instâncias de tarefas que podem ser executadas simultaneamente em cada DAG.
- **`worker_concurrency`** (**`CeleryExecutor`**). Número máximo de tarefas que um único *worker* pode processar.

    Portanto, se você tiver 4 workers em execução e `worker_concurrency=16`, poderá processar até 64 tarefas simultaneamente.

- **`max_active_runs_per_dag`**. Número máximo de DagRuns ao longo do tempo podem ser escalonadas para cada DAG específica.

    !!! note "Nota"
        Esse número deve depender de quanto tempo as DAGs levam para executar, seu intervalo de escalonamento e desempenho do **scheduler**.


## Referências

- [Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter](https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902)
- [Airflow Guides by Astronomer](https://www.astronomer.io/guides/)
- [Concepts $-$ Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/concepts/index.html)
