## Table of Contents  
**[Working Locally](#working-locally)**<br> 
**[Migrations](#migrations)**<br> 
**[Tests](#tests)**<br>
**[Test Data](#test-data)**<br>
**[DDL Data Definition Language](#ddl-data-definition-language)**<br>
**[CI/CD Process](#CICD-Process)**<br>

## Working Locally
* Clone needed repo from: https://github.hy-vee.cloud/shared-databases
* Ensure you have a Hy-Vee provisioned [GSuite](https://gsuite.google.com) account.  If you need an account or do not 
know if you have an account, reach out to [#wamdo](https://hy-vee.slack.com/messages/CAN6EJP9V).
* Ensure that you have [`make`](http://gnuwin32.sourceforge.net/downlinks/make.php), 
[`gcloud`](https://dl.google.com/dl/cloudsdk/channels/rapid/GoogleCloudSDKInstaller.exe), and 
[`docker`](https://download.docker.com/win/stable/Docker%20for%20Windows%20Installer.exe) installed and available on 
your `$PATH`.

### Commands
* **`make db-up`** - Pulls the latest docker image containing the database and all dependencies at equal to master for 
the same, runs it in the background, and binds it to local port 1433.
   * You can connect to it on localhost with the `sa` SQL login, `Password#123`.
* **`make migrate`(Mac Only)** - Executes `flyway migrate`.  If done without making any changes from master, only tests will run.
Else, any changes and then all test will run.  All tests will run every time.
   * The database can take a few seconds to start.  If you see a login timeout wait a few and try again.
* **`make migrate-windows`(Windows Only)** - Executes `flyway migrate`.  If done without making any changes from master, only tests will run.
Else, any changes and then all test will run.  All tests will run every time.
   * The database can take a few seconds to start.  If you see a login timeout wait a few and try again.
* **`make db-down`** - Tears down the database.
* **`make db-reset`** - Resets your database by tearing it down and pulling the latest.

#### Troubleshooting
* `docker-credential-gcloud not install or not available in PATH` during `make db-up` - Run `gcloud components update`.
If there aren't any updates to pull, uninstall and reinstall `gcloud`.
* `ERROR: Unable to load config file: /flyway/flyway-docker.conf` during `make migrate-windows` - Right click Docker 
icon from status bar -> Settings -> Shared Drives -> Check relevant drive (or uncheck -> apply -> recheck) -> Apply.
* `TCP Provider: No such host is known.` during `make migrate-windows` while on VPN - From `Command Prompt` execute 
`nslookup hvcorp10` and note the `Address` of the `Server`. (use the first server, which is the dns server, not the IP of `hvcorp10`)
Then right click Docker icon from status bar -> Settings -> Network -> Select `Fixed` DNS server and enter the `Address` from before.
So far it has been `10.215.1.50`, but that could change depending on which DNS server you are hitting.
* If you already have a SQL instance running locally on the default port (1433) it will need to be stopped or changed to
a different port.
* For Windows Users when running `make db-up` if you see the following warning below please ignore as long as it says Pulling Database at the bottom you should be good to go
      
      WARNING: `docker-credential-gcloud` not in system PATH.
      gcloud's Docker credential helper can be configured but it will not work until this is corrected.
      WARNING: `docker` not in system PATH.
      `docker` and `docker-credential-gcloud` need to be in the same PATH in order to work correctly together.
      gcloud's Docker credential helper can be configured but it will not work until this is corrected.
      gcloud credential helpers already registered correctly.
      Pulling database ... YourDatabaseName

## Migrations
Located within ./migrations.  Separated into two groups, repeatable and versioned.

### Repeatable
Located within subfolders of ./migrations/repeatable.  Procedural code (views, stored procedures, triggers, and 
functions).  Unlike non-procedural code, repeatable migrations require the entire object to change in order to update 
instead of just the part that is being updated.  Repeatable migrations execute any time they change in alphabetical 
order (case sensitive, capitals first) after versioned migrations.

`V1` is the baseline, so the first versioned migration is `V2`.

* **Adding** (CREATE) - Create a new versioned migration with a baseline to `CREATE` and a new repeatable migration to 
`ALTER`
* **Updating** (ALTER) - Update the existing repeatable migration file; do not create a new versioned migration
* **Deleting** (DROP) - Delete the existing repeatable migration file and create a new versioned migration to `DROP`

Files must reside within the subfolders of ./migrations/repeatable, be prefixed with `R__`, end with `.sql`, and 
be convertible to `UTF-8`.

Commenting the change log at the top of repeatable migrations is no longer needed as history will now be in GitHub.

### Versioned
Located within ./migrations/Versioned.  Non-procedural code (migrations for everything within ./zddl except table 
triggers which are included with the table).  Only new versioned migrations execute, in numerical order, prior to 
repeatable migrations.  Files must reside within ./migrations/Versioned, be prefixed with `V#__`, end with `.sql`, and 
be convertible to `UTF-8`.  Each new version should be sequential with the previous (`1`, `2`, `3`...) and individual 
migrations should be logically grouped so that they are not too large and the file name accurately and sufficiently 
describes the contents.  When separating into multiple versions utilize `dot` notation (`2.0`, `2.1`, `2.2`...).  Each 
PR should account for an entire major version; no two PRs should interact with the same major version.

Since only new versioned migrations execute, they cannot be edited once merged.  Instead, a new version must be created.  
In addition, new versions prior to the current version are invalid.  Edits to the current or prior versions are also 
invalid.

## Tests
Located within subfolders of ./test.  Unlike the migrations above, tests do not alter the schema of the database.  
Instead, tests utilize said schema (just like the contents of the repeatable migrations above).  Adding tests makes 
Flyway aware of queries so they, too, can be validated.

Files must reside within within subfolders of ./test and be prefixed with `R__z_`, end with `.sql`, and be convertible 
to `UTF-8`.  Tests only execute locally and against the ephemeral environment; never QA or PROD.  They execute 
in alphabetical order (case sensitive, capitals first).  The purpose of the `z_` secondary prefix is to ensure they 
always run last locally.

There are varying types of tests.  The simplest test is executing a query (or stored procedure) which validates that all 
the dependent schema.  If there are multiple logical paths through said query, multiple executions will be necessary to 
hit each path.  If said query returns data, we can then take it a step further by inserting said data into a variable 
and validate the expected return type.  We can then take it a step further to assert the actual return value(s).  If the 
query updates or deletes data, we can also assert the expected changes.  These varying types of tests are explain in 
greater detail [here](/readme/pr-review.md).

See [How to test](/readme/pr-review.md#how-to-test) for specific details on what we've learned so far in regard to 
testing in SQL.
 
Tests are organized into four groups, application queries, stored procedures, tables, and users.
 
#### Application Queries
Located within ./test/application-queries.  Application queries should be organized into folders per associated repo and 
the naming convention is `RepoName_ClassName_MethodName` as applicable.  Additionally, include a comment with a link to 
the relevant code.  Finally, ideally a comment with a link to the test file should be added to the source repo.
 
#### Stored Procedures
Located within ./test/stored-procedures.  Stored procedures tests are not organized into folders and the name is the 
name of the stored procedure.

#### Tables
Located within ./test/tables.  Table tests are not organized into folders and the name is the name of the table.  

#### Users
Located within ./test/users.  User tests are organized into folders per user where the folder bares the name of the user
and the file includes the user name and method.

## Test Data
Located within ./test-data.  Test data simply does not exist today and having it will give us the greatest benefit from 
this process.  Currently, standing up this database locally gives all empty tables except for those where test data is 
defined.  But, if we define test data, we'll then be able to write better tests and work completely locally.  Thus, you 
are highly encouraged to define test data for any tables interacted with so that overtime we'll have full coverage.

Test data migrations are like versioned migrations (and must follow the same number rules), but the name is 
`V#__test_data_tableName.sql`.  Unless your test data migrations are dependent upon versioned migrations that you are creating, version them as `dot` increments to the current version to avoid conflicts.  If they are dependent upon 
versioned migrations that you are creating, version them as `dot` increments to these future versions.  Test data 
migrations execute only locally and against the ephemeral environment; never QA or PROD.

See [Test data](/readme/pr-review.md#test-data) for specific details on what we've learned so far in regard to defining 
test data in SQL.

### Test Data Helper
Located withnin [test-data-helper](/resources/sql-scripts/test-data-helper). For helping create data for testing. 
Functions need to be created within the database they will be used in.
  * [Get_Random_Date](/resources/sql-scripts/test-data-helper/test_data_helper_get_random_date.sql): Will return a 
  random date given a start and end date. Also needs `NEWID()` to be given since it can't be called in functions.

### Placeholders
Placeholders are the mechanism by which migration scripts differ between environments.  The need for these should be 
exceedingly rare.  Utilizing them is very straightforward.  In the migration script, define the placeholder with 
`${PLACEHOLDERNAME}`. Then, specify the production value in `./flyway.conf` and the QA value in 
`./flyway-qa-placeholders.conf` with `flyway.placeholders.PLACEHOLDERNAME=VALUE`.  Note that the matching is case 
sensitive.  Note also that changing the value of a placeholder in a repeatable migration does not cause said repeatable 
migration to execute.  

One legit use case for placeholders is managing equivalent accounts, those that vary by name between environments.
./migrations/repeatable/triggers/R__DBA_tr_CICD_EvtLog.DdlTrigger.sql contains an example.  See the equivalent accounts 
at ./equivalentAccounts.md.

## DDL Data Definition Language
Located within ./zddl.  Code-view of the database objects maintained as versioned migrations (except for table 
triggers); those not maintained as repeatable migrations.  This directory is maintained entirely by Concourse as part of 
the CI/CD process.  Any changes to these objects will automatically be committed by Concourse.  Note that it is 
generated from the ephemeral environment.

Do not manually commit updates to the DDL.  This allows us to view DIFFs of our versioned migrations and also the 
history for these objects.  The purpose of the `z` prefix on the directory is to show the DIFFs at the end of each PR.

## CICD Process
Located at https://concourse-ci.hy-vee.com/team/shared-databases/pipelines/REPO_NAME
* **pr-verify-check-last-commit** - Triggered by all PRs
  1. Fail if the last commit was by `concourse-svc` with a message of `Updating DDL`.

* **pr-verify** - Triggered by all PRs which pass **pr-verify-check-last-commit and acts on an ephemeral SQL server 
deployed to GCP unless otherwise noted.
  1. Validate files are in the correct folders, named properly, and convertible to UTF-8
     * If you see an `iconv` error converting to UTF-8 and are working within SSMS, save the file as UTF-8 with 
     Signature from File -> Advanced Save options...
  1. Validate versioned migrations and test data against persisted database
  1. Destroy and restore database from back-up of database after executing only `./ci/scripts/baseline.sql`)
  1. Lint repeatable migrations
  1. Execute all migrations and test data
  1. Validate object naming standards
  1. Execute all pending migrations against QA and a copy of PROD
  1. Execute tests
  1. Generate DDL and push to pull request if changes
  1. Test dependent databases (if any)
  1. Post migrations to be executed against QA as a comment to pull request
  
* **pr-verify-flyway-migrate-docker** - Triggered by all PRs
  1. Executes all pending migrations and tests against the database docker image

* **pr-verify-sonar-scanner** - Triggered by all PRs
  1. Executes sonar-scanner against the pull request

* **flyway-migrate-qa** - Triggered by all merges to master.
  1. Execute new versioned migrations and updated repeatable migrations against QA
  1. Execute new versioned migrations, updated repeatable migrations, and new test data migrations against persisted 
  database (used in validation step 3 of pr-verify)
  1. Backs up the persisted database and pushes it to Artifactory
  1. Posts to [#shared-databases](https://hy-vee.slack.com/messages/CDQ6QASUB) if an error occurs.
  
* **build-docker-image** - Triggered by pushing a new database backup to Artifactory (**flyway-migrate-qa**)
  1. Builds and pushes a new version of the database docker container

* **flyway-migrate-prod** - Triggered by push-button deployment
  1. Execute new versioned migrations and updated repeatable migrations against PROD
  1. Execute new versioned migrations and updated repeatable migrations against the copy of PROD
  1. Execute new versioned migrations, update repeatable migrations, and new test data against copies of the database in
  dependent environments
  1. Posts to [#shared-databases](https://hy-vee.slack.com/messages/CDQ6QASUB) if an error occurs.
  
* **sonar-scanner** - Triggered by push-button deployment after **flyway-migrate-prod**
  1. Runs static code analysis against all objects
       *    Analysis is viewable [here](https://dev-sonar.hy-vee.com/projects) for each repo 
  
