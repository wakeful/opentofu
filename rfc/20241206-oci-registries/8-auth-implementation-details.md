# Authentication-related Implementation Details

---

This document is part of the [OCI registries RFC](../20241206-oci-registries.md).

| [« Previous](7-open-questions.md) | [Up](../20241206-oci-registries.md) | [Next »](9-provider-implementation-details.md) |

---

This appendix discusses implementation details related to [OCI Registry Authentication](6-authentication.md).

## Registry authentication is a cross-cutting concern

While most of what we've discussed in this RFC is defined separately for provider and module package installation, the _authentication_ implementation should be shared between both and designed so that it could potentially be used for any other OCI registry interactions we might implement in future, such as hypothetical support for storing OpenTofu state snapshots as OCI artifacts.

Therefore our primary concern for implementation is in centrally-managing the credentials settings and then making them available to the relevant components of both the provider and module package installers.

## OpenTofu CLI Configuration and `package main`

Although there is considerable existing legacy code not following this pattern, the current intended design for OpenTofu is to follow [the dependency inversion principle](https://en.wikipedia.org/wiki/Dependency_inversion_principle) with `package main` acting as the ultimate arbiter of how different subsystems are configured to work together. The main package in turn uses `package cliconfig` (`internal/command/cliconfig`) to decide most of the locally-user-configurable settings that can affect those dependency resolution decisions.

We will continue that design approach by teaching `package cliconfig` to decode and validate the `oci_default_credentials` and `oci_credentials` blocks in the CLI configuration, returning the discovered information as part of the overall `cliconfig.Config` object generated by that package.

The implicit configuration mode acts as an alternative way to populate the same settings from Docker CLI or other container system configuration files, and so would also be accessed through `package cliconfig` by mapping the concepts from the Docker CLI configuration language to the same internal data types that we would decode our explicit configuration into, so that the rest of the system does not need to be concerned about how that information was discovered.

`package main` is responsible for using the information returned from the CLI configuration to tell other subsystems how to configure and instantiate an OCI Distribution client, which will then encapsulate all of the OCI registry interactions, including the selection of and inclusion of appropriate credentials when making requests.

## OCI Registry Credentials Policy Layer

> [!NOTE]
> The Go signatures included below are intended to be illustrative of overall architecture rather than concrete/final. Although the final implementation will follow the same general structure described here, the specific methods, arguments, etc will be finalized during the implementation phase based on the discovered practical requirements, and thus will be discussed primarily during code review rather than RFC review.
>
> If you are reading this document after the implementation is complete, prefer to consult the real API documentation in the current Go packages in case the finer details have changed from what's described here.

Since the mechanisms for configuring credentials are non-trivial and the logic for selecting a single set of credentials based on the configuration are relatively complex, we will encapsulate the model of credentials-selection policy into a separate `package ociauthconfig`, placed at `internal/command/cliconfig/ociauthconfig` to reflect its close relationship with the CLI configuration language.

The main functionality in this package will be a function that takes a set of objects representing individual sources of credentials configuration (e.g. individal Docker CLI configuration files, or blocks from the main OpenTofu CLI Configuration), finds all of the credential sources relevant to a particular OCI repository address, and then chooses the one that matches the repository address most specifically as described in [Credentials Selection Precedence](6-authentication.md#credentials-selection-precedence).

That selection will work in terms of an interface `CredentialsConfig` which conceptually represents some sort of configuration artifact that can contain zero or more credential settings:

```go
// CredentialsConfig is implemented by objects that can provide zero or more
// [CredentialsSource] objects given a registry domain and repository path.
//
// This package has its own implementaion of this interface in terms of a
// Docker CLI-style credentials configuration file, accessible through
// [FindDockerCLIStyleCredentialsConfigs] and [FixedDockerCLIStyleCredentialsConfigs],
// but package cliconfig also implements this separately for OpenTofu's own
// OCI credentials config language included as part of the OpenTofu CLI
// configuration.
type CredentialsConfig interface {
    // CredentialsSourcesForRepository returns a sequence of all of the individual
    // credentials sources in the associated configuration that match the given
    // OCI registry domain and repository path.
    //
    // If multiple returned credential sources have the same specificity and that
    // specificity turns out to be the highest available, the one returned earlier
    // in the sequence "wins" for the sake of credentials selection.
	CredentialsSourcesForRepository(ctx context.Context, registryDomain string, repositoryPath string) iter.Seq2[CredentialsSource, error]
}
```

This in turn returns a sequence of objects implementing another interface `CredentialsSource`, which represents a method for obtaining a _single_ set of credentials:

```go
type CredentialsSource interface {
	CredentialsSpecificity() CredentialsSpecificity
	Credentials(ctx context.Context) (Credentials, error)
}
```

This additional indirection allows the main credentials-selection logic to first use the "specificity" value to choose a single credentials source to use, and only then to call `Credentials` on it to obtain the final concrete credentials to use. This is particularly important for the `CredentialsSource` implementation that wraps Docker-style credentials helpers, since we'll want to avoid executing any credentials helper until it's the finally-chosen single source.

`Credentials` is an opaque struct type encapsulating some concrete credentials. Since our initial implementation will rely on the OCI Distribution client implementation from the upstream library from the ORAS project, `Credentials` will initially offer just a single method for translating our internal representation into the representation expected by that library, with the expectation that we'll change this specific API later if we find a reason to use a different client library:

```go
import (
	orasauth "oras.land/oras-go/v2/registry/remote/auth"
)

func (c *Credentials) ForORAS() orasauth.Credential
```

`CredentialsSpecificity` is a type that represents the approximate levels of specificity used for selecting credentials in the Docker configuration language, whose rules we are also adopting for matching the `oci_credentials` blocks in OpenTofu's CLI configuration. There will initially be three constants of type `CredentialsSpecificity`, and one function for representing a dynamically-detected number of matching path segments:

```go
// NoCredentialsSpecificity is the zero value of CredentialsSpecificity,
// representing the total absense of a specificity value.
const NoCredentialsSpecificity CredentialsSpecificity

// GlobalCredentialsSpecificity is the lowest level of specificity, used for
// credentials that are defined globally, rather than domain-specific or
// repository-specific.
const GlobalCredentialsSpecificity CredentialsSpecificity

// DomainCredentialsSpecificity is the second-lowest level of specificity, used
// for credentials that are associated with an entire domain.
const DomainCredentialsSpecificity CredentialsSpecificity

// RepositoryCredentialsSpecificity returns a CredentialsSpecificity for a
// repository path with a given number of path segments.
//
// "path segments" means the number of segments that appear in the
// slash-separated repository path. For example, "foo/bar/baz" has three
// segments.
//
// If pathSegments is zero then the result is equal to
// [DomainCredentialsSpecificity], since that represents just a domain match
// without any path segment matches.
func RepositoryCredentialsSpecificity(pathSegments uint) CredentialsSpecificity
```

The declarations above are in increasing order of precedence, with `NoCredentialsSpecificity` representing no selection at all, and `GlobalCredentialsSpecificity` representing the lowest valid precedence. `RespositoryCredentialsSpecificity` results with higher values of `pathSegments` are more specific.

The internal representation of `CredentialsSpecificity` is not guaranteed by the API of `package ociauthconfig` so that we can evolve it in future if we discover new requirements, but for the initial implementation it will just be an integer where `0` represents `NoCredentialsSpecificity` and higher integers represent the gradually-higher levels of specificity. Code outside of `package ociauthconfig` is not allowed to rely on it being an integer, and so we can potentially make it a more complex representation in later releases if needed.

Bringing this all together, the main entry-point to `package ociauthconfig` is the concrete type `CredentialsConfigs`, which represents a sequence of `CredentialsConfig` implementations and offers one method:

```go
func (cc *CredentialsConfigs) CredentialsSourceForRepository(ctx context.Context, registryDomain, repositoryPath string) (CredentialsSource, error)
```

The pseudocode for the implementation of this function is:

- Let `result` be a `nil` value of type `CredentialsSource`, and `resultSpec` be initialized as `NoCredentialsSpecificity`.
- For each `CredentialsConfig` implementation, named `cfg`:
  - Let `sources` be the result of `cfg.CredentialsSourcesForRepository` with the given `registryDomain` and `repositoryPath`.
  - For each `CredentialsSource` implementation in `sources`, named `source`:
      - If `source.CredentialsSpecificity` (named `candidateSpec`) is greater than `resultSpec`:
        - Assign `source` to `result` and `candidateSpec` to `resultSpec`.
- If `resultSpec` is still `NoCredentialsSpecificity`, return a "no credentials available" error and terminate.
- Otherwise, return `result` with no error and terminate.

This therefore finds both the highest available specificity for the given repository address and, if multiple are available at that specificity, the "earliest-declared" credentials source at that specificity. `package cliconfig` will instantiate `CredentialsConfigs` with the explicitly-configured credentials blocks first and the ambiently-detected credentials configurations afterward, thus causing explicitly-configured credentials to be preferred over ambiently-detected ones whenever both have equal precedence.

The result is a `CredentialSource`, and so the caller must then finally call `Credentials` on that object to obtain the concrete credentials to use.

Once `package cliconfig` has constructed a `CredentialsConfigs` object based on the configured and/or detected credentials, the rest of the system may depend only on the exported API of `CredentialsConfigs`, `CredentialsSource`, and `Credentials`.

> [!NOTE]
> Ideally we would rely on a third-party library for this non-OpenTofu-specific concern, but in our review of some candidates we found that they are typically not extensible to allow us to integrate our own OpenTofu CLI Configuration-based explicit configuration method. Many of them also come only as part of larger libraries with a significant number of indirect dependencies that we would not otherwise need, and would likely cause false positives for naive security scanners that work only at a whole-Go-module granularity.
>
> Therefore we'll use our own implementation of this concern at least for the first round, but since all of this functionality will be in OpenTofu's `internal` packages for the foreseeable future we will have the freedom to swap for an upstream implementation of similar functionality later if we become aware of one, as long as it is sufficiently compatible. The matching rules above are based on those documented for the Docker-style configuration language, and so we can reasonably assume that third-party libraries would match it well enough.

## OCI Client for the Provider Installer

The provider installation components already follow the dependency inversion principle, with `package main` constructing various implementations of [`getproviders.Source`](https://pkg.go.dev/github.com/opentofu/opentofu/internal/getproviders#Source) based on the `provider_installation` block in the CLI configuration, or the implied fallbacks thereof.

The new `oci_mirror` block type in `provider_installation` will therefore be represented internally as a new implementation of `getproviders.Source`, whose constructor function will take the ORAS implementation of OCI Distribution client, which will in turn be configured to obtain credentials using the centrally-configured `ociauthconfig.CredentialsConfigs` object.

The existing logic for instantiating the provider installation methods in `package main` will then be extended to translate an `oci_mirror` installation method configuration into a suitably-configured instance of the new `getproviders.Source` implementation.

For more information, refer to [Provider implementation details](9-provider-implementation-details.md).

## OCI Client for the Module Package Installer

The module installation mechanisms in OpenTofu are considerably older and have not yet been completely adapted to follow the dependency inversion principle. Therefore some futher refactoring of that subsystem will be required to implement this proposal. We will take inspiration from the design of the provider installation process to improve the consistency between these two subsystems.

Currently `package getmodules` (`internal/getmodules`) contains some statically-initialized data structures that act as configuration for the third-party library [`go-getter`](https://pkg.go.dev/github.com/hashicorp/go-getter), which OpenTofu relies on for all module package retrieval. Those static data structures are exposed to external callers only indirectly through [`getmodules.PackageFetcher`](https://pkg.go.dev/github.com/opentofu/opentofu/internal/getmodules#PackageFetcher), whose instantiation function currently takes no arguments because all of its dependencies are statically configured inside the package.

`getmodules.PackageFetcher` instances are currently instantiated inline within some of the functions of [`package initwd`](https://pkg.go.dev/github.com/opentofu/opentofu/internal/initwd), as an implementation detail. `package initwd` has seen _some_ efforts to adopt a dependency-inversion-style approach, with `initwd.ModuleInstaller` taking the modules directory, config loader, and module registry protocol client as arguments rather than instantiating them directly itself.

To continue that evolution, we will extend `initwd.NewModuleInstaller` to also take a `getmodules.PackageFetcher` as an argument rather than instantiating it directly inline. We will then extend `package command`'s `Meta` type to include a field for a provided `getmodules.PackageFetcher`, alongside [the existing field for a `getproviders.Source`](https://github.com/opentofu/opentofu/blob/ffa43acfcdc4431f139967198faa2dd20a2752ea/internal/command/meta.go#L127-L130).

[`command.Meta.installModules` currently calls `initwd.NewModuleInstaller` directly](https://github.com/opentofu/opentofu/blob/ffa43acfcdc4431f139967198faa2dd20a2752ea/internal/command/meta_config.go#L294), and so we will extend that call to pass in the provided `getmodules.PackageFetcher` alongside the module registry client and other dependency objects.

[`package main` directly instantiates `command.Meta`](https://github.com/opentofu/opentofu/blob/ffa43acfcdc4431f139967198faa2dd20a2752ea/cmd/tofu/commands.go#L89-L115) as its primary way of injecting dependencies into the CLI command layer, including the population of the `ProviderSource` field described above. We will therefore also pass the centrally-instantiated `getmodules.PackageFetcher` in the same way, completing the chain of dependency passing all the way from `package main` to the module installer.

The support for OCI registries as a module installation source will involve the addition of a new implementation of `go-getter`'s `Getter` interface, which will include the preconfigured OCI Distribution client (from the ORAS library) as one of its fields.

For more information, refer to [Module implementation details](10-module-implementation-details.md).

---

| [« Previous](7-open-questions.md) | [Up](../20241206-oci-registries.md) | [Next »](9-provider-implementation-details.md) |

---