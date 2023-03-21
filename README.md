# project_management_statistics

The motivation for this project is based on the challenges and limitations encountered while generating metrics using Telegraf and directly querying APIs.
These challenges highlight the need for a more efficient and robust solution to capture historical data and calculate meaningful metrics.

We are not implying that the Telegraf approach is worst. Telegraf serves its intended purpose effectively; however, the project we are proposing offers a 
potentially more effective and comprehensive solution for designing and exploring new metric formulas. By capturing and storing historical data,
this project aims to enhance the process of generating metrics by addressing the limitations observed in the Telegraf approach.

Here are the key motivating factors for this project:

- **Delayed metric validation**: When calculating metrics from the moment a script starts running, it may take a day or two of collected data to verify
if the metric works as expected. With historical information, you can validate the metric during the design phase and deploy a well-crafted metric to production.
- **Difficulty in metric modification**: If you need to modify a metric, the changes will only apply from the moment of implementation,
resulting in inconsistent historical metrics. With historical data, you can recalculate the entire metric using the updated formula, ensuring consistency throughout.
- **API limitations**: Directly querying APIs often comes with security restrictions and response speed constraints, which can hinder the calculation of complex metrics.
- **Present moment limitations**: With these API limitations and using Telegraf with scripts that directly query APIs, the calculated metrics are based on the current state of the issues.
- **Missing metrics during downtime**: If Telegraf is offline for a period, the metrics generated during that time are lost,
as the scripts do not consider history and only capture the present moment.
- **Limited complexity and trends analysis**: Without historical data, it becomes challenging to create complex metrics based on trends or issue evolution over time.


## Structure

This project uses a database schema to store and manage issues from different issue tracking systems. 
The schema is illustrated in the following diagram:

```mermaid
graph TD
    A(fa:fa-cloud Progress) -->|Issues| B[fa:fa-gear Progress Mirroring]
    C(fa:fa-cloud Bugzilla) -->|Issues| D[fa:fa-gear Bugzilla Mirroring]
    B -->|std Issues| E(fa:fa-database Database)
    D -->|std Issues| E
    E -->|std Issues| F[fa:fa-gear Querying]
    F -->|std Issues| G[fa:fa-gear Metrics]
    G -->|statistics| H[fa:fa-gear Formating]
    G -->|statistics| H2[fa:fa-gear Formating]
    G -->|statistics| H3[fa:fa-gear Formating]
    H -->|influx input line format| I(fa:fa-database Influxdb)
    I --> J(fa:fa-chart-pie Grafana)
    H2 -->|CSV| LibreOffice(Libreoffice/Excel)
    H3 -->|JSON| Others
```

### Database

Using a database with a normalized model offers several benefits over directly connecting to APIs:

- **Enhanced performance**: By storing issues in a local (or remote) database, you can bypass limitations imposed by APIs, 
such as maximum query rates per second. This enables faster access to the information you need.
- **Scalable data retrieval**: By leveraging a database, you can fetch a larger number of issues per query,
allowing you to handle a growing number of tickets with ease. Avoiding an other one possible limitation of APIs.
- **Complete history for metric exploration**: A database enables you to store and maintain the complete history of issue changes for a project,
providing valuable context when exploring and designing new metrics. 
This comprehensive historical data allows you to instantly overview the evolution of a metric without waiting for additional data collection.
As a result, you can quickly refine and adjust the metric during the design phase, ensuring that it accurately reflects the desired insights and trends.
- **Simplified data extraction**: With a normalized model, you can consolidate multiple queries into a single request, 
streamlining the data extraction process and reducing the complexity of your code.
- **Cross-platform compatibility**: A normalized model ensures compatibility between different issue tracking systems,
making it easier to integrate multiple platforms and facilitating collaboration between teams using different tools.
- **Consistent data formats**: Normalizing the data in a database guarantees a consistent format for all issues,
simplifying data analysis and reporting while reducing the need for manual data normalization. 
This consistent format is beneficial not only for product owners and scrum masters who usally manage a single project,
but also for managers who need to maintain an overview of multiple projects across the department.
By having a unified data format, managers can efficiently track progress, evaluate performance, and make data-driven decisions across all projects,
ensuring better alignment with organizational goals and objectives.

### Normalized Model

By utilizing a normalized model for Issue, IssueEvent, and IssueAttribute, we can combine different sources into the same metrics.
This is particularly useful for teams working with different ticket systems and management perspectives. Or for managers to have an overview of the department

### Issue

The Issue model contains the common fields expected from a ticket system and provides the last status (and snapshot) of the ticket. These fields include:

- **issue_id**: A unique identifier for the issue within its specific ticket system.
- **created_on**: The date and time when the issue was created.
- **start_date**: The date and time when work on the issue began (usually is considered that the issue is "in progress" from this point).
- **closed_on**: The date and time when the issue was closed.
- **status**: The current status of the issue, such as "open", "in progress", or "closed".
- **priority**: The priority level of the issue, which can help in determining the order of work.
- **context_key**: A string used to filter issues that belong to different groups or contexts. In different ticket systems, this context can vary. 
For example, in the Progress ticket system, the context may be the project, while in Bugzilla, it could be a combination of the product, version, etc.

