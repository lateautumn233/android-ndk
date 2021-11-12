==============================================
llvm-rs-cc: Compiler for Renderscript language
==============================================


Introduction
------------

llvm-rs-cc compiles a program in the Renderscript language to generate the
following files:

* Bitcode file. Note that the bitcode here denotes the LLVM (Low-Level
  Virtual Machine) bitcode representation, which will be consumed on
  an Android device by libbcc (in
  platform/frameworks/compile/libbcc.git) to generate device-specific
  executables.

* Reflected APIs for Java. As a result, Android's Java developers can
  invoke those APIs from their code.

Note that although Renderscript is C99-like, we enhance it with several
distinct, effective features for Android programming. We will use
some examples to illustrate these features.

llvm-rs-cc is run on the host and performs many aggressive optimizations.
As a result, libbcc on the device can be lightweight and focus on
machine-dependent code generation for some input bitcode.

llvm-rs-cc is a driver on top of libslang. The architecture of
libslang and libbcc is depicted in the following figure::

    libslang   libbcc
        |   \   |
        |    \  |
     clang     llvm


Usage
-----

* *-o $(PRIVATE_RS_OUTPUT_DIR)/res/raw*

  This option specifies the directory for outputting a .bc file.

* *-p $(PRIVATE_RS_OUTPUT_DIR)/src*

  The option *-p* denotes the directory for outputting the reflected Java files.

* *-d $(PRIVATE_RS_OUTPUT_DIR)*

  This option *-d* sets the directory for writing dependence information.

* *-MD*

  Note that *-MD* will tell llvm-rs-cc to output dependence information.

* *-a $(EXTRA_TARGETS)*

  Specifies additional target dependencies.

Example Command
---------------

First::

  $ cd <Android_Root_Directory>

Using frameworks/base/tests/RenderScriptTests/Fountain as a simple app in both
Java and Renderscript, we can find the following command line in the build
log::

  $ out/host/linux-x86/bin/llvm-rs-cc \
    -o out/target/common/obj/APPS/Fountain_intermediates/src/renderscript/res/raw \
    -p out/target/common/obj/APPS/Fountain_intermediates/src/renderscript/src \
    -d out/target/common/obj/APPS/Fountain_intermediates/src/renderscript \
    -a out/target/common/obj/APPS/Fountain_intermediates/src/RenderScript.stamp \
    -MD \
    -I frameworks/base/libs/rs/script_api/include \
    -I external/clang/lib/Headers \
    frameworks/base/libs/rs/java/Fountain/src/com/android/fountain/fountain.rscript

This command will generate:

* **fountain.bc**

* **ScriptC_fountain.java**

* **ScriptField_Point.java**

The **Script\*.java** files above will be documented below.


Example Program: fountain.rscript
----------------------------

fountain.rscript is in the Renderscript language, which is based on the standard
C99. However, llvm-rs-cc goes beyond "clang -std=c99" and provides the
following important features:

1. Pragma
---------

* *#pragma rs java_package_name([PACKAGE_NAME])*

  The ScriptC_[SCRIPT_NAME].java has to be packaged so that Java
  developers can invoke those APIs.

  To do that, a Renderscript programmer should specify the package name, so
  that llvm-rs-cc knows the package expression and hence the directory
  for outputting ScriptC_[SCRIPT_NAME].java.

  In fountain.rscript, we have::

    #pragma rs java_package_name(com.android.fountain)

  In ScriptC_fountain.java, we have::

    package com.android.fountain

  Note that the ScriptC_fountain.java will be generated inside
  ./com/android/fountain/.

* #pragma version(1)

  This pragma is for evolving the language. Currently we are at
  version 1 of the language.


2. Basic Reflection: Export Variables and Functions
---------------------------------------------------

llvm-rs-cc automatically exports the "externalizable and defined" functions and
variables to Android's Java side. That is, scripts are accessible from
Java.

For instance, for::

  int foo = 0;

In ScriptC_fountain.java, llvm-rs-cc will reflect the following methods::

  void set_foo(int v)...

  int get_foo()...

This access takes the form of generated classes which provide access
to the functions and global variables within a script. In summary,
global variables and functions within a script that are not declared
static will generate get, set, or invoke methods.  This provides a way
to set the data within a script and call its functions.

