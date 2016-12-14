---
title: "A taste of .NET in Bee Smalltalk"
created_at: 2016-10-13 10:26:13 +0000
kind: article
published: true
---

Couple days ago, I finally managed to get .NET to do a job for me. Not much but 
still .NET did its job and did it well. The following text shows how to call
.NET code from Bee Smalltalk and retrieve results. I won't go into technical 
details on how it's done internally, rather show the usage from user point of
view. The goal is to give you an idea how it might feel to call .NET code from
Bee Smalltalk. 

<!-- more -->

## The Task

The task is very simple. I have a list of CDs in an XML file with custom XML 
format and I want to generate an HTML file, for instance to put it on my web site
or, say, to generate an HTML mail with price list. Whatever. The `cd.xml` files
looks like:

    #!xml
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <catalog>
      <cd>
        <title>Empire Burlesque</title>
        <artist>Bob Dylan</artist>
        <country>USA</country>
        <company>Columbia</company>
        <price>10.90</price>
        <year>1985</year>
      </cd>
      <cd>
        <title>Hide your heart</title>
        <artist>Bonnie Tyler</artist>
        <country>UK</country>
        <company>CBS Records</company>
        <price>9.90</price>
        <year>1988</year>
      </cd>
      <!-- ...more... -->
    </catalog>

I'd like to use XSL transformation to create resulting HTML. I can of course 
parse it and generate HTML myself, but that's too much manual work. I'd rather
write a simple XSL transformation [1] and let XSLT processor to do the hard job. 
This is, after all, what XSL was designed for. The `cd.xsl` looks like: 

    #!xslt
    <?xml version="1.0" encoding="ISO-8859-1"?>
    <xsl:stylesheet version="1.0" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
    <xsl:template match="/">
      <html>
        <body>
        <h2>My CD Collection</h2>
          <table border="1">
            <tr bgcolor="#9acd32">
              <th>Title</th>
              <th>Artist</th>
            </tr>
            <xsl:for-each select="catalog/cd">
            <tr>
              <td><xsl:value-of select="title"/></td>
              <td><xsl:value-of select="artist"/></td>
            </tr>
            </xsl:for-each>
          </table>
        </body>
      </html>
    </xsl:template>
    </xsl:stylesheet>

There's no (native) implementation of XSLT processor in Bee (and if 
fact in no Smalltalk I know of). However .NET contains XSLT processor [2] so 
let's "just" use it. 

Looking ant documentation of .NET `System.Xml.Xsl.XslCompiledTransform` class,
the plan is simple: 

 * load data (`cd.xml`),
 * create an output buffer (wrapped in `System.Xml.XmlWriter` for `XslCompiledTransform` convenience),
 * create an instance of `XslCompiledTransform`,
 * load stylesheet (`cd.xsl`),
 * let `XslCompiledTransform` instance work out the magic,
 * read contents of the buffer,
 * job done.

In Bee:

    #!smalltalk
    [
        "load data as instance"
        
        data := System.Xml.XPath.XPathDocument new: 'cd.xml'.
       
        "create an output buffer"
        buffer := System.Text.StringBuilder new.
        writer := System.Xml.XmlWriter Create: buffer.

        "create an XSLT processor and load stylesheet"
        xslt :=  System.Xml.Xsl.XslCompiledTransform new.
        xslt Load: 'cd.xsl'.
        
        "let the magic happen"    
        xslt Transform: data _: nil _: writer _: nil.

        "retrieve resulting HTML"
        ^ buffer ToString.
    ] on: System.IO.FileNotFoundException do: [:fnfex |
        "Not a great handler, but for a demo it will do"
        TaskDialog warning: 'OOPS, file''s missing: ', fnfex FileName.
    ] on: System.Xml.Xsl.XsltException do:[:xsltex |
        TaskDialog warning: 'OOPS, transformation failed: ' , xsltex Message
    ].
    "job done"

Much like in C#, isn't it? That's the goal. Bee's CLR interop does the
magic for you. Well, not quite, see below. However, you can see that: 

 * To refer to a .NET class, use fully qualified class name 
 * Method names matches .NET method names, including the capitalization (it is the convention of .NET that method names starts with uppercase letter). If the method takes arguments, the keyword preceding second and more argument is `_:`. This is fixed in oder to make the code more predictable. 
 * Constructors are exposed as class-side #new method, as it is customary in 
   Smalltalk
 * Static methods are exposed as class-side methods
 * Overloaded methods and constructors are properly handled, dispatching to 
   appropriate implementation based on *runtime* types of arguments.
 * .NET exception can be handled using Smalltalk idioms: just handle it
   using #on:do: as usual. The handler gets reference to .NET exception object.
   .NET exceptions can be rethrown but cannot be resumed (because the way CLR
   deals with exceptions).

 ## A Little Sweeter (yet) Hypothetical Code 

 Having to prefix classes is not so nice. Also having to use `_:` keyword makes 
 the code to look little ugly. It can be done in a much sweeter way. Imagine: 

    #!smalltalk    
    Object
      subclass: #CD2HTMLGenerator
      instanceVariableNames: ''
      classVariableNames: ''
      poolDictionaries: 'System.Xml System.Xml.Xsl System.Xml.XPath'
    "..."
    generate  
      | data buffer writer xslt |
      data := XPathDocument new('cd.xml').    
      buffer := System.Text.StringBuilder new().
      writer := XmlWriter Create(buffer).    
      xslt := XslCompiledTransform new().
      xslt Load('cd.xsl').      
      xslt Transform(data, nil, writer, nil).      
      ^ buffer ToString().

