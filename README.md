# Deploy DocSpring On Premise

## Requirements

* An [Aptible](https://www.aptible.com) account
* DocSpring License Key (must be set in the `DOCSPRING_LICENSE` environment variable.)
* Credentials for DocSpring’s Docker Registry

> You should have received the DocSpring license key and Docker credentials (AWS Access Token ID and Secret) via Slack or email.

## Run DocSpring on Aptible

1. Go to your Aptible dashboard and make sure you've added a public SSH key to your Aptible account.
2. Install the [Aptible CLI](https://deploy-docs.aptible.com/docs/cli)
3. Run `aptible login` to authenticate
4. Clone this repository. `git clone https://github.com/DocSpring/docspring-enterprise-aptible.git`
5. Change your working directory `cd docspring-enterprise-aptible`
6. If desired, update the `Dockerfile` with your desired version (replace `:latest` with a version tag)
7. Create a new app on Aptible `aptible apps:create docspring-enterprise`
8. Add a postgres database `aptible db:create docspring-enterprise-postgres --type postgresql --version 11`
9. Add a redis database `aptible db:create docspring-enterprise-redis --type redis --version 5.0`
10. Connect to your AWS bucket (see instructions below)
11. Set your app config variables. Use these instructions to set `REDIS_URL` and `DATABASE_URL`: https://deploy-docs.aptible.com/docs/database-credentials#using-database-credentials

```bash
$ ADMIN_PASSWORD="$(openssl rand -hex 6)"
$ echo "After app is deployed, you can log in with username: admin@example.com, password: $ADMIN_PASSWORD"

$ aptible config:set --app "docspring-enterprise" \
    APTIBLE_PRIVATE_REGISTRY_USERNAME="<AWS Access Token ID>" \
    APTIBLE_PRIVATE_REGISTRY_PASSWORD="<AWS Secret Access Token>" \
    DOCSPRING_LICENSE="<docspring license key>" \
    DATABASE_URL="<pg-connection-string>" \
    REDIS_URL="<redis-connection-string>" \
    DISABLE_EMAILS="true" \
    SECRET_KEY_BASE="$(openssl rand -hex 64)" \
    SUBMISSION_DATA_ENCRYPTION_KEY="$(openssl rand -hex 32)" \
    ADMIN_NAME="Admin" \
    ADMIN_EMAIL="admin@example.com" \
    ADMIN_PASSWORD="$ADMIN_PASSWORD"
```

12. Set "Optional configuration" variables (see instructions below)
13. Add aptible as a remote and push
```
git remote add aptible git@beta.aptible.com:<aptible-environment>/docspring-enterprise.git
git push aptible master
```
> You can find your environment name by running: `aptible environment:list`
> If you have any problems pushing to Aptible, take a look at this troubleshooting guide: https://deploy-docs.aptible.com/docs/permission-denied-git-push

14.  Create an Aptible endpoint for `web`. **Make sure to set the container port to 8000**.
15.  Set your custom host url (e.g., `docspring.company.com`) using the `SETTINGS__HOST_URL` config variable
16.  We recommend setting the **RAM for each container to ~2GB** and scaling up as needed.
```
aptible config:set --app <app-slug> \ 
    SETTINGS__HOST_URL=<aptible-endpoint-host>
```

### Troubleshooting

## Docker Image Authentication

If you are having problems authenticating with AWS ECR to fetch the Docker image, e.g.:

```
INFO -- : Using private repository credentials from APTIBLE_PRIVATE_REGISTRY_USERNAME and APTIBLE_PRIVATE_REGISTRY_PASSWORD
INFO -- : Fetching app image: 691950705664.dkr.ecr.us-east-1.amazonaws.com/docspring/enterprise...
ERROR -- : unauthorized: authentication required
```

Try authenticating on your local machine using the AWS CLI and Docker and ensure this works:

```bash
export AWS_ACCESS_KEY_ID="<AWS Access Token ID>"
export AWS_SECRET_ACCESS_KEY="<AWS Secret Access Token>"
aws ecr get-login-password | docker login --username AWS --password-stdin "691950705664.dkr.ecr.us-east-1.amazonaws.com"
# => Login Succeeded

docker pull "691950705664.dkr.ecr.us-east-1.amazonaws.com/docspring/enterprise:latest"
# => latest: Pulling from docspring/enterprise
# => Digest: sha256:dac6300bf72d964a09a8069c8c9005613cb7434af0d211535158805b5d83bbd1
# => ...
```


### Database users

When you ran `aptible db:create <...>`, you created a postgres database with 2 users: `postgres` and `aptible`. 
In order to create the database, perform rollbacks, execute all migrations in future releases, maintain the Superuser role for the user you specified in your `DATABASE_URL` connection string.

You can verify which users are in your database by creating an ephemeral database tunnel `aptible db:tunnel <aptible-db-name>`.
```bash
$ aptible db:tunnel <aptible-db-name> --type postgresql
Creating postgresql tunnel to <aptible-db-name>...
Use --type TYPE to specify a tunnel type
Valid types for pg-dev: postgresql
Connect at postgresql://aptible:<password>@localhost.aptible.in:<port>/db
Or, use the following arguments:
* Host: localhost.aptible.in
* Port: <port>
* Username: aptible
* Password: <password>
* Database: db
Connected. Ctrl-C to close connection.
```
And in a separate terminal
```postgresql
$ psql postgresql://aptible:<password>@localhost.aptible.in:<port>/db
db=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 aptible   | Superuser                                                  | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

You can create a new role if you so choose
```postgresql
db=# create user mynewrole;
CREATE ROLE
db=# alter user mynewrole with superuser;
ALTER ROLE
db=# \du
                                   List of roles
 Role name |                         Attributes                         | Member of
-----------+------------------------------------------------------------+-----------
 aptible   | Superuser                                                  | {}
 mynewrole | Superuser                                                  | {}
 postgres  | Superuser, Create role, Create DB, Replication, Bypass RLS | {}
```

If you want to use `mynewrole`, you'll need to change the `DATABASE_URL` connection string.

### Connect to your AWS bucket:

You will need an S3 bucket, a User, and an IAM policy.
1. Create a new S3 bucket for DocSpring (e.g., `docspring-data`)
2. Create a new User for DocSpring (e.g., `docspring`) in your AWS IAM settings. Keep your access key ID and secret access key somewhere safe. Those should be used to set `SETTINGS__AWS__ACCESS_KEY_ID` and `SETTINGS__AWS__SECRET_ACCESS_KEY`, respectively 
3. Attach an S3 bucket policy with the following configuration
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "ListObjectsInBucket",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::docspring-data"
            ]
        },
        {
            "Sid": "AllObjectActions",
            "Effect": "Allow",
            "Action": "s3:*Object",
            "Resource": [
                "arn:aws:s3:::docspring-data/*"
            ]
        }
    ]
}
```
4. Edit your bucket permissions by navigating to the bucket > "Permissions.
```json
[
    {
        "AllowedHeaders": [
            "Authorization"
        ],
        "AllowedMethods": [
            "GET"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    },
    {
        "AllowedHeaders": [
            "*"
        ],
        "AllowedMethods": [
            "PUT",
            "POST"
        ],
        "AllowedOrigins": [
            "*"
        ],
        "ExposeHeaders": [],
        "MaxAgeSeconds": 3000
    }
]
```

* Note: You may also set allowed origins to the domain at which you're hosting DocSpring (with no trailing forward slash)

## Optional configuration

See: [DocSpring Enterprise Configuration (Environment Variables)](https://docs.google.com/document/d/1av1w3JDvvQOxXcywQ-L7HHSVJONxiL94Ih_E2agLzOQ/edit)

## Google OAuth Details (Pending)

To set up OAuth for your internal team, start by logging into your the the Google Cloud Platform for your team's workspace. Then, do the following:

1. Create a new project
2. Go to `APIs and Services`
3. Create an `OAuth consent screen` and make it “internal”
4. Create a new `Credentials > OAuth client ID > web application`
5. Add `https://<yourdomain>` (from the endpoint creation step. should be the same as `SETTINGS__HOST_URL`) to "Authorized Javascript Origins"
6. Add `https://<yourdomain>/users/auth/google_oauth2/callback` to "Authorized redirect URIs"
7. Save
8. Back in your terminal, set:

```bash
aptible config:set --app <app-name> GOOGLE_OAUTH_CLIENT_ID=<new_client_id> GOOGLE_OAUTH_CLIENT_SECRET=<new_client_secret>
```

## Public API Considerations

If you're running DocSpring on-premise on a private network (or something like Cloudflare zero trust), then you'll need to allow-list public API endpoints
to confinue using them without the request being blocked.
Our api routes are `/api/*`. For example, the Generate PDF route is `https://api.docspring.com/api/v1/templates/<TEMPLATE_ID>/submissions` 
To permit future routes and versions, we suggest allow-listing `<your-domain>/api/*` routes.

