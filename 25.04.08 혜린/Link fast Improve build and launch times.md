# Link fast: Improve build and launch times

URL: https://developer.apple.com/videos/play/wwdc2022/110362/
category: BuildProcess
status: 진행 중
when: WWDC 2022

- 스크립트
    
    ♪ ♪ Nick Kledzik: Hi, I'm Nick Kledzik, lead Engineer on Apple's Linker team.
    Today, I'd like to share with you how to link fast.
    I'll tell you what Apple has done to improve linking, as well as help you understand what actually happens during linking so that you can improve your app's link performance.
    So what is linking? You've written code, but you also use code that someone else wrote in the form of a library or a framework.
    In order for your code to use those libraries, a linker is needed.
    Now, there are actually two types of linking.
    There is "static linking", which happens when you build your app.
    This can impact how long it takes for your app to build and how big your app ends up being.
    And there is "dynamic linking".
    This happens when your app is launched.
    This can impact how long your customers have to wait for your app to launch.
    
    - 목차
    
    In this session I'll be talking about both static and dynamic linking.
    First, I'll define what static linking is and where it came from, with some examples.
    Next, I'll unveil what is new in ld64, Apple's static linker.
    Then, with this background on static linking, I'll detail best practices for static linking.
    The second half of this talk will cover dynamic linking.
    I'll show what dynamic linking is and where it came from, and what happens during dynamic linking.
    Next, I'll reveal what is new in dyld this year.
    Then, I'll talk about what you can do to improve your app's dynamic link time performance.
    And lastly, we'll wrap up with two new tools that will help you peek behind the curtains.
    You'll be able to see what is in your binaries, and what is happening during dynamic linking.
    
    - static linking
    
    To understand static linking, let's go way back to when it all started.
    In the beginning, programs were simple and there was just one source file.
    Building was easy.
    You just ran the compiler on your one source file and it produced the executable program.
    But having all your source code in one file did not scale.
    How do you build with multiple source files? And this is not just because you don't want to edit a large text file.
    The real savings is not re-compiling every function, every time you build.
    What they did was to split the compiler into two parts.
    The first part compiles source code to a new intermediate "relocatable object" file.
    The second part reads the relocatable .
    o file and produces an executable program.
    We now call the second part 'ld', the static linker.
    So now you know where static linking came from.
    As software evolved, soon people were passing around .
    o files.
    But that got cumbersome.
    Someone thought, "Wouldn't it be great if we could package up a set of .
    o files into a 'library'?" At the time the standard way to bundle files together was with the archiving tool 'ar'.
    It was used for backups and distributions.
    So the workflow became this.
    You could 'ar' up multiple .
    o files into an archive, and the linker was enhanced to know how to read .
    o files directly out of an archive file.
    This was a great improvement for sharing common code.
    At the time it was just called a library or an archive.
    Today, we call it a static library.
    But now the final program was getting big because thousands of functions from these libraries were copied into it, even if only a few of those functions were used.
    So a clever optimization was added.
    Instead of having the linker use all the .
    o files from the static library, the linker would only pull .
    o files from a static library if doing so would resolve some undefined symbol.
    That meant someone could build a big libc.
    a static library, which contained all the C standard library functions.
    Every program could link with the one libc.
    a, but each program only got the parts of libc that the program actually needed.
    And we still have that model today.
    But the selective loading from static libraries is not obvious and trips up many programmers.
    To make the selective loading of static libraries a little clearer, I have a simple scenario.
    In main.
    c, there's a function called "main" that calls a function "foo".
    In foo.
    c, there is foo which calls bar.
    In bar.
    c, there is the implementation of bar but also an implementation of another function which happens to be unused.
    Lastly, in baz.
    c, there is a function baz which calls a function named undef.
    Now we compile each to its own .
    o file.
    You'll see foo, bar, and undef don't have gray boxes because they are undefined.
    That is, a use of a symbol and not a definition.
    Now, let's say you decide to combine bar.
    o and baz.
    o into a static library.
    Next, you link the two .
    o files and the static library.
    Let's step through what actually happens.
    First, the linker works through the files in command line order.
    The first it finds is main.
    o.
    It loads main.
    o and finds a definition for "main", shown here in the symbol table.
    But also finds that main has an undefined "foo".
    The linker then parses the next file on the command line which is foo.
    o.
    This file adds a definition of "foo".
    That means foo is no longer undefined.
    But loading foo.
    o also adds a new undefined symbol for "bar".
    Now that all the .
    o files on the command line have been loaded, the linker checks if there are any remaining undefined symbols.
    In this case "bar" remains undefined, so the linker starts looking at libraries on the command line to see if a library will satisfy that missing undefined symbol "bar".
    The linker finds that bar.
    o in the static library defines the symbol "bar".
    So the linker loads bar.
    o out of the archive.
    At that point there are no longer any undefined symbols, so the linker stops processing libraries.
    The linker moves on to its next phase, and assigns addresses to all the functions and data that will be in the program.
    Then it copies all the functions and data to the output file.
    Et voila! You have your output program.
    Notice that baz.
    o was in the static library but not loaded into the program.
    It was not loaded because the way the linker selectively loads from static libraries.
    This is non-obvious, but the key aspect of static libraries.
    Now you understand the basics of static linking and static libraries.
    Let's move on to recent improvements on Apple's static linker, known as ld64.
    By popular demand, we spent some time this year optimizing ld64.
    And this year's linker is.
    .
    .
    twice as fast for many projects.
    How did we do this? We now make better use of the cores on your development machine.
    We found a number of areas where we could use multiple cores to do linker work in parallel.
    That includes copying content from the input to the output file, building the different parts of LINKEDIT in parallel, and changing the UUID computation and codesigning hashes to be done in parallel.
    Next, we improved a number of algorithms.
    Turns out the exports-trie builder works really well if you switch to use C++ string_view objects to represent the string slices of each symbol.
    We also used the latest crypto libraries which take advantage of hardware acceleration when computing the UUID of a binary, and we improved other algorithms too.
    While working on improving linker performance, we noticed configuration issues in some apps that impacted link time.
    Next, I'll talk about what you can do in your project to improve link time.
    I'll cover five topics.
    First, whether you should use static libraries.
    And then three little known options that have a big effect on your link time.
    Finally, I'll discuss some static linking behavior that might surprise you.
    The first topic is if you are actively working on a source file that builds into a static library, you've introduced a slowdown to your build time.
    Because after the file is compiled, the entire static library has to be rebuilt, including its table of contents.
    This is just a lot of extra I/O.
    Static libraries make the most sense for stable code.
    That is, code not being actively changed.
    You should consider moving code under active development out of a static library to reduce build time.
    Earlier we showed the selective loading from archives.
    But a downside of that is that it slows down the linker.
    That is because to make builds reproducible and follow traditional static library semantics, the linker has to process static libraries in a fixed, serial order.
    That means some of the parallelization wins of ld64 cannot be used with static libraries.
    But if you don't really need this historical behavior, you can use a linker option to speed up your build.
    That linker option is called "all load".
    It tells the linker to blindly load all .
    o files from all static libraries.
    This is helpful if your app is going to wind up selectively loading most of the content from all the static libraries anyways.
    Using -all_load will allow the linker to parse all the static libraries and their content in parallel.
    But if your app does clever tricks where it has multiple static libraries implementing the same symbols, and depends on the command line order of the static libraries to drive which implementation is used, then this option is not for you.
    Because the linker will load all the implementations and not necessarily get the symbol semantics that were found in regular static linking mode.
    The other downside of -all_load is that it may make your program bigger because "unused" code is now being added in.
    To compensate for that, you can use the linker option -dead_strip.
    That option will cause the linker to remove unreachable code and data.
    Now, the dead stripping algorithm is fast and usually pays for itself by reducing the size of the output file.
    But if you are interested in using -all_load and -dead_strip, you should time the linker with and without those options to see if it is a win for your particular case.
    The next linker option is -no_exported_symbols.
    A little background here.
    One part of the LINKEDIT segment that the linker generates is an exports trie, which is a prefix tree that encodes all the exported symbol names, addresses, and flags.
    Whereas all dylibs need to have exported symbols, a main app binary usually does not need any exported symbols.
    That is, usually nothing ever looks up symbols in the main executable.
    If that is the case, you can use -no_exported_symbols for the app target to skip the creation of the trie data structure in LINKEDIT, which will improve link time.
    But if your app loads plugins which link back to the main executable, or you use xctest with your app as the host environment to run xctest bundles, your app must have all its exports, which means you cannot use -no_exported_symbols for that configuration.
    Now, it only makes sense to try to suppress the exports trie if it is large.
    You can run the dyld_info command shown here to count the number of exported symbols.
    One large app we saw had about one million exported symbols.
    And the linker took two to three seconds to build the exports trie for that many symbols.
    So adding -no_exported_symbols shaved two to three seconds off the link time of that app.
    I'll tell you more about the dyld_info tool later in this talk.
    The next option is: -no_deduplicate.
    A few years back we added a new pass to the linker to merge functions that have the same instructions but different names.
    It turns out, with C++ template expansions, you can get a lot of those.
    But this is an expensive algorithm.
    The linker has to recursively hash the instructions of every function, to help look for duplicates.
    Because of the expense, we limited the algorithm so the linker only looks at weak-def symbols.
    Those are the ones the C++ compiler emits for template expansions that were not inlined.
    Now, de-dup is a size optimization, and Debug builds are about fast builds, and not about size.
    So by default, Xcode disables the de-dup optimization by passing -no_deduplicate to the linker for Debug configurations.
    And clang will also pass the no-dedup option to the linker if you run clang link line with -O0.
    In summary, if you use C++ and have a custom build, that is, either you use a non-standard configuration in Xcode, or you use some other build system, you should ensure your debug builds add -no_deduplicate to improve link time.
    The options I just talked about are the actual command line arguments to ld.
    When using Xcode, you need to change your product build settings.
    Inside build settings, look for "Other Linker Flags".
    Here's what you would set for -all_load.
    And notice the "Dead Code Stripping" option is here as well.
    And there's -no_exported_symbols.
    And here's -no_deduplicate.
    
    ### Static linking surprises
    
    Now let's talk about some surprises you may experience when using static libraries.
    The first surprise is when you have source code that builds into a static library which your app links with, and that code does not end up in the final app.
    For instance, you added "attribute used" to some function, or you have an Objective-C category.
    Because of the selective loading the linker does, if those object files in the static library don't also define some symbol that is needed during the link, those object files won't get loaded by the linker.
    Another interesting interaction is static libraries and dead stripping.
    It turns out dead stripping can hide many static library issues.
    Normally, missing symbols or duplicate symbols will cause the linker to error out.
    But dead stripping causes the linker to run a reachability pass across all the code and data, starting from main, and if it turns out the missing symbol is from an unreachable code, the linker will suppress the missing symbol error.
    Similarly, if there are duplicate symbols from static libraries, the linker will pick the first and not error.
    The last big surprise with using static libraries, is when a static library is incorporated into multiple frameworks.
    Each of those frameworks runs fine in isolation, but then at some point, some app uses both frameworks, and boom, you get weird runtime issues because of the multiple definitions.
    The most common case you will see is the Objective-C runtime warning about multiple instances of the same class name.
    Overall, static libraries are powerful, but you need to understand them to avoid the pitfalls.
    That wraps up static linking.
    
    # Dynamic Linking
    
    Now, let's move on to dynamic linking.
    First, let's look at the original diagram for static linking with a static library.
    Now think about how this will scale over time, as there is more and more source code.
    It should be clear that as more and more libraries are made available, the end program may grow in size.
    That means the static link time to build that program will also increase over time.
    Now let's look at how these libraries are made.
    What if we did this switch? We change 'ar' to 'ld' and the output library is now an executable binary.
    This was the start of dynamic libraries in the '90s.
    As a shorthand, we call dynamic libraries "dylibs".
    On other platforms they are known as DSOs or DLLs.
    So what exactly is going on here? And how does that help the scalability? The key is that the static linker treats linking with a dynamic library differently.
    Instead of copying code out of the library into the final program, the linker just records a kind of promise.
    That is, it records the symbol name used from the dynamic library and what the library's path will be at runtime.
    How is this an advantage? It means your program file size is under your control.
    It just contains your code, and a list of dynamic libraries it needs at runtime.
    You no longer get copies of library code in your program.
    Your program's static link time is now proportional to the size of your code, and independent of the number of dylibs you link with.
    Also, the Virtual Memory system can now shine.
    When it sees the same dynamic library used in multiple processes, the Virtual Memory system will re-use the same physical pages of RAM for that dylib in all processes that use that dylib.
    I've shown you how dynamic libraries started and what problem they solve.
    But what are the "costs" for those "benefits"? First, a benefit of using dynamic libraries is that we have sped up build time.
    But the cost is that launching your app is now slower.
    This is because launching is no longer just loading one program file.
    Now all the dylibs also need to be loaded and connected together.
    In other words, you just deferred some of the linking costs from build time to launch time.
    Second, a dynamic library based program will have more dirty pages.
    In the static library case, the linker would co-locate all globals from all static libraries into the same DATA pages in the main executable.
    But with dylibs, each library has its DATA page.
    Lastly, another cost of dynamic linking is that it introduces the need for something new: a dynamic linker! Remember that promise that was recorded in the executable at build time? Now we need something at runtime that will fulfill that promise to load our library.
    That's what dyld, the dynamic linker, is for.
    Let's dive into how dynamic linking works at runtime.
    An executable binary is divided up into segments, usually at least TEXT, DATA, and LINKEDIT.
    Segments are always a multiple of the page size for the OS.
    Each segment has a different permission.
    For example, the TEXT segment has "execute" permissions.
    That means the CPU may treat the bytes on the page as machine code instructions.
    At runtime, dyld has to mmap() the executables into memory with each segments' permissions, as show here.
    Because the segments are page sized and page aligned, that makes it straightforward for the virtual memory system to just set up the program or dylib file as backing store for a VM range.
    That means nothing is loaded into RAM until there is some memory access on those pages, which triggers a page fault, which causes the VM system to read the proper subrange of the file and fill in the needed RAM page with its content.
    But just mapping is not enough.
    Somehow the program needs to be "wired up" or bound to the dylib.
    
    ### Dynamic linking, fixups
    
    **For that we have a concept called "fix ups".
    In the diagram, we see the program got pointers set up that point to the parts of the dylib it uses.
    Let's dive into what fix-ups are.
    Here is our friend, the mach-o file.
    Now, TEXT is immutable.
    And in fact, it has to be in a system based on code signing.
    So what if there is a function that calls malloc()? How can that work? The relative address of _malloc can't be known when the program was built.
    Well, what happens is, the static linker saw that malloc was in a dylib and transformed the call site.
    The call site becomes a call to a stub synthesized by the linker in the same TEXT segment, so the relative address is known at build time, which means the BL instruction can be correctly formed.
    How that helps is that the stub loads a pointer from DATA and jumps to that location.
    Now, no changes to TEXT are needed at runtime– just DATA is changed by dyld.
    In fact, the secret to understanding dyld is that all fixups done by dyld are just dyld setting a pointer in DATA.
    So let's dig more into the fixups that dyld does.
    Somewhere in LINKEDIT is the information dyld needs to drive what fixups are done.
    There are two kinds of fixups.
    The first are called rebases, and they are when a dylib or app has a pointer that points within itself.
    Now there is a security feature called ASLR, which causes dyld to load dylibs at random addresses.
    And that means those interior pointers cannot just be set at build time.
    Instead, dyld needs to adjust or "rebase" those pointers at launch.
    On disk, those pointers contain their target address, if the dylib were to be loaded at address zero.
    That way, all the LINKEDIT needs to record is the location of each rebase location.
    Dyld can then just add the actual load address of the dylib to each of the rebase locations to correctly fix them up.
    The second kind of fixups are binds.
    Binds are symbolic references.
    That is, their target is a symbol name and not a number.
    For instance, a pointer to the function "malloc".
    The string "_malloc" is actually stored in LINKEDIT, and dyld uses that string to look up the actual address of malloc in the exports trie of libSystem.
    dylib.
    Then, dyld stores that value in the location specified by the bind.
    This year we are announcing a new way to encode fixups, that we call "chained fixups".
    The first advantage is that is makes LINKEDIT smaller.
    The LINKEDIT is smaller because instead of storing all the fixup locations, the new format just stores where the first fixup location is in each DATA page, as well as a list of the imported symbols.
    Then the rest of the information is encoded in the DATA segment itself, in the place where the fixups will ultimately be set.
    This new format gets its name, chained fixups, from the fact that the fixup locations are "chained" together.
    The LINKEDIT just says where the first fixup was, then in the 64-bit pointer location in DATA, some of the bits contain the offset to the next fixup location.
    Also packed in there is a bit that says if the fixup is a bind or a rebase.
    If it is a bind, the rest of the bits are the index of the symbol.
    If it's a rebase, the rest of the bits are the offset of the target within the image.
    Lastly, runtime support for chained fixups already exists in iOS 13.
    4 and later.
    Which means you can start using this new format today, as long as your deployment target is iOS 13.
    4 or later.
    And the chained fixup format enables a new OS feature we are announcing this year.
    But to understand that, I need to talk about how dyld works.
    Dyld starts with the main executable– say your app.
    Parses that mach-o to find the dependent dylibs, that is, what promised dynamic libraries it needs.
    It finds those dylibs and mmap()s them.
    Then for each of those, it recurses and parses their mach-o structures, loading any additional dylibs as needed.
    Once everything is loaded, dyld looks up all the bind symbols needed and uses those addresses when doing fixups.
    Lastly, once all the fixups are done, dyld runs initializers, bottom up.**
    
    ## improvement in dynamic linking
    
    Five years ago we announced a new dyld technology.
    We realized the steps in green above were the same every time your app was launched.
    So as long as the program and dylibs did not change, all the steps in green could be cached on first launch and re-used on subsequent launches.
    This year we are announcing additional dyld performance improvements.
    We are announcing a new dyld feature called "page-in linking".
    Instead of dyld applying all the fixups to all dylibs at launch, the kernel can now apply fixups to your DATA pages lazily, on page-in.
    It has always been the case that the first use of some address in some page of an mmap()ed region triggered the kernel to read in that page.
    But now, if it is a DATA page, the kernel will also apply the fixup that page needs.
    We have had a special case of page-in linking for over a decade for OS dylibs in the dyld shared cache.
    This year we generalized it and made it available to everyone.
    This mechanism reduces dirty memory and launch time.
    It also means DATA_CONST pages are clean, which means they can be evicted and recreated just like TEXT pages, which reduces memory pressure.
    This page-in linking feature will be in the upcoming release of iOS, macOS, and watchOS.
    But page-in linking only works for binaries built with chained fixups.
    That is because with chained fixups, most of the fixup information will be encoded in the DATA segment on disk, which means it is available to the kernel during page-in.
    One caveat is that dyld only uses this mechanism during launch.
    Any dylibs dlopen()ed later do not get page-in linking.
    In that case, dyld takes the traditional path and applies the fixups during the dlopen call.
    With that in mind, let's go back to the dyld workflow diagram.
    For five years now, dyld has been optimizing the steps above in green by caching that work on first launch and reusing it on later launches.
    Now, dyld can optimize the "apply fixup" step by not actually doing the fixups, and letting the kernel do them lazily on page-in.
    Now that you have seen what's new in dyld, let's talk about best practices for dynamic linking.
    
    ## Best practices for dynamic linking
    
    What can you do to help improve dynamic link performance? As I just showed, dyld has already accelerated most of the steps in dynamic linking.
    One thing you can control is how many dylibs you have.
    The more dylibs there are, the more work dyld has to do to load them.
    Conversely, the fewer dylibs, the less work dyld has to perform.
    The next thing you can look at are static initializers, which is code that always runs, pre-main.
    For instance, don't do I/O or networking in a static initializer.
    Anything that can take more than a few milliseconds should never be done in an initializer.
    As we know, the world is getting more complicated, and your users want more functionality.
    So it makes sense to use libraries to manage all that functionality.
    Your goal is to find your sweet spot between dynamic and static libraries.
    Too many static libraries and your iterative build/debug cycle is slowed down.
    On the other hand, too many dynamic libraries and your launch time is slow and your customers notice.
    But we sped up ld64 this year, so your sweet spot may have changed, as you can now use more static libraries, or more source files directly in your app, and still build in the same amount of time.
    Lastly, if it works for your installed base, updating to a newer deployment target can enable the tools to generate chained fixups, making your binaries smaller, and improving launch time.
    The last thing I'd like you all to be aware of is two new tools that will help you peek inside the linking process.
    The first tool is dyld_usage.
    You can use it to get a trace of what dyld is doing.
    The tool is only on macOS, but you can use it to trace your app launching in the simulator, or if your app built for Mac Catalyst.
    Here is an example run against TextEdit on macOS.
    As you can tell by the top few lines, the launch took 15ms overall, but only 1ms for fixups, thanks to page-in linking.
    The vast majority of time is now spent in static initializers.
    The next tool is dyld_info.
    You can use it to inspect binaries, both on disk and in the current dyld cache.
    The tool has many options, but I'll show you how to view exports and fixups.
    Here the -fixup option shows all the fixup locations dyld will process and their targets.
    The output is the same regardless of if the file is old style fixups or new chained fixups.
    Here the -exports option will show all the exported symbols in the dylib, and the offset of each symbol from the start of the dylib.
    In this case, it is showing information about Foundation.
    framework which is the dylib in the dyld cache.
    There is no file on disk, but the dyld_info tool uses the same code as dyld and can thus find it.
    Now that you understand the history and tradeoffs of static versus dynamic libraries, you should review what you app does and determine if you have found your sweet spot.
    Next, if you have a large app and have noticed the build takes a while to link, try out Xcode 14 which has the new faster linker.
    If you still want to speed up your static link more, look into the three linker options I detailed and see if they make sense in your build, and improve your link time.
    Lastly, you can also try building your app, and any embedded frameworks, for iOS 13.
    4 or later to enable chained fixups.
    Then see if your app is smaller and launches faster on iOS 16.
    Thanks for watching, and have a great WWDC.
    

