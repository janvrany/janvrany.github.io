---
title: "Unmanaged delegates in CLR"
created_at: 2017-06-07 11:37:27 +0100
kind: article
published: true
---

As we progressed with [CLR interop for Bee Smalltalk][1], a need to create 
a delegate for an unmanaged code arose. By "a delegate for an unmanaged code" I mean
an instance of a [delegate][2] object that can be passed to CLR code which, when
called, would execute an unmanaged code. In our case a Smalltalk code. 

This can be handy, maybe necessary, if we want write a code that handles
events. And we may well would like to do so, writing some UI code using .NET
classes without use of event handlers seems to be impossible. Moreover, there
are other interesting events we may want to handle, such as 
[AppDomain.FirstChanceException][3].

Looking at the .NET documentation. this seem to be an easy task. Just use 
[Marshal.GetDelegateForFunctionPointer()][4] to create a delegate object for
an arbitrary function pointer. Looks easy but it turned out to be a little more
tricky. And not very well documented. The following text describes my findings
and my solution so far. Mainly for my (our) own record but someone else may find 
useful too.

<!-- more -->

*Disclaimer*: When I say "something cannot be done" or "something does not work",
I actually mean "No matter how long I tried, I could not find a way to do it.". 
If you know how to get around, please let me know. Pretty please!

## Creating a delegate (the easy bit)

In theory this is easy, all we need to do is: 

 * Create Smalltalk callback object: 
        
        callback := OsCallback new
            receiver: [ Transcript show: 'Hello, world!'; cr.]
            selector: #value:
            types: #()               " no parameters      "
            returnType: #long        " that is, int32_t   "
            callingConvention: #api; " that is, __stdcall "
            bind.
       
 * Get a hand on appropriate delegate type: 
        
        type := (appDomain Load: 'Some.Assembly') GetType: 'Some.Assembly.SomeDelegateType'.

 * And use `Marshal.GetDelegateForFunctionPointer()` to create a delegate: 
    
        delegate := System.Runtime.InteropServices.Marshal 
                        GetDelegateForFunctionPointer: callback asParameter
                                                    _: type
     

The problem is that we're using a COM interface to call .NET methods (such as
`GetDelegateForFunctionPointer()`) and the (unmanaged) function pointer argument 
is typed as `IntPtr`. Makes perfect sense, it's an pointer after all. Sadly, COM 
interface does not support marshaling of `IntPtr`. This is, unfortunately, a 
repeating pattern in Windows world. Sometimes it works, but not always and cases
when it does not work are not documented. 

The ugly way around this is to write a small wrapper in C# that is COM-callable:

    #!csharp
    public static Delegate GetDelegateForFunctionPointer32(Int32 ptr, Type t) 
    {
        return Marshal.GetDelegateForFunctionPointer((IntPtr)ptr, t);
    }

## Invocation of the delegate (the tricky bit)

So we managed to create a delegate object pass it to .NET code and at some point,
.NET will call the delegate. Again, in theory it's simple, CLR pushes arguments
on a stack as defined by calling convention and then issue a `call $callback_stub`.

The interesting question is in what form it pushes arguments to a stack. When calling 
normal .NET we use COM interface so all parameters are passed wrapped in a COM's 
[VARIANT][5] structure and return value is also a `VARIANT`. 

Hours of googling did not reveal anything useful, maybe I'm not very good at it. 
So in the end I ended up debugging the actual call in WinDBG on assembly level. 
What I've seen didn't make sense at first glance.

Let's have a look. Consider following delegate and code that call it. The
delegate takes one `int` argument. I'm starting with `int` as `int` is simple
enough.

    #!csharp
    // delegate type
    public delegate void DintRvoid(int arg);

    // method that takes a delegate ob above type and calls it 
    public static void CallDintRvoid(DintRvoid del, int arg) { del(arg); }