Take the addParticles function in fountain.rscript as an example::

  void addParticles(int rate, float x, float y, int index, bool newColor) {
    ...
  }

llvm-rs-cc will genearte ScriptC_fountain.java as follows::

  void invoke_addParticles(int rate, float x, float y,
                           int index, bool newColor) {
    ...
  }


3. Export User-Defined Structs
------------------------------

In fountain.rscript, we have::

  typedef struct __attribute__((packed, aligned(4))) Point {
    float2 delta;
    float2 position;
    uchar4 color;
  } Point_t;

  Point_t *point;

llvm-rs-cc generates one ScriptField*.java file for each user-defined
struct. In this case, llvm-rs-cc will reflect two files,
ScriptC_fountain.java and ScriptField_Point.java.

Note that when the type of an exportable variable is a structure, Renderscript
developers should avoid using anonymous structs. This is because llvm-rs-cc
uses the struct name to identify the file, instead of the typedef name.

For the generated Java files, using ScriptC_fountain.java as an
example we also have::

  void bind_point(ScriptField_Point v)

This binds your object with the allocated memory.

You can bind the struct(e.g., Point), using the setter and getter
methods in ScriptField_Point.java.

After binding, you can access the object with this method::

  ScriptField_Point get_point()

In ScriptField_Point_s.java::

    ...
    // Copying the Item, which is the object that stores every
    // fields of struct, to the *index*\-th entry of byte array.
    //
    // In general, this method would not be invoked directly
    // but is used to implement the setter.
    void copyToArray(Item i, int index)

    // The setter of Item array,
    // index: the index of the Item array
    // copyNow: If true, it will be copied to the *index*\-th entry
    // of byte array.
    void set(Item i, int index, boolean copyNow)

    // The getter of Item array, which gets the *index*-th element
    // of byte array.
    Item get(int index)

    set_delta(int index, Float2 v, boolean copyNow)

    // The following is the individual setters and getters of
    // each field of a struct.
    public void set_delta(int index, Float2 v, boolean copyNow)
    public void set_position(int index, Float2 v, boolean copyNow)
    public void set_color(int index, Short4 v, boolean copyNow)
    public Float2 get_delta(int index)
    public Float2 get_position(int index)
    public Short4 get_color(int index)

    // Copying all Item array to byte array (i.e., memory allocation).
    void copyAll()
    ...


4. Summary of the Java Reflection above
---------------------------------------

This section summarizes the high-level design of Renderscript's reflection.

* In terms of a script's global functions, they can be called from Java.
  These calls operate asynchronously and no assumptions should be made
  on whether a function called will have actually completed operation.  If it
  is necessary to wait for a function to complete, the Java application
  may call the runtime finish() method, which will wait for all the script
  threads to complete pending operations.  A few special functions can also
  exist:

  * The function **init** (if present) will be called once after the script
    is loaded.  This is useful to initialize data or anything else the
    script may need before it can be used.  The init function may not depend
    on globals initialized from Java as it will be called before these
    can be initialized. The function signature for init must be::

      void init(void);

  * The function **root** is a special function for graphics.  This function
    will be called when a script must redraw its contents.  No
    assumptions should be made as to when this function will be
    called.  It will only be called if the script is bound as a graphics root.
    Calls to this function will be synchronized with data updates and
    other invocations from Java.  Thus the script will not change due
    to external influence in the middle of running **root**.  The return value
    indicates to the runtime when the function should be called again to
    redraw in the future.  A return value of 0 indicates that no
    redraw is necessary until something changes on the Java side.  Any
    positive integer indicates a time in milliseconds that the runtime should
    wait before calling root again to render another frame.  The function
    signature for a graphics root functions is as follows::

      int root(void);

  * It is also possible to create a purely compute-based **root** function.
    Such a function has the following signature::

      void root(const T1 *in, T2 *out, const T3 *usrData, uint32_t x, uint32_t y);

    T1, T2, and T3 represent any supported Renderscript type.  Any parameters
    above can be omitted, although at least one of in/out must be present.
    If both in and out are present, root must only be invoked with types of
    the same exact dimensionality (i.e. matching X and Y values for dimension).
    This root function is accessible through the Renderscript language
    construct **forEach**.  We also reflect a Java version to access this
    function as **forEach_root** (for API levels of 14+).  An example of this
    can be seen in the Android SDK sample for HelloCompute.

  * The function **.rs.dtor** is a function that is sometimes generated by
    llvm-rs-cc.  This function cleans up any global variable that contains
    (or is) a reference counted Renderscript object type (such as an
    rs_allocation, rs_font, or rs_script).  This function will be invoked
    implicitly by the Renderscript runtime during script teardown.

