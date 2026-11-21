# platform-operations

Manual Jenkins jobs for cluster operations. Each job calls the shared library.
These jobs do not build an application repository.

## Job discovery

The root `Jenkinsfile` calls `discoverJobsPipeline()` from the shared library.
The existing `raulstechtips` organization folder discovers this repository.
Its `main` job creates and updates the operational jobs in a separate,
root-level `Jobs` folder:

```text
Jenkins home
+-- raulstechtips
|   +-- platform-operations
|       +-- main
+-- Jobs
    +-- Update Infisical Auth
    +-- Create Minio Bucket
    +-- Create Minio User
    +-- Rotate Minio Keys
    +-- Delete Minio User
```

Discovery reads filenames matching `pipelines/*.Jenkinsfile`. It does not
read or execute their Groovy contents. Use lowercase words separated by
hyphens, for example `update-infisical-auth.Jenkinsfile`. Files in nested
directories are not discovered.

The filename supplies the stable job ID, `Jobs/update-infisical-auth`.
The display name replaces hyphens with spaces and capitalizes each word:
`Update Infisical Auth`. No metadata file or display-name parameter is needed.

To add a job, add its Jenkinsfile and merge it to `main`. The normal GitHub
push build refreshes the job list. Only `main` can update the list; feature
branches, pull requests, and tags do not change generated jobs.
Each generated job loads its own Jenkinsfile from this repository's `main`
branch, using the repository's existing SCM credential.

Discovery does not start operational jobs or give them automatic build
triggers. It preserves the parameters and options initialized by each
operational pipeline. Run jobs manually from `Jobs`. Removing a file disables its job
and retains its history. Restoring the same filename re-enables the same
job. Renaming a file disables the old job and creates a new one.
Do not move or rename the discovery job: Job DSL tracks generated jobs
through that job.

## Update Infisical authentication

Entry point: `pipelines/update-infisical-auth.Jenkinsfile`.

The job updates only these existing Kubernetes resources:

| Input | Resource | Field |
| --- | --- | --- |
| `INFISICAL_CLIENT_ID` | Secret `infisical/universal-auth-credentials` | `data.clientId` |
| `INFISICAL_CLIENT_SECRET` | Secret `infisical/universal-auth-credentials` | `data.clientSecret` |
| `INFISICAL_URL` | InfisicalConnection `default/<INFISICAL_CONNECTION_NAME>` | `spec.address` |

All three inputs are optional. Supply at least one.
An empty or whitespace-only input leaves its field unchanged.
If all three inputs are empty, the job fails before it calls Kubernetes.
The client secret is a password parameter.
For the URL, supply the complete API address, for example
`https://infisical.dev.raulstechtips.dev/api`.

The library fixes the credentials namespace to `infisical`, the Secret name
to `universal-auth-credentials`, and the connection namespace to `default`.
`INFISICAL_CONNECTION_NAME` defaults to `infisical-prod`. Set it to another
existing connection, such as `infisical-dev`, when needed.
This parameter selects the target for a URL update; it does not rename a
connection or count as a replacement value when the other three inputs are empty.
The job does not create resources, change Git, search for other Infisical URLs,
or change an `InfisicalAuth` reference. The existing authentication resource
continues to use the same credentials Secret and connection.

### Prepare Jenkins

1. Merge the `jenkins-shared-library` changes, then this repository.
   The Jenkinsfile loads the library's `main` branch until its first release
   containing this job. When that release is available, replace `main`
   with its release tag.
2. Apply the Jenkins values in `cluster-config/apps/dev/jenkins.yaml` through
   the normal deployment process. The existing `deployNamespaces` list includes
   `redis`, `infisical`, and `default`. The chart binds the existing `admin`
   ClusterRole in each namespace. No custom Jenkins chart settings are needed.
   Before a URL update, check that this role includes `get` and `patch` for
   InfisicalConnections; the namespace binding does not add resource rules.
3. Open **Jenkins > raulstechtips > Scan Organization Now**. After the scan,
   open **platform-operations > main** and select **Build Now**.
   If `main` is not listed, run **Scan Multibranch Pipeline Now** first.
   The initial build is explicit because organization indexing does not
   automatically run newly discovered branches in this Jenkins setup.
