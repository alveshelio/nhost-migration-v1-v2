#A guide to migrate from Nhost V1 to V2

At the moment of writing this post, Nhost doesn’t have a migration tool so you’ll need to do this process manually.

What’s important to know is that you can’t yet create a project in Nhost and do a `nhost init --remote`; this isn’t available yet. You’ll need to create a project locally with the new Nhost CLI.
However, before you update Nhost CLI you need to make sure you migrate your metadata since it will not be compatible with the new version.

1. Clone your actual project in another folder

    This is particularly important because you don’t want to mess up your existing project.
Make sure your nhost CLI is still on version 0.4.7
If your Hasura console version is around 1.3.4 you’ll need to migrate your metadata to V3. https://hasura.io/blog/migrating-from-hasura-v1-3-to-v2-0/
Running `hasura scripts update-project-v3` didn't worked, I had to first run `hasura scripts update-project-v2` and then `hasura scripts update-project-v3`


2. Nhost CLI 0.5.0

    Once you’ve migrated your metadata it is time to upgrade your Nhost CLI. First uninstall nhost, `npm uninstall -g nhost`. To make sure it's uninstalled run `which nhost`. To install the latest version of Nhost CLI follow this guide: https://docs.beta.nhost.io/reference/cli/nhost-cli#installation


3. Migrate your database

    Nhost authentication has slightly changed in V2, the `users` table is now in the `auth` schema. Doing a database dump will not work for us, we'll need to export all tables in `SQL` format. I've used Table Plus, but you can use any other tool. Select all your tables, the `users` table too. If you've created functions, make sure you export them too.

    Modify your `SQL` script to make sure that you remove the `users` table creation from the script.
   Remove any script you have to create the table

```sql
   DROP TABLE IF EXISTS "public"."users";
   -- This script only contains the table creation statements and does not fully represent the table in the database. It's still missing: indices, triggers. Do not use it as a backup.
   
   -- Table Definition
   CREATE TABLE "public"."users" (
       "id" uuid NOT NULL DEFAULT gen_random_uuid(),
       "created_at" timestamptz NOT NULL DEFAULT now(),
       "updated_at" timestamptz NOT NULL DEFAULT now(),
       "display_name" text,
       "avatar_url" text,
       PRIMARY KEY ("id")
   );
```

   Then make sure you replace `"public"."users"` by `"auth"."users"`.
   You will need to change the script to insert data into the `users` table. The new users table has two changes, `avatar_url` isn't nullable anymore so if you have `NULL` values, you need to replace it by single quotes`''` and you need to add a required column `locale`. Here's an example of an `INSERT INTO "auth"."users"` script:

   ```sql
   INSERT INTO "auth"."users" ("id", "created_at", "updated_at", "display_name", "avatar_url", "locale")
   VALUES ('2c00e6cc-af92-4d8e-9ee2-df21d278f50e', '2021-10-21 01:40:22.389257+00', '2021-10-21 01:40:22.389257+00', 'My user', 'http://my-avatar-url.com/super-cool-avatar', 'en'),
```

4. Create your **V2** project
   
   You can either run `nhost --init` or `nhost`, follow the **Usage** guide https://docs.beta.nhost.io/reference/cli/nhost-cli#usage.


5. Import your database 
   
   It is time to import your database. You can either open Hasura console and use the SQL screen or connect to the database using your preferred tool and run your `SQL` script.


6. Export your metadata from your **V2 project**
   
   Once you've imported your database you'll need to export your metadata. This will be useful to update the metadata from your **V1 project**


7. Update your **V1 project's** metadata
   
   Now that you have metadata from both projects it's time to start updating your **V1 project's** metadata to be able to import it into your **V2 project**.

   In **V2** these tables in `auth` schema have changed. Tables `users` and `accounts` have both been merged together into `users`. Table `account_roles` has become `user_roles` and table `account_providers` has become `user_providers` .
   
   What I ended up doing was to copy all the `auth`  schema metadata from my **V2 project** into my **V1 project** and then if I had permissions into any of the tables I'd add the permissions.

   Here's an example of a table I had permissions on
```json
   {
            "table": {
              "schema": "auth",
              "name": "roles"
            },
            "configuration": {
              "custom_root_fields": {
                "insert": "insertAuthRoles",
                "select_aggregate": "authRolesAggregate",
                "insert_one": "insertAuthRole",
                "select_by_pk": "authRole",
                "select": "authRoles",
                "delete": "deleteAuthRoles",
                "update": "updateAuthRoles",
                "delete_by_pk": "deleteAuthRole",
                "update_by_pk": "updateAuthRole"
              },
              "custom_name": "authRoles",
              "custom_column_names": {
                "role": "role"
              }
            },
            "array_relationships": [
              {
                "name": "userRoles",
                "using": {
                  "foreign_key_constraint_on": {
                    "column": "role",
                    "table": {
                      "schema": "auth",
                      "name": "user_roles"
                    }
                  }
                }
              },
              {
                "name": "usersByDefaultRole",
                "using": {
                  "foreign_key_constraint_on": {
                    "column": "default_role",
                    "table": {
                      "schema": "auth",
                      "name": "users"
                    }
                  }
                }
              }
            ],
            "select_permissions": [
              {
                "role": "me",
                "permission": {
                  "columns": [
                    "role"
                  ],
                  "filter": {}
                }
              },
              {
                "role": "candidate",
                "permission": {
                  "columns": [
                    "role"
                  ],
                  "filter": {}
                }
              },
              {
                "role": "user",
                "permission": {
                  "columns": [
                    "role"
                  ],
                  "filter": {}
                }
              }
            ]
          }
```
   In this example I've simply copied all the table data fom V2 into V1, and then I've added the `select_permissions`, if you have other permissions, make sure you add them too.
   
   It is time now to import your newly updated metadata into your **V2 project** 

9. Frontend
   
   Once you've imported your database and metadata you can now start taking care of your frontend. In my V1 project I had my frontend and nhost both at the root of my project, however, for V2 I'd suggest you to create a `web` folder and clone your project there and remove anything related to nhost, so you'd end up only with your frontend code. 
   
   You should have this the structure

```
project_root
│   
└───nhost
│   
└───web
    │   src
    │   package.json
```

