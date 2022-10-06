# Crix: Detecting Missing-Check Bugs in OS Kernels

Missing a security check is a class of semantic bugs in software programs where erroneous execution states are not validated. Missing-check bugs are particularly common in OS kernels because they frequently interact with external untrusted user space and hardware, and carry out error-prone computation. Missing-check bugs may cause a variety of critical security consequences, including permission bypasses, out-of-bound accesses, and system crashes.

The tool, Crix, can quickly detect missing-check bugs in OS kernels. It evaluates whether any security checks are missing for critical variables, using an inter-procedural, semantic- and context-aware cross-checking. We have used Crix to find 278 new missing-check bugs in the Linux kernel. More details can be found in the paper shown at the bottom.

## How to use Crix

### Build LLVM
```sh
	$ cd llvm
	$ ./build-llvm.sh
	# The installed LLVM is of version 10.0.0
```

### Build the Crix analyzer
```sh
	# Build the analysis pass of Crix
	$ cd ../analyzer
	$ make
	# Now, you can find the executable, `kanalyzer`, in `build/lib/`
```

### Prepare LLVM bitcode files of OS kernels

* Replace error-code definition files of the Linux kernel with the ones in "encoded-errno"
* The code should be compiled with the built LLVM
* Compile the code with options: -O0 or -O2, -g, -fno-inline
* Generate bitcode files
	- We have our own tool to generate bitcode files: https://github.com/sslab-gatech/deadline/tree/release/work. Note that files (typically less than 10) with compilation errors are simply discarded
	- We also provided the pre-compiled bitcode files - https://github.com/umnsec/linux-bitcode

### Run the Crix analyzer
```sh
	# To analyze a single bitcode file, say "test.bc", run:
	$ ./build/lib/kanalyzer -sc test.bc
	# To analyze a list of bitcode files, put the absolute paths of the bitcode files in a file, say "bc.list", then run:
	$ ./build/lib/kalalyzer -mc @bc.list
```

## More details
* [The Crix paper (USENIX Security'19)](https://www-users.cs.umn.edu/~kjlu/papers/crix.pdf)
```sh
@inproceedings{crix-security19,
  title        = {{Detecting Missing-Check Bugs via Semantic- and Context-Aware Criticalness and Constraints Inferences}},
  author       = {Kangjie Lu and Aditya Pakki and Qiushi Wu},
  booktitle    = {Proceedings of the 28th USENIX Security Symposium (Security)},
  month        = August,
  year         = 2019,
  address      = {Santa Clara, CA},
}
```

## GAB's notes and chnages:

 This has been slightly modified to work with LLVM 10.0.1 and Linux 5.10.
 Originally it claims to have worked with LLVM 10.0.0 and 5.3. Not much changed
 from 10.0.0 that I could tell except some bug fixes, but much more has changed
 in Linux.

Planning on adding the ability to detect locks and the branching heursitics here
and ability to output all the info including callgraph, CFG, etc into some sort
of database including any source/IR/machine code translation and location info.
May also want to do our dataflow analysis in this repo too.

The changes I made to get it working were:

-disable inline patch for llvm and the option for kernel is not necessary and
probably better to not use, we can compile with -O2

-linux compiled with SYMOBLIC_ERRNAME disabled. Crix patches Linux to change the
enums of errors to larger and unique values for easy detection. Linux 5.4 adds
an errname.c which basically allows printing the actual enum error name instead
of the int value for debug purposes. But annoyingly they also added a range
check to restrict the possible enum values, so that large values would cause
compilation crash. Smaller values are preferred for embedded systems to reduce
the table size they are stored in so it would force designers to reevaluate
their decision if they tried to add an err with large enum.

-In Crix, added checks for isLiteral() in a couple places where compound literal
structs are encountered. Structs were iterated over to get their names, but
literals don' have names so it would crash. Either some compound literals were
added to linux or LLVM changed the way it compiled them to IR. I believe it
creates two entities, the one we care about and the literal, which we can
ignore, so it shouldn't affect our results

-In Crix, added checks to the hash function if the object is opaque. Also not
sure if this is a kernel or LLVM change. The hash would try to expand structs to
create a more detailed hash, but opaque structs can't be expanded so it woudl
crash. Instead we treat as not a struct. I'm not sure what precision impact this
has or what cases this happens for, but I assume it isn't a lot. Might change
this as I understand the code better

-In Crix, output is disabled by default. Can enable with some of the DEBUG_PRINT
#defines in SecurityChecks.cc and alter the outputs.

-note they use "deadline" instead of gllvm to compile whole kernel, which may
result in some of the differences seen. I believe deadline works similarly but
perhaps handles things a little differently

 Don't worry about the instructions below, we have our own in our llvm and
 kernel directories. However we do still need to build this to use it (cd
 ../analyzer, make)

 As a note, the instructions and files given by this project are not complete,
 see the cheq repo and Typedive:Where does it go? paper as well (same underlying
 code/structure)
