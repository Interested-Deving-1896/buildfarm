[update-readmes]   Mode: rewrite — migrating to template structure...
# buildfarm

[![Built with Ona](https://ona.com/build-with-ona.svg)](https://app.ona.com/#https://github.com/Interested-Deving-1896/buildfarm)

<!-- AI:start:what-it-does -->
_Description pending._
<!-- AI:end:what-it-does -->

## Architecture

<!-- AI:start:architecture -->
_Architecture documentation pending._
<!-- AI:end:architecture -->

## Install

<!-- Add installation instructions here. This section is yours — the AI will not modify it. -->

```bash
git clone https://github.com/Interested-Deving-1896/buildfarm.git
cd buildfarm
```

## Usage


All commandline options override corresponding config settings.

### Redis

Run via

```shell
$ docker run -d --rm --name buildfarm-redis -p 6379:6379 redis:7.2.4
redis-cli config set stop-writes-on-bgsave-error no
```

### Buildfarm Server

Run via

```shell
$ bazelisk run //src/main/java/build/buildfarm:buildfarm-server -- <logfile> <configfile>

Ex: bazelisk run //src/main/java/build/buildfarm:buildfarm-server -- --jvm_flag=-Djava.util.logging.config.file=$PWD/examples/logging.properties $PWD/examples/config.minimal.yml
```
**`logfile`** has to be in the [standard java util logging format](https://docs.oracle.com/cd/E57471_01/bigData.100/data_processing_bdd/src/rdp_logging_config.html) and passed as a --jvm_flag=-Dlogging.config=file:
**`configfile`** has to be in [yaml format](https://buildfarm.github.io/buildfarm/docs/configuration).

### Buildfarm Worker

Run via

```shell
$ bazelisk run //src/main/java/build/buildfarm:buildfarm-shard-worker -- <logfile> <configfile>

Ex: bazelisk run //src/main/java/build/buildfarm:buildfarm-shard-worker -- --jvm_flag=-Djava.util.logging.config.file=$PWD/examples/logging.properties $PWD/examples/config.minimal.yml

```
**`logfile`** has to be in the [standard java util logging format](https://docs.oracle.com/cd/E57471_01/bigData.100/data_processing_bdd/src/rdp_logging_config.html) and passed as a --jvm_flag=-Dlogging.config=file:
**`configfile`** has to be in [yaml format](https://buildfarm.github.io/buildfarm/docs/configuration).

### Bazel Client

To use the example configured buildfarm with bazel (version 1.0 or higher), you can configure your `.bazelrc` as follows:

```shell
$ cat .bazelrc
$ build --remote_executor=grpc://localhost:8980
```

Then run your build as you would normally do.

### Debugging

Buildfarm uses [Java's Logging framework](https://docs.oracle.com/javase/10/core/java-logging-overview.htm) and outputs all routine behavior to the NICE [Level](https://docs.oracle.com/javase/8/docs/api/java/util/logging/Level.html).

You can use typical Java logging configuration to filter these results and observe the flow of executions through your running services.
An example `logging.properties` file has been provided at [examples/logging.properties](examples/logging.properties) for use as follows:

```shell
$ bazel run //src/main/java/build/buildfarm:buildfarm-server -- --jvm_flag=-Djava.util.logging.config.file=$PWD/examples/logging.properties $PWD/examples/config.minimal.yml
```

and

``` shell
$ bazel run //src/main/java/build/buildfarm:buildfarm-shard-worker -- --jvm_flag=-Djava.util.logging.config.file=$PWD/examples/logging.properties $PWD/examples/config.minimal.yml
```

To attach a remote debugger, run the executable with the `--debug=<PORT>` flag. For example:

```shell
$ bazel run //src/main/java/build/buildfarm:buildfarm-server -- --debug=5005 $PWD/examples/config.minimal.yml
```


### Third-party Dependencies

Most third-party dependencies (e.g. protobuf, gRPC, ...) are managed automatically via
[rules_jvm_external](https://github.com/bazelbuild/rules_jvm_external). These dependencies are enumerated in
the WORKSPACE with a `maven_install` `artifacts` parameter.

Things that aren't supported by `rules_jvm_external` are being imported as manually managed remote repos via
the `WORKSPACE` file.

### Deployments

Buildfarm can be used as an external repository for composition into a deployment of your choice.
See also the documentation site in the [Worker Execution Environment](https://buildfarm.github.io/buildfarm/docs/architecture/worker-execution-environment/) section.

Add the following to your `MODULE.bazel` to get access to buildfarm targets, filling in the `<COMMIT-SHA>` values:

```starlark
bazel_dep(name = "build_buildfarm")
git_override(
    module_name = "build_buildfarm",
    commit = "<COMMIT-SHA>",
    remote = "https://github.com/buildfarm/buildfarm.git",
)

# Transitive!
# TODO: remove this after https://github.com/bazelbuild/remote-apis/pull/293 is merged
bazel_dep(name = "remoteapis", version = "eb433accc6a666b782ea4b787eb598e5c3d27c93")
archive_override(
    module_name = "remoteapis",
    integrity = "sha256-68wzxNAkPZ49/zFwPYQ5z9MYbgxoeIEazKJ24+4YqIQ=",
    strip_prefix = "remote-apis-eb433accc6a666b782ea4b787eb598e5c3d27c93",
    urls = [
        "https://github.com/bazelbuild/remote-apis/archive/eb433accc6a666b782ea4b787eb598e5c3d27c93.zip",
    ],
)

bazel_dep(name = "googleapis", version = "0.0.0-20240326-1c8d509c5", repo_name = "com_google_googleapis")
bazel_dep(name = "grpc-java", version = "1.62.2")

googleapis_switched_rules = use_extension("@googleapis//:extensions.bzl", "switched_rules")
googleapis_switched_rules.use_languages(
    grpc = True,
    java = True,
)
use_repo(googleapis_switched_rules, "com_google_googleapis_imports")

```

You can then use the existing layer targets to build your own OCI images, for example:

``` starlark
oci_image(
    name = "mycompany-buildfarm-server",
    base = "@<YOUPROVIDE>",
    entrypoint = [
        "/usr/bin/java",
        "-jar",
        "/app/build_buildfarm/buildfarm-server_deploy.jar",
    ],
    tars = [
        "@build_buildfarm//:layer_buildfarm_server",
    ],
)
```

where `@<YOUPROVIDE>` is the name of an `oci.pull(name = 'YOUPROVIDE', ...)` in your `MODULE.bazel`

### Helm Chart

To install OCI bundled Helm chart:

```bash
helm install \
  -n bazel-buildfarm \
  --create-namespace \
  bazel-buildfarm \
  oci://ghcr.io/buildfarm/buildfarm \
  --version "0.4.1"
```

## Configuration

<!-- Document configuration options here. This section is yours — the AI will not modify it. -->

## CI

<!-- AI:start:ci -->
_CI documentation pending._
<!-- AI:end:ci -->

## Mirror chain

<!-- AI:start:mirror-chain -->
This repo is maintained in [`Interested-Deving-1896/buildfarm`](https://github.com/Interested-Deving-1896/buildfarm) and mirrored through:

```
Interested-Deving-1896/buildfarm  ──►  OpenOS-Project-OSP/buildfarm  ──►  OpenOS-Project-Ecosystem-OOC/buildfarm
```

Changes flow downstream automatically via the hourly mirror chain in
[`fork-sync-all`](https://github.com/Interested-Deving-1896/fork-sync-all).
Direct commits to OSP or OOC are detected and opened as PRs back to `Interested-Deving-1896`.
<!-- AI:end:mirror-chain -->

## Contributors

<!-- AI:start:contributors -->
_Contributors pending._
<!-- AI:end:contributors -->

## Origins

<!-- AI:start:origins -->
_Original project — no upstream fork._
<!-- AI:end:origins -->

## Resources

<!-- AI:start:resources -->
_No additional resource files found._
<!-- AI:end:resources -->

## License

<!-- AI:start:license -->
[Apache-2.0](https://github.com/Interested-Deving-1896/buildfarm/blob/main/LICENSE) © 2026 [Interested-Deving-1896](https://github.com/Interested-Deving-1896)
<!-- AI:end:license -->