# 들어가기에 앞서

**1. 링커가 무엇인가요?**

우리가 작성한 코드가 라이브러리를 사용하려면 링커가 필요합니당
링커에는 두 가지 타입이 있어용

**2. 링커의 타입들**

1. static linking (정적 링크)
앱을 구축하는데 걸리는 시간과 앱의 크기에 영향을 줄 수 있음.
2. dynamic linking (동적 링크) 
유저들이 앱 시작을 기다리는데 걸리는 시간에 영향을 줄 수 있음

---

# ✅ Static linking

# What is static linking?

![스크린샷 2025-04-09 오전 12.50.16.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.50.16.png)

하나 이상의 소스 파일이 있을 때 이걸 어떻게 프로그램으로 만들지?

1. 먼저 앞부분은 소스 파일을 relocatable object로 컴파일해요(.o 파일)
2. .o 파일들을 하나의 프로그램으로 한 번 더 컴파일해요
    1. 이때 이 두 번째 과정을 ld라고 부릅니다. (static linker)

![스크린샷 2025-04-09 오전 12.52.53.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.52.53.png)

근데 .o파일이 많아지면 귀찮으니까 라이브러리로 묶어버리면 어때 ?! 라는 의견이 나왔어용

![.o 파일들을 ar화 해서 아카이브로 만듦. 이게 지금의 정적 라이브러리.](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_12.54.32.png)

