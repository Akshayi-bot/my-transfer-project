## Analysis and Solution

1. **dbt_project.yml**  
   The central configuration file for the dbt project, specifying the project's name, version, and overall model configurations. This file ties together various parts of the project and coordinates dbt's execution process.

2. **manifest.yml**  
   Defines environment-specific variables such as project IDs, service accounts, and the paths to the table configuration files, ensuring dbt can dynamically access the appropriate settings.

3. **models/schema.yml**  
   Defines the sources used in the dbt project, mapping the source table names to their identifiers in BigQuery, allowing dbt to access and use the correct datasets.

4. **models/m0_report/mo_report.sql**  
   This dbt model creates the `reconciliation_mo_report` table, performing data transformations and deduplication on various source tables from the `coreruby` dataset.

5. **models/m1_report/m1_report.sql**  
   Similar to the `mo_report.sql`, this model processes data from the `coreruby` tables, handles tax information, and prepares the data for reporting in the `reconciliation_m1_report` table.

6. **models/single/default.sql**  
   A generic dbt model that performs simple `SELECT *` operations, often used as a baseline for querying data without complex transformations.

7. **table_config.yml**  
   Used for the prod and dev environments, this file is similar to `preprod_table_config.yml` but tailored to different environments, defining source and target BigQuery details for each model.

8. **preprod_table_config.yml**  
   Contains configuration for the preprod (pre-production) environment, specifying source and target BigQuery details for each model to ensure proper data handling in a staging setup.

9. **dbt_command_loop.sh**  
   A shell script that automates the execution of dbt commands for each model, iterating through configuration files and executing the necessary transformations for each specified table.

## Issue and suggestions
1. **Excessive `dbt run` Commands**: The current approach requires running a separate `dbt run` command for each model, which is inefficient and increases overhead, ultimately slowing down the process. Dbt is optimized to build all models within a project in a single run, so this method can be streamlined.

2. **Redundant Configuration Files**: The use of separate configuration files for different environments adds unnecessary complexity and makes ongoing maintenance more difficult. It is more efficient to use dbt profiles and environment variables to manage environment-specific configurations, reducing redundancy.

## Expected outcome
Currently, the project utilizes `dbt_command_loop.sh` to execute individual `dbt run` commands. It takes the environment (prod, preprod, or dev) as an argument and loops through the respective configuration file (either `table_config.yml` or `preprod_table_config.yml`). For each table defined in the configuration, it constructs a `dbt run` command with the appropriate environment variables and model names. The expected outcome is the creation of the target tables in BigQuery, as specified in the configuration files.


## Solution to Eliminate Shell Script and Environment-Specific Config Files in dbt

## Objective

The goal is to eliminate the need for shell scripts (`dbt_command_loop.sh`) and separate configuration files for different environments (preprod, prod, dev) in your dbt project. By utilizing dbt's native configuration management tools like `profiles.yml`, `vars` in `dbt_project.yml`, and centralized YAML files, we can streamline the project setup and make it more maintainable.

## Steps to Achieve the Solution

### 1. Consolidate Configuration in `profiles.yml`

The `profiles.yml` file is used to define environment-specific configurations such as BigQuery project IDs, datasets, and credentials. Instead of creating separate configuration files for each environment, you can define multiple targets within a single profile, with different configurations for `dev`, `prod`, and `preprod`.

#### Example `profiles.yml`:

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: bigquery
      project: "{{ env_var('DBT_SRC_GCP_PROJECT') }}"
      dataset: "{{ env_var('DBT_GCP_DATASET') }}"
      threads: 4
      keyfile: "{{ env_var('DBT_KEYFILE') }}"
    prod:
      type: bigquery
      project: "{{ env_var('DBT_SRC_GCP_PROJECT') }}"
      dataset: "{{ env_var('DBT_GCP_DATASET') }}"
      threads: 8
      keyfile: "{{ env_var('DBT_KEYFILE') }}"
    preprod:
      type: bigquery
      project: "{{ env_var('DBT_SRC_GCP_PROJECT') }}"
      dataset: "{{ env_var('DBT_GCP_DATASET') }}"
      threads: 4
      keyfile: "{{ env_var('DBT_KEYFILE') }}"
```

In this setup, we have defined three targets (`dev`, `prod`, `preprod`) under the same profile. The values are dynamically fetched using the `env_var` function, which allows you to manage different environments by setting environment variables at runtime.

### 2. Use `vars` in `dbt_project.yml` for Table-Specific Configuration

To manage table-specific configurations (e.g., target table names), we can use the `vars` feature in `dbt_project.yml`. This approach eliminates the need for separate configuration files for each table.

#### Example `dbt_project.yml`:

```yaml
name: 'my_project'
version: '1.0'
profile: 'my_project'

