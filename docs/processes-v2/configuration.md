---
layout: wmt/docs
title:  Configuration
side-navigation: wmt/docs-navigation.html
---

# {{ page.title }}

The `configuration` sections contains [dependencies](#dependencies),
[arguments](#arguments) and other process configuration values.

- [Merge Rules](#merge-rules)
- [Entry Point](#entry-point)
- [Arguments](#arguments)
- [Dependencies](#dependencies)
- [Requirements](#requirements)
- [Process Timeout](#process-timeout)
- [Exclusive Execution](#exclusive-execution)

## Merge Rules

Process `configuration` values can come from different sources: the section in
the `concord.yml` file, request parameters, policies, etc. Here's the order in
which all `configuration` sources are merged before the process starts:

- environment-specific [default values](../getting-started/configuration.html#default-process-variables);
- [defaultCfg](../getting-started/policies.html#default-process-configuration-rule) policy values;
- the current organization's configuration values;
- the current [project's configuration](../api/project.html#get-project-configuration) values;
- values from current active [profiles](./profiles.html)
- configuration file send in [the process start request](../api/process.html#start);
- [processCfg](../getting-started/policies.html#process-configuration-rule) policy values.

## Entry Point

The `entryPoint` configuration sets the name of the flow that will be used for
process executions. If no `entryPoint` is specified the flow labelled `default`
is used automatically, if it exists.

```yaml
configuration:
  entryPoint: "main" # use "main" instead of "default"

flows:
  main:
    - log: "Hello World"
```

**Note:** some flow names have special meaning, such as `onFailure`, `onCancel`
and `onTimeout`. See the [error handling](./flows.html#error-handling) section
for more details.

## Arguments

Default values for arguments can be defined in the `arguments` section of the
configuration as simple key/value pairs as well as nested values:

```yaml
configuration:
  arguments:
    name: "Example"
    coordinates:
      x: 10
      y: 5
      z: 0

flows:
  default:
    - log: "Project name: ${name}"
    - log: "Coordinates (x,y,z): ${coordinates.x}, ${coordinates.y}, ${coordinates.z}"
```

Values of `arguments` can contain [expressions](#expressions). Expressions can
use all regular tasks:

```yaml
configuration:
  arguments:
    listOfStuff: ${myServiceTask.retrieveListOfStuff()}
    myStaticVar: 123
```

Concord evaluates arguments in the order of definition. For example, it is
possible to use a variable value in another variable if the former is defined
earlier than the latter:

```yaml
configuration:
  arguments:
    name: "Concord"
    message: "Hello, ${name}"
```

A variable's value can be [defined or modified with the set step](#set-step)
and a [number of variables](./index.html#provided-variables) are automatically
set in each process and available for usage.

## Dependencies

The `dependencies` array allows users to specify the URLs of dependencies such
as:

- plugins ([tasks](../getting-started/tasks.html)) and their dependencies;
- dependencies needed for specific scripting language support;
- other dependencies required for process execution.

```yaml
configuration:
  dependencies:
  # maven URLs...
  - "mvn://org.codehaus.groovy:groovy-all:2.4.12"
  # or direct URLs
  - "https://repo1.maven.org/maven2/org/codehaus/groovy/groovy-all/2.4.12/groovy-all-2.4.12.jar"
  - "https://repo1.maven.org/maven2/org/apache/commons/commons-lang3/3.6/commons-lang3-3.6.jar"
```

Concord downloads the artifacts and adds them to the process' classpath.

Multiple versions of the same artifact are replaced with a single one,
following standard Maven resolution rules.

Usage of the `mvn:` URL pattern is preferred since it uses the centrally
configured [list of repositories](../getting-started/configuration.html#dependencies)
and downloads not only the specified dependency itself, but also any required
transitive dependencies. This makes the Concord project independent of access
to a specific repository URL, and hence more portable.

Maven URLs provide additional options:

- `transitive=true|false` - include all transitive dependencies
  (default `true`);
- `scope=compile|provided|system|runtime|test` - use the specific
  dependency scope (default `compile`).

Additional options can be added as "query parameters" parameters to
the dependency's URL:
```yaml
configuration:
  dependencies:
  - "mvn://com.walmartlabs.concord:concord-client:{{ site.concord_core_version }}?transitive=false"
```

The syntax for the Maven URL uses the groupId, artifactId, optionally packaging,
and version values - the GAV coordinates of a project. For example the Maven
`pom.xml` for the Groovy scripting language runtime has the following
definition:

```xml
<project>
  <groupId>org.codehaus.groovy</groupId>
  <artifactId>groovy-all</artifactId>
  <version>2.4.12</version>
  ...
</project>
```

This results in the path
`org/codehaus/groovy/groovy-all/2.4.12/groovy-all-2.4.12.jar` in the
Central Repository and any repository manager proxying the repository.

The `mvn` syntax uses the short form for GAV coordinates
`groupId:artifactId:version`, so for example
`org.codehaus.groovy:groovy-all:2.4.12` for Groovy.

Newer versions of groovy-all use `<packaging>pom</packaging>` and define
dependencies. To use a project that applies this approach, called Bill of
Material (BOM), as a dependency you need to specify the packaging in between
the artifactId and version. For example, version 2.5.2 has to be specified as
`org.codehaus.groovy:groovy-all:pom:2.5.2`:

```yaml
configuration:
  dependencies:
  - "mvn://org.codehaus.groovy:groovy-all:pom:2.5.2"
```

The same logic and syntax usage applies to all other dependencies including
Concord [plugins](../plugins/index.html).

## Requirements

A process can have a specific set of `requirements` configured. Concord uses
requirements to control where the process should be executed and what kind of
resources it gets. For example, if the process specifies

```yaml
configuration:
  requirements:
    agent:
      favorite: true
``` 

and if there is an agent with

```
concord-agent {
  capabilities = {
    favorite = true
  }
}
```

in its configuration file then it is a suitable agent for the process.

Following rules are used when matching `requirements.agent` values of processes
and agent `capabilities`:
- if the value is present in `capabilities` but missing in `requirements.agent`
is is **ignored**;
- if the value is missing in `capabilities` but present in `requirements.agent`
then it is **not a match**;
- string values in `requirements.agent` are treated as **regular expressions**,
i.e. in pseudo code `capabilities_value.regex_match(requirements_value)`;
- lists in `requirements.agent` are treated as "one or more" match, i.e. if one
or more elements in the list must match the value from `capabilities`;
- other values are compared directly.

More examples:

```yaml
configuration:
  requirements:
    agent:
      size: ".*xl"
      flavor:
        - "vanilla"
        - "chocolate"
```

matches agents with:

```
concord-agent {
  capabilities = {
    size = "xxl"
    flavor = "vanilla"
  }
}
```

### Process Timeout

You can specify the maximum amount of time the process can spend in
the `RUNNING` state with the `processTimeout` configuration. It can be useful
to set specific SLAs for deployment jobs or to use it as a global timeout:

```yaml
configuration:
  processTimeout: "PT1H"
flows:
  default:
    # a long running process
```

In the example above, if the process runs for more than 1 hour it is
automatically cancelled and marked as `TIMED_OUT`.

The parameter accepts duration in the
[ISO 8601](https://en.wikipedia.org/wiki/ISO_8601) format.

A special `onTimeout` flow can be used to handle such processes:

```yaml
flows:
  onTimeout:
  - log: "I'm going to run when my parent process times out"
```

The way Concord handles `processTimeout` is described in more details in
the [error handling](./flows.html#handling-cancellations-failures-and-timeout)
section.

**Note:** forms waiting for input and other processes in `SUSPENDED` state
are not affected by the process timeout. I.e. a `SUSPENDED` process can stay
`SUSPENDED` indefinitely -- up to the allowed data retention period.

## Exclusive Execution

The `exclusive` section in the process `configuration` can be used to configure
exclusive execution of the process:

```yaml
configuration:
  exclusive:
    group: "myGroup"
    mode: "cancel"

flows:
  default:
    - ${sleep.ms(60000)} # simulate a long-running task
```

In the example above, if another process in the same project with the same
`group` value is submitted, it will be immediately cancelled.

If `mode` set to `wait` then only one process in the same `group` is allowed to
run.

**Note:** this feature available only for processes running in a project.