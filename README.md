# Apache Airflow — Simple End-to-End Notes

This guide explains **Apache Airflow** in very simple terms, with a hands-on example you can run. Perfect if you are new to Airflow.

---

## What is Airflow?

Airflow is a tool that lets you **automate workflows** (like ETL pipelines).

* You write Python code to describe your workflow.
* The workflow is a **DAG** = Directed Acyclic Graph (a chain of tasks).
* Airflow runs tasks in the right order, retries if they fail, and gives you a nice web UI to monitor.

Think of Airflow as a **smart scheduler for data pipelines**.

---

## How Airflow Works

![Airflow architecture](https://airflow.apache.org/docs/apache-airflow/stable/_images/diagram_basic_airflow_architecture.png)

* **Webserver** → the UI you use.
* **Scheduler** → decides when tasks run.
* **Workers** → actually run your tasks.
* **Database** → keeps track of what happened.
* **DAG folder** → where you keep your Python workflow files.

---

## Installing Airflow (quick way)

```bash
# Create virtual environment
python -m venv .venv
source .venv/bin/activate

# Install airflow with right constraints
pip install "apache-airflow==2.9.3" \
  --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.9.3/constraints-3.10.txt"

# Setup airflow home
airflow db init

# Create admin user
airflow users create \
  --username admin --password admin \
  --role Admin --email admin@example.com

# Start webserver & scheduler
airflow webserver --port 8080
airflow scheduler
```

Now open: [http://localhost:8080](http://localhost:8080) and login with `admin/admin`.

---

## Example Project Structure

```
├─ dags/
│  └─ etl_orders.py   # our DAG file
├─ data/
│  ├─ raw/            # input files created by DAG
│  └─ output/         # output reports created by DAG
```

---

## Example: Mini ETL Pipeline

This simple pipeline will:

1. Create some fake order data (CSV file).
2. Summarize total revenue per country.
3. Save a report (CSV + Markdown summary).

### Install extra Python library

```bash
pip install pandas
```

### The DAG (`dags/etl_orders.py`)

```python
from datetime import datetime
from pathlib import Path
import pandas as pd
import random

from airflow.decorators import dag, task

BASE = Path.cwd()
DATA = BASE / "data"
RAW = DATA / "raw"
OUT = DATA / "output"
RAW.mkdir(parents=True, exist_ok=True)
OUT.mkdir(parents=True, exist_ok=True)

@dag(
    dag_id="etl_orders",
    start_date=datetime(2024, 1, 1),
    schedule="@daily",   # runs daily
    catchup=False,
)
def etl_orders():

    @task
    def generate_orders():
        rows = []
        for i in range(10):
            rows.append({
                "order_id": i+1,
                "customer": random.choice(["Alice", "Bob", "Cara"]),
                "country": random.choice(["US", "IN", "DE"]),
                "amount": round(random.uniform(10,200),2),
            })
        df = pd.DataFrame(rows)
        csv_path = RAW / "orders.csv"
        df.to_csv(csv_path, index=False)
        return str(csv_path)

    @task
    def transform(csv_path: str):
        df = pd.read_csv(csv_path)
        report = df.groupby("country")["amount"].sum().reset_index()
        out_path = OUT / "daily_report.csv"
        report.to_csv(out_path, index=False)
        return str(out_path)

    @task
    def publish(report_path: str):
        df = pd.read_csv(report_path)
        print("Report saved:", report_path)
        print(df)

    csv_path = generate_orders()
    report_path = transform(csv_path)
    publish(report_path)

etl_orders()
```

---

## What Happens When You Run It

* **generate\_orders** → creates `data/raw/orders.csv`
* **transform** → makes `data/output/daily_report.csv`
* **publish** → prints summary in logs

Example report CSV:

```csv
country,amount
DE,350.25
IN,500.10
US,420.30
```

---

## Operators in Airflow

Operators are building blocks for tasks. Some common ones:

* **PythonOperator** → run Python functions.
* **BashOperator** → run shell commands.
* **EmailOperator** → send emails.
* **DummyOperator** → placeholder task.
* **Sensor** → wait for a condition (like file exists).

Example:

```python
from airflow.operators.bash import BashOperator

list_files = BashOperator(
    task_id="list_files",
    bash_command="ls -l data/output"
)
```

---

## Dependencies Setup

Dependencies define the order of execution.

Ways to set dependencies:

```python
# Method 1: chain style
t1 >> t2 >> t3

# Method 2: set_downstream/set_upstream
t1.set_downstream(t2)
t2.set_upstream(t3)

# Method 3: TaskFlow API style
csv_path = generate_orders()
report_path = transform(csv_path)
publish(report_path)
```

Example Graph in UI:
![Airflow DAG Graph](https://airflow.apache.org/docs/apache-airflow/stable/_images/dags.png)

---

## Scheduling

* Airflow can run tasks **manually** or on a **schedule**.
* Common schedules:

  * `@daily` → once per day
  * `@hourly` → once per hour
  * Cron format → e.g. `0 6 * * *` = 6 AM every day

---

## XCom (Passing Data Between Tasks)

* Airflow tasks can pass small pieces of data with **XCom**.
* In TaskFlow API (`@task`), return values are stored automatically as XComs.

Example:

```python
@task
def step1():
    return {"filename": "data/raw/orders.csv"}

@task
def step2(info):
    print("Reading", info["filename"])
```

---

## Variables & Connections

* **Variables** = key/value pairs stored in Airflow (for configs like `bucket_name`).
* **Connections** = store database credentials or API keys.
* Manage them in the **Admin → Variables/Connections** UI.

Example:

```python
from airflow.models import Variable
my_key = Variable.get("MY_API_KEY")
```

---

## Executors (How Tasks Run)

* **SequentialExecutor**: runs tasks one at a time (good for learning).
* **LocalExecutor**: runs tasks in parallel locally.
* **CeleryExecutor**: uses workers across machines.
* **KubernetesExecutor**: runs each task as a Kubernetes pod.

---

## Monitoring & Logs

* UI → shows **Tree View**, **Graph View**, **Logs**.
* Logs stored under `airflow_home/logs`.
* Can be sent to S3/GCS for production.

---

## Reliability Features

* **Retries**: Airflow retries tasks if they fail.
* **SLA**: flag if a task takes too long.
* **Trigger Rules**: e.g., run task only if all upstream succeed or fail.

---

## Real-World Example Operators Together

```python
from airflow.operators.email import EmailOperator
from airflow.operators.bash import BashOperator
from airflow.operators.python import PythonOperator

# Example Python task
py_task = PythonOperator(
    task_id="process_data",
    python_callable=lambda: print("Processing done")
)

# Example Bash task
bash_task = BashOperator(
    task_id="list_outputs",
    bash_command="ls -lh data/output"
)

# Example Email task
email_task = EmailOperator(
    task_id="send_email",
    to="user@example.com",
    subject="Pipeline Complete",
    html_content="<p>Your ETL finished successfully!</p>",
)

py_task >> bash_task >> email_task
```

---

## Useful Commands

```bash
# List all DAGs
airflow dags list

# Trigger our DAG manually
airflow dags trigger etl_orders

# See task logs
airflow tasks logs etl_orders publish 2024-01-01
```

---

## Key Ideas (remember!)

* **DAG = workflow**
* **Task = step in workflow**
* **Operator = how the task runs**
* **Dependencies = order of tasks**
* **Scheduler + Workers** run tasks automatically
* **UI** shows DAGs, logs, reports
* **Variables/Connections** store config & credentials
* **XCom** passes small data between tasks

---

Now you can run your first Airflow DAG 🎉