4. If discovery stops with **script not yet approved for use**, open
   **Manage Jenkins > In-process Script Approval** as an administrator.
   Inspect and approve the Job DSL script that starts with `folder('Jobs')`
   and creates SCM pipelines from `jobDefinitions`. Run
   **raulstechtips > platform-operations > main > Build Now** again.
   The script is fixed; filenames and SCM settings are passed as data.
   Adding or removing pipeline files does not require another approval.
   A later library change to the DSL script requires its own approval.
5. Return to Jenkins home. Open **Jobs > Update Infisical Auth**.
   The generated job uses **Pipeline script from SCM**, branch `*/main`,
   and script path `pipelines/update-infisical-auth.Jenkinsfile`.
   It has no automatic build trigger. No controller restart is required.
6. On a newly created operational job, select **Build Now** once to register its
   Declarative Pipeline parameters. This first build fails because no input
   was supplied. It makes no Kubernetes change. Refresh the job page.

The deployer agent must have Bash, jq, and kubectl. The configured
`alpine/k8s:1.30.14` container supplies these tools.
The job uses its Kubernetes service account, not an Infisical login.

### Run the job

1. Open **Jenkins > Jobs > Update Infisical Auth > Build with Parameters**.
2. Check `INFISICAL_CONNECTION_NAME` if you will change the URL.
   Fill in only the replacement fields you want to change.
3. Select **Build**.
4. Open the build's **Console Output**. A successful build prints the names
   of the resources it patched, not their credential values.

Examples:

- Replace only the client secret: fill in `INFISICAL_CLIENT_SECRET`.
  Leave the other two inputs empty.
- Replace the client identity: fill in both client ID and client secret.
- Change the server address: fill in `INFISICAL_URL`.
- Update the development connection: set `INFISICAL_CONNECTION_NAME` to
  `infisical-dev` and fill in `INFISICAL_URL`.
- Replace all three: fill in all three inputs.

The job validates its inputs and submits server-side dry runs for every
requested patch before it writes anything. Kubernetes merge patches preserve
all omitted fields and unrelated data.

The two resource updates are not one transaction. If an actual update fails
after another update succeeded, the build fails. Check the reported resource
names, then rerun the required fields.

A successful build confirms that Kubernetes accepted the updates. It does not
prove that the replacement identity can authenticate with Infisical or read
every project. Check operator readiness and secret synchronization afterwards.

### Keep Git aligned

The operator is responsible for keeping Git configuration aligned with a URL
change. A later Argo sync can restore the URL declared in Git.
This job deliberately does not update Git.

Changing this connection does not update separate `hostAPI` values on other
resources. Those changes remain a separate operator action.

## Create Minio Bucket

Entry point: `pipelines/create-minio-bucket.Jenkinsfile`.

The job creates bucket `{NAME}-{ENVIRONMENT}` when it does not exist. It also
creates read-only and read-write policies and users for that bucket. It stores
one access key pair for each user in Infisical:

| Input or result | Value |
| --- | --- |
| `NAME` | Bucket base name |
| `ENVIRONMENT` | `dev`, `staging`, or `prod` |
| Bucket | `{NAME}-{ENVIRONMENT}` |
| Policy and user | `{NAME}-{ENVIRONMENT}-{level}` for `ro` and `rw` |
| Infisical project and environment | Project `k3s`, environment `{ENVIRONMENT}` |
| Infisical folder | `/minio/users/{NAME}/{level}` |
| Infisical keys | `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` |

The job does not change an existing bucket or its data. It does not remove a
bucket.

### Prepare Jenkins

The Jenkins credentials `jenkins-minio` and `jenkins-infisical` must exist.
On the first build, Jenkins registers the job parameters. That build has no
input and changes nothing. Refresh the job page after the build.

### Run the job

1. Open **Jenkins > Jobs > Create Minio Bucket > Build with Parameters**.
2. Enter `NAME` and select `ENVIRONMENT`.
3. Select **Build**.
4. Check the build's **Console Output** for the result.

## Create Minio User

Entry point: `pipelines/create-minio-user.Jenkinsfile`.

