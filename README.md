# snowflake_terraform

Create a new repository for GitHub
```sh
mkdir <directory-name> 
cd <directory-name>
git init
git add README.md
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/sfguide-terraform-sample.git
$ git push -u origin main
```

Create the public and private keys
```sh 
mkdir ~/.ssh
cd ~/.ssh
openssl genrsa 2048 | openssl pkcs8 -topk8 -inform PEM -out snowflake_tf_snow_key.p8 -nocrypt
openssl rsa -in snowflake_tf_snow_key.p8 -pubout -out snowflake_tf_snow_key.pub
```
In a Snowflake account
```sql
-- Replace RSA_PUBLIC_KEY_HERE with the public key generated
CREATE USER "tf-snow" RSA_PUBLIC_KEY='<RSA_PUBLIC_KEY_HERE>' DEFAULT_ROLE=PUBLIC MUST_CHANGE_PASSWORD=FALSE;

GRANT ROLE SYSADMIN TO USER "tf-snow";
GRANT ROLE SECURITYADMIN TO USER "tf-snow";

-- Find the account credentials
SELECT LOWER(current_organization_name() || '-' || current_account_name()) as YOUR_SNOWFLAKE_ACCOUNT;
```
```sh 
export SNOWFLAKE_USER="tf-snow"
export SNOWFLAKE_AUTHENTICATOR=JWT
export SNOWFLAKE_PRIVATE_KEY=`cat ~/.ssh/snowflake_tf_snow_key.p8`
export SNOWFLAKE_ACCOUNT="<YOUR_SNOWFLAKE_ACCOUNT>"

# open vscode
code .
```
In the repository create a new file called main.tf
```tf
<!-- this creates the resources -->
terraform {
  required_providers {
    snowflake = {
      source  = "Snowflake-Labs/snowflake"
      version = "~> 0.87"
    }
  }
}

provider "snowflake" {
  role = "SYSADMIN"
}

resource "snowflake_database" "db" {
  name = "TF_DEMO"
}

resource "snowflake_warehouse" "warehouse" {
  name           = "TF_DEMO"
  warehouse_size = "xsmall"
  auto_suspend   = 60
}
```
Next is to set up these resources
```sh
terraform init
``` 
Use a .gitignore file and save these into it
```sh
*.terraform*
*.tfstate
*.tfstate.*
```
Run the following for a check on the future chnage tracking
```sh 
git checkout -b dbwh
git add main.tf
git add .gitignore
git commit -m "Add Database and Warehouse"
git push origin HEAD
```
In your shell run 
```sh
# this is to seeif there are errors or warnings
terraform plan
# if its all good run this
terraform apply
```
Log in to your Snowflake account for confirmation of this running correctly.

Next is to add the schema services, update the file below by replacing the account_role_name to the one being used

```tf
provider "snowflake" {
  alias = "security_admin"
  role  = "SECURITYADMIN"
}

resource "snowflake_account_role" "role" {
  provider = snowflake.security_admin
  name     = "TF_DEMO_SVC_ROLE"
}

resource "snowflake_grant_privileges_to_account_role" "database_grant" {
  provider          = snowflake.security_admin
  privileges        = ["USAGE"]
  account_role_name = "<role_name>"
  on_account_object {
    object_type = "DATABASE"
    object_name = snowflake_database.db.name
  }
}

resource "snowflake_schema" "schema" {
  database   = "TF_DEMO"
  name       = "TF_ACQUIRED"
}

resource "snowflake_grant_privileges_to_account_role" "schema_grant" {
  provider          = snowflake.security_admin
  privileges        = ["USAGE"]
  account_role_name = "<role_name>"
  on_schema {
    schema_name = "\"${snowflake_database.db.name}\".\"${snowflake_schema.schema.name}\""
  }
}

resource "snowflake_grant_privileges_to_account_role" "warehouse_grant" {
  provider          = snowflake.security_admin
  privileges        = ["USAGE"]
  account_role_name = "<role_name>"
  on_account_object {
    object_type = "WAREHOUSE"
    object_name = "TF_DEMO"
  }
}

resource "tls_private_key" "svc_key" {
  algorithm = "RSA"
  rsa_bits  = 2048
}

resource "snowflake_user" "user" {
    provider          = snowflake.security_admin
    name              = "tf_demo_user"
    default_warehouse = snowflake_warehouse.warehouse.name
    default_role      = "<role_name>"
    default_namespace = "${snowflake_database.db.name}.${snowflake_schema.schema.name}"
    rsa_public_key    = substr(tls_private_key.svc_key.public_key_pem, 27, 398)
}

resource "snowflake_grant_privileges_to_account_role" "user_grant" {
  provider          = snowflake.security_admin
  privileges        = ["MONITOR"]
  account_role_name = "<role_name>"  
  on_account_object {
    object_type = "USER"
    object_name = snowflake_user.user.name
  }
}

resource "snowflake_grant_account_role" "grants" {
  provider  = snowflake.security_admin
  role_name = "<role_name>"
  user_name = snowflake_user.user.name
}

```
Create a new file named outputs.tf
```tf 
output "snowflake_svc_public_key" {
  value = tls_private_key.svc_key.public_key_pem
}

output "snowflake_svc_private_key" {
  value     = tls_private_key.svc_key.private_key_pem
  sensitive = true
}
```
Check for future change tracking
```sh
git checkout -b svcuser
git add main.tf
git add outputs.tf
git commit -m "Add Service User, Schema, Grants"
git push origin HEAD

``` 
In the terminal run 
```sh
terraform apply
```
Confirm changes and fix errors that appear, then go onto the Snowflake account to see if the changes have happened.

Finally if you no longer need this terraform and are looking at creating your own run 
```sh 
terraform destroy
```
ANd remove the tf_user you created in your Snowflake
```sql
DROP USERT "tf_user";
```