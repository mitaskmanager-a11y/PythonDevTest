# Complete Guide: Installing Apache Airflow on WSL (Windows 11)

This guide provides a clean, production-style setup for installing WSL, Ubuntu, and Apache Airflow on Windows 11. It uses a virtual environment and constraint files to avoid the `externally-managed-environment` error and package conflicts.

## 1. Install WSL on Windows 11

Open **PowerShell as Administrator** and run:
```powershell
wsl --install
```
*This installs WSL2, Ubuntu (default), and virtualization components. Restart your PC after installation.*

Verify the installation (in PowerShell):
```powershell
wsl -l -v
```
*Ensure the state is "Running" and the version is "2".*

## 2. Initialize Ubuntu

Search for "Ubuntu" in your Windows Start menu and open it.
The first time you open it, you will be prompted to create a user:
*   **Enter new UNIX username:** (e.g., `data_engineer`)
*   **Enter new password:** (your password)

## 3. Update System & Install Python Prerequisites

Inside your Ubuntu terminal, update your package lists and install required Python development tools. Note the inclusion of `python3-full` and `python3-venv` to prevent PEP 668 (`externally-managed-environment`) errors.

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install python3-pip python3-venv python3-full python3-dev build-essential libssl-dev libffi-dev -y
```

Verify your Python version:
```bash
python3 --version
```
*(Expected: Python 3.10 or higher)*

## 4. Create and Activate a Virtual Environment

**Crucial Step:** Ubuntu blocks system-wide pip installations. You *must* install Airflow inside an active virtual environment. Never use the `--break-system-packages` flag.

```bash
# Create project folder
mkdir -p ~/airflow_project
cd ~/airflow_project

# Create the virtual environment
python3 -m venv airflow_venv

# Activate it
source airflow_venv/bin/activate
```
*Your terminal prompt should now start with `(airflow_venv)`.*

## 5. Install Airflow Using Constraints

With your virtual environment active, upgrade pip and install Airflow using the official constraint files to prevent dependency conflicts (e.g., SQLAlchemy, Flask, Pendulum mismatches).

```bash
# Upgrade pip
pip install --upgrade pip setuptools wheel

# Set version variables
AIRFLOW_VERSION=2.9.1
PYTHON_VERSION="$(python3 --version | cut -d ' ' -f 2 | cut -d '.' -f 1-2)"
CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# Install Airflow
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

## 6. Initialize Airflow

Set your Airflow home directory and initialize the backend database (SQLite by default).

```bash
export AIRFLOW_HOME=~/airflow
airflow db migrate
```
*(Note: `airflow db init` is deprecated in newer versions; use `airflow db migrate`)*

Create an Admin user for the web UI:
```bash
airflow users create \
    --username admin \
    --firstname Airflow \
    --lastname Admin \
    --role Admin \
    --email admin@example.com \
    --password admin
```

## 7. Start Airflow Services

You need two separate terminal windows/tabs to run Airflow locally.

**Terminal 1 (Scheduler):**
```bash
# Ensure you are in the virtual environment
source ~/airflow_project/airflow_venv/bin/activate
export AIRFLOW_HOME=~/airflow

airflow scheduler
```

**Terminal 2 (Webserver):**
Open a new Ubuntu terminal window.
```bash
# Activate environment again
source ~/airflow_project/airflow_venv/bin/activate
export AIRFLOW_HOME=~/airflow

airflow webserver -p 8080
```

## 8. Access the UI and Test a DAG

1. Open your Windows web browser and navigate to: `http://localhost:8080`
2. Log in using `admin` / `admin`.

**Create a test DAG:**
Your DAG files belong in `~/airflow/dags`.

```bash
mkdir -p ~/airflow/dags
nano ~/airflow/dags/test_dag.py
```

Paste the following Python code into the file:
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def hello():
    print("Hello from Airflow in WSL!")

with DAG(
    dag_id="test_dag",
    start_date=datetime(2024,1,1),
    schedule_interval="@daily",
    catchup=False
) as dag:

    task1 = PythonOperator(
        task_id="hello_task",
        python_callable=hello
    )
```
Save (`Ctrl+O`, `Enter`) and Exit (`Ctrl+X`). Refresh the Airflow UI to see your new DAG!

---

## 🚀 Next Steps for Production (Data Engineers)

To make this WSL setup behave closer to a real AWS/Production environment, consider configuring:
1. **PostgreSQL Backend:** Swap the default SQLite database for Postgres to allow parallel task execution.
2. **LocalExecutor:** Change the executor in `airflow.cfg` from `SequentialExecutor` to `LocalExecutor`.
3. **WSL Memory Limits:** Create a `.wslconfig` file in Windows to prevent the Airflow scheduler from consuming all system RAM and crashing.