* In terms of a script's global data, global variables can be written
  from Java.  The Java instance will cache the value or object set and
  provide return methods to retrieve this value.  If a script updates
  the value, this update will not propagate back to the Java class.
  Initializers, if present, will also initialize the cached Java value.
  This provides a convenient way to declare constants within a script and
  make them accessible to the Java runtime.  If the script declares a
  variable const, only the get methods will be generated.

  Globals within a script are considered local to the script.  They
  cannot be accessed by other scripts and are in effect always 'static'
  in the traditional C sense.  Static here is used to control if
  accessors are generated.  Static continues to mean *not
  externally visible* and thus prevents the generation of
  accessors.  Globals are persistent across invocations of a script and
  thus may be used to hold data from run to run.

  Globals of two types may be reflected into the Java class.  The first
  type is basic non-pointer types.  Types defined in rs_types.rsh may also be
  used.  For the non-pointer class, get and set methods are generated for
  Java.  Globals of single pointer types behave differently.  These may
  use more complex types.  Simple structures composed of the types in
  rs_types.rsh may also be used.  These globals generate bind points in
  Java.  If the type is a structure they also generate an appropriate
  **Field** class that is used to pack and unpack the contents of the
  structure.  Binding an allocation in Java effectively sets the
  pointer in the script.  Bind points marked const indicate to the
  runtime that the script will not modify the contents of an allocation.
  This may allow the runtime to make more effective use of threads.


5. Vector Types
---------------

Vector types such as float2, float4, and uint4 are included to support
vector processing in environments where the processors provide vector
instructions.

On non-vector systems the same code will continue to run but without
the performance advantage.  Function overloading is also supported.
This allows the runtime to support vector version of the basic math
routines without the need for special naming.  For instance,

* *float sin(float);*

* *float2 sin(float2);*

* *float3 sin(float3);*

* *float4 sin(float4);*


===============================================================
libbcc: A Versatile Bitcode Execution Engine for Mobile Devices
===============================================================


Introduction
------------

libbcc is an LLVM bitcode execution engine that compiles the bitcode
to an in-memory executable. libbcc is versatile because:

* it implements both AOT (Ahead-of-Time) and JIT (Just-in-Time)
  compilation.

* Android devices demand fast start-up time, small size, and high
  performance *at the same time*. libbcc attempts to address these
  design constraints.

* it supports on-device linking. Each device vendor can supply their
  own runtime bitcode library (lib*.bc) that differentiates their
  system. Specialization becomes ecosystem-friendly.

libbcc provides:

* a *just-in-time bitcode compiler*, which translates the LLVM bitcode
  into machine code

* a *caching mechanism*, which can:

  * after each compilation, serialize the in-memory executable into a
    cache file.  Note that the compilation is triggered by a cache
    miss.
  * load from the cache file upon cache-hit.

Highlights of libbcc are:

* libbcc supports bitcode from various language frontends, such as
  Renderscript, GLSL (pixelflinger2).

* libbcc strives to balance between library size, launch time and
  steady-state performance:

  * The size of libbcc is aggressively reduced for mobile devices. We
    customize and improve upon the default Execution Engine from
    upstream. Otherwise, libbcc's execution engine can easily become
    at least 2 times bigger.

  * To reduce launch time, we support caching of
    binaries. Just-in-Time compilation are oftentimes Just-too-Late,
    if the given apps are performance-sensitive. Thus, we implemented
    AOT to get the best of both worlds: Fast launch time and high
    steady-state performance.

    AOT is also important for projects such as NDK on LLVM with
    portability enhancement. Launch time reduction after we
    implemented AOT is signficant::


     Apps          libbcc without AOT       libbcc with AOT
                   launch time in libbcc    launch time in libbcc
     App_1            1218ms                   9ms
     App_2            842ms                    4ms
     Wallpaper:
       MagicSmoke     182ms                    3ms
       Halo           127ms                    3ms
     Balls            149ms                    3ms
     SceneGraph       146ms                    90ms
     Model            104ms                    4ms
     Fountain         57ms                     3ms

    AOT also masks the launching time overhead of on-device linking
    and helps it become reality.

  * For steady-state performance, we enable VFP3 and aggressive
    optimizations.

