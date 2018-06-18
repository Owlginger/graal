## JVM Operations Manual

### Difference between running Graal in the native image vs on the JVM

When running Graal on the JVM, it goes through the same warmup phase that the
rest of the Java application does. That is, it is first interpreted before
its hot methods are compiled. This can translate into slightly longer times
until the application reaches peak performance when compared to the native compilers
in the JVM such as C1 and C2.

Currently, this is the only way Graal can be used with the JVM. We are
developing a mode whereby Graal itself is compiled ahead of time into a native shared
library using SubstrateVM. This will provide startup times on par with the
native compilers. It will also remove any impact Graal's use of the Java heap
may have on the application as Graal will have its own heap and memory manager.

## Measuring Performance

The first thing to be sure of when measuring the performance of Graal is to ensure
the JVM is using Graal. In a GraalVM binary, the JVM is configured to use Graal
as the top tier compiler by default. You can confirm this by adding `-Dgraal.ShowConfiguration=info`
to the command line. It will produce a line of output similar to the one below
when Graal is initialized:
```
Using Graal compiler configuration 'community' provided by org.graalvm.compiler.hotspot.CommunityCompilerConfigurationFactory loaded from jar:file:/Users/dsimon/graal/graal/compiler/mxbuild/dists/graal.jar!/org/graalvm/compiler/hotspot/CommunityCompilerConfigurationFactory.class
```
Note that Graal is only initialized on the first top tier JIT compilation request
so if your application is short lived, you may not see this output.

Optimizing JVM-based applications is a science in itself. Compilation may not
even be a factor in the case of poor performance as the problem may
lie in any other part of the VM (I/O, garbage collection, threading etc) or in
poorly written application or 3rd party library code. For this reason, it's
worth using profilers such as [Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html) to
diagnose application behavior.

You can also compare performance against the native top tier compiler in the JVM by
adding `-XX:-UseJVMCICompiler` to the command line.

