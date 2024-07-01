# Hasura Tech Eval

## Task One

#### Briefly describe the steps taken to configure your Hasura GraphQL Engine

1. Followed steps 1 to 4 in  <https://hasura.io/docs/latest/hasura-cli/quickstart/> to set up a Hasura instance and add a data source.
2. Made the following changes to `docker-compose.yml`:
      1. Exposed the Postgres port to the host to allow connection from a Postgres client.
      2. Moved `HASURA_GRAPHQL_ADMIN_SECRET` and `POSTGRES_PASSWORD` to `.env` file.
3. Created tables and added data from chinook database
      1. Created a migration

            ```bash
            hasura migrate create "create_tables" --database-name chinook
            ```

      2. Copied the DDL statements to create tables and foreign keys from <https://github.com/lerocha/chinook-database/releases/download/v1.4.5/Chinook_PostgreSql_SerialPKs.sql> to `up.sql`, updated `down.sql` with scripts to drop the corresponding tables, and applied the migration.

            ```bash
            hasura migrate apply --database-name chinook-db
            ```

      3. Created a seed file to populate data

            ````bash
            hasura seed create seed_tables --database-name chinook
            ````

      4. Copied the scripts to populate tables from <https://github.com/lerocha/chinook-database/releases/download/v1.4.5/Chinook_PostgreSql_SerialPKs.sql> and applied the seed file.

            ```bash
            hasura seed apply --database-name chinook
            ```

      5. Configured row-level permissions on the `album` table by adding this configuration to the file `metadata\databases\chinook\tables\public_album.yaml`

          ```yaml
          select_permissions:
            - role: artist
              permission:
                columns:
                  - album_id
                  - title
                  - artist_id
                filter:
                  artist_id:
                    _eq: X-Hasura-Artist-Id
              comment: ""
          ```

#### GraphQL query and results set for each of the above statements

1. How many artists are in the database?

      **Query**:

      ```graphql
      query artist_count {
        artist_aggregate {
          aggregate {
            count
          }
        }
      }
      ```

      **Response**:

      ```json
      {
        "data": {
          "artist_aggregate": {
            "aggregate": {
               "count": 275
            }
          }
        }
      }
      ```

2. List the first track of every album by every artist in ascending order.

      Added `artist -> album` and `album -> track` foreign key references on the Hasura Console. Since the *track* table does not have a column that indicates the track number within the album, I will use the first track in the album by ascending order of track title.

      **Query**:

      ``` graphql
      query first_track {
        artist(order_by: {name: asc}) {
          artist_id
          name
          albums(order_by: {title: asc}) {
            album_id
            title
            tracks(limit: 1, order_by: {name: asc}) {        
              track_id
              name
            }
          }
        }
      }
      ```

      **Response**:

      ```json
      {
        "data": {
          "artist": [
            {
              "artist_id": 230,
              "name": "Aaron Copland & London Symphony Orchestra",
              "albums": [
                {
                  "album_id": 296,
                  "title": "A Copland Celebration, Vol. I",
                  "tracks": [
                    {
                      "track_id": 3427,
                      "name": "Fanfare for the Common Man"
                    }
                  ]
                }
              ]
            },
            {
              "artist_id": 202,
              "name": "Aaron Goldberg",
              "albums": [
                {
                  "album_id": 267,
                  "title": "Worlds",
                  "tracks": [
                    {
                      "track_id": 3357,
                      "name": "OAM's Blues"
                    }
                  ]
                }
              ]
            }
            ***ABBREVIATED FOR BREVITY***
          ]
        }
      }
      ```

3. Get all albums for artist id = 5 without specifying a where clause.

    Configured row-level permissions on the artist table using the header `X-Hasura-Artist-Id` as described earlier.

    Passed `X-Hasura-Artist-Id` and `X-Hasura-Role` as HTTP headers to the GraphQL Query

    ```http
    X-Hasura-Artist-Id: 5
    X-Hasura-Role: artist
    ```

    **Query**:

    ```graphql
    query AlbumsByArtist {
      album {
        artist_id
        album_id
        title
      }
    }
    ```

    **Response**:

    ```json
    {
      "data": {
        "album": [
          {
            "artist_id": 5,
            "album_id": 7,
            "title": "Facelift"
          }
        ]
      }
    }
    ```

    This could be accomplished using a user-defined SQL function as well, but you will have to use a *WHERE* clause in the SQL query.

4. Using a GraphQL mutation, add your favorite artist and one of their albums that isn’t in the dataset.

    **Mutation**:

    ``` graphql
    mutation AddArtistAndAlbum {
      insert_artist(objects: {name: "Sir Mix-a-Lot", albums: {data: {title: "Mack Daddy"}}}) {
        returning {
          artist_id
          name
          albums {
            album_id
            title
          }
        }
      }
    }
    ```

    **Response**:

    ```json
    {
      "data": {
        "insert_artist": {
          "returning": [
            {
              "artist_id": 276,
              "name": "Sir Mix-a-Lot",
              "albums": [
                {
                  "album_id": 348,
                  "title": "Mack Daddy"
                }
              ]
            }
          ]
        }
      }
    }
    ```

