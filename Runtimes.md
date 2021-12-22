# JRE system overhaul

The idea is to do a complete rework of Java runtime management.

The game now requires different versions of the JRE and our current approach no longer works.

The complete shape of the system is undecided at this point.

Format decisions should aim for future extensibility - don't close doors that don't need to be closed forever.

## Runtime references
Runtime references would be used in version metadata files.
Along those, a hierarchy of options / variable defaults

Currently, vanilla hardcodes these for `x86`:
```
-Xmx800M -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M
```

And these for `amd64`:
```
-Xmx2G -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=32M
```

### Runtime reference in version metadata

This is how a Java runtime could be referenced in each Minecraft version:
```json
{
    "runtime": {
        "minVersion": 7,
        "variables": {
            "all": {
                "x86": {
                    "maxHeap": 800,
                    "heapRegionSize": 16
                },
                "amd64": {
                    "maxHeap": 2048,
                    "heapRegionSize": 32,
                    "minStack": 1
                }
            }
        }
    }
}
```

This is how a Java runtime could be referenced in each Minecraft version, with slightly different defaults:
```json
{
    "runtime": {
        "minVersion": 17,
        "variables": {
            "windows": {
                "x86": {
                    "minHeap": 512,
                    "maxHeap": 1450
                }
            },
            "all": {
                "x86": {
                    "minHeap": 512,
                    "maxHeap": 2048
                },
                "amd64": {
                    "minHeap": 512,
                    "maxHeap": 2048,
                    "minStack": 1
                }
            }
        }
    }
}
```

`minVersion` and `maxVersion` together define the range of Java versions a particular component can accept:

- `"minVersion": 17, "maxVersion": 17` means only version 17.
- `"minVersion": 16, "maxVersion": 17` means either 16 or 17. Highest will be preferred when resolving versions.
- `"minVersion": 7` means anything above and including version 7. Highest will be preferred.
- `"maxVersion": 8` means anything below and including version 8. Highest will be preferred.

If there are multiple components being composed together, range intersection operation is used to resolve the final set of eligible Java versions.

**Hypothetical example:**

- Minecraft X has `"minVersion": 6`
- Minecraft Forge for X can only work with `"minVersion": 7, "maxVersion": 8`
- When the ranges are composed together, the eligible versions are 7 and 8.
- MultiMC will try to use these, in order:
    - Mojang Java 8, as long as there is one for this platform/architecture)
    - System Java 8, if present
    - System Java 7, if present
    - FAIL to resolve and show an error

### Runtime reference in a modpack

In a modpack (or instance), we could say there is an optimal heap size and minimal heap size:
```json
{
    "runtime": {
        "minHeap": "2048M",
        "optimalHeap": "4096M"
    }
}
```

With this, we could:

- Warn the user if they don't have enough memory for optimal use
- Warn them harder if they have below minimum available memory
- Determine the actual heap size dynamically based on the runtime, hardware, architecture and the instance/modpack variables.

Actually specifying a Java version is probably not needed, but could be a thing.

## Runtimes

Every major Java version has some properties.

Some versions can be mapped to a Mojang runtime, which is ready to use and available.

Every runtime has a major version - even locally installed ones - and it can be probed.

We would have a simple runtime registry that ties major versions of Java to piston coordinates and various other properties.

The piston components can be discovered from:
https://launchermeta.mojang.com/v1/products/java-runtime/2ec0cc96c44e5a76b9c8b7c39df7210883d12871/all.json

This can be internally hardcoded, and only the name of the component needs to be in the MultiMC metadata.

On launch, we would load the piston index and runtime metadata files from local cache and then attempt to fetch updated ones (with usual online/offline/missing states).

### Example of a possible metadata file describing all runtimes:
```json
[
    {
        "majorVersion": 7,
        "managedArgs": {
            "minHeap": "-Xms${value}m",
            "maxHeap": "-Xmx${value}m",
            "minStack": "-Xss${value}m",
            "permSize": "-XX:PermSize=${value}m",
            "startOnFirstThread": "-XstartOnFirstThread"
        },
        "hardArgs": [
            "-XX:+UseG1GC",
            "-XX:G1NewSizePercent=20",
            "-XX:G1ReservePercent=20",
            "-XX:MaxGCPauseMillis=50",
            "-XX:G1HeapRegionSize=${heapRegionSize}m",
        ]
    },
    {
        "majorVersion": 8,
        "pistonComponent": "jre-legacy",
        "managedArgs": {
            "minHeap": "-Xms${value}M",
            "maxHeap": "-Xmx${value}M",
            "minStack": "-Xss${value}M",
            "startOnFirstThread": "-XstartOnFirstThread"
        },
        "hardArgs": [
            "-XX:+UseG1GC",
            "-XX:G1NewSizePercent=20",
            "-XX:G1ReservePercent=20",
            "-XX:MaxGCPauseMillis=50",
            "-XX:G1HeapRegionSize=${heapRegionSize}m",
        ]
    },
    {
        "majorVersion": 16,
        "pistonComponent": "java-runtime-alpha",
        "managedArgs": {
            "minHeap": "-Xms${value}m",
            "maxHeap": "-Xmx${value}m",
            "minStack": "-Xss${value}m",
            "startOnFirstThread": "-XstartOnFirstThread"
        },
        "hardArgs": [
            "-XX:+UseZGC"
        ]
    },
    {
        "majorVersion": 17,
        "pistonComponent": "java-runtime-beta",
        "managedArgs": {
            "minHeap": "-Xms${value}m",
            "maxHeap": "-Xmx${value}m",
            "minStack": "-Xss${value}m",
            "startOnFirstThread": "-XstartOnFirstThread"
        },
        "hardArgs": [
            "-XX:+UseZGC"
        ]
    }
]
```

