# pydabs-genie-agent-retarget

This project allows to rewrite Unity Catalog references in [Databricks Genie
Agent](https://docs.databricks.com/aws/en/genie-agents/) (formerly Genie space)
definitions when you deploy them with Declarative Automation Bundles (DABs).

A serialized Genie space hardcodes `catalog.schema.table` names in table identifiers, join
specs, SQL snippets, example and benchmark SQL, and text instructions.
`genie_agent_retarget` is a Python mutator for DABs that rewrites all of them per target
with prefix rules:

| Rule level | Example                                              |
|------------|------------------------------------------------------|
| table      | `main.intel.oss_iocs: ${var.catalog}.intel.iocs`     |
| schema     | `main.gold: ${var.catalog}.${var.gold_schema}`       |
| catalog    | `main: ${var.catalog}`                               |

The most specific rule wins, and each name is rewritten once (rules never chain). Matching
is case-insensitive and handles backtick-quoted names, in the space and in the rules. A
rule whose two sides have a different number of parts fails `databricks bundle validate`.

After a rewrite, the mutator sorts `data_sources.tables` by identifier, because the Genie
API rejects an unsorted list. If a table rule renames a table, the mutator also renames
the short table qualifiers in SQL fields: join specs, SQL snippets (filters, measures,
expressions), example SQL, and benchmark SQL (`oss_iocs.ioc_type = 'URL'` becomes
`iocs.ioc_type = 'URL'`). It skips a short name that another table of the space also has,
because it is ambiguous.

## Usage

1. Add the package to the bundle's virtual environment (with `uv`):

   ```toml
   [dependency-groups]
   dev = [
       "databricks-bundles==1.19.0",
       "pydabs-genie-agent-retarget",
   ]
   ```

   Or copy `src/genie_agent_retarget.py` next to `databricks.yml`. It has no
   dependencies except `databricks-bundles`.

2. Register the mutator and define the `genie_table_mapping` variable in
   `databricks.yml`:

   ```yaml
   python:
     venv_path: .venv
     mutators:
       - 'genie_agent_retarget:retarget_genie_spaces'

   variables:
     genie_table_mapping:
       type: complex
       default:
         main.gold: ${var.catalog}.${var.gold_schema}
   ```

3. Keep each space as `src/<name>.geniespace.json` (the output of
   `databricks bundle generate genie-space`) and reference it with `file_path`.

Use `genie_table_mapping: {}` in a target that needs no rewriting.

## Command line

The same code works on a single file. This is useful when you export a space from an
environment that is not the authoring one, and must convert it back to the authoring
names:

```sh
python -m genie_agent_retarget src/x.geniespace.json --in-place \
  -m dev_catalog.gold_dev=main.gold
```

## Example

`example/` deploys 3 Genie spaces (CVE analysis, IoC analysis, and Lakewatch security
operations over 22 OCSF gold tables). Each data source location is a variable
(`ocsf_schema`, `cve_schema`, `iocs_table`):

- `dev` moves the spaces to other schemas and renames one table.
- `azure` deploys them to another workspace, where the data is in other
  catalogs and schemas.
- `prod` keeps the names, so the files are deployed without changes.

```sh
cd example
uv sync
databricks bundle validate -t dev
databricks bundle deploy -t dev
```

The example uses names from our environments: the CLI profiles (`lw-field`, `az-alexott`),
the catalogs and schemas of each target, and the warehouse name.  The JSON files keep the
names they were authored against (`fe_lakewatch_catalog...`), because the rules map from
these names. To try the example in your workspace, change `workspace.profile`,
`warehouse_name`, and the `ocsf_schema`, `cve_schema`, and `iocs_table` variables of a
target to your tables. The tables must have the same columns, otherwise the instructions
and example SQL of the spaces do not match the data.

The Genie API checks that all tables exist when it creates or updates a space, so the
tables must exist before you deploy.

If the `DATABRICKS_CONFIG_PROFILE` environment variable is set, the CLI uses that profile
instead of the target's `workspace.profile`. Unset it when you deploy targets that use
different workspaces.

## Tests

```sh
uv sync && uv run pytest
```

## License

This project is licensed under the [Databricks License](LICENSE).