.o 파일들을 ar화 해서 아카이브로 만듦. 이게 지금의 정적 라이브러리.

프로그램이 점점 커지다보니 최적화를 위해
링커는 정적 라이브러리의 .o 파일을 모두 사용하지 않고
**정의되지 않은 심볼 문제가 해결되는 경우에만** .o 파일을 사용하기로 함.

![이렇게 4개의 파일이 있어요.
main.c → foo 함수 호출, foo.c → bar 호출, bar.c → bar 및 unused 구현, baz.c → undef 함수 호출하는 baz 함수 정의](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.00.01.png)

이렇게 4개의 파일이 있어요.
main.c → foo 함수 호출, foo.c → bar 호출, bar.c → bar 및 unused 구현, baz.c → undef 함수 호출하는 baz 함수 정의

![.o 파일로 컴파일 완!
정의되지 않은 foo, bar, undef에는 회색 박스가 없음.](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.00.14.png)

.o 파일로 컴파일 완!
정의되지 않은 foo, bar, undef에는 회색 박스가 없음.

![bar.o, bar.z를 하나의 정적 라이브러리로 만들고 나머지 두 .o파일들을 이 라이브러리와 링킹할 때 어떤 일이 일어날까?](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.01.07.png)

bar.o, bar.z를 하나의 정적 라이브러리로 만들고 나머지 두 .o파일들을 이 라이브러리와 링킹할 때 어떤 일이 일어날까?