The job creates a policy and user for each selected access level. It stores
each user's access key pair in Infisical:

| Input or result | Value |
| --- | --- |
| `USER_NAME` | User base name |
| `ACCESS` | `RO`, `RW`, or `BOTH` |
| `BUCKETS` | Comma, space, or newline-separated bucket names |
| `ENVIRONMENT` | `dev`, `staging`, or `prod` |
| Policy and user | `{USER_NAME}-{ENVIRONMENT}-{level}` |
| Infisical project and environment | Project `k3s`, environment `{ENVIRONMENT}` |
| Infisical folder | `/minio/users/{USER_NAME}/{level}` |
| Infisical keys | `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` |

`BOTH` creates `ro` and `rw` users. The job does not create the buckets in
`BUCKETS` or remove any bucket.

### Prepare Jenkins

The Jenkins credentials `jenkins-minio` and `jenkins-infisical` must exist.
On the first build, Jenkins registers the job parameters. That build has no
input and changes nothing. Refresh the job page after the build.

### Run the job

1. Open **Jenkins > Jobs > Create Minio User > Build with Parameters**.
2. Enter the user name and bucket names. Select the access level and
   environment.
3. Select **Build**.
4. Check the build's **Console Output** for the result.

## Rotate Minio Keys

Entry point: `pipelines/rotate-minio-keys.Jenkinsfile`.

The job creates a new access key for each selected existing user, stores it
in Infisical, then removes the user's other access keys:

| Input or result | Value |
| --- | --- |
| `NAME` | User base name |
| `ENVIRONMENT` | `dev`, `staging`, or `prod` |
| `ACCESS` | `RO`, `RW`, or `BOTH` |
| User | `{NAME}-{ENVIRONMENT}-{level}` |
| Infisical project and environment | Project `k3s`, environment `{ENVIRONMENT}` |
| Infisical folder | `/minio/users/{NAME}/{level}` |
| Infisical keys | `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` |

Consumers fail to access MinIO after rotation until they reload the new secret.
The job does not change buckets, policies, or users.

### Prepare Jenkins

The Jenkins credentials `jenkins-minio` and `jenkins-infisical` must exist.
On the first build, Jenkins registers the job parameters. That build has no
input and changes nothing. Refresh the job page after the build.

### Run the job

1. Open **Jenkins > Jobs > Rotate Minio Keys > Build with Parameters**.
2. Enter the user name. Select the environment and access level.
3. Select **Build**.
4. Make each consumer reload its secret. Check the build's **Console Output**
   for the result.

## Delete Minio User

Entry point: `pipelines/delete-minio-user.Jenkinsfile`.

The job removes the selected users, their policies, and their Infisical
folders:

| Input or result | Value |
| --- | --- |
| `NAME` | User base name |
| `ENVIRONMENT` | `dev`, `staging`, or `prod` |
| `ACCESS` | `RO`, `RW`, or `BOTH` |
| `CONFIRM_NAME` | Must equal `NAME` |
| Policy and user | `{NAME}-{ENVIRONMENT}-{level}` |
| Infisical project and environment | Project `k3s`, environment `{ENVIRONMENT}` |
| Infisical folder | `/minio/users/{NAME}/{level}` |

The job never removes a bucket. It logs: `Bucket is not removed by this job.`

### Prepare Jenkins

The Jenkins credentials `jenkins-minio` and `jenkins-infisical` must exist.
On the first build, Jenkins registers the job parameters. That build has no
input and changes nothing. Refresh the job page after the build.

### Run the job

1. Open **Jenkins > Jobs > Delete Minio User > Build with Parameters**.
2. Enter the user name in both `NAME` and `CONFIRM_NAME`. Select the
   environment and access level.
3. Select **Build**.
4. Check the build's **Console Output** for the result.

## Local verification

From `jenkins-shared-library`, run:

```text
./gradlew test
```

The unit tests check discovery gating, filename mapping, job-generation
settings, authentication input validation, fixed targets, connection selection,
and selective command generation. They do not access a cluster.

From `jenkins`, run:

```text
helm lint charts -f ../cluster-config/apps/dev/jenkins.yaml
helm unittest charts
```
