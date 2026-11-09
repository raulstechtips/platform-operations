# platform-operations

Manual Jenkins jobs for cluster operations. Each job calls the shared library.
These jobs do not build an application repository.

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
3. In Jenkins, select **New Item**. Enter `update-infisical-auth`, select
   **Pipeline**, then select **OK**.
4. Under **Pipeline**, set **Definition** to **Pipeline script from SCM**.
   Select **Git** and enter
   `https://github.com/raulstechtips/platform-operations.git`.
   Select the existing `github-app` credential, set **Branch Specifier** to
   `*/main`, and set **Script Path** to
   `pipelines/update-infisical-auth.Jenkinsfile`.
   Leave build triggers disabled, then select **Save**.
   Adding another manual job uses the same UI steps with its own script path.
   It requires no Jenkins chart field or controller restart.
5. On a newly created job, select **Build Now** once to register its
   Declarative Pipeline parameters. This first build fails because no input
   was supplied. It makes no Kubernetes change. Refresh the job page.

The deployer agent must have Bash, jq, and kubectl. The configured
`alpine/k8s:1.30.14` container supplies these tools.
The job uses its Kubernetes service account, not an Infisical login.

### Run the job

1. Open **Jenkins > update-infisical-auth > Build with Parameters**.
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

## Local verification

From `jenkins-shared-library`, run:

```text
./gradlew test
```

The unit tests check input validation, fixed targets, connection selection,
and selective command generation. They do not access a cluster.

From `jenkins`, run:

```text
helm lint charts -f ../cluster-config/apps/dev/jenkins.yaml
helm unittest charts
```
