# Agendamento e Sensores

Para que as DAGs sejam executáveis, elas precisam de:

- Uma data de início a partir do qual podem ser executadas.
- Uma política de disparo que pode ser (além do disparo manual): um intervalo de execução e/ou um critério condicional.

Essas configurações são definidas durante a inicialização (i.e. criação do objeto) da DAG.

## Configurando o Agendamento de DAGs

Chamamos de agendamento (ou escalonamento) o procedimento executado pelo Airflow de definir quando uma DAG deve ser executada.

O agendamento é configurado através de três parâmetros principais: `start_date`, `schedule_interval` e `end_date`. Todos estes parâmetros são atribuidos durante a inicialização da DAG.

- **`start_date` [obrigatório]**. Momento a partir do qual a DAG em questão estará disponível para ser executada. Note que o argumento `start_date` é obrigatório durante a inicialização pois sem uma data de início é impossível para o Airflow saber se a DAG pode ou não ser executada.
- **`schedule_interval`**. Intervalo de execução da DAG. Por padrão, o valor de `schedule_interval` é `None`, o que significa que a DAG em questão só será executada quando triggada manualmente.
- **`end_date`**. Momento até onde a DAG deve ser executada.

Abaixo, um exemplo de uma DAG com escalonamento diário.

```python
dag = DAG(
    dag_id="02_daily_schedule",
    schedule_interval="@daily",
    start_date=dt.datetime(2019, 1, 1),
    ...
)
```

Uma vez definida a DAG, o Airflow irá agendar sua primeira execução para o primeiro intervalo a partir da data de início.

!!! example "Exemplo"

    Se definirmos que uma DAG está disponível para ser executada a partir do dia **09 de Agosto de 2021** às **00hrs** com **intervalo de execução de 15 minutos**, a primeira execução será agendada para 00:15, a segunda execução será agendada para 00:30 e assim sucessivamente.

Se não definirmos uma data final, o Airflow irá agendar e executar a DAG em questão eternamente. Assim, caso exista uma data final definitiva a partir do qual a DAG não deverá ser executada, podemos utilizar o argumento `end_date` da mesma forma que `start_date` para limitar os agendamentos.

```python
dag = DAG(
    dag_id="03_with_end_date",
    schedule_interval="@daily",
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
)
```

### Intervalos Baseados em Cron