1. main.o를 찾음 → 파일을 로드함 → main에 대한 정의를 찾음
    
    ![스크린샷 2025-04-09 오전 1.03.45.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.03.45.png)
    
2. 명령어 라인의 다음 파일인 foo.o을 찾아서 파싱함 → 로드되면서 foo의 정의까지 한 번에 같이 로드~ → 하지만 정의되지 않은 심볼인 bar도 추가됨
    
    ![스크린샷 2025-04-09 오전 1.05.20.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.05.20.png)
    
3. 명령어 라인의 모든 .o 파일은 로드됐기 때문에 정의되지 않은 심볼이 있는지 확인 → bar는 아직 정의 안됨 → 명령어 라인의 라이브러리를 보면서 이 심볼을 충족하는 라이브러리를 찾음
4. bar.o가 충족시키는걸 알아서 bar.o를 로드함
5. 라이브러리 수색 중단 후 프로그램에 포함될 모든 데이터와 메서드에 주소를 할당함
    
    ![스크린샷 2025-04-09 오전 1.19.32.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.19.32.png)
    
6. 이 모든 함수와 데이터를 출력 파일에 복사함 → 출력 프로그램 완성~~
    
    ![스크린샷 2025-04-09 오전 1.19.43.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.19.43.png)
    

<aside>
⚠️

이때 baz.o는 로드되지 않은 것을 볼 수 있음!! 링커가 정적 라이브러리에서 선택적으로 bar.o만 로드했기 때문에~~(bar.o에 있는 심볼만 필요해서!)

➡️ 이렇게 선택적으로 로드하는 것이 **정적 라이브러리**의 특징!

</aside>

# Recent ld64(static linker) improvements

최적화 작업을 하셨다네요, 2ㅂㅐ로 빨라졌대요 → 어떻게? **코어를 활용해서!**

![링크 관련 태스크들을 병렬로 처리하고자 함 + 알고리즘도 개선.
이게 14였으니까 뭔가 더 빨라졌으려나…?](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.22.45.png)

링크 관련 태스크들을 병렬로 처리하고자 함 + 알고리즘도 개선.
이게 14였으니까 뭔가 더 빨라졌으려나…?

# Static linking best practices

> 어떻게 하면 static linking에서 최고의 성능을 낼 수 있을까?(5가지)
> 
- when to use static libraries(적절한 곳에서 정적 라이브러리를 사용하기)
    - 코드가 변경되면 static library 전체가 모두 rebuild돼야함
    - 그렇기 때문에 static library 코드에 막 작업을 많이 하면 추가적인 I/O로 인해 빌드 시간이 느려짐
    - **그래서 정적 라이브러리는 안정적인 코드에 적합하다~**
