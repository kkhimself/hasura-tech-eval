# Hasura Tech Eval

## Task One

#### Briefly describe the steps taken to configure your Hasura GraphQL Engine

1. Followed steps 1 to 4 in  https://hasura.io/docs/latest/hasura-cli/quickstart/ to set up a Hasura instance and add a data source.
2. Made the following changes to `docker-compose.yml`:
      1. Exposed the Postgres port to the host to allow connection from a Postgres client. 
      2. Moved `HASURA_GRAPHQL_ADMIN_SECRET` and `POSTGRES_PASSWORD` to `.env` file.
3. Created tables and added data from chinook database
      1. Created a migration
            ```bash
            hasura migrate create "create_tables" --database-name chinook
            ```
      2. Copied the DDL statements to create tables and foreign keys from https://github.com/lerocha/chinook-database/releases/download/v1.4.5/Chinook_PostgreSql_SerialPKs.sql to `up.sql`, updated `down.sql` with scripts to drop the corresponding tables, and applied the migration.
            ```bash
            hasura migrate apply --database-name chinook-db
            ```
      3. Created a seed file to populate data

            ````bash
            hasura seed create seed_tables --database-name chinook
            ````
      4. Copied the scripts to populate tables from https://github.com/lerocha/chinook-database/releases/download/v1.4.5/Chinook_PostgreSql_SerialPKs.sql and applied the seed file.
            ```bash
            hasura seed apply --database-name chinook
            ```

#### GraphQL query and results set for each of the above statements

#### Metadata in YAML format

#### Describe any challenges you encountered and your troubleshooting steps to address them

#### SQL statements used directly against Postgres


## Task Two