Anything below lowest major version (currently Java 7) is not supported by MultiMC.
Args from a major version propagate to any higher version that lacks a definition in the metadata, with the exception of `pistonComponent`.

So, for example, when JRE overrides are enabled:
- User selects system Java 7.
    - Args from `7` are used.

- User selects system Java 11.
    - Args from `8` are used.

- User selects system Java 18.
    - Args from `17` are used.

Java older than 7 will not be supported (Java code in MultiMC is built for Java 7 at the minimum). When detected, we will simply present an error.

`managedArgs` are managed by MultiMC and not directly visible to the user. They depend on the left hand side (variable by name) being set in order to be passed to Java. If there is no defined value of `minHeap`, `-Xms${value}m` is not passed to the JRE.

`startOnFirstThread` is then only specified on macOS, and replaces the current trait-based logic.

`hardArgs` are hardcoded, but do show up in the UI as 'default'/'ghost' and can be customized. Variables can be substituted into hardcoded args (like `heapRegionSize` for G1GC).

How the values for the variables are determined is internal to MultiMC and tuned to work well on all machines over time.

## Java settings in MultiMC

Existing Java settings in MultiMC have two layers -- global and per-instance.

The settings poorly accomodate resolving which Java to use, and/or determining the heap size and other java args dynamically.

For best user experience, the JRE settings should never have to be touched.

### Global settings

These affect every instance, unless overriden, and are present in `multimc.cfg`:
```
JavaArchitecture=64
JavaPath=java
JavaTimestamp=1635003862000
JavaVersion=17.0.1
JvmArgs=-verbose:gc
LastHostname=peterix
MaxMemAlloc=8192
MinMemAlloc=4096
PermGen=128
```

This allows only selecting one version of Java tied to one specificy JRE binary.

**Caching:**
When going between machines, or when the selected Java runtime disappears/changes, MultiMC asks you which Java to use again.

These variables stored in global settings are used to support that functionality.

- `JavaArchitecture` records the selected JRE's CPU architecture.
- `JavaTimestamp` records the last changed timestamp of the selected JRE's binary.
- `JavaVersion` records the selected JRE's version.
- `LastHostname` is used as a factor to pop up the JRE selection wizard.

**Actual options:**
- `JavaPath` is an absolute or relative path to the JRE binary. MultiMC can use a JRE stored in its data folder via the relative path, making it slightly more portable.
- `JvmArgs` are extra args added to the JRE args, by default empty.
- `MinMemAlloc` directly translates to minimum heap size (`-Xms${value}m` argument).
- `MinMemAlloc` directly translates to maximum heap size (`-Xmx${value}m` argument).
- `PermGen` directly translates to Java 7 and older 'Permanent Generation' size (`-Xmx${value}m` argument).

    The default for PermGen in MultiMC is 128m.

    ```
    if (permgen != 64)
    {
        args << QString("-XX:PermSize=%1m").arg(permgen);
    }
    ```

### Per-Instance settings

Per-instance settings mirror the global settings, but need to be toggled on with override settings.

With `OverrideLocation`, the instance uses its own Java runtime:
```
OverrideJavaLocation=true
JavaPath=/usr/lib/jvm/java-17-openjdk/bin/java
JavaArchitecture=64
JavaTimestamp=1635003862000
JavaVersion=17.0.1
```

With `OverrideMemory`, the instance will start with different memory amounts than specified in global settings:
```
OverrideMemory=true
MaxMemAlloc=8192
MinMemAlloc=4096
PermGen=128
```

With `OverrideJavaArgs`, the instance has its own extra runtime arguments (they are **not combined** with global):
```
OverrideJavaArgs=true
JvmArgs=-verbose:gc
```

There is also a legacy setting that overrides both location and args, but not memory.

This:
```
OverrideJava=true
JavaPath=/usr/lib/jvm/java-17-openjdk/bin/java
JavaArchitecture=64
JavaTimestamp=1635003862000
JavaVersion=17.0.1
JvmArgs=-verbose:gc
```

Is equivalent to this:
```
OverrideJavaArgs=true
OverrideJavaLocation=true
JavaPath=/usr/lib/jvm/java-17-openjdk/bin/java
JavaArchitecture=64
JavaTimestamp=1635003862000
JavaVersion=17.0.1
JvmArgs=-verbose:gc
```

The `OverrideJava` turns into `OverrideJavaArgs` and `OverrideJavaLocation` on instance settings save.