- 링크 시간에 큰 영향을 끼치는 옵션 3가지
    - 정적 라이브러리의 선택적 로딩은 링크 시간이 느려지게 함. 왜?
    빌드를 재현가능하게 하고 전통적인 정적 라이브러리의 의미를 따르기 위해서는
    링커가 라이브러리를 고정된 직렬 순서로 처리해야하기 때문임.
        - 빌드를 재현가능하게 해야하는 이유
            
            링커가 정적 라이브러리를 고정된 순서로 처리하는 이유는 바로 이 재현성을 보장하기 위해서입니다. 만약 링커가 매번 다른 순서로 라이브러리를 처리한다면, 동일한 소스 코드로 빌드해도 결과물이 조금씩 달라질 수 있습니다. 특히 심볼 해결(symbol resolution) 과정에서 동일한 심볼이 여러 라이브러리에 정의되어 있을 경우, 처리 순서에 따라 어떤 라이브러리의 심볼이 최종적으로 선택될지 달라질 수 있기 때문입니다.
            
        - **이러한 선택적 로드가 필요가 없다면 링커 옵션을 수정해서 속도를 개선할 수 있음**
    - `-all_load` or `-force_load`
        
        ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/ee4af152-5099-4b16-bbf5-31c17e439fda.png)
        
        - **링커로 하여금 정적라이브러리의 모든 .o 파일들을 로드하게함**
        - 결국 앱이 정적 라이브러리의 모든 코드를 사용하게 되는 경우라면 이 옵션도 ㄱㅊ
        - 하지만 여러 정적 라이브러리들이 동일한 심볼을 정의해두고 (명령어 라인의 정적 라이브러리 순서에 의존해 어떤 구현을 사용할지 정해서) 사용하고 있다면 duplicated symbol 오류가 날 수도 있어서 적합한 옵션이 아님
        - 프로그램이 무거워질 수도 있다는 단점이 있음(사용되지 않는 코드의 추가)
            - 그럼 여기서 `-dead_strip`이라는 옵션을 추가하면 연결한 불가한 코드와 데이터를 링커가 제거함
            - -all_load와 -dead_strip 모두 사용할거면 어떤 경우가 본인 상황에 유리한지 시간 측정이 필요함
    - `-no_exported_symbols`
        - 링커가 생성하는 LINKEDIT 세그먼트의 한 부분은 하나의 **exports-trie**로 접두사 트리임
        내보낸 모든 심볼 이름, 주소 및 플래그를 인코딩함
            
            ![스크린샷 2025-04-09 오전 1.19.43.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%8C%E1%85%A5%E1%86%AB_1.19.43.png)
            
        
        **LINKEDIT 세그먼트란**
        
        - **링커가 생성하는 실행 파일의 한 부분으로, 다양한 링킹 관련 정보를 포함합니다.**
        - 이 중 하나가 **"exports trie"**입니다.
        
        **exports trie의 특징**
        
        - 프리픽스 트리(prefix tree) 구조를 사용합니다.
            - 유사한 이름을 가진 심볼들이 공통 접두사를 공유하므로 공간을 절약할 수 있습니다.
        - **바이너리에서 외부로 내보내는(export) 모든 심볼(함수, 변수 등)에 대한 정보를 저장**
        - **이 정보는 다른 바이너리(다이나믹 라이브러리나 애플리케이션)가 해당 심볼을 참조할 때 필요합니다.**
        - 링커가 기본적으로는 심볼 내보내기가 없더라도 exports trie 구조 자체는 생성한다는 것입니다. 이는 일종의 '빈' 데이터 구조일 수 있지만, 그래도 LINKEDIT 세그먼트에 포함됩니다.
        - 하지만 `no_exported_symbols` 옵션을 사용하면 아예 구조 자체도 안만들어지기 때문에 시간&용량이 더 단축된다~
        
        **다이나믹 라이브러리(dylibs)와 메인 앱 바이너리의 차이**
        
        - 다이나믹 라이브러리: 반드시 내보내진 심볼이 필요합니다.
        - 메인 앱 바이너리: 일반적으로 내보내진 심볼이 필요하지 않습니다.
        
        **메인 앱에서 내보내진 심볼이 필요 없는 이유**
        
        - 보통 다른 코드가 메인 실행 파일에서 심볼을 찾아 사용하지 않기 때문입니다.
        
        > **결론: 메인 애플리케이션이 자신의 심볼을 외부로 내보낼 필요가 없다면, `-no_exported_symbols` 옵션을 사용하여 이 trie 구조 생성을 건너뛰어 링크 시간을 단축할 수 있습니다.**
        > 
        - 하지만 만약 앱이 메인 실행파일로 다시 연결되는 플러그인을 로드하거나 호스트로 xctest를 사용하면 앱에 모든 exported symbol이 있어야함 → 이때는 이 옵션 사용하기 어려움
            - **플러그인(plugins)이 메인 실행 파일에 링크백(link back)하는 경우**:
                - 플러그인은 별도로 컴파일된 동적 라이브러리/번들로, 애플리케이션 실행 중에 로드됩니다.
                - 플러그인이 메인 애플리케이션의 함수나 변수를 사용해야 할 경우가 있습니다.
                - 예를 들어:
                    - 애플리케이션이 제공하는 API를 플러그인이 호출해야 할 때
                    - 애플리케이션의 데이터 구조나 상태에 접근해야 할 때
                - 이때 플러그인은 메인 실행 파일에서 해당 심볼을 찾아야 하므로, 메인 실행 파일은 이러한 심볼을 내보내야(export) 합니다.
            - **xctest로 앱을 호스트 환경으로 사용하는 경우**:
                - xctest는 Apple의 테스트 프레임워크입니다.
                - 앱을 호스트 환경으로 사용한다는 것은, 테스트 코드가 실제 앱 내에서 실행된다는 의미입니다.
                - 테스트 코드(xctest 번들)는 앱의 내부 요소들을 테스트해야 하므로:
                    - 앱의 클래스, 함수, 변수 등에 접근해야 합니다.
                    - 이들 요소를 테스트하기 위해 메인 앱에서 이러한 심볼을 내보내야 합니다.
        
        ```jsx
        dyld_info -exports /path/to/binary | wc -l
        ```
        
        이 코드를 사용하면 몇 개의 심볼이 내보내졌는지 볼 수 있음
        
    - `-no_deduplicate`
        - 이름은 다르지만 명령은 동일한 함수들의 병합을 위한 패스가 추가됨.
        - 하지만 이런 알고리즘은 비쌈 중복찾으려면 모든 함수를 다 뒤져야함
        - 그래서 링커가 약한 정의 심볼까지만 찾아볼 수 있도록 알고리즘이 제한돼있음
        - **de-dup 중복 제거는 사이즈 최적화임. 하지만 디버그 빌드에서 중요한 점은 사이즈가 아닌 속도 빠른 빌드에 관한 것.**
        - **그래서 Xcode는 기본적으로 디버그 구성을 위해 링커에 `-no_dedeuplicate`를 전달해 de-dup 최적화를 비활성화함(중복을 허용)**
        - clang 또한 no-dedup 옵션을 링커에 전달함, 링크 라인이 -O0로 설정ㄷ