Note that:

 * One can "import" classes from particular namespace by simply using a pool 
   dictionary with the same name as the namespace. This has more or less the 
   same effect as C# `using` declaration [3]. 
 * One can use "parenthesis" syntax for calling method with arguments just 
   like in C-based languages. If found this nicer because in those languages
   method names do not play well with keywords and it makes it clear that you're
   entering a different realm. 

As I said, the above code is hypothetical, it's not possible to write the code 
like this  right now. Bee's parser and compiler would have to be extended. 
Changes are rather cosmetic - other compilers have been changed to allow such 
extended  syntax. Also I agree it's a matter of a taste. Personally I'd prefer 
this extended syntax. 

## Is it Really that Simple? 

No, as usual with computers, it is not. There's a little more preparatory work
that has to be carried before running the example. 

First of all, a suitable CLR runtime has to be loaded into a running Bee process. 
So far I worked only with .NET 4.5 but generally speaking, .NET 4.x should work
just the same. To load CLR, perform following Mumbo Jumbo:

    #!smalltalk
    runtimeInfo := ICLRMetaHostClient newInstance defaultRuntime.
    runtimeInfo isLoaded ifFalse:[
      runtimeInfo getCLRHost Start
    ].
    runtimeInfo getCorHost DefaultDomain.

Another question is from where the classes like `System.Xml.Xsl.XslCompiledTransform`
come from? Obviously, these are just a little proxy classes for real .NET types,
but how does Bee Smalltalk know what types are there in what assemblies? 

It does not (for now). You have to first manually import class you want to use 
in your code. Class `CLRTranslator` does most of the job for you, so the process
of importing the code boils down to executing something like: 

    #!smalltalk
    ClrTranslator 
      importType: 'System.Xml.Xsl.XslCompiledTransform' 
      from:'System.Xml, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089'

Basically one has to tell which type (a fully qualified name is required here) 
and from which assembly (System.Xml in the above example). Underlying code uses
`AppDomain.Load(String)` [4] to load the assembly. This means that CLR runtime
roles apply when resolving assembly name [5]. In short: 

  * For application-provided assemblies (shipped with the application) one can
    use short name i.e., `Bee.CLRInterop.Tests.Mocks`. Also for CLR core library
    one may use just `mscorlib`. 
  * For assemblies stored in GAC [6], one has to use full name, including
    version, culture and pubkey information, i.e., `System.Xml, Version=2.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089`. To list all assemblies
    registered in GAC, use `gacutil.exe -l` (`gacutil.exe` can be found in 
    Microsoft SDK for Windows). If you won't use full name, you'd get an error
    stating that assembly or one of its dependencies could not be loaded. 

When a class is imported into Bee as shown above it is automatically put into
a Bee package corresponding with it's assembly. For example, the class 
`System.Xml.Xsl.XslCompiledTransform` would be put into package `System.Xml`, class
`System.AppDomain` would be in package `mscorlib`. In other words, there's 
one-to-one mapping between .NET assemblies and Bee packages hosting proxy 
classes. Such package is created if it does not exist already. One can save such 
a package for future reuse so there's no need to import classes every time Bee 
starts up. 

The need for manual imports is, I hope, temporary until a mechanism to do it 
lazily on demand is implemented. 

## Conclusion

In this article I have demonstrated a use of .NET classes from Bee Smalltalk 
as implemented in a prototype interop package. Since it was a prototype, a lot 
of things are not implemented at all (such as exception propagation, object 
identity management, Smalltalk delegates, generic types, threading and so on) 
or implemented partially (proper marshaling of all basic value types). 

Nonetheless the above example actually runs on my machine. 

## Acknowledgment

This prototype has been implemented as part of my work at [CaesarSystems][7].
Many thanks for letting me to work on this and for their support. It has been
- and still is - a great fun!

[1]: https://www.w3.org/TR/xslt20/
[2]: https://msdn.microsoft.com/en-us/library/system.xml.xsl.xslcompiledtransform(v=vs.110).aspx
[3]: https://msdn.microsoft.com/en-gb/library/dfb3cx8s.aspx
[4]: https://msdn.microsoft.com/en-us/library/8wcywcds(v=vs.110).aspx
[5]: https://msdn.microsoft.com/en-us/library/yx7xezcf(v=vs.110).aspx
[6]: https://msdn.microsoft.com/en-us/library/yf1d93sz(v=vs.110).aspx
[7]: https://www.petrovr.com/