The other fields are:

- **author**: The name of the user who created the issue.
- **assigned_to**: The name of the user to whom the issue is currently assigned.
- **subject**: A brief description or title of the issue.
- **due_date**: The date and time when the issue is expected to be completed.
- **estimated_hours**: The estimated number of hours required to complete the issue.
- **updated_on**: The date and time when the issue was last updated.

By storing these fields in the Issue model, you can efficiently filter, sort, and analyze issues from various ticket systems in a consistent format.

#### Context key
The context_key is designed to help you filter and group issues that belong to different contexts or groups, such as projects, products, or versions.
It is essential to define the context_key in a consistent and meaningful way to make it easy to work with issues from various ticket systems.

To define the context_key, you can follow a standard format, combining relevant context elements and separating them with a delimiter
like a period (.) or a hyphen (-). In the example you provided, the context_key is created by concatenating the product,
version, and an additional numeric identifier:

e.g. product1.version1-4

Here's a step-by-step guide to defining a context_key:

- Identify the relevant context elements for your issue tracking system. For example, in Bugzilla, you may want to include the product and version,
while in the Progress, you might only need the project.
- Choose a consistent delimiter to separate the elements in the context_key. Common delimiters include periods (.), hyphens (-), or underscores (_).
- Combine the context elements using the chosen delimiter to form the context_key.

By consistently defining the context_key in this manner, you can easily filter, sort, and analyze issues that belong to different contexts while 
maintaining compatibility between different ticket systems.

### IssueAttribute

In addition to the main fields stored in the Issue model, you may encounter other attributes specific to a particular ticket system or relevant to your unique use case.
To accommodate these additional attributes, you can use the IssueAttribute model.

The IssueAttribute model is designed to store key-value pairs associated with an issue. Each entry in this model includes an issue_id (which links it to the corresponding issue),
a key representing the attribute's name, and a value representing the attribute's content.

By storing these additional attributes in the IssueAttribute model, you can capture and preserve valuable information that is not part of the common fields in the Issue model.
This flexible approach allows you to store custom attributes from various ticket systems, enabling more in-depth filtering, analysis, and reporting.

For example, if you encounter a custom attribute called "Version" in one of the ticket systems, you can store it in the IssueAttribute model as follows:

- issue_id: The ID of the issue that has the "Version" attribute.
- key: "Version"
- value: The actual value of the "Version" attribute.

This approach ensures that all relevant data is captured and stored, allowing you to create more accurate and comprehensive metrics and reports.

### IssueEvent

The IssueEvent model plays a crucial role in capturing the history of changes that occur to an issue throughout its lifecycle.
By recording these events, you can better understand the evolution of issues, analyze trends, and assess the effectiveness of your team's processes.

The IssueEvent model includes the following fields:

- **issue_id**: A foreign key reference to the associated issue in the Issue model.
- **user_name**: The name of the user who performed the change.
- **created_on**: The date and time when the event occurred.
- **type**: The type of change, such as "status change", "priority change", or "assignment".
- **field**: The field that was modified during the event.
- **old_value**: The previous value of the modified field before the event occurred.
- **new_value**: The updated value of the modified field after the event occurred.

By storing issue events in the IssueEvent model, you can analyze the historical changes and trends for each issue, such as the frequency of status changes,
priority adjustments, or user assignments. This information can help you identify potential areas for improvement, and the effectiveness of your team's processes.
Additionally, maintaining a record of issue events can provide valuable context when exploring and designing new metrics, ensuring that they accurately reflect the desired insights and trends.

### Database System

The current database system is SQLite, which is native to Python and easily portable. However, this can be replaced with any SQL (MariaDB, PostgreSQL, etc.) or NoSQL (MongoDB, etc.) system.

## Usage from user perspective

### Dumping progress issues into local DB for further processing

In this section, we'll focus on setting up the environment to dump Progress issues into a local database for further processing. Follow these steps to get started:

Create a Redmine configuration file as a Python file, for example, config.py.
Define the following variables in this file:
- **REDMINE_UR**: The URL to target Redmine instance from where tickets will be dumped.
- **REDMINE_KEY**: The API key that allows access to the REDMINE_URL.
- **REDMINE_QUERY_RATIO**: The maximum number of requests per minute to the Redmine API.

Execute the following command, replacing <your_redmine_config_path>, <project_name>, and <sqlite_file> with appropriate values

```bash
podman run  -ti --rm -v <your_redmine_config_path>:/pms/redmine/config.py pms "./dumper.py --project <project_name> --database <sqlite_file> "
```

Once you've completed the steps, you will obtain an SQLite file containing the dumped database with all the relevant Progress issues. If the database already exists, 
the dumper script will intelligently update it with the latest information from Redmine, ensuring that your local database remains up-to-date for further processing and analysis.

Consider reusing the previous dumped DB file as this script is designed to update the existing database in a way that minimizes API calls. By optimizing the update process,
the script reduces the load on the Redmine API and helps prevent reaching any imposed rate limits, ensuring smooth and efficient data synchronization.


### Generating metrics

