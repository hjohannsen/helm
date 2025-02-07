Acrolinx Platform 2021.05 Helm Chart
===========================================================

This Helm chart installs version 2021.05 of the Acrolinx Platform and version 1.0.5 of the Acrolinx Acrolinx Core Platform operator in a single-node Kubernetes cluster.

About
-------

[Acrolinx][acrolinx-platform] improves content quality.
It helps your writers with standard and custom guidelines and scores your content based on these guidelines.
The higher your Acrolinx Score, the better your content.
It works with over 30 authoring tools and helps writers enhance texts.
Visit our [blog][acrolinx-blog] to learn why better content results in better business.
See the [release notes][acrolinx-release-notes] for what's new in 2021.05.

Since version 2021.02, the Acrolinx Platform has used a [containerized][docker-what-is-a-container] architecture that runs in a Kubernetes cluster.
[Kubernetes][kubernetes-home] (K8s) is an open-source system for automating deployment, scaling, and management of containerized applications.
It has a large, rapidly growing ecosystem.
Kubernetes services, support, and tools are widely available.

The Acrolinx Core Platform operator is an [extension][kubernetes-operator-pattern] to Kubernetes.
It bootstraps the Core Platform and keeps it in the preferred state.
You can think of it as the knowledge of a human operator written in code.

[Helm][helm-home] is a package manager for Kubernetes.
Helm helps define, package, and ship complex Kubernetes applications as so-called [charts][helm-charts].
You can install, upgrade, uninstall, or roll back charts with simple commands.


Prerequisites
--------------

### Software and Environment

You need administrative access to a Kubernetes cluster, [Helm 3][helm-3] and [`kubectl`][kubernetes-kubectl].

You also need access to the Acrolinx Platform images.
Acrolinx distributes the Core Platform images via a private container registry.
The credentials for that registry are temporary.
You can obtain and refresh them through the [Acrolinx download area][acrolinx-docs-download-area].
[The Helm installation can refresh them automatically][this-creds].

### Namespaces

