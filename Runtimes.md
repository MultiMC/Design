# JRE system overhaul

The idea is to do a complete rework of Java runtime management.

The game now requires different versions of the JRE and our current approach no longer works.

The complete shape of the system is undecided at this point.

Format decisions should aim for future extensibility - don't close doors that don't need to be closed forever.

## Runtime references
Runtime references are used in version metadata files
Along those, a hierarchy of options / variable defaults

Currently, vanilla hardcodes these for `x86`:
```
-Xmx2G -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=32M
```

And these for `amd64`:
```
-Xmx800M -XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M
```

This is how a Java runtime could be referenced in a Minecraft version:
```json
{
    "runtime": {
        "majorVersion": 17,
        "mojangReference": "java-runtime-beta",
        "variables": {
            "x86": {
                "maxHeap": "800M",
                "minStack": "1M"
            },
            "amd64": {
                "maxHeap": "2048M"
            }
        }
    }
}
```

This is how a Java runtime could be referenced in a Minecraft version, with slightly different defaults:
```json
{
    "runtime": {
        "majorVersion": 17,
        "mojangReference": "java-runtime-beta",
        "variables": {
            "windows": {
                "x86": {
                    "minHeap": "512M",
                    "maxHeap": "1450M",
                }
            },
            "all": {
                "x86": {
                    "minHeap": "512M",
                    "maxHeap": "2048M",
                    "minStack": "1M"
                },
                "amd64": {
                    "minHeap": "512M",
                    "maxHeap": "2048M",
                    "minStack": "1M"
                }
            }
        }
    }
}
```

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

## Runtimes

- Every runtime has a major version.
- Runtimes can come from Mojang
    This is what we want to use by default.
- Runtimes can be local
    This is what we had up to now.

## Major versions

The major version describes the basic properties of the runtime:
- Feature set
    - Is it valuable to model these in detail?
- Possible garbage collectors?
    - Is it valuable to model these in detail?
- Basic (not vendor specific) set of arguments
- Mapping of MultiMC traits and/or variables to arguments

## Garbage collectors and runtime parameters

This is a deep topic. We may simply want to have some default GC options for every major version of Java and let people override them.

Alternative is to map out all of these options and pick based on:
- Java version
- Available number of cores
- Size of heap
- etc.

Mojang practically uses Java 8, 16 and 17. Others do not have to be mapped out completely.

We can use the Java 8 options for all major versions up to 16.

We can use the Java 17 options for all versions above 17, until specified otherwise.

### Java 8

https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/collectors.html

#### Unusable GCs

- **SerialGC** - `-XX:+UseSerialGC` - Serial collection of up to 100MB heap space. Useless for Minecraft.
- **ParallelGC** - `-XX:+UseParallelGC` - high latency parallel collector. Useless for Minecraft.
- **ParallalGC with parallel compaction disabled** - `-XX:+UseParallelGC -XX:-UseParallelOldGC` - Why? It's just slower...

#### Potentially usable GCs

https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/concurrent.html#mostly_concurrent

- **UseConcMarkSweepGC** - `-XX:+UseConcMarkSweepGC` - parallel, shares processing time with GC.
   - Incremental Mode (deprecated):
     This is useful on machines with one core, if we want to guarantree low latency.
     https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/cms.html#CJAGIIEJ
- **G1GC** - `-XX:+UseG1GC` - parallel, GC cycles run next to the actual computation.

#### Conclusion

By the looks of it, for Minecraft, only G1GC makes sense.

We could just say that `-XX:+UnlockExperimentalVMOptions -XX:+UseG1GC -XX:G1NewSizePercent=20 -XX:G1ReservePercent=20 -XX:MaxGCPauseMillis=50 -XX:G1HeapRegionSize=16M` are the default args here, and combine that with stack, permgen and heap sizes.

The argument for modeling this would be running on VERY old hardware - think single core system with 2GB of memory. Then **ConcMarkSweepGC** with **Incremental Mode** could make sense.

Knowing we run with such constraints and selecting different GC based on it would be helpful to users of such systems.

### Java 16 and 17

https://docs.oracle.com/en/java/javase/16/gctuning/available-collectors.html

This adds the the Z Garbage Collector, which is fully parallel and aims for low latency (a few ms, still a lot for games, but better than the alternatives).

#### Conclusion

Practically, ZGC should be used: `-XX:+UseZGC`.

See: https://docs.oracle.com/en/java/javase/16/gctuning/z-garbage-collector.html

Interesting parameters:
- Number of GC threads - `-XX:ConcGCThreads` - possibly the default heuristics would be OK.
- Returning memory to the system is enabled by default, and can be disabled with `-XX:-ZUncommit`.
