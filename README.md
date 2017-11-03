# Hasura Hello World
> Get started with a free Hasura Project

## Installation

OS X, Linux

```
$ curl https://storage.googleapis.com/hasuractl/install-dev.sh | bash
```

Windows

```
https://storage.googleapis.com/hasuractl/dev/windows-amd64/hasura.exe
```

## Quickstart

#### Getting a free project

Once you have **hasura** installed,

```
$ hasura login
```

Authenticating is required to allow hasura to setup the free trial.

```
$ hasura quickstart hasura/hello-world
$ cd hello-world
```

#### Getting cluster status

After the free cluster is installed, get brief information about the cluster via the following command,

```
$ hasura cluster status
```

#### Deploying on a hasura cluster

After you are done with quickstart, you will have a free cluster on Hasura, up and running. You need to **git push** your project to the cluster and apply the configuration. This step will apply migrations and deploy custom services that are applicable.

```
$ git add .
$ git commit -m "Hasura Hello World App"
$ git push hasura master
```

#### Accessing Console

Now that you have deployed the project on your cluster, you would want to manage the schema and explore APIs.

Access the **api-console** via the following command:

```
$ hasura api-console
```

This will open up Console UI on the browser. You can access it at [http://localhost:8080](http://localhost:8080)

## Usage

Using the **api-console**, you can explore different Hasura APIs.

You will land up on the API Explorer where you can try out APIs (Data, Auth, Filestore and Notify) using the API Collections.

### Data APIs


The Hasura Data API provides a HTTP/JSON API backed by a PostgreSQL database.

These APIs are designed to be used by any client capable of making HTTP requests, especially
and are carefully optimized for performance.

The Data API provides the following features:
* CRUD APIs on PostgreSQL tables with a MongoDB-esque JSON query syntax.
* Rich query syntax that supports complex queries using relationships.
* Role based access control to handle permissions at a row and column level.

The following examples use a sample schema for a blog application that consists of two tables, article and author. To run the following examples, install hasura, login and run
```
$ hasura quickstart hello-world
$ cd hello-world
$ git add .
$ git commit -am "Initial Commit" && git push hasura master
$ hasura api-console
```
This will open the Hasura api-console, with the sample schema discussed above already loaded.
You can just paste the queries shown below into the json textbox in the API explorer and hit send to test them out.
(The following is a short set of examples to show the power of the Hasura Data APIs, check out our [documentation](https://docs.hasura.io/) for more when you're done here!)

Let's look at some sample queries to explore the Data APIs:


#### CRUD
Simple CRUD Operations are supported via an intuitive JSON query language.

* Select all entries in the article table, ordered by rating:
```json
{
    "type": "select",
    "args": {
        "table": "article",
        "columns": ["*"],
        "order_by": [
            {
                "column": "rating"
            }
        ]
    }
}
```

* Update a particular entry in the author table:
```json
{
    "type": "update",
    "args": {
        "table": "author",
        "where": {
            "name": {
                "$eq": "Adams"
            }
        }
    }
}
```

* The where clause on the Data API is a very expressive boolean expression, and can be arbitrarily complex. For example:
```json
{
    "type": "select",
    "args": {
        "table": "article",
        "columns": [
            "content"
        ],
        "where": {
            "$and": [
                {
                    "$or": [
                        {
                            "author_id": {
                                "$eq": "7"
                            }
                        },
                        {
                            "title": {
                                "$like": "Editorial%"
                            }
                        }
                    ]
                },
                {
                    "rating": {
                        "$gte": "3"
                    }
                }
            ]
        },
        "order_by": [
            {
                "column": "rating",
                "order": "asc"
            }
        ]
    }
}
```

  This query will select all the articles with ratings above 3, which were either written by an author with author_id 7 or, which have a title starting with "Editorial". This can be used to construct complex queries that feel very intuitive.

* Pagination on queries is supported through limit and offset parameters:
```json
{
    "type": "select",
    "args": {
        "table": "article",
        "columns": ["*"],
        "limit": "10",
        "offset": "20"
    }
}
```

* Raw SQL:  The APIs support running arbitrary SQL queries through a run_sql type key.

This can be used to perform queries directly on the postgres db:
```json
{
    "type" : "run_sql",
    "args" : {
        "sql" : "CREATE TABLE category (
                     id SERIAL NOT NULL PRIMARY KEY,
                     name TEXT NOT NULL
                 );"
    }
}
```

#### Relationships

Modelling data in an RDBMS involves establishing connections between various tables through foreign key constraints. These can be used to build more complex relationships, which can be used to fetch related data alongside the columns queried, as pseudo columns.

In the standard article-author sample schema, the relationships we can define are:
1. Articles have an object/many-to-one relationship with authors
2. Authors have an array/one-to-many relationship with articles.

We can define these relationships on the database, and use them to get related data by expanding the relationships in the columns array:
```json
{
    "type": "select",
    "args": {
        "table": "author",
        "columns": [
            "name",
            {
                "name": "articles",
                "columns": [
                    "content",
                    "title",
                    "rating"
                ]
            }
        ]
    }
}
```

This query will add an array of article objects alongside the name of the author.

You can also use the standard where/order_by/offset/limit conditions on the article objects:
```json
{
    "type": "select",
    "args": {
        "table": "author",
        "columns": [
            "name",
            {
                "name": "articles",
                "columns": [
                    "content",
                    "title",
                    "rating"
                ],
                "where": {
                    "rating": {
                        "$gte": "3"
                    }
                },
                "order_by": [
                    {
                        "column": "rating",
                        "order": "desc"
                    }
                ]
            }
        ],
        "where": {
            "name": {
                "$like": "A%"
            }
        }
    }
}
```

This will get us a list of all articles with rating greater than 3 by authors with names starting with A, ordered by rating among articles by the same author.

All this and more can be done with a single query!

#### Aggregations

The JSON based query language is designed to be simple yet powerful. That said, there will be queries that cannot be expressed through the select query, like getting the number of ratings given for each article, if you have the ratings by user data stored in another table.

To express complex queries like aggregations, window functions, custom joins etc, you can directly use SQL.
```json
{
  "type" : "run_sql",
  "args" : {
    "sql" : "CREATE VIEW article_rating_count AS...",
  }
}
```

If you can define a view with your query in SQL, you can then track it with the Data APIs, and use the JSON query language to access it.
```json
{
  "type" : "add_existing_table_or_view",
  "args" : {
    "name" : "article_rating_count"
  }
}
```

  Note that views are read only, so you can only perform select queries on them. You can also manually define object relationships on views, in order to easily obtain them from select queries on other tables.

#### Role based access control
Permissions on the Data APIs are designed to restrict the operations that can be performed on the database by various users/roles. The Data APIs support setting permissions on various CRUD operations at a row/column granularity.  By default, the admin role has access to all operations.

This is accomplished through the session middleware that Hasura provides. This session middleware provides the Data API with the role and user id of the current user with every request, and this lets the Data service apply the permissions as appropriate.

* The permissions can be based on a user id check from the information provided by the session middleware:
```json
{
    "type" : "create_insert_permission",
    "args" : {
        "table" : "article",
        "role" : "user",
        "permission" : {
            "check" : {
                "author_id" : "REQ_USER_ID"
            }
        }
    }
}
```

  This query will set select permissions on the article table for the user role so that users will be able only insert entries into the article table with author_ids matching their user ids. This means that the database will not permit a user to write an article in another user's name.
  This sort of a constraint is a property of the data, and therefore should be accomplished in the database, and the permission layer provides the perfect tools for the job.
  Apart from create_insert_permissions, the Data API also provides other types of queries to create select/update and delete permissions. This way, permissions can be set on all CRUD operations.

* The permission object in the json query uses syntax very similar to a where clause in the select query, making it extremely expressive,  as shown here:
```json
{
    "type" : "create_update_permission",
    "args" : {
        "table" : "article",
        "role" : "user",
        "permission" : {
            "check" : {
                "author_id" : "REQ_USER_ID",
                "$or" : [
                    {
                        "category" : "editorial",
                        "is_reviewed" : false
                    },
                    {
                        "category" : { "$neq" : "editorial"}
                    }
                ]
            }
        }
    }
}
```

This query sets insert permissions on the article table for the user role so that users can only insert entries into the table if the author_id is the same as their user id, and if is_reviewed is false when the category is editorial.

* This permissions setup can be further improved by creating custom roles. For example, the above schema can be improved by having an author role that can be given permissions to edit only the article table, and nothing else.

This is very useful for a more complex schema, say a forum, with several types of users like admins, moderators, thread owners, and normal users.


#### Performance

The Hasura Data APIs are heavily optimized for performance, and a comparison can be found on our [ website ](https://hasura.io/compare).

Explore the Data APIs further using the learning-center in the API console!
Bring up the API console using
```bash
$ hasura api-console
```
And then navigate to the [ Learning center ](http://localhost:8080/learning-center) tab.


### File APIs

The File API on Hasura lets you upload and store files, and download them when required. This is done via simple POST, GET and DELETE requests on a single endpoint.

Just like the Data service, the File API supports Role based access control to the files, along with custom authorization hooks. (Check out our [ documentation ](https://docs.hasura.io/) for more!)

#### Uploading files

Uploading a file requires you to generate a file_id and make a post request with the content of the file in the request body and the correct mime type as the content-type header.

```http
POST https://filestore.project-name.hasura-app.io/v1/file/05c40f1e-cdaf-4e29-8976-38c899 HTTP/1.1
Content-Type: image/png
Authorization: Bearer <token>

<content-of-file-as-body>
```

This is a very simple to use system, and lets you directly add an Upload button on your frontend, without spending time setting up the backend.

#### Downloading files
Downloading a file requires the unique file id that was used to upload it. This can be stored in the database and retrieved for download.

To download a particular file, what is required is a simple GET query.
```http
GET https://filestore.project-name.hasura-app.io/v1/file/05c40f1e-cdaf-4e29-8976-38c899 HTTP/1.1
Authorization: Bearer <token>
```

#### Permissions
By default, the File API provides three hooks to choose from

  1. Private: Only logged in users can upload/download.
  2. Public: Anyone can download, but only logged in users can upload.
  3. Read Only: Anyone can download, but no one can upload.

You can also set up your own authorization webhook!
(Check out our [ documentation ](https://docs.hasura.io/) for more!)

### Explore the Auth and Notify APIs

Check out the [ Learning center ](http://localhost:8080/learning-center) tab on the API Console for short tutorials on the Auth and Notify APIs!

### Node.js Express Hello World App

This quickstart project has a simple Node.js Express Hello World app.

You can access the app source code inside

```
$ cd services/app/src
```

The URL for accessing the app is [https://app.your-cluster-name.hasura-app.io](https://app.<your-cluster-name>.hasura-app.io)

## Add your own custom microservice

#### Docker microservice

```
$ hasura service add <service-name> -i <docker-image> -p <port>
```

#### git push microservice

```bash
$ hasura service add <service-name>
```

Once you have added a new service, you need to add a route to access the service.

#### Add route for the service created.

```bash
$ hasura route generate <service-name>
```

It will output the route information that needs to be put into conf/routes.yaml file.

#### Add a remote for the service

```bash
$ hasura remotes generate <service-name>
```

This will output the remotes configuration, which should be added to the conf/remotes.yaml file under the {{ cluster.name }} key.
    Make sure it is properly indented!

#### Apply your changes

```
$ git add .
$ git commit -m "Added a new service"
$ git push hasura master
```
