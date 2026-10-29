# Spring Petclinic with azd

This Azure Developer CLI (`azd`) template deploys the Spring Petclinic sample
application to Azure Container Apps. The service host is `springbootapp`, while
the extension ID remains `azure.springboot`. Native azd container services build
and publish the image; the extension prepares the Maven container and PostgreSQL
configuration, deploys the template-authored service Bicep module, and waits for
readiness. The provider has no base target. No deployment hooks or authored
Dockerfile are required.

> [!IMPORTANT]
> `azure.springboot` 0.1.0 is an unpublished, unsupported development
> prototype. It is not available from a public azd extension registry. Build
> and install the extension from the pinned public source described below.
> The earlier `azure.springboot` service host is unsupported and has no
> compatibility alias. `host: springbootapp` requires a matching rebuilt
> extension bundle.

## Prerequisites

- [Azure Developer CLI](https://aka.ms/azd) 1.37.2 or newer
- [Git](https://git-scm.com/)
- [Go](https://go.dev/) 1.26.4
- Java 17 or newer (full JDK); this repository includes the Maven Wrapper
- A running Docker-compatible engine
- Azure Bicep CLI (azd can install it when it is missing)
- An Azure subscription, only when you are ready to run `azd up`

## Install the prototype extension

The extension source is pinned to
[`XiaofeiCao/azure-dev@9e095c5`](https://github.com/XiaofeiCao/azure-dev/tree/9e095c5064fb277c05ef7be3d20a8016e2dc66d3/cli/azd/extensions/azure.springboot).
The pinned module uses the released azd SDK 1.37.2 and does not require a
patched azd executable.

First install the official azd extension developer kit. If you already have a
source named `dev`, inspect it with `azd extension source list` and only add the
source below when it is absent or points to the same official URL.

```shell
azd extension source add -n dev -t url -l https://aka.ms/azd/extensions/registry/dev
azd extension install microsoft.azd.extensions
```

Then clone the public source at the immutable commit, build a portable bundle,
and install it.

### PowerShell

```powershell
git clone --filter=blob:none --sparse https://github.com/XiaofeiCao/azure-dev.git
Set-Location .\azure-dev
git sparse-checkout set cli/azd/extensions/azure.springboot
git checkout --detach 9e095c5064fb277c05ef7be3d20a8016e2dc66d3
Set-Location .\cli\azd\extensions\azure.springboot

azd x build --skip-install
$bundle = Join-Path $PWD 'azure.springboot_0.1.0.zip'
azd x pack --bundle --output $bundle
azd extension install --force $bundle
```

### Bash

```bash
git clone --filter=blob:none --sparse https://github.com/XiaofeiCao/azure-dev.git
cd azure-dev
git sparse-checkout set cli/azd/extensions/azure.springboot
git checkout --detach 9e095c5064fb277c05ef7be3d20a8016e2dc66d3
cd cli/azd/extensions/azure.springboot

azd x build --skip-install
bundle="$PWD/azure.springboot_0.1.0.zip"
azd x pack --bundle --output "$bundle"
azd extension install --force "$bundle"
```

The bundle is self-contained after it is created. `--force` is required when
replacing an older installed 0.1.0 bundle because the version constraint alone
cannot distinguish builds with different host metadata. Keep the bundle if you
need to install the same extension build in another isolated azd configuration.

## Initialize the template

Start each example in its own empty directory. The default branch of this
repository is the published template.

### H2

```shell
mkdir petclinic-h2
cd petclinic-h2
azd init -t https://github.com/XiaofeiCao/azd-spring-petclinic . -e petclinic-h2 --no-prompt
```

H2 is the default. Its application data is lost when the running instance is
replaced.

### PostgreSQL

```shell
mkdir petclinic-postgresql
cd petclinic-postgresql
azd init -t https://github.com/XiaofeiCao/azd-spring-petclinic . -e petclinic-postgresql --no-prompt
azd env set SPRING_DATABASE_TYPE postgresql
```

Use a new azd environment to change database mode. The template owns the
Container Apps virtual network and dedicated infrastructure subnet with its
attached network security group, so this topology requires a fresh environment;
do not retrofit or switch an already provisioned Container Apps environment in
place. The subnet disables default outbound access, and its network security
group allows public inbound HTTP and HTTPS traffic to the public Container Apps
ingress. PostgreSQL also remains public and is restricted to observed Container
Apps egress addresses, not private database connectivity.

For a new PostgreSQL environment, supply `SPRING_DATABASE_PASSWORD` through the
process or CI environment before the first provision, or let the extension
generate and persist it locally. Reruns reuse the original password. If local
secret state is lost, restore the original password from your secret store
through that process variable before provisioning again; it is never silently
regenerated or rotated. Do not print, commit, or pass real passwords as
command-line arguments.

This demo uses the PostgreSQL administrator login, local azd secret state, and a
native Container Apps secret, not Key Vault or passwordless authentication. If
replacement changes the observed Container Apps egress addresses, retry
`azd deploy` to reconcile the PostgreSQL firewall rules. These are demo
trade-offs, not production guidance. PostgreSQL TLS validates the CA chain using
the JRE trust store and checks the hostname without a per-user `root.crt` file.

## Deploy and package

Authenticate before using Azure:

```shell
azd auth login
```

From an initialized H2 or PostgreSQL directory:

```shell
azd up
```

To build the application package locally without provisioning Azure resources:

```shell
azd package
```

Select your Azure subscription and location when prompted by `azd up`, or set
`AZURE_SUBSCRIPTION_ID` and `AZURE_LOCATION` in the azd environment beforehand.

## Cleanup

Run cleanup from the initialized environment directory:

```shell
azd down --force --purge --no-prompt
```

This removes the template-owned virtual network, subnet, and network security
group with the environment. Container Apps platform-managed networking
resources may incur charges while provisioned.

Local azd state, including the PostgreSQL password, is retained after cleanup;
protect it.

## Validation status

The extension source build, local bundle install, template initialization,
exact-driver PostgreSQL CA and hostname checks, local packaging, and ARM preview
have been exercised without Azure provisioning. The current two-literal JDBC
TLS configuration replaces an earlier attempt that caused PGJDBC to look for a
missing `/app/.postgresql/root.crt`.

An end-to-end Azure run previously reached application image startup, but the
full updated Azure chain has not yet been rerun. Live PostgreSQL CRUD,
persistence, password reuse/recovery, and cleanup acceptance remain pending.
This repository does not claim that the complete cloud lifecycle has passed.

## Source and license

Application source is unchanged from
[`spring-projects/spring-petclinic@500158f`](https://github.com/spring-projects/spring-petclinic/tree/500158f732419217507c7656904b8e6aa1bcc0d6).
[`source-revision.json`](source-revision.json) records its provenance and
selected LF-normalized SHA-256 checksums.

Spring Petclinic is released under version 2.0 of the Apache License. See
[`LICENSE.txt`](LICENSE.txt). This template preserves the upstream project
attribution and license notices.
