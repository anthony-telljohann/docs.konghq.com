---
title: Kong for Kubernetes deployment options
---

- [Deployment options](#deployment-options)
- [DB-less versus database-backed deployments](#db-less-versus-database-backed-deployments)
- [Choosing between DB-less or database-backed deployments](#choosing-between-db-less-or-database-backed-deployments)
  * [Feature availability](#feature-availability)
  * [Plugin compatibility](#plugin-compatibility)
  * [Manual configuration](#manual-configuration)
- [Migrating between deployment types](#migrating-between-deployment-types)

# Deployment options

Kong for Kubernetes provides two options for its Kong Enterprise container
image:

* [kong-enterprise-k8s][k8s-bintray]
* [kong-enterprise-edition][enterprise-bintray]

_These repositories require a login. If you see a 404, log in through the [Kong
repository home page](https://bintray.com/kong) first._

The `kong-enterprise-k8s` image is recommended for most deployments. It
provides most Kong Enterprise plugins and runs without a database, but does not
provide other Kong Enterprise features (Kong Manager, Dev Portal, Vitals,
etc.).

The `kong-enterprise-edition` image is recommended for deployments that require
features not supported by `kong-enterprise-k8s`. It supports all Kong
Enterprise plugins and features, but cannot run without a database.

# DB-less versus database-backed deployments

When using Kong for Kubernetes, the source of truth for Kong's configuration is
the Kubernetes configuration in etcd: Kong's custom Kubernetes resources,
ingresses, and services provide the information necessary for the ingress
controller to configure Kong. This differs from Kong deployments that do not
use an ingress controller, where configuration in the database or DB-less
config file is the source of truth.

In traditional deployments, Kong's database (PostgreSQL or Cassandra) provides
a persistent store of configuration available to all Kong nodes to ensure
consistent proxy behavior across the cluster that is not affected by node
restarts. Because etcd provides this functionality in Kong for Kubernetes
deployments, it is not necessary to run an additional database, reducing
maintenance and infrastructure requirements.

While Kong for Kubernetes does not require a database, it is fully compatible
with PostgreSQL and requires it for some features. etcd remains the source of
truth, but controller can translate Kubernetes resources there into either
DB-less or database configuration.

# Choosing between DB-less or database-backed deployments

In general, DB-less deployments are simpler to maintain and require less
resources to run, and as such are the preferred option for Kong for Kubernetes.
These deployments use the `kong-enterprise-k8s` image and must set
`KONG_DATABASE=off` in their environment variables.

Database-backed deployments offer a wider range of features using the
`kong-enterprise-edition` image. Review the sections below to determine if your
use case requires a feature that is not available in DB-less deployments.

## Feature availability

Some Kong Enterprise features are not available in DB-less deployments. Users
should use the `kong-enterprise-edition` image and a database-backed deployment
if they wish to use:

* Kong Manager
* Dev Portal
* Brain
* Immunity
* Teams (RBAC)
* Vitals
* Workspaces

Because Kong for Kubernetes is configured by the ingress controller, some
functionality in these features is different from traditional deployments:

* Users will not typically create proxy configuration through Kong Manager;
  proxy configuration is normally managed by the controller and users provide
  configuration via Kubernetes resources.
* Because the controller creates proxy configuration on behalf of users, users
  do not need to interact with the Admin API directly. Typical Kong for
  Kubernetes deployments do not expose the Admin API to protect it in lieu of
  RBAC: only the controller can access it. Kubernetes-level RBAC rules and
  namespaces should be used to restrict what configuration administrators can
  create.
* Ingress controller instances create configuration in a single workspace only
  (`default` by default). To use multiple workspaces, users should deploy
  multiple controller instances, setting the `CONTROLLER_KONG_WORKSPACE`
  environment variable to the workspace that instance should use. These
  instances should set `CONTROLLER_INGRESS_CLASS` to unique values, to avoid
  creating duplicate configuration in workspaces. Note that if controller
  instances are deployed outside the Kong pod the Admin API must be exposed,
  and users should enable RBAC with workspace admin users for the controllers.
  Set `CONTROLLER_KONG_ADMIN_TOKEN` to the RBAC user's token.
* The controller cannot manage configuration for the features above: it cannot
  create workspaces, Dev Portal content, admins, etc. These features must be
  configured manually through the Admin API.

## Plugin compatibility

Not all plugins are compatible with DB-less operation, and as such not all
plugins are available in the `kong-enterprise-k8s` image. In addition to the
[compatible core plugins][core-plugins], the `kong-enterprise-k8s` image
includes the following Enterprise plugins:

* canary release
* collector
* degraphql
* forward-proxy
* graphql-proxy-cache-advanced
* graphql-rate-limiting-advanced
* jwt-signer
* kafka-log
* kafka-upstream
* ldap-auth-advanced
* mtls-auth
* oauth2-introspection
* openid-connect
* proxy-cache-advanced
* rate-limiting-advanced
* request-validator
* response-transformer-advanced
* vault-auth

The following Enterprise plugins are not included:

* application-registration (only used with Dev Portal)
* exit-transformer
* key-auth-enc
* request-transformer-advanced
* route-transformer-advanced
* statsd-advanced (only used with Vitals)

Third-party plugins are generally compatible with DB-less as long as they do
not create custom entities (i.e. they do not add new entities that users can
create and modify through the Admin API).

## Manual configuration

DB-less configuration must be supplied as a complete unit: it is not possible
to add or modify entities individually through the Admin API, or provide
partial configuration that is added to existing configuration. As such, all
configuration must be sourced from Kubernetes resources so that the ingress
controller can render it into a complete configuration.

On database-backed deployments, users can create or modify configuration
through the Admin API. The ingress controller uses a tag (set by the
`CONTROLLER_KONG_ADMIN_FILTER_TAG` environment variable) to to identify
configuration that it manages. While the controller will revert changes to
configuration with its tag, other configuration is left as-is.

Although database-backed deployments can use controller-generated and
manually-added configuration simultaneously, Kong's recommended best practice
is to still manage as much configuration through Kubernetes resources as
possible, primarily using manual configuration for temporary or test resources.

# Migrating between deployment types

Because etcd is the source of truth for Kong's configuration, the ingress
controller can re-create Kong's proxy configuration even if the underlying
datastore changes.

While most Kubernetes resources can be left unchanged when migrating between
deployment types, users must remove any KongPlugin resources that use
unavailable plugins when migrating from a database-backed deployment using the
`kong-enterprise-edition` image to a DB-less deployment using the
`kong-enterprise-k8s` image. No changes to Kubernetes resources are required if
migrating in the opposite direction.

[k8s-bintray]: https://bintray.com/kong/kong-enterprise-k8s
[enterprise-bintray]: https://bintray.com/kong/kong-enterprise-edition-docker
[core-plugins]: /latest/db-less-and-declarative-config/#plugin-compatibility