Currently, this project includes a single script, metrics_by_period.py, which allows users to extract metrics for a specified period or the entire dataset. 
The script supports the status_count metric and can output the results in various formats.

The script accepts the following parameters:

- **start_date**: Start date (YYYY-MM-DD). If not provided, defaults to the epoch.
- **end_date**: End date (YYYY-MM-DD). If not provided, defaults to today.
- **database**: Name of the database file to write results to.
- **metric**: Name of the method to apply to each date. Defaults to "status_count".
- **output_format**: Output format. Valid options are "json", "influxdb", and "csv". Defaults to "influxdb".
- **measurement_name**: The name of the measurement used in InfluxDB.

With these parameters, users can customize the metric extraction process according to their needs, making it easy to generate the desired output format and apply the appropriate metric method.

As the project expands, more scripts can be added to further enhance the user experience and capabilities.


```bash
python metrics_by_period.py --start_date 2022-03-01 --end_date 2023-03-30 --database <sqlite_file>  --metric status_count --output_format influxdb --measurement_name my_measurement
```

## For Developers

## Querying

Import SQLiteQuery

```
from db.sqlite_query import SQLiteQuery
```

Create the object

```
query = SQLiteQuery(<file>)
```

### Methods


#### query.issues(**filters)
Get all issues
params:
* filter: A dict of filters e.g. project=project1
return: A list of issues

#### query.status_snapshot(date, **filters)
Get all the active and resolved issues with the status in a especific moment
params:
* date (datetime): The date for the snapshot. None means the last values
* *filter: A dict of filters e.g. project=project1
return: A list of issues

e.g.

```
result = query.status_snapshot(date=datetime(2023, 1, 1), project=1)
```

#### query.issues_active_in_period(date_in, date_out, **filters)
Get all the issues that are active (not closed) during a period of time
params:
* date_in (datetime): The begining of the period
* date_out (datetime): The end of the period
* filter: A dict of filters e.g. project=project1
return: A list of issues


## Metrics

Import Metrics and SQLiteQuery if is necessary
```
from metrics.metrics import Metrics
from db.sqlite_query import SQLiteQuery
```

Create the objects
```
query = SQLiteQuery(<file>)
metrics = Metrics(query)
```

### Methods

#### metrics.status_count(**filters)
Classify the issues by status and return the ammount in each category
params:
* filter: A dict of filters e.g. project=project1

return: A dict with the format <status:count>

### status_count_by_date(date, **filters)
Classify the issues by status and return the ammount in each category
params:
* filter: A dict of filters e.g. project=project1

return: A dict with the format <status:count>


## Formatters

The formatters allow to convert a result from the metrics to an specific format.

- InfluxDB input line format
- CSV
- JSON

Use the static method format to convert a MetricsResult (result from metrics queries)
into the format recognized by influxdb, CSV, or JSON for populating a database or other systems.

#### MetricsInfluxdbFormatter
```
from metrics.influxdb_formatter import MetricsInfluxdbFormatter
MetricsInfluxdbFormatter.format(measurement_name: str, metrics: MetricsResults)
```

params:
* measurement_name: Measurement name for the influx db
* metrics: MetricsResults
return: A formated string compabible with influxDB

#### MetricsCSVFormatter
```
from formatters.csv_formatter import MetricsCSVFormatter
MetricsInfluxdbFormatter.format(measurement_name: str, metrics: MetricsResults, separator: str = ",")
```

params:
* measurement_name: Measurement name for the influx db
* metrics: MetricsResults
* separator: str
return: A CSV formated string

#### MetricsJSONFormatter
```
from formatters.csv_formatter import MetricsCSVFormatter
MetricsInfluxdbFormatter.format(measurement_name: str, metrics: MetricsResults)
```

params:
* measurement_name: Measurement name for the influx db
* metrics: MetricsResults
* separator: str
return: A JSON string

JSON schema
```
{
    name: str,
    filters: dict,
    values: [ {
        values: dict,
        date: str(ISO 8601 time format)
    }]
}
```
e.g. `{'name': 'test', 'filters': {'f1': 1}, 'values': [{'values': {'v1': 1}, 'date': '2023-03-14T13:31:06.596142'}]}`


## Containers

### Create the containers

There are four alternatives:

- podman-container: Create the main container using podman
- podman-container-test: Create the container for testing using podman
- docker-container: Create the main container with docker
- docker-container-test: Create the container for testing using docker

e.g.

```bash
make docker-container
```

The containers created are:

- **pms**: For the main container
- **pms_test**: For the testing container

### Execution

With podman or docker

```bash
podman run --rm pms <command>
```

### Testing

With podman or docker

```bash
podman run --rm pms_test make test
```

### Development in a container

With podman or docker

```bash
podman run -ti --rm -v <your_code_path>:/pms pms_test bash
```

### Development in situ

#### Prepare the venv

```bash
python3 -m venv venv
source venv/bin/activate
pip3 install -r requirements_test.txt
```

#### Testing with redmine

If you have already a redmine/config.py skip this step
```bash
cp ./redmine/config_example.py ./redmine/config.py
```

Run the tests with
```bash
make test
```