- 정적 라이브러리 사용 시 발생할 수 있는 놀라운 점들
    - **소스 코드가 정적 라이브러리로 빌드되고 앱이 이를 링크하지만, 해당 코드가 최종 앱에 포함되지 않을 수 있음**
        - **링커의 선택적 로딩 기능 때문 (+ 옵젝씨의 Dynamic dispatch 때문에)**
        - `attribute used`를 함수에 추가한 경우 / Objective-C 카테고리를 사용한 경우
        - 정적 라이브러리의 오브젝트 파일이 링크 도중 필요한 심볼을 정의하지 않으면(정적 라이브러리의 메코드가 호출되지 않으면), 링커가 해당 오브젝트 파일을 로드하지 않음
            
            ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image.png)
            
    - dead stripping
        - **사용되지 않는 코드를 제거하는 최적화 기술인데 이 과정에서 정적 라이브러리의 문제가 숨겨지는 문제가 발생할 수 있음(missing symbols, duplicated symbols)**
        - 링커가 main부터 시작해 모든 코드와 데이터에 도달 가능성(reachability) 검사를 수행
        - 누락된 심볼이 도달 불가능한 코드에 있으면 에러를 표시하지 않음
        - 정적 라이브러리에서 중복 심볼이 있을 경우 첫 번째 것을 선택하고 에러를 표시하지 않음
    - 하나의 정적 라이브러리가 여러 프레임워크에 통합되는 경우
        - **각 프레임워크는 개별적으로 잘 작동하지만, 앱이 두 프레임워크를 모두 사용할 경우 런타임 문제 발생**
        - Objective-C 런타임에러(동일한 클래스 명의 여러 인스턴스에 대한 경고), Duplicate symbols at runtime가 발생할 수 있음

# ✅  Dynamic linking

# What is dynamic linking?

![정적라이브러리](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.47.18.png)

정적라이브러리

![스크린샷 2025-04-09 오후 3.47.37.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.47.37.png)

- ar → ld로 변경
- output도 executable library로 변경 (dynamic library, dylibs)

➡️ dynamic library의 시작

## 정적 라이브러리와 뭐가 다르지?

![스크린샷 2025-04-09 오후 3.49.22.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.49.22.png)

static linker는 동적 라이브러리와의 링킹을 다르게 취급함.
**프로그램 내부에 코드 복사 x. 일종의 약속을 기록.
→ 여기에는 동적 라이브러리에서 사용되는 심볼, 런타임 시 라이브러리의 경로 등이 포함됨.**

## 이것이 해결한 문제점들(장점)

- **프로그램 파일 사이즈가 줄어듦** ! (우리가 작성한 코드 + 런타임에 필요한 동적 라이브러리 목록만 포함)
- (가상 메모리) 여러 프로세스에서 동일한 동적 라이브러리가 사용되는 경우 **동일한 물리적 RAM 페이지를 재사용할 수 있음**

## 이걸 사용하면 드는 비용(단점)

- **이점: 빌드 시간 단축 → 비용: 앱 시작 속도의 둔화**
    - dylib들을 모두 로드하고 연결 후 앱이 시작할 수 있기 때문
- **more dirty pages(메모리에서 변경된 페이지)**
    - 링커는 모든 정적 라이브러리의 전역 변수들을 메인 실행파일의 동일한 DATA페이지에 함께 배치함
        - 즉 정적 라이브러리는 전역 데이터가 하나의 메모리에 집중됨
    - 하지만 동적 라이브러리는 모든 라이브러리 별로 DATA 페이지가 있음.
- **dynamic linker가 필요함**
    - 이전에 일종의 약속을 기록한다고 했는데 이걸 실행할 무언가가 필요함
    - 이것이 dyld 동적 링커의 용도

## 동적 링커 dyld

> **동적 링커**(dynamic linker) 는 라이브러리 내용을 영구적 기억 장치에서 RAM 으로 복사하고 점프 테이블을 채우고 포인터의 위치를 재조정함으로써 런타임 중에 실행파일에 필요한 공유 라이브러리를 로드하고 링크함
> 
> 
> ![스크린샷 2025-04-09 오후 3.55.33.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.55.33.png)
> 

### 이 동적 링킹은 어떻게 작동할까?

- executable binary는 세그먼트로 분할됨.
    
    ![스크린샷 2025-04-09 오후 3.57.08.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.57.08.png)
    
- **이때 각 세그먼트는 각자 다른 권한을 지님**. ex. TEXT → 실행 (하여튼 이렇게 각 세그먼트별로 권한이 다 다르다는 것만 알고 있으면 될듯)
이는 CPU가 페이지의 바이트를 machine code instructions 으로 처리할 수 있다는 것을 의미합니다.
    - 뭥미?? 무슨 소리임?
    - TEXT는 프로그램의 실행 가능한 기계어 명령어(machine code instructions)들이 저장되는 메모리 영역입니다.
    - "execute" 권한은 CPU가 이 영역의 바이트들을 실제 프로그램 명령어로 해석하고 실행할 수 있음을 의미합니다.
- **런타임시 동적 링커는 각 세그먼트의 권한을 기반으로 실행 파일들을 메모리에 매핑(mmap)해야합니다. 가상 메모리에 이를 매핑해놓고 실질적인 접근이 있을 때 실질적인 메모리에 로드되게 됩니다.**
- 그럼 런타임에 dyld가 각 세그먼트의 권한을 받아서 실행 파일들을 메모리에 mmap() 해야함(dyld가 실행 파일의 세그먼트들을 메모리에 로드한다는 뜻)
    - mmap() 이란: 파일이나 디바이스를 프로세스의 가상 메모리 공간에 매핑하는 함수, 파일의 내용을 메모리에 직접 연결함
    
    ![Because the segments are page sized and page aligned, that makes it straightforward for the virtual memory system to just set up the program or dylib file as backing store for a VM range.
    세그먼트들은 페이지를 기준으로 크기 설정 및 정렬되기 때문에 (page sized and page aligned), 가상 메모리 시스템이 **프로그램**이나 **dylib 파일**을 **VM 범위**의 **backing store 에 설정하는것이 간단**합니다.](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_3.59.19.png)
    
    Because the segments are page sized and page aligned, that makes it straightforward for the virtual memory system to just set up the program or dylib file as backing store for a VM range.
    세그먼트들은 페이지를 기준으로 크기 설정 및 정렬되기 때문에 (page sized and page aligned), 가상 메모리 시스템이 **프로그램**이나 **dylib 파일**을 **VM 범위**의 **backing store 에 설정하는것이 간단**합니다.
    
- 해당 페이지에 메모리 액세스가 있을 때까지 RAM에 아무것도 로드되지 않음. 이로 인해 페이지 폴트가 발생. 그러면 VM 시스템이 파일의 적절한 하위 범위를 읽고 그 콘텐츠로 필요한 RAM 페이지를 채우도록 만듦
    
    ![스크린샷 2025-04-09 오후 5.51.31.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_5.51.31.png)
    
    - 프로그램 A를 실행할 때 처음에는 아무 것도 RAM에 로드되지 않습니다.
    - 프로그램 A의 특정 부분을 처음 사용하려고 하면 그제서야 해당 페이지가 RAM에 로드됩니다.
    - 사용하지 않는 코드나 라이브러리 부분은 계속 디스크에 남아있게 됩니다.
