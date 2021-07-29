---
title: "Developing OpenJ9 JIT for RISC-V"
created_at: 2021-07-29 08:37:29 +0000
kind: article
published: true
---

This post describes (my) workflow when working on RISC-V JIT for OpenJ9. I have wrote
it mainly as documentation for myself but I may be usefull for anyone who wants work
on RISC-V JIT in OpenJ9. Especially after I had to put it - hopefully temporarily - 
on hold.

<!-- more -->

## Prerequsities


## (Cross) compiling

*TL;DR:* use [openj9-openjdk-jdk11-devscripts][1] to cross-compile:

    git clone -b jv/riscv-devel https://github.com/janvrany/openj9-openjdk-jdk11-devscripts
    cd openj9-openjdk-jdk11-devscripts
    ./prepare.sh
    ./configure.sh --target riscv64-linux-gnu
    ./compile.sh --target riscv64-linux-gnu

---

Since not everything is upstreamed at the time of writing, you'll need need my RISC-V development branch of OpenJDK 11, OMR and OpenJ9:

  * [OpenJDK 11](https://github.com/janvrany/openj9-openjdk-jdk11/tree/jv/riscv-devel)

        git checkout -b jv/riscv-devel https://github.com/janvrany/openj9-openjdk-jdk11 openj9-openjdk-jdk11

  * [OpenJ9](https://github.com/janvrany/openj9/tree/jv/riscv-devel)

        git checkout -b jv/riscv-devel https://github.com/janvrany/openj9 openj9-openjdk-jdk11/openj9

  * [OMR](https://github.com/janvrany/omr/tree/jv/riscv-devel)

        git checkout -b jv/riscv-devel https://github.com/janvrany/omr openj9-openjdk-jdk11/omr

Use CMake (`--with-cmake`) to compile OpenJ9 (as of now, there's no support for RISC-V JIT component when using UMA).
See [configure.sh](https://github.com/janvrany/openj9-openjdk-jdk11-devscripts/blob/9a1f317e0cdb4763b7f35c0afdb758b722989919/configure.sh#L25-L35) 
for details on how to configure OpenJDK. 

## Testing 

To test just (cross) compiled OpenJ9, we need to copy the JDK image RISC-V machine - either real hardware or
virtual machine running under QEMU. I use `rsync` (`unleashed` is name of my testing RISC-V machine). :

    rsync ./openj9-openjdk-jdk11/build/linux-riscv64-normal-server-slowdebug/images/jdk unleashed:/tmp

Now, connect to testing machine (`unleashed` in my case) and continue there. First, create (or edit) a
test code, for example `Test.java`:

    public class Test {
        public static int jitMeaningOfWorld() {
            return 42;
        }

        public static void main(String[] args) {
            System.out.println("Meaning of world is " + jitMeaningOfWorld());
        }
    }

...and compile it:

    javac -cp . Test.java

And finally, run OpenJ9 to see whether it works (or not):

    /tmp/jdk/bin/java "-Xjit:verbose,count=0,limit={Test.jit*}" -cp . Test

The `-Xjit` option is important:

 * `verbose` tells the VM to print compilation log so you can see (most importantly) which method has been compiled and so on,
 * `count=0` tells the VM to compile on first execution (normally the JVM compiles only methods that are executed 'often')
 * `limit={Test.jit*}` is very important in this context, it tells the JVM to ever attempt to compile *only* methods whose signature matches the pattern.

So, when run, you should see something like:

    #INFO:  _______________________________________
    #INFO:  Version Information:
    #INFO:       JIT Level  - j9jit_20210112_0658_jv
    #INFO:       JVM Level  - 20210111_000000
    #INFO:       GC Level   - 3bf6fa788
    #INFO:
    #INFO:  _______________________________________
    #INFO:  AOT
    #INFO:  options specified:
    #INFO:       verbose,count=0,limit={Test.jit*}
    #INFO:
    #INFO:  options in effect:
    #INFO:       verbose=1

    #INFO:       compressedRefs shiftAmount=0
    #INFO:       compressedRefs isLowMemHeap=1
    #INFO:  StartTime: Jan 12 10:40:12 2021
    #INFO:  Free Physical Memory: 8215 MB
    #INFO:  CPU entitlement = 400.00
    + (cold) Test.jitMeaningOfWorld()I @ 0000004016DF6024-0000004016DF6044 OrdinaryMethod - Q_SZ=2 Q_SZI=2 QW=6 j9m=0000000000166AB8 bcsz=3 sync compThreadID=0 CpuLoad=208%(52%avg) JvmCpu=103%
    Meaning of world is 42

Most importantly, this line:

    + (cold) Test.jitMeaningOfWorld()I @ 0000004016DF6024-0000004016DF6044 OrdinaryMethod - Q_SZ=2 Q_SZI=2 QW=6 j9m=0000000000166AB8 bcsz=3 sync compThreadID=0 CpuLoad=208%(52%avg) JvmCpu=103%


means that the JVM compiled method `Test.jitMeaningOfWorld()I`. And the one below:

    Meaning of world is 42

means that this compiled method has been executed and returned correct answer. *Yay!*

## Debugging

One can certainly use GDB on RISC-V machine to debug the JIT:

    /opt/gdb/bin/gdb -ex "r" --args /tmp/jdk/bin/java "-Xjit:breakOnEntry,verbose,count=0,limit={Test.jit*}" -cp . Test

**However**, here I'm going to use `gdbserver` and debug from local development machine connecting to remote RISC-V machine. This way, I have the convenience of local development machine (so I can use my favourite GDB frontend). On the other hand, the setup is more complex.

1. On development machine, start `gdb` and load `java` binary into it:

        (gdb) file openj9-openjdk-jdk11/build/linux-riscv64-normal-server-slowdebug/images/jdk/bin/java
        Reading symbols from openj9-openjdk-jdk11/build/linux-riscv64-normal-server-slowdebug/images/jdk/bin/java
        ...
        Reading symbols from openj9-openjdk-jdk11/build/linux-riscv64-normal-server-slowdebug/images/jdk/bin/java.debuginfo
        ...
        (gdb)

2. Optional. While recent GDB can download debug info from remote host automagically, this is not always viable. For example, I connect to RISC-V host from my office over slow VPN so download takes ages.

   To make GDB to load debug symbols from local filesystem, we need tell GDB where's the "sysroot": where it an find (on a local filesystem) the filesystem of the remote target (the on of remote RISC-V machine). It does not have to be *excactly* the same, but similar enough. Luckily, we have RISC-V "sysroot" on a local machine - we needed it for cross compilation.

   The only issue is that the sysroot used for compilation does not have built jdk in `/tmp` (or wherever one
   uploads the built JDK, see section *Testing* above). The easiest way to create symlink:

        bash $ (cd /opt/riscv/sysroot/tmp && ln -s /.../openj9-openjdk-jdk11/build/linux-riscv64-normal-server-slowdebug/images/jdk .)

   And then in GDB just do:

        (gdb) set sysroot /opt/riscv/sysroot
        (gdb)

3. Finally, run a `gdbserver` on RISC-V machine and connect to it. In GDB, do:

        (gdb) target extended-remote | ssh unleashed /opt/gdb/bin/gdbserver - /tmp/jdk/bin/java '-Xjit:breakOnEntry,verbose,count=0,limit={RISCVJitTest.jit*}' -cp . Test

Happy debugging!

[1]: https://github.com/janvrany/openj9-openjdk-jdk11-devscripts