Here's the code in full that creates the delegate and call `CallDintRvoid()` 
method (which in turn call the passed delegate):

    | delegateType delegate callback mock |
    delegateType := (appDomain Load: 'Bee.CLRInterop.Tests.Mocks')
        GetType: 'Bee.CLRInterop.Tests.Mocks.DintRvoid'.
    callback := OsCallback new
        receiver: [:a1 | 
            Transcript show: 'Called by CLR, a1: ' , a1 printString; cr]
        selector: #value:
        types: #(long)
        returnType: #long
        callingConvention: #api;
        bind.
    delegate := (Smalltalk at: #'Bee.CLRInterop.Support')
        GetDelegateForFunctionPointer32: (callback asParameter longAtOffset: 0)
        _: delegateType.
    mock := appDomain
        CreateInstanceAndUnwrap: 'Bee.CLRInterop.Tests.Mocks'
        _: 'Bee.CLRInterop.Tests.Mocks.CLRDelegateTestMock'.
    mock class CallDintRvoid: delegate _: -10.   

Once executed, voila, we get a message *Called by CLR, a1: -10* on a Transcript.

So far, so good. More interesting would be what would happen when we call it
with some real object (i.e., with a reference type in CLR terminology):

    #!csharp    
    public delegate void DObjectRvoid(Object arg);
    
    public static void CallDObjectRvoid(DObjectRvoid del, Object arg) { del(arg); }

And the code that call the above: 

    delegateType := (appDomain Load: 'Bee.CLRInterop.Tests.Mocks')
        GetType: 'Bee.CLRInterop.Tests.Mocks.DObjectRvoid'.
    callback := OsCallback new
        receiver: [:a1 | 
            a1saved := a1.
            Transcript show: 'Called by CLR, a1: ' , a1 printString; cr]
        selector: #value:
        types: #(ulong)" <-- change here"
        returnType: #long
        callingConvention: #api;
        bind.
    delegate := (Smalltalk at: #'Bee.CLRInterop.Support')
        GetDelegateForFunctionPointer32: (callback asParameter longAtOffset: 0)
        _: delegateType.
    mock := appDomain
        CreateInstanceAndUnwrap: 'Bee.CLRInterop.Tests.Mocks'
        _: 'Bee.CLRInterop.Tests.Mocks.CLRDelegateTestMock'.
    mock class CallDObjectRvoid: delegate _: appDomain.
    self assert: a1saved = appDomain


When executed, it prints 'Called by CLR, a1: 13'. We passed an instance of
and `System.AppDomain` to a delegate and we got `13`. Indeed I'd expect a
number, likely a numeric value of a *pointer* to the object in CLR heap or
to some proxy or something. But `13` is weird. Okay, let's try to pass something 
else, say an instance of `System.RuntimeType`. Again, it prints 
'Called by CLR, a1: 13'! Funny, very funny. 

If you look at the `OsCallback` invocation machinery in Bee, it is quite complex
and a lot of code is involved including some generated assembly and a VM routine. 
Let's first check what is CLR passing to delegate function to rule out all problems
that might be in Bee code.

The idea is simple: lets put a machine level breakpoint to a (generated) callback
function entry point and then examine the (machine) stack. We're on i386 and 
using `__stdcall` calling convention so all parameters are on the stack.  

First, create a new test delegate and corresponding calling method: 

    #!csharp
    public delegate void DuintObjectuintRvoid(uint arg1, Object arg2, uint arg3);

    public static void CallDuintObjectuintRvoid(DuintObjectuintRvoid del, uint arg1, Object arg2, uint arg3) 
    { 
        del(arg1, arg2, arg3); 
    }       

This is essentially the same as in previous case only that the `Object` argument (`arg2`)
is surrounded by two `uint` arguments (`arg1` and `arg2`). When calling, we'll
pass some interesting numbers such as `0xDEADBEEF` and `0xCAFEBABE`. They will 
serve as sentinels when analyzing stack. We can use any value, but these are 
easier for humans to spot in raw memory dumps. Just a little trick. 

Second, we need to place a breakpoint at callback stub (machine) entry point. Since
callback stub is generated, we have to modify the test code to print the entry
point so we can put a (machine) breakpoint on it: 

    ...
    callback := OsCallback new
        receiver: [:a1 :a2 :a3 | 
            a2saved := a1.
            Transcript show: 'Called by CLR, a2: ' , a2 printString; cr]
        selector: #value:value:value:
        types: #(ulong ulong ulong)
        returnType: #long
        callingConvention: #api;
        bind.
    "   v---- new code   "
    Transcript
        show: '>> thunk address: %' , (callback asParameter longAtOffset: 0) hex;
        cr.
    KernelLibrary DebugBreak.
    "   ^---- new code   "
    ...

To make it easier in debugger to set the breakpoint, we issue a [DebugBreak()][6]
to stop execution as soon as callback stub is generated.

So, time to fire a [WinDBG][7], attach it to a running Bee and run the modified
test (the one with entry point debug print and a `DebugBreak()`). 
The debugger should stop at debug breakpoint:

    (a60.ddc): Break instruction exception - code 80000003 (first chance)
    *** ERROR: Symbol file could not be found.  Defaulted to export symbols for H:\Projects\Bee\sources3\vvm31w.dll - 
    eax=75414935 ebx=00000001 ecx=00000000 edx=001cf300 esi=00000001 edi=104fd1a0
    eip=74c4338d esp=001cf2b8 ebp=001cf2d8 iopl=0         nv up ei pl zr na pe nc
    cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00200246
    KERNELBASE!DebugBreak+0x2:
    74c4338d cc              int     3

Now we can create a breakpoint on callback. Its entry point address can be found
in Bee transcript window. It'd be so much nicer to have it printed in console
window because now if you're not careful enough and obscure transcript window
by another window while Bee is stopped, you're screwed as Bee windows don't 
redraw. Some day somebody may add support for console output. Anyways, let's
add the breakpoint and run it:

    0:000> bp %4DD0000
    0:000> bl
    2 e 04dd0000     0001 (0001)  0:****
    0:000> g

The breakpoint hits immediately: 

    Breakpoint 2 hit
    eax=04dd0000 ebx=00000000 ecx=00000040 edx=04fded54 esi=001ccc4c edi=001ccc34
    eip=04dd0000 esp=001ccbf4 ebp=001ccc88 iopl=0         nv up ei pl nz na pe nc
    cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00200206
    04dd0000 68c4024600      push    4602C4h
    *** WARNING: Unable to verify checksum for C:\Windows\assembly\NativeImages_v4.0.30319_32\mscorlib\50bcbedc6ed7027bd709339d3ec4c388\mscorlib.ni.dll

Now let's examine machine stack: 

    0:000> r esp
    esp=0028cc44
    0:000> dd %28cc44
    0028cc44  04a3064a deadbeef 0000000d 00000000
    0028cc54  0490002c 00000000 cafebabe 1f07125d
    0028cc64  73fca404 0028d000 00000018 0028cc48
    0028cc74  04a3064a 0028ccd8 04dead98 00737810
    0028cc84  0000000d 00000000 0490002c 00000000
    0028cc94  04a60000 cafebabe 0000000d 00000000
    0028cca4  0490002c 00000000 deadbeef 00000002
    0028ccb4  deadbeef 00000000 0028cc60 00000000

The first machine word (`04a3064a`) is a return address, arguments follow. 
As we can see, first argument is `deadbeef`, as expected. Second argument
is `0000000d`, that is the funny 13 we got into a Smalltalk is our initial
experiments. Then I'd expect to see `cafebabe`, the third argument. But no,
there's just plain `NULL`. Followed by something that look like a pointer,
then another `NULL`. Finally, the value of third argument - presumably. 
So, an reference type is passed in a four consecutive (machine) words. 
Interesting. 

After a while, I though it could be some sort of handle. Perhaps, 
a .NET [GCHandle][8] holding (hopefully) a pinned reference to
a real object. It would make a perfect sense, wouldn't it? GCHandle is a structure
and structures are passed by value (as opposite to classes which are passed by 
reference). If you look to reference implementation
to GCHandle class, you'll find out that it has 4 members. Here we see 4 machine
words being on a stack. That can't be a coincidence! 

Except it is. If you look closely at GCHandle, first field is an enum 
[GCHandleType][9]. However, 13 (0000000d) is not a valid enum value. 
Third argument is a *reference* to a GCHandleCookieTable class. Although
0490002c looks like a pointer to CLR object, it's not. Check in debugger:

    !do 0490002c
    <Note: this object has an invalid CLASS field>
    Invalid object

It took me a long time to abandon the idea that what's on the stack is a
GCHandle. 

Let's start from the beginning. The first two and the last (machine) words seem
to be useless, they're always the same. Third word must be a key. We know that
we passed an instance of `System.AppDomain`, so this must be somehow related. 
Let's dump all appdomain first. 

The way to do so is to "simply" iterate over all objects on a CLR heap, check if
it's an instance of `System.AppDomain` (or whatever class one looks for). 
We need to know that first (machine) word of each CLR object points to a
method table. We can get the method table by `!name2ee`. So: 

    0:000> !name2ee *!System.AppDomain
    Module:      71451000
    Assembly:    mscorlib.dll
    Token:       0200008c
    MethodTable: 7188edd0
    EEClass:     714553d8
    Name:        System.AppDomain
    --------------------------------------
    Module:      047d49a0
    Assembly:    Bee.CLRInterop.dll
    --------------------------------------
    Module:      047d5018
    Assembly:    Bee.CLRInterop.Tests.Mocks.dll

`System.Appdomain`s method table is: at`0x7188edd0`. So:

    0:000> .foreach ( adr {!dumpheap -short}) { .if (poi(${adr}) == 7188edd0) { .printf "INST AT %x\n", ${adr}  } } 
    INST AT 4db12f8

The appdomain seem to be at 0x4db12f8 - nowehere near to the 0490002c, value 
passed to the delegate. Let's inspect the appdomain anyway:

    0:000> !do %4db12f8
    Name:        System.AppDomain
    MethodTable: 7188edd0
    EEClass:     714553d8
    CCW:         04900020
    Size:        112(0x70) bytes
    File:        C:\Windows\Microsoft.Net\assembly\GAC_32\mscorlib\v4.0_4.0.0.0__b77a5c561934e089\mscorlib.dll
    Fields:
          MT    Field   Offset                 Type VT     Attr    Value Name
    7188ecb8  4000577        4        System.Object  0 instance 00000000 __identity
    718914a8  400033b        8 ....AppDomainManager  0 instance 00000000 _domainManager
    71461b08  400033c        c ...ect[], mscorlib]]  0 instance 00000000 _LocalStore
    7188f184  400033d       10 ...em.AppDomainSetup  0 instance 04db13d0 _FusionStore
    7188f568  400033e       14 ...y.Policy.Evidence  0 instance 00000000 _SecurityIdentity
    7188ed0c  400033f       18      System.Object[]  0 instance 00000000 _Policies
    718791a0  4000340       1c ...yLoadEventHandler  0 instance 00000000 AssemblyLoad
    7187e77c  4000341       20 ...solveEventHandler  0 instance 00000000 _TypeResolve
    7187e77c  4000342       24 ...solveEventHandler  0 instance 00000000 _ResourceResolve
    7187e77c  4000343       28 ...solveEventHandler  0 instance 00000000 _AssemblyResolve
    7187e77c  4000344       2c ...solveEventHandler  0 instance 00000000 ReflectionOnlyAssemblyResolve
    71870a94  4000345       30 ....Contexts.Context  0 instance 00000000 _DefaultContext
    71879d94  4000346       34 ...ActivationContext  0 instance 00000000 _activationContext
    71879dec  4000347       38 ...plicationIdentity  0 instance 00000000 _applicationIdentity
    718903a8  4000348       3c ....ApplicationTrust  0 instance 00000000 _applicationTrust
    71844318  4000349       40 ...ncipal.IPrincipal  0 instance 00000000 _DefaultPrincipal
    71870c00  400034a       44 ...cificRemotingData  0 instance 00000000 _RemotingData
    7188b958  400034b       48  System.EventHandler  0 instance 00000000 _processExit
    7188b958  400034c       4c  System.EventHandler  0 instance 00000000 _domainUnload
    71881584  400034d       50 ...ptionEventHandler  0 instance 00000000 _unhandledException
    7188f46c  400034e       54      System.String[]  0 instance 00000000 _aptcaVisibleAssemblies
    7146108c  400034f       58 ...bject, mscorlib]]  0 instance 00000000 _compatFlags
    7145bb64  4000350       5c ...tArgs, mscorlib]]  0 instance 04dead5c _firstChanceException
    7188d0f0  4000351       60        System.IntPtr  1 instance   703078 _pDomain
    71846a40  4000352       64         System.Int32  1 instance        0 _PrincipalPolicy
    71888998  4000353       68       System.Boolean  1 instance        0 _HasSetPolicy
    71888998  4000354       69       System.Boolean  1 instance        1 _IsFastFullTrustDomain
    71888998  4000355       6a       System.Boolean  1 instance        1 _compatFlagsInitialized
    7185d61c  4000357      cf0         System.Int32  1   shared   static s_flags
        >> Domain:Value  00703078:NotInit  <<