- **하지만 프로그램은 이렇게 매핑만으로 끝나느게 아니라 연결되거나 dylib에 바인딩돼야함.
➡️ 연결 과정이 필요함. (프로그램에서 동적 라이브러리의 실질적 메모리 주소를 가리킬 수 있도록 하는 과정)**

## fixup

- 프로그램의 포인터들은 최종적으로 각 프로그램이 사용하는 dylib 부분들을 향하고 있음
    
    ![스크린샷 2025-04-09 오후 4.01.43.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_4.01.43.png)
    

<aside>
💡

C 표준 라이브러리(동적 라이브러리)에 위치한 malloc()을 호출하는 케이스를 예시로 'fixup' 과정에 대해 알아봅시다.

</aside>

- 자 먼저 mach-o 파일이 있어요.
    - mach-o 파일이란 Apple 운영체제의 기본 실행 파일 형식임. 앱 빌드 시 컴파일러가 소스코드를 Mach-O 형식의 파일로 변환함
    
    ![스크린샷 2025-04-09 오후 5.56.47.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2025-04-09_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_5.56.47.png)
    
- 만약 malloc()을 호출하는 함수가 있다면?
    - malloc() 이란 동적 메모리 할당을 위해 사용되는 C 함수. 힙 메모리 영역에서 사용자가 요청한 크기만큼의 메모리 공간을 할당함. (런타임에 확보)
    - **동적링커(dyld)는 DATA 세그먼트에 있는 포인터를 업데이트하여 malloc 함수의 실제 메모리 주소를 기록해놓습니다.**
    - **_malloc의 상대 주소는 빌드 시 알 수 없음.**
    
    ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%201.png)
    
- **여기서 실제로 발생하는 일은, static linker(ld) 가 malloc이 dylib 에 있는 것을 보고, 컴파일 시 call site(호출 지점)을 특별한 “stub” 코드(일종의 임시 코드 조각)로 변환하는 것입니다.**
- **stub은 간접적으로 함수의 실제 주소를 찾아가는 방법을 제공합니다.**
    
    ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%202.png)
    
- **`BL`(Branch with Link) 명령어를 사용해 스텁을 호출**
- **stub은 `DATA` 세그먼트에 있는 포인터를 로드합니다.**
- **이 포인터는 실제 `malloc()` 함수의 주소를 가리킵니다. DATA 세그먼트에 저장되어 있는 포인트를 로드하여 malloc 함수의 위치로 점프합니다.**
- **이렇게 런타임에서는 `Text` 를 변경할 필요가 없으며 dyld에 의해 `DATA` 만 변경됩니다.**

---

1. 동적링커(dyld)는 DATA 세그먼트에 있는 포인터를 업데이트하여 malloc 함수의 실제 메모리 주소를 기록해놓습니다.
2. malloc 함수의 상대주소는 프로그램이 빌드될 때 알 수 없습니다.
3. 따라서 정적 링커는 컴파일시 malloc에 해당하는 부분을 '스텁'이라는 일종의 임시 코드 조각으로 대체해 놓습니다.
4. 그리고 링커는 'bl' 명령어를 통해 스텁을 호출합니다.
5. 스텁은 동적 링커와 상호작용하여 malloc의 실제 주소를 찾아냅니다.
6. DATA 세그먼트에 저장되어 있는 포인트를 로드하여 malloc 함수의 위치로 점프합니다. 여기서 확인할 부분은 런타임에서 TEXT 세그먼트는 변경되지 않습니다. 변경되는 부분은 단지 동적 링커에 의한 DATA에 대한 포인터 설정 뿐입니다.

> 가장 중요한 것은 
dyld에 의한 모든 fixup은 그저 DATA에 대한 포인터 설정뿐이라는 것
> 

## fixup의 종류

fixup에는 두 가지 타입이 있고
두가지 타입 모두 수행해야 프로그램 혹은 라이브러리가 정상적으로 동작할 수 있습니다.

리베이스(Rebase)와 바인드(Binding) 모두 **프로그램이나 라이브러리가 메모리에 올바르게 로드되고, 실행 중에 필요한 모든 코드와 데이터에 정확하게 접근할 수 있도록 하는 필수적인 과정입니다.**

### rebase

> dylib 또는 앱이 **자체적으로 내부를 가리키는 포인터**를 갖는 경우에 필요합니다.
앞선 예시에서 DATA 세그먼트에 있는 포인터를 업데이트하는 과정
> 

![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%203.png)

- dyld 는 ASLR 을 통해 임의의 주소에서 dylib을 로드하기 때문에 내부 포인터는 빌드 시에 설정할 수 없습니다.
    - ASLR: 실행 파일이나 라이브러리를 메모리에 로드할 때마다 그 위치를 무작위로 변경하는 보안 기능
- **빌드 시점**:
    - 모든 내부 포인터는 마치 프로그램이 주소 0에 로드된다고 가정하고 설정됩니다.
    - 즉, 포인터 값은 "대상의 위치가 시작점으로부터 얼마나 떨어져 있는지"를 의미합니다.
- **실행 시점**:
    - dyld(동적 링커/로더)가 실제로 프로그램을 메모리의 어느 위치에 로드할지 결정합니다. dyld는 LINKEDIT 세그먼트의 정보를 참조하여, 각 내부 포인터가 현재의 실제 메모리 위치를 올바르게 가리키도록 조정합니다.
    - 그런 다음 모든 내부 포인터에 "실제 로드 주소"를 더합니다.
    - 이 과정을 "rebase"라고 합니다.
- **최적화**:
    - `LINKEDIT` 섹션에는 조정이 필요한 포인터의 위치만 기록됩니다. 포인터들이 원래 가리키려고 했던 대상 주소(예를 들어, dylib가 메모리 주소 0에서 시작한다고 가정했을 때의 주소)를 포함합니다.
    - 실제 조정 작업은 dyld가 프로그램을 로드할 때 수행합니다.

### bind

![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%204.png)

- symbolic reference (symbol을 기준으로 삼음)
프로그램 코드가 특정 메모리 주소를 직접 참조하는 대신, 심볼을 사용하여 참조하는 방식입니다.
- LINKEDIT 세그먼트에는 심볼의 이름이(ex. _malloc) 저장됩니다.
동적 링커(dyld)는 실행 시 이 정보를 참조하여 심볼의 실제 메모리 주소를 찾아냅니다. 예를 들어 'libSystem.dylib'의 exports trie에서 '_malloc' 심볼의 실제 메모리 주소를 검색합니다
- dyld가 심볼의 실제 메모리 주소를 찾으면, 그 주소를 프로그램이나 라이브러리 내에서 해당 심볼을 참조하는 위치에 연결(바인드)합니다.

### chained fixup(새로운 수정방법)