5. How did you identify which ID to include in the mutation?

      I did not have to use an ID. The database tables are configured to auto-generate IDs and the `artist` table has an *array relationship* configured to the `album` table through a foreign key. I inserted the artist and the album together in a single mutation.

#### Metadata in YAML format

[metadata](./task-one/metadata/)

#### Describe any challenges you encountered and your troubleshooting steps to address them

1. When trying to connect a DB client to the database, I realized that the port of the postgres service was not exposed to the host, so I had to update *docker-compose.yml* to expose it.
2. Listing the first track of every album by every artist, if implementing SQL, is usually accomplished with a *GROUP BY* clause. Since Hasura does not support *GROUP BY*, I considered creating a database view, before realizing that I could just set up object relationships between artist, album and track tables and using a nested GraphQL query.
3. When filtering albums by artist id using row-level permissions, upon passing *X-Hasura-Artist-Id* in the HTTP Header, the results did not filter. Then I realized I had to send *X-Hasura-Role* as well.

#### SQL statements used directly against Postgres

1. Return the artist with the most number of albums

    ```sql
    SELECT ar.artist_id, ar."name"
    FROM artist ar
    INNER JOIN album al ON
      al.artist_id = ar.artist_id
    GROUP BY
      ar.artist_id, ar."name"
    ORDER BY
      count(al.album_id) DESC
    LIMIT 1;
    ```

2. Return the top three genres found in the dataset in descending order

    ```sql
    SELECT t.genre_id, g."name", count(t.track_id)
    FROM track t
    INNER JOIN genre g ON
      t.genre_id = g.genre_id
    GROUP BY
      t.genre_id, g."name"
    ORDER BY
      count(t.track_id) DESC
    LIMIT 3;
    ```

3. Return the number of tracks and average run time for each media type

    ```sql
    SELECT t.media_type_id, mt."name", count(t.track_id), avg(t.milliseconds)
    FROM track t
    INNER JOIN media_type mt ON
      mt.media_type_id = t.media_type_id
    GROUP BY
      t.media_type_id, mt."name"
    ORDER BY
      t.media_type_id;
    ```

## Task Two

#### Describe any issues you discovered with the shared deployment artifacts and the steps you took to remediate the issue(s)

1. **Issue**: `docker-compose.yaml` did not include initialization scripts for the database.

    **Remediation**:

    Created an `init.sql` file using the Chinook dataset and mapped it to the `/docker-entrypoint-initdb.d` directory of the *postgres* image in `docker-compose.yaml`

2. **Issue**: Password and database name were incorrect in the `HASURA_GRAPHQL_METADATA_DATABASE_URL` environment variable of `graphql-engine` service.

    **Remediation**:

    Moved the Postgres password to a `.env` file and corrected all connection strings in `docker-compose.yaml`.

3. **Issue**: Docker Compose Error: `manifest for hasura/graphql-engine:v2.48.2-pro not found: manifest unknown: manifest unknown`

    **Remediation**:

    Changed the image in `docker-compose.yaml` to `hasura/graphql-engine:v2.40.2`, the latest available non-beta version.

4. **Issue**: Container port of the graphql-engine service in `docker-compose.yaml` was incorrect and the host port did not match `endpoint` URL in `config.yaml`.

    **Remediation**:

    Correct container port of the graphql-engine service in `docker-compose.yaml` to `8080` and correct `endpoint` in `config.yaml` to `http://localhost:8111`.

5. **Issue**: `hasura metadata apply` throws this error: `error: open C:\\...\\hasura\\metadata\\databases\\Neon\\tables\\tables.yaml: The system cannot find the path specified."`

    **Remediation**:

    Corrected paths in `metadata\databases\databases.yaml`

6. **Issue**: `"error applying metadata \n{\n  \"error\": \"parsing boolean expression failed, expected Object, but encountered Array\",\n  \"path\": \"$.args.metadata.sources[0].tables[1].select_permissions[0].permission.filter\",\n  \"code\": \"parse-failed\"\n}"`

    **Remediation**: Corrected the value of `select_permissions.permission.filter` by removing the `-` before the colum name in the file `metadata\databases\PG\tables\public_artists.yaml`. While I was there, I noticed that the correct column name should be `album_id` and not `id`, so I corrected it too.