* Currently we disable Lazy JITting.



API
---

**Basic:**

* **bccCreateScript** - Create new bcc script

* **bccRegisterSymbolCallback** - Register the callback function for external
  symbol lookup

* **bccReadBC** - Set the source bitcode for compilation

* **bccReadModule** - Set the llvm::Module for compilation

* **bccLinkBC** - Set the library bitcode for linking

* **bccPrepareExecutable** - *deprecated* - Use bccPrepareExecutableEx instead

* **bccPrepareExecutableEx** - Create the in-memory executable by either
  just-in-time compilation or cache loading

* **bccGetFuncAddr** - Get the entry address of the function

* **bccDisposeScript** - Destroy bcc script and release the resources

* **bccGetError** - *deprecated* - Don't use this


**Reflection:**

* **bccGetExportVarCount** - Get the count of exported variables

* **bccGetExportVarList** - Get the addresses of exported variables

* **bccGetExportFuncCount** - Get the count of exported functions

* **bccGetExportFuncList** - Get the addresses of exported functions

* **bccGetPragmaCount** - Get the count of pragmas

* **bccGetPragmaList** - Get the pragmas


**Debug:**

* **bccGetFuncCount** - Get the count of functions (including non-exported)

* **bccGetFuncInfoList** - Get the function information (name, base, size)



Cache File Format
-----------------

A cache file (denoted as \*.oBCC) for libbcc consists of several sections:
header, string pool, dependencies table, relocation table, exported
variable list, exported function list, pragma list, function information
table, and bcc context.  Every section should be aligned to a word size.
Here is the brief description of each sections:

* **Header** (MCO_Header) - The header of a cache file. It contains the
  magic word, version, machine integer type information (the endianness,
  the size of off_t, size_t, and ptr_t), and the size
  and offset of other sections.  The header section is guaranteed
  to be at the beginning of the cache file.

* **String Pool** (MCO_StringPool) - A collection of serialized variable
  length strings.  The strp_index in the other part of the cache file
  represents the index of such string in this string pool.

* **Dependencies Table** (MCO_DependencyTable) - The dependencies table.
  This table stores the resource name (or file path), the resource
  type (rather in APK or on the file system), and the SHA1 checksum.

* **Relocation Table** (MCO_RelocationTable) - *not enabled*

* **Exported Variable List** (MCO_ExportVarList) -
  The list of the addresses of exported variables.

* **Exported Function List** (MCO_ExportFuncList) -
  The list of the addresses of exported functions.

* **Pragma List** (MCO_PragmaList) - The list of pragma key-value pair.

* **Function Information Table** (MCO_FuncTable) - This is a table of
  function information, such as function name, function entry address,
  and function binary size.  Besides, the table should be ordered by
  function name.

* **Context** - The context of the in-memory executable, including
  the code and the data.  The offset of context should aligned to
  a page size, so that we can mmap the context directly into memory.

For furthur information, you may read `bcc_cache.h <include/bcc/bcc_cache.h>`_,
`CacheReader.cpp <lib/bcc/CacheReader.cpp>`_, and
`CacheWriter.cpp <lib/bcc/CacheWriter.cpp>`_ for details.



JIT'ed Code Calling Conventions
-------------------------------

1. Calls from Execution Environment or from/to within script:

   On ARM, the first 4 arguments will go into r0, r1, r2, and r3, in that order.
   The remaining (if any) will go through stack.

   For ext_vec_types such as float2, a set of registers will be used. In the case
   of float2, a register pair will be used. Specifically, if float2 is the first
   argument in the function prototype, float2.x will go into r0, and float2.y,
   r1.

   Note: stack will be aligned to the coarsest-grained argument. In the case of
   float2 above as an argument, parameter stack will be aligned to an 8-byte
   boundary (if the sizes of other arguments are no greater than 8.)

2. Calls from/to a separate compilation unit: (E.g., calls to Execution
   Environment if those runtime library callees are not compiled using LLVM.)

   On ARM, we use hardfp.  Note that double will be placed in a register pair.