- 참고
    
    ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%205.png)
    
    - 기존에는 수정이 필요한 모든 위치의 정보를 LINKEDIT 세그먼트에 저장해야 했습니다. 하지만 'chained fixups' 방식에서는 **각 DATA 페이지의 첫 번째 수정 위치와 해당 페이지에서 필요한 심볼 목록만을 LINKEDIT에 저장**합니다. 이로 인해 LINKEDIT 세그먼트의 크기가 줄어듭니다.
    - 나머지 수정 정보는 DATA 세그먼트 자체에 직접 인코딩됩니다. 이는 수정이 최종적으로 적용될 메모리 위치에 대한 정보를 포함합니다.
    - **'Chained fixups'의 핵심은 수정 위치가 서로 연결되어 있다는 것입니다. LINKEDIT는 첫 번째 수정 위치만 알려주고, DATA 세그먼트 내의 각 64비트 포인터에는 다음 수정 위치로의 오프셋이 포함됩니다.**
    - DATA 세그먼트에 저장된 정보는 또한 해당 수정이 바인드인지 리베이스인지를 구분합니다. 이는 포인터의 일부 비트를 사용하여 표시됩니다. 바인드인 경우 나머지 비트는 필요한 심볼의 인덱스를 나타냅니다. 리베이스인 경우 나머지 비트는 해당 이미지 내의 대상 주소로의 오프셋을 나타냅니다.
    - 이 방식은 **수정 정보를 보다 효율적으로 저장하고 처리할 수 있게 해주어, 동적 링킹 과정을 최적화합니다.** 결과적으로 LINKEDIT 세그먼트의 크기 감소되고 수정 위치가 효율적으로 인코딩되어 프로그램 로드 시간을 단축시키고 전반적인 시스템 성능을 향상시킵니다.

# Recent dyld improvements

- 참고
    
    ## 기존 처리 방식
    
    ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%206.png)
    
    - **parse mach-o** : dyld는 애플리케이션의 실행 파일을 분석하여 시작합니다. (cf. Mach-O는 macOS와 iOS에서 사용되는 실행 파일 포맷입니다.)
    - **find dependent dylibs** : 애플리케이션 실행에 필요한 다른 dylib(동적 라이브러리)들의 목록을 확인합니다.
    - **mmap segments** : mmap() 시스템 호출을 사용하여 필요한 세그먼트(코드, 데이터 등)를 메모리에 매핑합니다.
    - **lookup symbols** : 애플리케이션에 필요한 심볼(함수나 변수 등)의 주소를 찾아냅니다.
    - **apply fixups** : 리베이스와 바인드 과정을 수행하여, 심볼들이 올바른 메모리 주소를 가리키도록 만듭니다.
    - **run initializers** : 애플리케이션과 라이브러리 내부에 정의된 초기화 코드를 실행합니다.
    
    2017년에 애플은 이중 parse mach-o, find dependent dylibs, lookup symbols 과정을 캐싱하여 첫번째 실행이후엔 재사용될 수 있도록 처리하여 개선했습니다.
    
    ## **page-in linking**
    
    ![image.png](Link%20fast%20Improve%20build%20and%20launch%20times%201caaba3750d780deb28cc4ebe5b4e0c0/image%207.png)
    
    이번에 또 새로운 개선이 있었습니다. dyld의 속성인 '페이지 인 링킹(page-in linking)'입니다. **페이지 인 링킹은 메모리에 로드된 애플리케이션의 데이터 세그먼트에 필요한 수정 사항들을 게으르게(즉, 필요한 시점에) 적용하는 기능입니다.** 따라서 그림에서는 파란색(apply fixups) 단계의 개선에 해당하겠죠.
    
    - **전통적으로, dyld는 애플리케이션을 실행할 때 필요한 모든 바인드(Binding)와 리베이스(Rebase) 수정 사항을 즉시 적용했습니다. 하지만 'page-in linking' 기능을 사용하면, 이런 수정 사항을 모든 dylib에 미리 적용하는 대신, 실제로 해당 데이터 페이지에 접근할 때까지 기다렸다가 수정을 적용합니다.**
    - 애플리케이션과 라이브러리의 세그먼트들은 mmap() 시스템 호출을 통해 메모리에 매핑됩니다. 사용자가 특정 페이지에 처음 접근할 때 페이지 폴트가 발생하고, 이 때 커널이 필요한 페이지를 메모리로 읽어들입니다.
    - 'page-in linking'에서는 데이터 페이지가 메모리로 로드될 때 커널이 페이지에 필요한 수정을 적용합니다. 이것은 이전에는 dyld가 수행했던 작업을 커널이 대신하는 것을 의미하며, 메모리 사용을 최적화하고 실행 시간을 단축시키는 데 기여합니다. (동적 링커는 앱 최초 실행시에 동작하기 때문에 이 작업을 수행할 수 없습니다.)
    - 런타임 중 동작하는 이러한 동작은 사용자가 인지할 수 없을 정도로 짧기 때문에 사용성에 악영향을 주지도 않습니다.
    - **'**page-in linking'을 사용함으로써 데이터 페이지(DATA_CONST)는 변경되지 않은 상태(즉, "깨끗한" 상태)를 유지하게 됩니다. 이는 페이지를 메모리 압력이 있을 때 추출하고 필요할 때 다시 가져오는(TEXT 페이지와 유사한) 방식으로 처리할 수 있다는 것을 의미합니다.
    - **'page-in linking'은 'chained fixups'(연결 수정)을 사용하여 빌드된 바이너리에서만 사용할 수 있습니다. 'chained fixups'는 대부분의 수정 정보를 디스크의 데이터 세그먼트에 저장하기 때문에, 이 정보를 페이지-인 링크 과정에서 쉽게 사용할 수 있습니다.**
    - 주의할점은 런타임 도중 dlopen()을 통해 로드된 dylib들은 이 매커니즘을 활용하지 못합니다. (dlopen : 런타임 중 동적라이브러리를 메모리에 로드하도록 도와주는 UNIX 기반 시스템의 동적 링킹 라이브러리(Dynamic Linking Library, DLL)에서 제공하는 함수 중 하나)

# Dynamic linking best practices

- 동적 링킹은 이미 개선이 많이 됐음. 우리가 이제 성능을 위해 제어할 수 있는 것 → dylib의 개수 (dylib이 적어야 dyld가 수행해야하는 작업이 줄어들기 때문)
- 정적 이니셜라이저
    - main 이전에 실행되는 코드
    - 여기서 I/O 또는 네트워킹을 수행하면 안됨
- static을 사용할지, dynamic을 사용할지 적절하게 선택하는 것이 중요!
    - ld64의 속도가 빨라졌기 때문에 static 더 쓰는것도 ㄱㅊ을지도?
- chained fixup 사용해보기~

# New tools(2개)

> 바이너리에 무엇이 있는지, 동적 링크 중에 무슨 일이 일어나고 있는지 볼 수 있음
> 
1. dyld_usage: dyld가 하는 일 추적 가능(시뮬레이터면 가능)
2. dyld_info: mach-o 파일들과 dyld 캐쉬에 있는 dylibs들에 대한 정보를 볼 수 있음

---

https://inuplace.tistory.com/1393