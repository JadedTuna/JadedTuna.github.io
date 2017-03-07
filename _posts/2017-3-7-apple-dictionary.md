---
layout: post
title: Progress on accessing data from Apple Dictionaries
tag: apple-dictionary reverse-engineering trie
---
This blog post is an update to my adventures on attempting to read data from Apple Dictionaries. I was finally able to find the function responsible for reading index values from the index file. The bad news is the code is pretty bad looking (pseudocode from IDA Pro) and it doesn't have any real variable/function names so one ends up having to guess from the context.

The file I was working with is `DictionaryServices`, found in `/System/Library/Frameworks/CoreServices.framework/Versions/A/Frameworks/DictionaryServices.framework`. It is a shared library that includes a whole lot of functions and data structures, but only three functions can be used by the programmer - `DCSGetTermRangeInString`, `DCSCopyTextDefinition` and `HIDictionaryWindowShow`, which allow you to either get the definition for a specific word or pop up dictionary window. I started working with `DCSCopyTextDefinition`, which apparently searches for query string in all the active dictionaries. It calls an undocumented function of `DCSDictionary`, `searchByString`, which accepts four arguments - some dictionary structure, the query string and two magic numbers. `searchByString` disappoints as it calls some internal function that does not even have a name - probably a function pointer. Here is the pseudocode:
<pre><code>
...
if (DCSDictionary::createDictionaryObj(this, stringToSearch))
{
    (*(**(this + 1) + 312LL))();
    result = (*(**(this + 1) + 256LL))(*(this + 1), stringToSearch, v5, v4);
}
...
return result;
</pre><code>
Marvelous.

I decided to explore `DCSDictionary` and `DCSEvironment` instead and learned quite a lot of stuff about them. `DCSEnvironment` manages loading dictionaries, stores active dictionaries, etc. and store metadata for them, such as paths. While I did learn a lot of stuff about how they are managed, I didn't find the code used for loading stuff from the dictionary until later. Apparently there are several ways to access dictionary data, depending on it's type. Thus `Body.data` is accessed using `HeapAccessMethod`, `KeyData.index` and `KeyData.data` using `TrieAccessMethod`. A big thanks here to `josephg`, since his research saved me from trouble of trying to guess the data type in those index files.

Alright, so we know which classes manage the data, should be easy now! But that's where the problems start. As aforementioned, the code is really garbled, there are almost no normal functions calls, and lots, LOTS of pointers. Pseudocode for the function responsible for reading Apple Tries is >500 lines long, with over 100 variables used throughout the file. I might attempt to tidy it up a bit, but I doubt I can understand it by myself.

<h2>Some notes</h2>
If anyone is interested, I can share my notes on several classes - mostly my guesses of which variables are located at specific addresses and some info of what some functions do. If anyone has any info on how Apple Tries are stored in the file and how the reading algorithm works, I'd really appreciate the help.

This library uses a lot of `CF` classes, especially `CFURL`s. They are apparently used to represent locations, both on the HDD and on the Internet. The main classes of interest are `DCSDictionary`, `DCSEnvironment`, `DCSDictionaryManager`, `Heap/TrieAccessContext`, `IDXAccessMethodManager` and `IDXBuiltInAccessMethod<Heap/TrieAccessContext>`. The latter two seem to be the ones processing different types of data - index (`TrieAccessContext`) and actual dictionary data (`HeapAccessData`). The cache file is located in `$HOME/Library/Caches/com.apple.DictionaryServices/DictionaryCache.plist`. The dictionaries themselves were located in `/Library/Dictionaries` on my machine. The main files of interest there are `Body.data`, `Key` and `Key`. One can also take a look at `Info.plist`, as it provides a lot of information about data stored in these files, as well as the `AccessMethods` used to access them. And to anyone looking for these dictionaries, they can be found online fairy easily. As for the library itself, I can share it if anyone needs it.