7. **Issue**: `WARN Metadata is inconsistent` with several errors like `Inconsistent object: no such table/view exists in source: "media_types"`

    **Remediation**: The table names in the database are singulars while the metadata refers to the plural forms of the names. Corrected the table names in the .yaml files under `metadata\databases\PG\tables\`

8. **Issue**: `WARN Metadata is inconsistent` with several errors like `Inconsistent object: in table  "album": in permission for role "artist": "id" does not exist`

    **Remediation**: The *id* column names in the database are prefixed with the table name like `artist_id` while the metadata refers columns without the prefix like `id`. Corrected the column names in the .yaml files under `metadata\databases\PG\tables\`.

9. **Issue**: `Inconsistent object: in function "custom_sql_artists": no such function exists: "custom_sql_artists"`

    **Remediation**:

    I have not fixed this issue yet as I could not find the DDL for the SQL function `custom_sql_artists`.

10. **Issue**: Errors related to *admin_secret*

    **Remediation**:

    Moved the `HASURA_GRAPHQL_ADMIN_SECRET` in `docker-compose.yaml` to an environment variable and removed `admin_secret` from `config.yaml`.

#### Include the requested artifacts from each above question 

1. `query getTracks` as *administrator*

    In the metadata, all tables are prefixed with *music_*, table names are plurals and *id* columns are prefixed with the table name. Corrected the query to account for these.

    **Headers**:

    None other than `x-hasura-admin-secret` and `content-type`.

    **Query**:

    ```graphql
    query getTracks($genre: String, $limit: Int, $offset: Int) {
      music_track(limit: $limit, offset: $offset, where: {genre: {name: {_eq: $genre}}}) {
        name
        track_id
      }
    }
    ```

    **Variables**:

    ```json
    {
      "genre": "Metal",
      "limit": 5,
      "offset": 50
    }
    ```

    **Result**:

    ```json
    {
      "data": {
        "music_track": [
          {
            "name": "Trupets Of Jericho",
            "track_id": 190
          },
          {
            "name": "Machine Men",
            "track_id": 191
          },
          {
            "name": "The Alchemist",
            "track_id": 192
          },
          {
            "name": "Realword",
            "track_id": 193
          },
          {
            "name": "Free Speech For The Dumb",
            "track_id": 408
          }
        ]
      }
    }
    ```

2. `query getAlbumsAsArtist` as *artist*

    **Headers**:

    ```http
    X-Hasura-Role: artist
    X-Hasura-Artist-Id: 9
    ```

    **Query**:

    ```graphql
    query getAlbumsAsArtist {
      music_album {
        title
      }
    }
    ```

    **Result**:

    ```json
    {
      "data": {
        "music_album": [
          {
            "title": "BackBeat Soundtrack"
          }
        ]
      }
    }
    ```

3. `query trackValue` as *artist*

    **Headers**:

    ```http
    X-Hasura-Role: artist
    ```

    **Query**:

    ```graphql
    query trackValue {
      music_track_aggregate {
        aggregate {
          sum {
            unit_price
          }
        }
      }
    }
    ```

    **Result**:

    ```json
    {
      "errors": [
        {
          "message": "field 'music_track_aggregate' not found in type: 'query_root'",
          "extensions": {
            "path": "$.selectionSet.music_track_aggregate",
            "code": "validation-failed"
          }
        }
      ]
    }
    ```

- Execute a complex query of your choice, with and without caching. Share the query, the response and the response time for each.

    **Headers**:

    None other than `x-hasura-admin-secret` and `content-type`.

    **Query**:

    ```graphql
    query first_track @cached(ttl: 60) {
      music_artist(order_by: {name: asc}) {
        artist_id
        name
        album(order_by: {title: asc}) {
          album_id
          title
          track(limit: 1, order_by: {name: asc}) {
            track_id
            name
          }
        }
      }
    }
    ```

    **Result**:

    The response times ranged between 160 and 170ms irrespective of caching. The HTTP response did not include the `X-Hasura-Query-Cache-Key` either. Caching does not seem to be enabled because it is available only on the Cloud and Enterprise Editions.


#### docker-compose.yaml for your working environment

[docker-compose.yaml](./task-two/docker-compose.yaml)

#### Metadata in YAML format for your working environment

[metadata](./task-two/hasura/metadata/)

#### Describe any challenges you encountered when executing the above queries and your troubleshooting steps to address them

1. Query *getAlbumsAsArtist* returned albums of other artists too because the filter in *metadata\databases\PG\tables\public_albums.yaml* was set up to return albums of the specified artist and those of all artists with an id greater than 10. Corrected this by removing the incorrect part of the filter.
2. Query *getAlbumsAsArtist* required the header *x-hasura-user-id* instead of *x-hasura-artist-id*. Corrected the name of the session variable in *public_album.yaml* and *public_artists.yaml* to *X-Hasura-Artist-Id*.
3. Query *trackValue* allowed users in the role *artist* to run aggregate queries on the track table. Corrected by changing `select_permissions.role.permission.allow_aggregations` to `false`.