models:
  my_project:
    mo_report:
      materialized: table
      target_table_name: "{{ var('DBT_TGT_TABLE_NAME', 'reconciliation_mo_report') }}"
    m1_report:
      materialized: table
      target_table_name: "{{ var('DBT_TGT_TABLE_NAME', 'reconciliation_m1_report') }}"
    consultation:
      materialized: table
      target_table_name: "{{ var('DBT_TGT_TABLE_NAME', 'appointment_consultation') }}"
    consultation_notes:
      materialized: table
      target_table_name: "{{ var('DBT_TGT_TABLE_NAME', 'appointment_consultation_notes') }}"
    patient:
      materialized: table
      target_table_name: "{{ var('DBT_TGT_TABLE_NAME', 'appointment_patient') }}"
```

In the above configuration, the target table name for each model is set using the `var()` function. This allows you to define the table names dynamically and configure them easily for different environments using `vars`.

### 3. Centralize Table Configuration Using a Single `config.yml`

Rather than having separate configuration files for each environment (e.g., `preprod_table_config.yml`, `table_config.yml`), you can consolidate all the table-specific configurations into a single `config.yml`. This can then be loaded into your dbt models dynamically using the `fromyaml()` function.

#### Example `config.yml`:

```yaml
mo_report:
  DBT_SRC_GCP_PROJECT: my-proprod-project
  DBT_SRC_GCP_DATASET: event
  DBT_GCP_DATASET: my_operational
  DBT_SRC_TABLE_NAME: coreruby_transactions_native
  DBT_TGT_TABLE_NAME: reconciliation_mo_report

m1_report:
  DBT_SRC_GCP_PROJECT: my-proprod-project
  DBT_SRC_GCP_DATASET: event
  DBT_GCP_DATASET: my_operational
  DBT_SRC_TABLE_NAME: coreruby_transactions_native
  DBT_TGT_TABLE_NAME: reconciliation_m1_report

consultation:
  DBT_SRC_GCP_PROJECT: my-proprod-project
  DBT_SRC_GCP_DATASET: my_domain_appointment
  DBT_GCP_DATASET: my_operational
  DBT_SRC_TABLE_NAME: consultation
  DBT_TGT_TABLE_NAME: appointment_consultation

consultation_notes:
  DBT_SRC_GCP_PROJECT: my-proprod-project
  DBT_SRC_GCP_DATASET: my_domain_appointment
  DBT_GCP_DATASET: my_operational
  DBT_SRC_TABLE_NAME: consultation_notes
  DBT_TGT_TABLE_NAME: appointment_consultation_notes

patient:
  DBT_SRC_GCP_PROJECT: my-proprod-project
  DBT_SRC_GCP_DATASET: my_domain_appointment
  DBT_GCP_DATASET: my_operational
  DBT_SRC_TABLE_NAME: patient
  DBT_TGT_TABLE_NAME: appointment_patient
```

This `config.yml` can be loaded into your dbt models as follows:

```sql
{% set config = fromyaml(load_file('config.yml'))[this.name] %}

SELECT *
FROM {{ source(config.DBT_SRC_GCP_PROJECT, config.DBT_SRC_TABLE_NAME) }}
```

By using `load_file()` and `fromyaml()`, you can dynamically load the configurations for each model directly from the `config.yml` file.

### 4. Remove Shell Script and Use dbt CLI with Environment Variables

Instead of relying on the shell script (`dbt_command_loop.sh`) to execute dbt commands for each model, you can directly use the `dbt run` command with the appropriate environment variables set for the environment.

#### Example Command to Run Models:

```bash
export DBT_ENVIRONMENT=dev
export DBT_SRC_GCP_PROJECT=my-proprod-project
export DBT_GCP_DATASET=my_operational
export DBT_KEYFILE=/path/to/your/keyfile.json
dbt run
```

This approach eliminates the need for the shell script. You simply run `dbt run` with the required environment variables set for the target environment (`dev`, `prod`, etc.).

## Conclusion

By consolidating environment-specific configurations in `profiles.yml`, using `vars` in `dbt_project.yml` for table-specific parameters, and centralizing table configurations in a single `config.yml`, you can eliminate redundant configuration files and shell scripts. This approach makes your dbt project easier to maintain and improves the overall efficiency of your workflow.

### Benefits:
- **Centralized Configuration**: All environment-specific settings are managed in `profiles.yml` and `dbt_project.yml`.
- **Dynamic Variables**: Use of `vars` and environment variables makes it easier to handle different configurations without redundant files.
- **No Shell Script Needed**: The dbt CLI can handle all models without the need for a looping shell script.

This solution streamlines the configuration management and enhances maintainability while reducing complexity.
