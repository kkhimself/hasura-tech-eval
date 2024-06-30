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

      Query:

      ```graphql
      query artist_count {
        artist_aggregate {
          aggregate {
            count
          }
        }
      }
      ```

      Response:

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

      Query:

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

      Response:

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

    Query:

    ```graphql
    query AlbumsByArtist {
      album {
        artist_id
        album_id
        title
      }
    }
    ```

    Response:

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

    Mutation:

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

    Response:

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
