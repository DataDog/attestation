# Predicate type: Provenance v0.1

Type URI: https://in-toto.io/Provenance/v0.1

## Purpose

Describe how an artifact or set of artifacts was produced.

## Model

Provenance is a claim that some entity (`builder`) produced one or more software
artifacts ([Statement]'s `subject`) by executing a series of steps (`recipe`),
using some other artifacts as input (`materials`). The builder is trusted to
have faithfully recorded the provenance; there is no option but to trust the
builder. However, the builder may have performed this operation at the request
of some external, possibly untrusted entity. These untrusted parameters are
captured in the recipe's `entryPoint`, `arguments`, and some of the `materials`.
Finally, the build may have depended on various environmental parameters
(`environment`) that are needed for [reproducing][reproducible] the build
but that are not under external control.

See [Example](#example) for a concrete example.

![Model Diagram](../../images/provenance.svg)

## Schema

```jsonc
{
  // Standard attestation fields:
  "type": "https://in-toto.io/Statement/v0.1",
  "subject": [{ ... }],

  // Predicate:
  "predicateType": "https://in-toto.io/Provenance/v0.1",
  "predicate": {
    "builder": {
      "id": "<URI>"
    },
    "recipe": {
      "type": "<URI>",
      "definedInMaterial": /* integer */,
      "entryPoint": "<STRING>",
      "arguments": { /* object */ },
      "environment": { /* object */ }
    },
    "metadata": {
      "buildStartedOn": "<TIMESTAMP>",
      "buildFinishedOn": "<TIMESTAMP>",
      "materialsComplete": true/false
    },
    "materials": [
      {
        "uri": "<URI>",
        "digest": { /* DigestSet */ },
        "mediaType": "<MEDIA_TYPE>",
        "tags": [ "<STRING>" ]
      }
    ]
  }
}
```

_(Note: This is a Predicate type that fits within the larger
[Attestation](../README.md) framework.)_

### Fields

<a id="type"></a>
`type` _string ([TypeURI]), required_

> Standard [Predicate](../README.md#predicate) field. Always
> `https://in-toto.io/Provenance/v0.1` for this version of the spec.

<a id="builder"></a>
`builder` _object, required_

> Identifies the entity that executed the recipe, which is trusted to have
> correctly performed the operation and populated this provenance.
>
> The identity MUST reflect the trust base that consumers care about. How
> detailed to be is a judgement call. For example, [GitHub Actions] supports
> both GitHub-hosted runners and self-hosted runners. The GitHub-hosted runner
> might be a single identity because, it's all GitHub from the consumer's
> perspective. Meanwhile, each self-hosted runner might have its own identity
> because not all runners are trusted by all consumers.
>
> Consumers MUST accept only specific (signer, builder) pairs. For example, the
> "GitHub" can sign provenance for the "GitHub Actions" builder, and "Google"
> can sign provenance for the "Google Cloud Build" builder, but "GitHub" cannot
> sign for the "Google Cloud Build" builder.
>
> Design rationale: The builder is distinct from the signer because one signer
> may generate attestations for more than one builder, as in the GitHub Actions
> example above. The field is required, even if it is implicit from the signer,
> is to aid readability and debugging. It is an object to allow additional
> fields in the future, in case one URI is not sufficient.

<a id="builder.id"></a>
`builder.id` _string ([TypeURI]), required_

> URI indicating the builder's identity.

<a id="recipe"></a>
`recipe` _object, optional_

> Identifies the configuration used for the build. When combined with
> `materials`, this SHOULD fully describe the build, such that re-running this
> recipe results in bit-for-bit identical output (if the build is
> [reproducible]).
>
> MAY be unset/null if unknown, but this is DISCOURAGED.

<a id="recipe.type"></a>
`recipe.type` _string ([TypeURI]), required_

> URI indicating what type of recipe was performed. It determines the meaning of
> `recipe.entryPoint`, `recipe.arguments`, `recipe.environment`, and
> `materials`.

<a id="recipe.definedInMaterial"></a>
`recipe.definedInMaterial` _integer, optional_

> Index in `materials` containing the recipe steps that are not implied by
> `recipe.type`. For example, if the recipe type were "make", then this would
> point to the source containing the Makefile, not the `make` program itself.
>
> Omit this field (or use null) if the recipe doesn't come from a material.
>
> TODO: What if there is more than one material?

<a id="recipe.entryPoint"></a>
`recipe.entryPoint` _string, optional_

> String identifying the entry point into the build. This is often a path to a
> configuration file and/or a target label within that file. The syntax and
> meaning are defined by `recipe.type`. For example, if the recipe type were
> "make", then this would reference the directory in which to run `make` as well
> as which target to use.
>
> Consumers SHOULD accept only specific `recipe.entryPoint` values. For example,
> a policy might only allow the "release" entry point but not the "debug" entry
> point. (This is REQUIRED for [SLSA 2][SLSA] policies.)
>
> MAY be omitted if the recipe type specifies a default value.
>
> Design rationale: The `entryPoint` is distinct from `arguments` to make it
> easier to write secure policies without having to parse `arguments`.

<a id="recipe.arguments"></a>
`recipe.arguments` _object, optional_

> Collection of all external inputs that influenced the build on top of
> `recipe.definedInMaterial` and `recipe.entryPoint`. For example, if the recipe
> type were "make", then this might be the flags passed to `make` aside from the
> target, which is captured in `recipe.entryPoint`.
>
> Consumers SHOULD accept only "safe" `recipe.arguments`. The simplest and
> safest way to achieve this is to disallow any `arguments` altogether. (This is
> REQUIRED for [SLSA 3][SLSA] policies.)
>
> This is an arbitrary JSON object with a schema is defined by `recipe.type`.
>
> Omit this field (or use null) to indicate "no arguments."

<a id="recipe.environment"></a>
`recipe.environment` _object, optional_

> Collection of all builder-controlled inputs that influenced the build.
> `definedInMaterial` and `recipe.entryPoint`. Usually this contains the
> information necessary to [reproduce][reproducible] the build but not
> meaningful to apply a policy against.
>
> This is an arbitrary JSON object with a schema is defined by `recipe.type`.
>
> TODO: Is there a better name for this? "Reproducibility" sounds more like a
> property (enum or bool) rather than a set of things needed for reproduction.

<a id="metadata"></a>
`metadata` _object, optional_

> Other properties of the build.

<a id="metadata.buildStartedOn"></a>
`metadata.buildStartedOn` _string ([Timestamp]), optional_

> The timestamp of when the build started.

<a id="metadata.buildFinishedOn"></a>
`metadata.buildFinishedOn` _string ([Timestamp]), optional_

> The timestamp of when the build completed.

<a id="metadata.materialsComplete"></a>
`metadata.materialsComplete` _boolean, optional_

> If true, `materials` is claimed to be complete, usually through some controls
> to prevent network access.

<a id="materials"></a>
`materials` _array of objects, optional_

> The collection of artifacts that influenced the build including sources,
> dependencies, build tools, base images, and so on.

<a id="materials.uri"></a>
`materials[*].uri` _string ([ResourceURI]), optional_

> The method by which this artifact was referenced during the build.
>
> TODO: Should we differentiate between the "referenced" URI and the "resolved"
> URI, e.g. "latest" vs "3.4.1"?
>
> TODO: Should wrap in a `locator` object to allow for extensibility, in case we
> add other types of URIs or other non-URI locators?

<a id="materials.digest"></a>
`materials[*].digest` _object ([DigestSet]), optional_

> Collection of cryptographic digests for the contents of this artifact.

<a id="materials.mediaType"></a>
`materials[*].mediaType` _string (Media Type), optional_

> The [Media Type](https://www.iana.org/assignments/media-types/) for this
> artifact, if known.

<a id="materials.tags"></a>
`materials[*].tags` _array (of strings), optional_

> Unordered set of labels whose meaning is dependent on `recipe.type`. SHOULD be
> sorted lexicographically.
>
> TODO: Recommend specific conventions, e.g. `source` and `dev-dependency`.

## Example

WARNING: This is just for demonstration purposes.

Suppose the builder downloaded `example-1.2.3.tar.gz`, extracted it, and ran
`make -C src foo CFLAGS=-O3`, resulting in a file with hash `5678...`. Then the
provenance might look like this:

```jsonc
{
  "type": "https://in-toto.io/Statement/v0.1",
  // Output file; name is "_" to indicate "not important".
  "subject": [{"name": "_", "digest": {"sha256": "5678..."}}],
  "predicateType": "https://in-toto.io/Provenance/v0.1",
  "predicate": {
    "builder": { "id": "mailto:person@example.com" },
    "recipe": {
      "type": "https://example.com/Makefile",
      "definedInMaterial": 0,                 // material containing the Makefile
      "entryPoint": "src:foo",                // target "foo" in directory "src"
      "arguments": {"CFLAGS": "-O3"}          // extra args to `make`
    },
    "materials": [{
      "uri": "https://example.com/example-1.2.3.tar.gz",
      "digest": {"sha256": "1234..."}
    }]
  }
}
```


## More examples

### GitHub Actions

WARNING: This is only for demonstration purposes. The GitHub Actions team has
not yet reviewed or approved this design, and it is not yet implemented. Details
are subject to change!

GitHub-Hosted runner:

```json
"builder": {
  "id": "https://github.com/Attestations/GitHubHostedActions@v1"
}
```

Self-hosted runner: Not yet supported. We need to figure out a URI scheme that
represents what system hosted the runner, or perhaps add additional properties
in `builder`.

GitHub Actions Workflow:

```jsonc
"recipe": {
  // Build steps were defined in a GitHub Actions Workflow file ...
  "type": "https://github.com/Attestations/GitHubActionsWorkflow@v1",
  // ... in the git repo described by `materials[0]` ...
  "definedInMaterial": 0,
  // ... at the path .github/workflows/build.yaml, using the job "build".
  "entryPoint": "build.yaml:build",
  // The only possible user-defined parameters that can affect the build are the
  // "inputs" to a workflow_dispatch event. This is unset/null for all other
  // events.
  "arguments": {
    "inputs": { ... }
  }
}
```

### Google Cloud Build

WARNING: This is only for demonstration purposes. The Google Cloud Build team
has not yet reviewed or approved this design, and it is not yet implemented.
Details are subject to change!

Google-hosted worker:

```json
"builder": {
  "id": "https://cloudbuild.googleapis.com/GoogleHostedWorker@v1"
}
```

Custom worker: Not yet supported. We need to figure out a URI scheme that
represents what system hosted the worker, or perhaps add additional properties
in `builder`.

Cloud Build config-as-code
([BuildTrigger](https://cloud.google.com/build/docs/api/reference/rest/v1/projects.triggers)
with `filename`):

```jsonc
"recipe": {
  // Build steps were defined in a cloudbuild.yaml file ...
  "type": "https://cloudbuild.googleapis.com/CloudBuildYaml@v1",
  // ... in the git repo described by `materials[0]` ...
  "definedInMaterial": 0,
  // ... at the path path/to/cloudbuild.yaml.
  "entryPoint": "path/to/cloudbuild.yaml",
  // The only possible user-defined parameters that can affect a BuildTrigger
  // are the subtitutions in the BuildTrigger.
  "arguments": {
    "substitutions": {...}
  }
}
```

Cloud Build with steps defined in a trigger or over RPC:

```jsonc
"recipe": {
  // Build steps were provided as an argument. No `definedInMaterial` or
  // `entryPoint`.
  "type": "https://cloudbuild.googleapis.com/CloudBuildSteps@v1",
  "arguments": {
    // The steps that were performed. (Format TBD.)
    "steps": [...],
    // The substitutions in the build trigger.
    "substitutions": {...}
    // TODO: Any other arguments?
  }
}
```

### Manually run commands

WARNING: This is just a proof-of-concept. It is not yet standardized.

Execution of arbitrary commands:

```jsonc
"recipe": {
  // There was no entry point, and the commands were run in an ad-hoc fashion.
  // There is no `definedInMaterial` or `entryPoint`.
  "type": "https://example.com/ManuallyRunCommands@v1",
  "arguments": {
    // The list of commands that were executed.
    "commands": [
      "tar xvf foo-1.2.3.tar.gz",
      "cd foo-1.2.3",
      "./configure --enable-some-feature",
      "make foo.zip"
    ],
    // Indicates how to parse the strings in `commands`.
    "shell": "bash"
  }
}
```

## Appendix: Review of CI/CD systems

See [ci_survey.md](../../ci_survey.md) for a list of well-known CI/CD systems,
to make sure they all map cleanly into this schema.

[DigestSet]: ../field_types.md#DigestSet
[GitHub Actions]: #github-actions
[Reproducible]: https://reproducible-builds.org
[ResourceURI]: ../field_types.md#ResourceURI
[SLSA]: https://github.com/slsa-framework/slsa
[Statement]: ../README.md#statement
[Timestamp]: ../field_types.md#Timestamp
[TypeURI]: ../field_types.md#TypeURI