Podemos definir intervalos de execução complexos usando a mesma sintaxe que usamos no [cron](https://cron-job.org/en/).

Basicamente, a sintaxe é composta por cinco componentes organizados da seguinte forma:

```bash
# ┌─────── minute (0 - 59)
# │ ┌────── hour (0 - 23)
# │ │ ┌───── day of the month (1 - 31)
# │ │ │ ┌───── month (1 - 12)
# │ │ │ │ ┌──── day of the week (0 - 6) (Sunday to Saturday;
# │ │ │ │ │	7 is also Sunday on some systems)
# * * * * *
```
<p style="text-align: center; font-size: 0.75rem; margin-bottom: 1.5rem;">
    <b>Fonte:</b> <a target="_blank" href="https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902">Data Pipelines with Apache Airflow (2021) by Bas    Harenslak and Julian de Ruiter</a>
</p>

O carácter `*` significa que o valor do campo em questão não importa. Assim, podemos definir desde intervalos simples e convencionais:

- `0 * * * *`. Executa a cada hora

Até intervalos mais complexos

- `0 0 1 * *`. Executa a cada primeiro dia do mês (às 00hrs)

Também podemos utilizar de vírgulas (`,`) para definir conjuntos de valores e hífen (`-`) para intervalos de valores. Por exemplo:

- `0 0 * * MON, WED, FRI`. Executa toda segunda, quarta e sexta-feira (às 00hrs)
- `0 0,12 * * MON-FRI`. Executa às 00hrs e 12hrs de segunda à sexta-feira

Alternativamente, podemos recorrer à ferramentas como [crontab.guru](https://crontab.guru/) e [crontab-generator](https://crontab-generator.org/) para definirmos as expressões de forma mais fácil.

O Airflow também fornece alguns macros que podemos utilizar com mais facilidade. Os mais comuns são:

|   Macro   | Descrição                      |
|:---------:|--------------------------------|
| `@once`   | Executa uma única vez          |
| `@hourly` | Executa a cada começo de hora  |
| `@daily`  | Executa todos os dias às 00hrs |
| `@weekly` | Executa todo domingo às 00hrs  |

### Intervalos Baseados em Frequência

Embora poderosas, expressões cron são incapazes de representar agendamentos baseados em frequência. Por exemplo, não é possível definir (de forma adequada) um intervalo de "três em três em dias".

Por conta disso, o Airflow também aceita instâncias `timedelta` para definir intervalos de execução. Com isso, podemos definir uma DAG que é executada a cada três dias a partir da data de início.

```python
dag = DAG(
    dag_id="04_time_delta",
    schedule_interval=dt.timedelta(days=3),
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
)
```

## Catchup & Backfill

Por padrão, o Airflow sempre irá escalonar e executar toda e qualquer execução **passada** que **deveria ter sido executada** mas que por algum motivo **não foi**. Este comportamento é denominado *"backfill"* e é controlado pelo argumento `catchup` $-$ presente na inicialização da DAG.

Caso este comportamento não seja desejado, basta desativá-lo atribuindo `False` ao parâmetro. Dessa forma, o Airflow irá executar a DAG apenas a partir do primeiro intervalo de execução mais próximo.

```python
dag = DAG(
    dag_id="09_no_catchup",
    schedule_interval="@daily",
    start_date=dt.datetime(year=2019, month=1, day=1),
    end_date=dt.datetime(year=2019, month=1, day=5),
    catchup=False,
)
```

!!! example "Example"
    Se uma DAG for ativada dentro de um período intervalar, o Airflow irá agendar a execução da DAG para o início deste período.

    Por exemplo, considerando uma DAG com `start_date=datetime(2021, 08, 10)` e `schedule_interval="@daily"`; se ativarmos a DAG no dia 15 do mesmo mês às 12hrs, o Airflow irá definir que a primeira execução da DAG deverá ter ocorrido às 00hrs do dia 15 e irá executar a DAG imediatamente.

!!! note "Nota"
     Para mudar o valor padrão do `catchup` de `True` para `False`, basta acessar o arquivo de configuração e modificar o parâmetro `catchup_by_default`.

Embora o *backfilling* possa ser indesejado em algumas situações, seu uso é muito útil para a reexecução de tarefas históricas.

Por exemplo, suponha as seguintes tarefas `download_data >> process_data`. Considerando que os dados adquiridos através da tarefa `download_data` ainda sejam reacessáveis localmente, podemos realizar as alterações desejadas em `process_data` e então limparmos as execuções passadas (através do botão ++"Clear"++) para que assim o Airflow reagende e execute a nova implementação de `process_data`.

!!! warning "Atenção"
    A reexecução das DAGs não ocorre de forma ordenada (i.e. de acordo com a data de execução), mas sim de forma paralela.

    Para que o *backfilling* ocorra de forma ordenada, é necessário que o argumento `depends_on_past` presente na inicialização das tarefas seja `True`.

    Detalhes sobre o argumento serão apresentados a diante.

### Datas de execução de uma DAG

Em diversas situações é útil sabermos as datas de execução de uma DAG. Por conta disso, o Airflow nos permite acessar tanto a data da execução corrente da DAG, quanto a data da execução imediatamente anterior e posterior.

- `execution_date.` Data da execução corrente de DAG.

    Contudo, diferente do que o nome sugere, a data de execução marcada na DAG não é a data em que ela foi executada, mas sim no momento em que ela deveria ser executada, com base no intervalo de execução.

    !!! example "Exemplo"
        Suponha que temos uma DAG configurada para executar diariamente a partir do dia 2020-01-01.

        Após dez dias de execução, fizemos algumas alterações de implementação e agora precisamos que as execuções dos últimos dez dias sejam refeitas. Neste caso, ao acionarmos a opção ++"Clear"++, as datas de execução da DAG permanecerão de acordo com a data em que foram agendadas originalmente.

        Agora, se triggarmos manualmente a DAG antes da próxima execução agendada, a `execution_date` será o momento em que a DAG foi de fato disparada.

- `previous_execution_date`. Data de execução (`execution_date`) da DAG imediatamente anterior.
- `next_execution_date`. Próxima data de execução (`execution_date`) agendada para a DAG.

O acesso a essas informações pode ser feito através do contexto da DAG[^1] ou por meio dos macros pré-definidos:

=== "Macros"

    | Macro             | Descrição                                       |
    |-------------------|-------------------------------------------------|
    | `{{ ds }}`        | Data de execução no formato YYYY-MM-DD          |
    | `{{ ds_nodash }}` | Data de execução no formato YYYYMMDD            |
    | `{{ prev_ds }}`   | Data de execução anterior no formato YYYY-MM-DD |
    | `{{ next_ds }}`   |  Próxima data de execução no formato YYYY-MM-DD |

=== "Uso"

    ```python
    fetch_events = BashOperator(
        task_id="fetch_events",
        bash_command=(
            f"curl -o /data/events.json {source}"
            "start_date={{ ds }}"
            "end_date={{ next_ds }}"
        ),
        ...
    )
    ```

## Sensores

Além de execuções automáticas realizadas em intervalos de tempo, podemos querer disparar uma DAG sempre que um critério condicional for atendido. No Airflow, podemos fazer isso através de Sensores.

Um sensor é um tipo especial de operador que verifica continuamente (em um intervalo de tempo) se uma certa condição é verdadeira ou falsa.

- Se verdadeira, o sensor tem seu estado alterado para bem-sucedido e o restante do pipeline é executado.
- Se falsa, o sensor continua tentando até que a condição seja verdadeira ou um tempo limite (timeout) for atingido.

Um sensor muito utilizado é o `FileSensor` que verifica a existência de um arquivo e retorna verdadeiro caso o arquivo exista. Caso contrário, o `FileSensor` retorna `False` e refaz a checagem após um intervalo de 60 segundos (valor padrão). Este ciclo permanece até que o arquivo venha a existir ou um tempo limite seja atigindo (por padrão, 7 dias).

Podemos configurar o intervalo de reavalição através do argumento `poke_interval` que espera receber, em segundos, o período de espera entre cada checagem. Já o timeout é configurado por meio do argumento `timeout`.

```python
from airflow.sensors.filesystem import FileSensor

wait_for_file = FileSensor(
    task_id="wait_for_file",
    filepath="/data/file.csv",
    poke_interval=10,  # 10 seconds
    timeout=5 * 60,  # 5 minutes
)
```

!!! note "Nota"
	O `FileSensor` suporta [wildcards](#) (e.g. astericos [`*`]), o que nos permite criar padrões de correspondência nos nomes dos arquivos.

!!! info "Informação"
	Chamamos de *poking* a rotina realizada pelo sensor de execução e checagem contínua de uma condição.

### Condições Personalizadas

Há diversos cenários em que as condições de execução de um pipeline são mais complexas do que a existência ou não de um arquivo. Em situações como essa, podemos recorrer a uma implementação baseada em Python através do `PythonSensor` ou então criarmos nosso próprio sensor.

O `PythonSensor` assim como o `PythonOperator` executa uma função Python. Contudo, essa função deve retornar um valor booleano: `True` indicando que a condição foi cumprida, `False` caso contrário.

Já para criarmos nosso próprio sensor, basta estendermos a classe `BaseSensorOperator`. Informações sobre a criação de componentes personalizados são apresentados na seção [Criando Componentes Personalizados](https://ahayasic.github.io/apache-airflow-in-a-nutshell/content/going_deeper/creating_custom_components/)

### Sensores & Deadlock

O tempo limite padrão de sete dias dos sensores possui uma falha silenciosa.

Suponha uma DAG cujo `schedule_interval` é de um dia. Se ao longo do tempo as condições não foram atendidas, teremos um acumulo de DAGs e tarefas para serem executadas considerável.

O problema deste cenário é que o Airflow possui um limite máximo de tarefas que ele consegue executar paralalemente. Portanto, caso o limite seja atingido, as tarefas ficarão bloqueadas e nenhuma execução será feita. Este comportamento é denominado *sensor deadlock*.

Embora seja possível aumentar o número de tarefas executáveis em paralelo, as práticas recomendadas para evitar esse tipo de situação são:

- Definir um *timeout* menor que o `schedule_interval`. Com isso, as execuções irão falhar antes da próxima começar.
- Alterar a forma como os sensores são acionados pelo *scheduler*. Por padrão, os sensores trabalham no modo `poke` (o que pode gerar o *deadlock*). Através do argumento `mode` presente na classe do sensor em questão, podemos alterar o acionamento do sensor para `reschedule`.

    No modo `reschedule`, o sensor só permanecerá ativo como uma tarefa enquanto estiver fazendo as verificações. Ao final da verificação, caso o critério de sucesso não seja cumprido, o *scheduler* colocará o sensor em estado de espera, liberando assim uma posição (*slot*) para outras tarefas serem executadas.

## Triggando outras DAGs

## Exemplos
Para exemplos e casos de uso dos tópicos abordados nesta seção, acesse [Estudos de Caso](#).

Para detalhes técnicos e práticas de uso, acesse [Receitas & Códigos Padrões](#).

## Referências

- [Data Pipelines with Apache Airflow (2021) by Bas Harenslak and Julian de Ruiter](https://www.amazon.com.br/Data-Pipelines-Apache-Airflow-Harenslak/dp/1617296902)
- [Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/concepts/index.html)
- [Airflow Guides by Astronomer](https://www.astronomer.io/guides/)
- [Marc Lamberti Blog](https://marclamberti.com/blog/)