Quite a lot, but following is very interesting and familiar: 

    ...
    CCW:         04900020
    ...

The value we've seen on the stack is 0490002c, three (machine) words higher than
CCW. CCW stands for *COM-callable wrapper*. Very promising indeed. CCW itself is
not necessarily a pointer to COM interface. So let's have a look the value of 
COM object we're using in Bee that represents the appdomain: 

    ClrApplication current appDomain asInteger hex "-> 490002C"

Very good indeed. The third word of those four words pushed onto a stack is
actually a pointer to `_AppDomain` COM interface. 

Now we can adapt the experiment taking all of this into an account to make it
working: 

    
    | delegateType delegate callback mock |

    delegateType := (appDomain Load: 'Bee.CLRInterop.Tests.Mocks')
        GetType: 'Bee.CLRInterop.Tests.Mocks.DuintObjectuintRvoid'.
    callback := OsCallback new
        receiver: [:a1 :ignored1 :ignored2 :a2raw :ignore3 :a3 | | a2 |
            "   v---- new code   "
            a2 := ClrObject
                automationFrom: (_ObjectClient
                    objectFromIUnknown: (IUnknownClient fromAddress: a2raw)).
            "   ^---- new code   "            
            Transcript show: 'Called by CLR, a2: ' , a2 printString; cr]
        selector: #value:value:value:value:value:value:
        types: #(ulong ulong ulong ulong ulong ulong)
        returnType: #long
        callingConvention: #api;
        bind.
    delegate := (Smalltalk at: #'Bee.CLRInterop.Support')
        GetDelegateForFunctionPointer32: (callback asParameter longAtOffset: 0)
        _: delegateType.
    mock := appDomain
        CreateInstanceAndUnwrap: 'Bee.CLRInterop.Tests.Mocks'
        _: 'Bee.CLRInterop.Tests.Mocks.CLRDelegateTestMock'.
    mock class
        CallDuintObjectuintRvoid: delegate
        _: 16rDEADBEEF
        _: appDomain
        _: 16rCAFEBABE.

The difference is in the callback. It now takes 6 raw ulong (uint32) arguments,
one for first real argument, then four ulongs for second arguments and finally
one for the third. Then inside the callback we convert the raw pointer into 
a interop object, an instance of a smalltalk proxy for `System.AppDomain` 
instances so we can send it messages and so on. 

Indeed, in the end you need not to manually fiddle about all this yourself. All
we need is to create custom subclass of `OsCallback` and override its 
`#getArgumentsFrom:` to do the conversion itself. Also, since we know the delegate
type, we may ask it for number and types of arguments so you don't need to know
that on a machine level there are four words passed for given parameter. 
`ClrCallback` can do the magic for you. 

Job done!

Except it is not. I has been long time I learned to trust no one, including
myself, when doing things like this. Let's do couple more experiments. 

First, pass something else than an appdomain. Say, the delegate itself, to
make it easy. Just a little change in the code above:

    ...
    mock class
        CallDuintObjectuintRvoid: delegate
        _: 16rDEADBEEF
        _: delegate    " <-- change here"
        _: 16rCAFEBABE.

It works fine, but this is what you get on a (machine) stack:
   
    05763044  ........ deafbeef 00000009 00000000 
    05763044  04b1ed6c 00000000 cafebabe ........ 

As we can see, we have magical `9` in first (out of four) machine words, 
not `13` as before. Not a big deal, we don't use this value anyway. 9 is
as good as 13. Good to know, all the same. 

Let's do some other little experiment, just to have peace of mind. Let's create
a delegate that takes, `System.AppDomain` as a formal parameter:

    #!csharp

    public delegate void DuintAppDomainuintRvoid(uint arg1, AppDomain arg2, uint arg3);

    public static void CallDuintAppDomainuintRvoid(DuintAppDomainuintRvoid del, uint arg1, AppDomain arg2, uint arg3) { del(arg1, arg2, arg3); }

The test code is the same as in previous case, all we need is just to create
a delegate typed as `DuintAppDomainuintRvoid` (rather than `DuintObjectuintRvoid`
as we did so far) and call it via `CallDuintAppDomainuintRvoid` (rather than 
`CallDuintObjectuintRvoid`). 

When executed, it...crashes. Hard. Segmentation violation. Sigh, let's take
a step back and see what's on the (machine) stack: 

    058f0dc0  ........ deafbeef 04810030 cafebabe 
    05763044  1ebbc170 73fca404 003ecf90 ........ 

Wait, where's my magical 13. Or 9. Where are my 4 machine words? Everything is
exactly the same, the only difference here is the *formal* type of a delegate
parameter. If we change it back to expect one (machine) word per object-reference
argument as we did initially, it works. Try:


    | delegateType delegate callback mock |
    delegateType := (appDomain Load: 'Bee.CLRInterop.Tests.Mocks')
        GetType: 'Bee.CLRInterop.Tests.Mocks.DuintAppDomainuintRvoid'.
    callback := DebugOsCallback new
        "   v---- changed code   "
        receiver: [:a1 :a2raw :a3 | | a2 |
        "   ^---- changed code   "
            a2 := ClrObject
                automationFrom: (_ObjectClient
                    objectFromIUnknown: (IUnknownClient fromAddress: a2raw)).            
            Transcript show: 'Called by CLR, a2: ' , a2 printString; cr]
        "   v---- changed code   "
        selector: #value:value:value:        
        types: #(ulong ulong ulong )
        "   ^---- changed code   "
        returnType: #long
        callingConvention: #api;
        bind.
    delegate := (Smalltalk at: #'Bee.CLRInterop.Support')
        GetDelegateForFunctionPointer32: (callback asParameter longAtOffset: 0)
        _: delegateType.
    mock := appDomain
        CreateInstanceAndUnwrap: 'Bee.CLRInterop.Tests.Mocks'
        _: 'Bee.CLRInterop.Tests.Mocks.CLRDelegateTestMock'.
    mock class
        CallDuintAppDomainuintRvoid: delegate
        _: 16rDEADBEEF
        _: appDomain
        _: 16rCAFEBABE.

What's so special about type `object` (or `System.Object`) still remains hidden 
to me. 

Well, enough now. Let's call it a day! 

## Wrap-up

To wrap it up, we have found that: 

 * Simple data types such as integer, booleans, floats (presumably) are passed
   in raw form (i.e., uint as 32bit unsigned integer value).
 * Struct type are passed by value, the order of fields is generally undefined 
   and may vary (this has not been shown in this article).
 * reference types are passed as pointers `_Object` COM interface. 
 * Except if the formal argument is declared as `object` (or `System.Object`). 
   Then the value is passed in four consecutive machine words, the pointer
   to `_Object` COM interface being the third. Meaning of other remains 
   a mystery.

I find last two findings a bit scary. If there's one exception, how can I know
there are no others? I'm concerned about this - the consequence of getting this
"just a little wrong" is instant VM death. Very bad indeed. Finding a cause
may be tedious, time consuming work. We better get it right.

## Future Work

So far, we only checked `uint` and reference types. We should add support for
structures and check and validate support for all "primitive" types as well 
as (multidimensional) arrays of those. Like I did for COM marshaling / 
unmarshaling. 

[1]: /2016/10/a-taste-of-dotNET-in-Bee-Smalltalk.html
[2]: https://docs.microsoft.com/en-us/dotnet/articles/csharp/programming-guide/delegates/
[3]: https://msdn.microsoft.com/en-us/library/system.appdomain.firstchanceexception%28v=vs.110%29.aspx
[4]: https://msdn.microsoft.com/en-us/library/zdx6dyyh%28v=vs.110%29.aspx
[5]: https://msdn.microsoft.com/en-gb/library/windows/desktop/ms221627%28v=vs.85%29.aspx
[6]: https://msdn.microsoft.com/en-us/library/windows/desktop/ms679297%28v=vs.85%29.aspx
[7]: https://msdn.microsoft.com/en-gb/library/windows/hardware/ff551063%28v=vs.85%29.aspx
[8]: https://msdn.microsoft.com/en-us/library/system.runtime.interopservices.gchandle%28v=vs.110%29.aspx
[9]: https://msdn.microsoft.com/en-us/library/83y4ak54%28v=vs.110%29.aspx