Before you install the Core Platform, you need to create two [namespaces][kubernetes-namespaces]:
* `acrolinx-system`: The namespace in which the operator is running.
* `acrolinx`: The namespace in which the platform is running.
* Optional: A third namespace for the [Helm release][helm-concepts] data.
```shell
kubectl create namespace acrolinx-system
kubectl create namespace acrolinx
```
(You can customize the namespaces in the chart's [values][helm-value-files].)

With the current version of this Helm chart, you **must** install the operator and the Core Platform to different namespaces.

[Helm 3 stores release data in the same namespace as the release][helm-release-data-namespace].
This data is neecessary for the [Helm history][helm-history], for rollbacks and for deinstallation.
The "release namespace" is the one you specify with the `--namespace <namespace>` flag when you install a Helm chart on the [command line][this-usage].
If you leave out that flag, your release data gets stored in the `default` namespace.
Use `--namespace acrolinx-system` to store your release data in the same namespace as the Acrolinx Core Platform operator.
Of course, you can also use a namespace of your own choosing.

This Helm chart only needs the `--namespace` cmd-line flag to specify the location for the release data.
The namespaces for platform and operator are specified via the [values][helm-value-files] `operator.namespace` and `platform.namespace`, respectively.

The Helm chart doesn't manage  your namespaces. This is because the namespaces would get deleted if you tried to uninstall a release.
Instead, it's safer to presume that the namespaces already exist and are the responsibility of your cluster administrator.


Usage
-------

### The Deliverable

You can find the Acrolinx Helm repository and all Acrolinx Helm charts in the [Acrolinx Helm Repository][acrolinx-helm-repo].

Helm charts can be distributed as plain archives, such as `acrolinx-platform-1.0.5+2021.05.tgz`, or via a [Helm repository][helm-repos].
If you plan to install from a repository, you may have to [add that repository][helm-repo-add] first to make sure that the client knows which repositories to search for Helm charts.
You can even unpack the archive and run the installation from the resulting directory.
For the [installation command-line syntax][helm-install], it makes no difference.
The chart name from the repository, the `.`, and the `.tgz` file all appear in the same position.

Add The Repo
-------------

Add the Acrolinx Helm repository as described [here][acrolinx-helm-repo-add].

### Exploration

If you aren't already familiar with the Acrolinx Standard Stack Helm deployment, it's a good idea to start with a quick exploration of the chart.

To [see basic information about the chart][helm-show-chart], run:
```shell
helm show chart acrolinx/acrolinx-platform
```

To [see this README][helm-show-readme], run:
```shell
helm show readme acrolinx/acrolinx-platform
```

To [see the effective default values][helm-show-values], run
```shell
helm show values acrolinx/acrolinx-platform
```

### Configuration

Before you install the Acrolinx Platform to Kubernetes, you may want to change some settings.

#### Customize the Values File
Run
```shell
helm show values acrolinx/acrolinx-platform > custom-values.yaml
```
Move the resulting custom [values][helm-value-files] file to a location where it can persist, for example, in the `acrolinx.yaml` file.
The comments above the individual settings should give good tips about what you can do.
Make your modifications.

#### Registry Credentials

Acrolinx distributes the Core Platform images via a private container registry.
You can find temporary credentials for that registry in the [Acrolinx download area][acrolinx-docs-download-area].
The Helm chart refreshes those credentials automatically, but it needs your credentials for the download area to do so.
You can configure these in the `images.downloadAreaUser` and `images.downloadAreaPwd` properties:
```yaml
images:
  downloadAreaUser: "<user name for Acrolinx download area>"
  downloadAreaPwd: "<password for Acrolinx download area>"
```

If you copied the images to your own private registry and are pulling from there, just leave out the `download_area_.*` settings.
Instead, use:
```yaml
images:
  username: "<username for private registry>"
  password: "<password for private registry>"
```

#### User
For security reasons, the Core Platform services shouldn't run with `root` privileges.
Instead, you need to create a dedicated unprivileged user, say `acrolinx`.
You'll then use that user's ID and GID for the platform services:
```yaml
platform:
  spec:
    securityContext:
      runAsUser: <id -u acrolinx>
      runAsGroup: <id -g acrolinx>
```
The `acrolinx` user should also own the [mounted configuration directory described below][this-mount-overlay].

#### Mount the Configuration Directory

If you have an existing Acrolinx installation, we recommend that you mount your [configuration directory][acrolinx-docs-configuration-directory] into the cluster.
Copy the directory to the cluster node if necessary.
This used to be `.config/Acrolinx/ServerConfiguration`.
If you plan to create an Acrolinx installation from scratch, dedicate an empty directory to the Acrolinx configuration on your single cluster node.
For example, if you call it `/home/acrolinx/config`, you'd set the following in your custom values file:
```yaml
platform:
  spec:
    securityContext:
      runAsUser: <acrolinx user id>
      runAsGroup: <acrolinx group id>
    coreServer:
      overlayDirectory:
        volumeSource:
          hostPath:
            ### Path to the custom configuration directory on the host system.
            path: /home/acrolinx/config
```

#### Database Connections

Add the settings for the Targets database to `persistence.credentials`.
You can configure settings for the [terminology database][acrolinx-docs-term-db-settings], the [reporting database][acrolinx-docs-reporting-db-settings], and the [JReport databases][acrolinx-docs-jreport-db-settings] in the `server/bin/persistence.properties` file in the configuration directory.

If you want to test the Acrolinx Platform without any external databases, remove any `persistence.credentials` and set `installTestDB` to `true`:
```yaml
platform:
  persistence:
    installTestDB: true
```

#### TLS for Ingress

We strongly recommend that you only access the Core Platform with a secure connection.
Transport Layer Security (TLS) support is integrated with Træfik and configured by the operator.
You first need to add your server certificate as a secret to the cluster.
[Have a look at the Kubernetes documentation][kubernetes-tls-secrets] to make sure that your certificate is in the right format.
After that, you can configure the Helm chart with the name of that secret.
Note that you need to place the secret in the `acrolinx` namespace.

You can upload your certificate like this:
```shell
kubectl -n acrolinx create secret tls acrolinx-tls-cert --cert=path/to/tls.cert --key=path/to/tls.key
```

Then configure the name of the secret in your values.yaml file:
```yaml
platform:
  spec:
    ingress:
      tlsSecretName: "acrolinx-tls-cert"
```

This will also enable an automatic redirect from port 80 to 443 to ensure the use of the secure connection.

### Installation and Upgrades

The canonical command to install the Acrolinx Platform via Helm is:
```shell
helm install acrolinx --values <custom values file> acrolinx/acrolinx-platform
```
You can also set individual values on the command line. (See the [Helm install reference][helm-install]).

Run
```shell
helm upgrade acrolinx --values <custom values file> acrolinx/acrolinx-platform --namespace acrolinx-system
```

Acrolinx will deliver a new Helm chart for every Standard Stack Acrolinx Platform version.

### Get Status Information

After an installation or upgrade you might need to wait a while for the Core Platform to start.
The length of your wait will depend on the number of images need to be pulled.
```shell
kubectl wait coreplatform acrolinx  --for condition=Ready --timeout=10m -n acrolinx
```

To see a summary of the platform status, run:
```shell
kubectl get coreplatform acrolinx -o jsonpath="{.status.summary}" -n acrolinx | jq
```
For all status details:
```shell
kubectl get coreplatform acrolinx -o jsonpath="{.status}" -n acrolinx | jq
```

### Suspend the Acrolinx Platform

There might be cases where you want to shut down an Acrolinx Platform instance without deleting it completely.
This allows a quick restart and keeps resources such as physical volumes claimed.
You might do this for backups or to free resources that are used by a test or staging instance.

You can do this with the `acrolinx.com/suspend` annotation.
When you set it to the string `"true"`, it will stop all workloads created for the annotated instance:

```yaml
metadata:
  annotations:
    acrolinx.com/suspend: "true"
```

To restart the instance, remove the annotation or set it to `"false"`.

### Troubleshooting

To identify issues, you can use our [support package script][acrolinx-helm-repo-support-package-script].
Just execute the script as described [here][acrolinx-helm-repo-support-package-script-manual].
Send the resulting `.tgz` file to [Acrolinx Support][acrolinx-support].


[acrolinx-blog]: https://www.acrolinx.com/blog/
[acrolinx-docs]: https://docs.acrolinx.com/doc/en
[acrolinx-docs-configuration-directory]: https://docs.acrolinx.com/coreplatform/latest/en/advanced/the-configuration-directory
[acrolinx-docs-download-area]: https://docs.acrolinx.com/coreplatform/latest/en/acrolinx-on-premise-only/maintain-the-core-platform/download-updated-software
[acrolinx-docs-jreport-db-settings]: https://docs.acrolinx.com/coreplatform/2021.05/en/acrolinx-standard-stack-only/external-databases/connect-to-an-external-analytics-database
[acrolinx-docs-reporting-db-settings]: https://docs.acrolinx.com/coreplatform/2021.05/en/acrolinx-standard-stack-only/external-databases/connect-to-an-external-reporting-database
[acrolinx-docs-term-db-settings]: https://docs.acrolinx.com/coreplatform/2021.05/en/acrolinx-standard-stack-only/external-databases/connect-to-an-external-terminology-database
[acrolinx-helm-repo]: https://acrolinx.github.io/helm/
[acrolinx-helm-repo-add]: https://acrolinx.github.io/helm/#add
[acrolinx-helm-repo-support-package-script]: https://acrolinx.github.io/helm/resources/core-platform/tools/support-package.sh
[acrolinx-helm-repo-support-package-script-manual]: https://acrolinx.github.io/helm/resources/core-platform/tools/support-package-user-manual.md
[acrolinx-home]: https://www.acrolinx.com
[acrolinx-platform]: https://www.acrolinx.com/the-acrolinx-content-strategy-governance-platform/
[acrolinx-release-notes]: https://docs.acrolinx.com/coreplatform/2021.05/en/acrolinx-core-platform-releases/acrolinx-release-notes-including-subsequent-service-releases
[acrolinx-support]: https://support.acrolinx.com/hc/en-us
[docker-what-is-a-container]: https://www.docker.com/resources/what-container
[helm-3]: https://helm.sh/blog/helm-3-released/
[helm-charts]: https://helm.sh/docs/topics/charts/
[helm-concepts]: https://helm.sh/docs/intro/using_helm/#three-big-concepts
[helm-history]: https://helm.sh/docs/helm/helm_history/
[helm-home]: https://helm.sh/
[helm-install]: https://helm.sh/docs/helm/helm_install/
[helm-release-data-namespace]: https://helm.sh/docs/faq/#release-names-are-now-scoped-to-the-namespace
[helm-repo-add]: https://helm.sh/docs/intro/quickstart/#initialize-a-helm-chart-repository
[helm-repos]: https://helm.sh/docs/topics/chart_repository/
[helm-value-files]: https://helm.sh/docs/chart_template_guide/values_files/
[helm-show-chart]:https://helm.sh/docs/helm/helm_show_chart/
[helm-show-readme]: https://helm.sh/docs/helm/helm_show_readme/
[helm-show-values]: https://helm.sh/docs/helm/helm_show_values/
[kubernetes-home]: https://kubernetes.io/
[kubernetes-kubectl]: https://kubernetes.io/docs/reference/kubectl/overview/
[kubernetes-namespaces]: https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
[kubernetes-operator-pattern]: https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
[kubernetes-tls-secrets]: https://kubernetes.io/docs/concepts/configuration/secret/#tls-secrets
[this-creds]: #Registry-Credentials
[this-mount-overlay]: #Mount-the-Configuration-Directory
[this-usage]: #Usage
