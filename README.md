# Sandbox Experiment

This is an implementation of sandboxed code using the Java SecurityManager, written in Scala.

It takes inspiration from Jens Nordahl's [Sandboxing plugins in Java](http://www.jayway.com/2014/06/13/sandboxing-plugins-in-java/), although it took some poking to see what the parameters are.

It consists of a Main class that starts up a sandbox, then starts a script from within the sandbox.

## Running

Install SBT ("brew install sbt" or similar package manager).  Then type `sbt run`.  If all goes well, the script should execute.

You may need to change `testscript.sh` to be executable.

In `build.sbt` there are some command aliases that will let you run the code inside of SBT

```
addCommandAlias("runThread", "run com.tersesystems.sandboxexperiment.sandbox.ThreadSpawner")
addCommandAlias("runDeserializer", "run com.tersesystems.sandboxexperiment.sandbox.ObjectDeserializer")
addCommandAlias("runPrivileged", "run com.tersesystems.sandboxexperiment.sandbox.PrivilegedScriptRunner")
```

i.e. `sbt runThread` which will let you try things as well.

## How it works

The Java SecurityManager / architecture can prevent sensitive operations (like file execution) to sandboxed code, even if it's a server side application.

The SandboxPolicy class is where all the magic happens.  Specifically, this line:

```
permissions.add(new java.io.FilePermission(scriptName, "execute"))
```

If you comment out that line and recompile (or just run `sbt run` again), then you'll see errors.

## DoPrivilegedAction

You can create a set of library utilities that can operate with code in the sandbox, using the AccessController.doPrivileged method.  Although this is meant to be used with the sandbox, it must be outside of the sandbox itself so that the immediate caller is in the right ProtectionDomain.

Because Scala has closures and magic "apply" method, we can use a cleaner DoPrivilegedAction: 

```
AccessController.doPrivileged(DoPrivilegedAction {
  if (! canonicalFile.exists()) {
    throw new IllegalStateException(s"$canonicalFile does not exist!")
  }
}, AccessController.getContext, new FilePermission(absolutePath, "read"))
```

Note that there's no need for a DoPrivilegedExceptionAction, as all exceptions are runtime in Scala.

Unfortunately, the doPrivileged method is magic: it relies on its immediate caller for the security check, and so you can't wrap it in a generally available utility class without giving it the permissions of that utility class.

There's much more that can be done in that area: for example, having typed permissions instead having everything be RuntimePermission.

## Logging

There's also a complete set of logs using Logback, containing all the security information in the policy, in JSON format, using the SLF4J marker API.

You don't need all the logging information, but it does show how you'd log information (especially if someone is trying an exploit).

## This Looks Complicated

It's not as bad as it looks, but it is not well laid out for application developers.  This system was originally designed for applets, so it makes sense that it assumes discrete packaging.  The tough bit is adding all the permissions to the sandbox.

If you have an existing application and you just want to blacklist some operations, you can use [Pro-Grade](http://pro-grade.sourceforge.net/) as an [easy way to secure applications](https://tersesystems.com/2015/12/22/an-easy-way-to-secure-java-applications/).

## Why do it this way?

Because this is the defined way to do it.  Per [Evaluating the Flexibility of the Java Sandbox](https://www.cs.cmu.edu/~clegoues/docs/coker15acsac.pdf):

> Java provides an “orthodox” mechanism to achieve this goal while aligning with intended sandbox usage: a custom class loader that loads untrusted classes into a constrained protection domain. This approach is more clearly correct and enables a self-protecting sandbox.

This example uses a custom security policy rather than a constrained protection domain, but the idea is the same.

## Limitations and Warnings

Note that the security policy can only distinguish by code location (packaging) and by class loader (which means URLClassLoader, which also means packaging).  This means that you probably need to package sandbox code in a different jar -- I have been completely unable to effectively sandbox code that was in the same package. 

There are notes about the ProtectionDomain in the Java Security book that suggest that packaging sandboxed code together with the Main, "AllPermissions" class is a Bad Idea and will compromise the system, so this is probably for the best.

## Other projects

Check out https://github.com/johnlcox/sandbox-runtime which uses Byte Buddy for more fine grained permissions.

Check out https://github.com/wsargent/securityfixer for overriding SecurityManager so it can't be overridden.
