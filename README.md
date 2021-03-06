# Scala iOS Demo

This is an extremely simple demo app that chiefly exists to show the complex plumbing that can be
used to allow one to write iOS applications in Scala. Here's a quick breakdown of the process:

  * Code is written in [Scala], and compiled with scalac.
  * Java bytecodes (for app, scala-library.jar and any other third party jars) are converted from
    Java bytecode to CLR bytecode by [IKVM].
  * CLR bytecodes are compiled by [Xamarin.iOS] \(nee MonoTouch) into ARM x86 assembly.

## Obligatory Screenshot

![Obligatory screenshot](screenshot.png)

## See the code!

Here's the [Scala code](core/src/main/scala/iscala/AppDelegate.scala) that is
responsible for the above app.

## Building

To build and run this demo app, you will need the following:

  * [Maven]
  * [Scala]
  * [ikvm-monotouch] \(building it is hard, so just get the [pre-built version])
  * [Xamarin.Studio]

You need to edit `app/pom.xml` and set `ivkmPath` to the path at which you installed
`ikvm-monotouch`. Otherwise no manual tweaks should be needed.

With all of the above installed, you can build the app thusly (on a Mac, you can't develop iOS apps
on any other platform):

    % mvn clean package
    % open app/iscala.sln
    # click the Run arrow in Xamarin.Studio

## Hacking

This is mainly a proof of concept. I did a bunch of work to make it possible to make iOS games in
Java using the [PlayN] cross-platform game framework, and I knew that it should in theory be
possible to use the same set of tools to make Scala run on iOS. Indeed it is.

If you are seriously going to undertake to write an iOS app in Scala, here are a grab bag of
caveats and recommendations:

### Put together a better build system

Build using SBT or something less slow than Maven. You'll need to implement the moral equivalent of
what my [ikvm-maven-plugin] does, but if you just run "mvn package -X" you'll see the big fat
command line it puts together at the end, and you can work it out from there.

You might also want to investigate using IKVM's "on the fly" bytecode converter to enable a more
efficient edit/compile/debug cycle. Currently, to test your app, you have to run the Scala code
through Proguard every time you want to run it in the iOS simulator (otherwise MonoTouch will
require ages to grind through all of `scala-library.jar`). However, MonoTouch uses the normal Mono
VM when running code in the simulator, so you can do things there (like on-the-fly byte-code
compilation) that you can't do in an app that's built for the device. Thus it is probably possible
to wire up IKVM's on-the-fly Java bytecode to CLR bytecode translator, plus the existing Mono VM,
to enable a development environment where you can incrementally recompile Scala classes and simply
restart your simulated app to incorporate the new code.

### Learn about IKVM's magic

Learn about how IKVM translates Java bytecode to C# bytecode, because you're going to see the iOS
APIs through the lens of C#-ness. Xamarin did a great job of converting the Objective-C APIs into
C# APIs, which are easy for a C# programmer to use. You will now be using those C# APIs "as seen"
from Java, and what's worse, it's Java as seen from Scala, which is not always smooth sailing.

For example, IKVM generates synthetic inner interfaces for C# delegates as well as C# attributes,
but it fails to mark them static. `javac` doesn't care because it knows that all inner-interfaces
are implicitly static, but `scalac` thinks they require an outer-this pointer to be constructed
which requires hackery (shown in the example code) to work around.

Unfortunately there's not a comprehensive list of all the things IKVM does to map between the JVM's
view of the world and the CLR's view of the world. You kind of have to plow through ten plus years
of [Jeroen's blog posts] to find whatever particular thing you're looking for. Of course, Google is
your friend here.

In a rare case of "two wrongs do make a right", using C# generics from Scala by way of IKVM is
horribly unpleasant, but the iOS API originates from Objective-C code in the first place, so
Xamarin rarely makes use of C# generics in their C# version of the API. So you don't often run into
that unpleasantry.

### You can use Interface Builder

The demo app happens to use [MonoTouch.Dialog], which is a nice programmatic way of creating UIs on
iOS. However, it is also possible to use Interface Builder to create apps. The [MonoTouch UI docs]
explain how you wire up IB UIs via C#, and anything you can do in C# you can do in Scala, modulo a
variable degree of cumbersome translation.

### Don't use JDK classes

[ikvm-monotouch] uses a hacked up version of OpenJDK where 90% of the crap from the JDK was excised
to get things working with the limited CLR profile made available by MonoTouch. This means you
cannot use things like `java.net` or most of `java.nio`, and there's no bundled implementation of
CORBA or LDAP or any of the other two dozen random bits of enterprise crap that Sun/Oracle has
piled in there over the years. By extension, anything in Scala that builds on those bits is not
going to work. You're probably not planning on writing an enterprise web server for the iPhone, but
you should still bear in mind that you're not in Kansas anymore.

Some of the JDK stuff *does* work, but that still doesn't mean that you should use it. Using
something like `java.io.File` means that you're using IKVM's implementation of Java files on top of
the CLR libraries, and the CLR libraries are then implemented via the iOS API. Bad idea. Just use
the iOS API in the first place and you'll save yourself a great deal of pain and applications size.
`NSUrl`, `NSData` and friends are not that difficult to use, and they'll provide the best
performance.

Hopefully you were not thinking you would be able to use AWT and/or Swing. Even if it did work,
which it doesn't, Steve Jobs would likely come back from the dead just to smack you.

The Scala collections library works, and that (and all the fun things you can do with Scala
yourself) is pretty much the extent of what you can bring over from the Java world. Everything else
should be built on the iOS APIs directly. Of course, if you're extremely motivated, you can improve
my [ikvm-monotouch] port and make more Java stuff work out of the box, but you'll still be running
everything through two layers of emulation, on under-powered, power-starved mobile CPUs. Think of
the children.

### iOS API docs

You can often figure out what you want to do by either looking at the [Objective-C documentation]
directly (or reading about how someone did something in Objective-C) and then looking at the
[MonoTouch API docs] to see whether and how the API differs in C#.

## Licensing

As far as I can tell (IANAL), the entire tool-chain involved here is AOK for developing apps and
selling them on the app store. A number of parties (myself included) have shipped commercial games
based on this toolchain (minus Scala). Xamarin.Studio is a commercial product, so you have to cough
up for that, but everything else is open source and licensed sufficiently liberally for commercial
use.

## Questioning

I'm on [scala-tools], so feel free to post there with questions or whatnot.

[Scala]: http://scala-lang.org
[IKVM]: http://ikvm.net
[Xamarin.iOS]: http://xamarin.com
[Maven]: http://maven.apache.org/
[ikvm-monotouch]: https://github.com/samskivert/ikvm-monotouch
[Xamarin.Studio]: http://xamarin.com/download
[pre-built version]: https://dl.dropboxusercontent.com/u/404021/ikvm-monotouch.zip
[PlayN]: https://code.google.com/p/playn
[ikvm-maven-plugin]: https://github.com/samskivert/ikvm-maven-plugin
[MonoTouch.Dialog]: http://docs.xamarin.com/guides/ios/user_interface/monotouch.dialog
[MonoTouch UI docs]: http://docs.xamarin.com/guides/ios/user_interface
[Objective-C documentation]: http://developer.apple.com/library/ios/navigation/
[MonoTouch API docs]: http://docs.go-mono.com/
[Jeroen's blog posts]: http://weblog.ikvm.net/
[scala-tools]: https://groups.google.com/forum/?fromgroups=#!forum/scala-tools
