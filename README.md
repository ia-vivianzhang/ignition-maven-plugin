# Ignition Maven Plugin

This is the maven plugin used in the Ignition SDK examples to build Ignition Modules.

# Build and Install

Check out the repo, and execute `mvn clean install`

# Goals

## `ignition:modl`

Builds the `.modl` file from the configured module structure. This is the primary goal, bound to the `package` phase by default.

## `ignition:post`

POSTs a built `.modl` file to a running Ignition Gateway for hot deployment during development.

## `ignition:write-dev-descriptor`

Generates a JSON dev module descriptor for IDE classloader isolation. Bound to the `compile` phase by default.

The descriptor enables a dev Ignition gateway to load your module directly from Maven build outputs (`target/classes` + resolved dependency JARs) instead of requiring a full `.modl` build. This provides:

- **Classloader isolation** matching production behavior (each module gets its own `ModuleClassLoader`)
- **HotSwap support** for method-body changes via JDWP
- **No .modl build required** — no compile+zip+sign cycle for code changes

> For Gradle-based modules, the [`ignition-module-tools`](https://github.com/inductiveautomation/ignition-module-tools) Gradle plugin provides the equivalent `writeDevModuleDescriptor` task, which emits the same descriptor format.

### Usage

```bash
mvn ignition:write-dev-descriptor
```

This produces `target/dev/{moduleId}.json`. Copy it to your gateway's `user-lib/modules/dev/` directory:

```bash
mkdir -p /path/to/ignition/user-lib/modules/dev
cp target/dev/*.json /path/to/ignition/user-lib/modules/dev/
```

### Gateway Configuration

Your dev gateway needs these JVM flags in `ignition.conf` (or wrapper config):

```
# Required: allow unsigned modules
-Dignition.allowunsignedmodules=true

# Required: point gateway at dev descriptor directory
-Dignition.dev.moduleDir=user-lib/modules/dev

# Recommended: enable remote debugging for breakpoints + HotSwap
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
```

> **Note:** These flags only activate on **dev builds** of Ignition. The dev module loading path is completely inert on production gateways.

### Development Workflow

1. `mvn compile` then `mvn ignition:write-dev-descriptor` — generate and copy descriptor (first time, or after dependency changes)
2. Start/restart the gateway
3. In IntelliJ: **Run → Attach to Process** (or create a **Remote JVM Debug** config on port 5005)
4. Make code changes
5. **Build → Recompile** (Ctrl+Shift+F9 / Cmd+Shift+F9) — HotSwap applies method-body changes immediately
6. Test in the browser — changes are live

Repeat steps 4-6 without restarting. Only restart the gateway when:
- Module dependencies change (added/removed/version bumped)
- Module metadata changes (hooks, module dependencies)
- Structural class changes that HotSwap can't handle (new methods, new fields)

### Configuration

The goal reads the same `<configuration>` block as [`ignition:modl`](#ignitionmodl) — no additional configuration is needed. It uses `moduleId`, `moduleName`, `moduleVersion`, `requiredIgnitionVersion`, `projectScopes`, `hooks`, and `depends` to build the descriptor, and resolves compile-scope dependencies from each sub-project.

`<projectScopes>` maps each sub-project to the Ignition scope(s) it targets. The `<name>` is matched against the `<name>` element of the child module's POM (not its `<artifactId>`), and `<scope>` is one or more Ignition scope letters — `G` (gateway), `C` (client/vision), `D` (designer):

```xml
<projectScopes>
    <projectScope>
        <name>my-module-gateway</name>   <!-- matches <name> in the child pom -->
        <scope>G</scope>
    </projectScope>
    <projectScope>
        <name>my-module-designer</name>
        <scope>CD</scope>                <!-- may combine scopes -->
    </projectScope>
</projectScopes>
```

The `required` flag on a `<depend>` is only written to the descriptor (and to `module.xml`) when `requiredIgnitionVersion` is 8.3 or newer, since earlier gateways don't understand it. It defaults to `false`.

> **Portability:** the descriptor bakes in absolute filesystem paths (class directories and dependency JARs) specific to the machine that generated it. Regenerate it on each dev machine — **do not check it into source control.**