If you observe a significant performance regression when using Graal, please
open an issue on GitHub. Attaching a Java Flight Recorder log and instructions
to reproduce the issue makes investigation easier and thus
chances of a fix higher. Even better is if you can submit a [JMH](http://openjdk.java.net/projects/code-tools/jmh/)
benchmark that represents the hottest parts of your application (as identified
by a profiler). This allows us to very quickly pinpoint missing optimization
opportunities or to offer suggestions on how to restructure the code to
avoid or reduce performance bottlenecks.

## Troubleshooting Graal

Like all software, Graal is not guaranteed to be bug free so it is useful to
know how to diagnose and submit useful bug reports if you encounter issues.

### Compilation Exceptions

One advantage of Graal being written in Java is that runtime exceptions during
compilation are not fatal VM errors. Instead, each compilation has an exception
handler that takes an action based on the `graal.CompilationFailureAction`
property.

The default value is `Diagnose` and causes failing compilations to be retried
with extra diagnostics enabled. Just before the VM exits, all diagnostic output
captured during retried compilations is written to a .zip file and output such
as the following is printed to the console:
```
Graal diagnostic output saved in /Users/demo/dumps/1499768882600/graal_diagnostics_64565.zip
```

You can then attach the .zip file to an issue on [GitHub](https://github.com/{{ site.github_username }}).

You can also set the following values for the `graal.CompilationFailureAction`
property:
* `Silent`: prints nothing to the console.
* `Print`: prints a message to the console but does not do re-compilation.
* `ExitVM`: does the same thing as `Diagnose` but the VM process exits after
re-compilation.

### Code Generation Errors

The other type of error compilers can have is producing incorrect machine code.
This error can cause a VM crash, which should produce a file that starts with
`hs_err_pid` in the current working directory of the VM process. In most cases,
there is a section in the file that shows the stack at the time of the crash,
including the type of code for each frame in the stack, as in the following
example:

```
Stack: [0x00007000020b1000,0x00007000021b1000],  sp=0x00007000021af7a0,  free space=1017k
Native frames: (J=compiled Java code, j=interpreted, Vv=VM code, C=native code)
J 761 JVMCI org.graalvm.compiler.core.gen.NodeLIRBuilder.matchComplexExpressions(Ljava/util/List;)V (299 bytes) @ 0x0000000108a2fc01 [0x0000000108a2fac0+0x141] (null)
j  org.graalvm.compiler.core.gen.NodeLIRBuilder.doBlock(Lorg/graalvm/compiler/nodes/cfg/Block;Lorg/graalvm/compiler/nodes/StructuredGraph;Lorg/graalvm/compiler/core/common/cfg/BlockMap;)V+211
j  org.graalvm.compiler.core.LIRGenerationPhase.emitBlock(Lorg/graalvm/compiler/nodes/spi/NodeLIRBuilderTool;Lorg/graalvm/compiler/lir/gen/LIRGenerationResult;Lorg/graalvm/compiler/nodes/cfg/Block;Lorg/graalvm/compiler/nodes/StructuredGraph;Lorg/graalvm/compiler/core/common/cfg/BlockMap;)V+65
```

This example shows that the top frame was compiled (J) by the JVMCI compiler,
which is the Graal compiler. The crash occurred at offset 0x141 in the machine
code produced for:
```
org.graalvm.compiler.core.gen.NodeLIRBuilder.matchComplexExpressions(Ljava/util/List;)V
```

The next two frames in the stack were executing in the interpreter (j). The
location of the crash is also often indicated near the top of the file with
something like this:
```
# Problematic frame:
# J 761 JVMCI org.graalvm.compiler.core.gen.NodeLIRBuilder.matchComplexExpressions(Ljava/util/List;)V (299 bytes) @ 0x0000000108a2fc01 [0x0000000108a2fac0+0x141] (null)
```

In this example, there is likely an error in the code produced by Graal for `NodeLIRBuilder.matchComplexExpressions`.

When filing an issue on [GitHub](https://github.com/{{ site.github_username }})
for such a crash, you should first attempt to reproduce the crash with extra
diagnostics enabled for the compilation of the problematic method.
In this example, you would add the following to your command line:
```
-Dgraal.MethodFilter=NodeLIRBuilder.matchComplexExpressions, -Dgraal.Dump=:2
```

These options are described in more detail [here](https://github.com/oracle/graal/blob/master/compiler/docs/Debugging.md).
In brief, these options tell Graal to capture snapshots of the compiler state at
verbosity level 2 while compiling any method named `matchComplexExpressions` in
a class with a simple name of `NodeLIRBuilder`. The complete format of the
`MethodFilter` option is described in the output of `java -XX:+JVMCIPrintProperties`.

Quite often, the crash location does not exist directly in the problematic method
mentioned in the crash log but comes from an inlined method.

In such a case, simply filtering for the problematic method might not capture an
erroneous compilation causing a crash.

To improve the likelihood of capturing an erroneous compilation, you need to
broaden the `MethodFilter` value. To guide this, add `-Dgraal.PrintCompilation=true`
when trying to reproduce the crash so you can see what was compiled just before
the crash.

The following shows sample output from the console:
```
HotSpotCompilation-1218        Lorg/graalvm/compiler/core/amd64/AMD64NodeLIRBuilder;                  peephole                                      (Lorg/graalvm/compiler/nodes/ValueNode;)Z           |   87ms   428B   447B  1834kB
HotSpotCompilation-1212        Lorg/graalvm/compiler/lir/LIRInstructionClass;                         forEachState                                  (Lorg/graalvm/compiler/lir/LIRInstruction;Lorg/graalvm/compiler/lir/InstructionValueProcedure;)V  |  359ms    92B   309B  6609kB
HotSpotCompilation-1221        Lorg/graalvm/compiler/hotspot/amd64/AMD64HotSpotLIRGenerator;          getResult                                     ()Lorg/graalvm/compiler/hotspot/HotSpotLIRGenerationResult;  |   54ms    18B   142B  1025kB
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x000000010a6cafb1, pid=89745, tid=0x0000000000004b03
#
# JRE version: OpenJDK Runtime Environment (8.0_121-b13) (build 1.8.0_121-graalvm-olabs-b13)
# Java VM: OpenJDK 64-Bit GraalVM (25.71-b01-internal-jvmci-0.30 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# J 1221 JVMCI org.graalvm.compiler.hotspot.amd64.AMD64HotSpotLIRGenerator.getResult()Lorg/graalvm/compiler/hotspot/HotSpotLIRGenerationResult; (18 bytes) @ 0x000000010a6cafb1 [0x000000010a6caf60+0x51] (null)
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
```
Here we see that the crash happened in a different method than the first crash.
As such, we expand the filter argument to be `-Dgraal.MethodFilter=
NodeLIRBuilder.matchComplexExpressions,AMD64HotSpotLIRGenerator.getResult`
and run again.

When the VM crashes in this way, it does not execute the shutdown code that
archives the Graal diagnostic output or delete the directory it was written to.
This must be done manually after the crash.

By default, the directory is `$PWD/dumps/<timestamp>`; for example, `./dumps/1499938817387`.
However, you can set the directory with `-Dgraal.DumpPath=<path>`.

A message, such as the following, is printed to the console when this
directory is first used by the Graal compiler:
```
Dumping debug output in /Users/demo/dumps/1499768882600
```

This directory should contain content related to the crashing method, such as:
```
$ ls -l /Users/demo/dumps/1499768882600
-rw-r--r--  1 demo  staff    144384 Jul 13 11:46 HotSpotCompilation-1162[AMD64HotSpotLIRGenerator.getResult()].bgv
-rw-r--r--  1 demo  staff     96925 Jul 13 11:46 HotSpotCompilation-1162[AMD64HotSpotLIRGenerator.getResult()].cfg
-rw-r--r--  1 demo  staff  12600725 Jul 13 11:46 HotSpotCompilation-791[NodeLIRBuilder.matchComplexExpressions(List)].bgv
-rw-r--r--  1 demo  staff   1727409 Jul 13 11:46 HotSpotCompilation-791[NodeLIRBuilder.matchComplexExpressions(List)].cfg
```
You should attach a zip of this directory to an issue on [GitHub](https://github.com/{{ site.github_username }}).