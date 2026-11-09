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
| `INFISICAL_URL` | InfisicalConnection `default/infisical-prod` | `spec.address` |

All three inputs are optional. Supply at least one.
An empty or whitespace-only input leaves its field unchanged.
If all three inputs are empty, the job fails before it calls Kubernetes.
The client secret is a password parameter.
For the URL, supply the complete API address, for example
`https://infisical.dev.raulstechtips.dev/api`.

The resource names are fixed in the Jenkinsfile. They are not build parameters.
The job does not create resources, change Git, search for other Infisical URLs,
or change an `InfisicalAuth` reference. The existing authentication resource
continues to use the same credentials Secret and connection.

### Prepare Jenkins

1. Merge the `jenkins-shared-library` changes, then this repository.
   The Jenkinsfile loads the library's `main` branch until its first release
   containing this job. When that release is available, replace `main`
   with its release tag.
2. Publish a Jenkins chart containing `infisicalAuthUpdate` and `pipelineJobs`.
3. Deploy that chart with the settings in
   `cluster-config/apps/dev/jenkins.yaml`.
   The new `infisicalAuthUpdate` settings grant `get` and `patch` on only
   the named credentials Secret and connection. They do not add either
   namespace to `deployNamespaces`.
4. Jenkins configuration as code creates the `update-infisical-auth` job.
   It loads this repository's Jenkinsfile through SCM. There is no automatic
   build trigger.
5. On a newly created job, select **Build Now** once to register its
   Declarative Pipeline parameters. This first build fails because no input
   was supplied. It makes no Kubernetes change. Refresh the job page.

The deployer agent must have Bash, jq, and kubectl. The configured
`alpine/k8s:1.30.14` container supplies these tools.
The job uses its Kubernetes service account, not an Infisical login.

### Run the job

1. Open **Jenkins > update-infisical-auth > Build with Parameters**.
2. Fill in only the fields you want to replace.
3. Select **Build**.
4. Open the build's **Console Output**. A successful build prints the names
   of the resources it patched, not their credential values.

Examples:

- Replace only the client secret: fill in `INFISICAL_CLIENT_SECRET`.
  Leave the other two inputs empty.
- Replace the client identity: fill in both client ID and client secret.
- Change the server address: fill in `INFISICAL_URL`.
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
./gradlew test operationsIntegrationTest
```

The integration checks run the Groovy command helper with Bash and jq against
a local Kubernetes fixture. They do not access a cluster.

From `jenkins`, run:

```text
helm lint charts -f ../cluster-config/apps/dev/jenkins.yaml
helm unittest charts
```
