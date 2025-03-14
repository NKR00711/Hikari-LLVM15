# Hikari-LLVM15
 A fork of HikariObfuscator [WIP]
 
 ## Link to the original project
 [https://github.com/HikariObfuscator/Hikari](https://github.com/HikariObfuscator/Hikari)
 
 ## Build
 
 [https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm](https://llvm.org/docs/GettingStarted.html#getting-the-source-code-and-building-llvm)

If you're using macOS, it can be downloaded from [actions](https://github.com/61bcdefg/Hikari-LLVM15/actions)

### Obfuscate Swift

Due to the large number of closed-source changes in Xcode's LLVM compared to the original LLVM, Hanabi has been unable to compile Swift since Xcode 15.

You can use [Swift Toolchain](https://github.com/61bcdefg/Hikari-Swift)。

Note that the obfuscation parameter is added in **Other Swift Flags** in **Swift Compiler-Other Flags**, and need to pass `-Xllvm`, not `-mllvm`.

Where Optimization is turned off is in **Optimization Level** in **Swift Compiler-Code Generation**, set to *No Optimization [-Onone]*. Due to the nature of the swift language, StringEncryption may doesn't work if optimization is not turned off

Each time **Other Swift Flags** are modified, Shift+Command+K(Clean Build Folder) is required before compilation, because Swift does not recompile when it detects changes to the project CFlags as Objective-C does

### PreCompiled IR

PreCompiled IR is a custom LLVM Bitcode file that is generated by adding `-emit-llvm` to the compile command (C Flags) of the source file where the callback function exists, and then placing it in the specified location

###  Some modifications

#### AntiClassDump (This pass has design flaws, the program may crash after enabling it, not recommended for use)

Support for arm64e

#### BogusControlFlow

Skip basic blocks containing the MustTailCall to avoid errors

Skip presplit coroutine and basic blocks containing CoroBeginInst to support swift

Fixed missing opaque predicates

#### Flattening

Skip presplit coroutine to support swift

Indirect modification of the state variable, it can make some scripts unable to properly deobfuscate (e.g. d810)

#### FunctionCallObfuscate

The Objc Call will only be obfuscated where obfuscation is enabled, not the whole module

#### FunctionWrapper

Skip some functions that are not currently handled to support swift

Supports obfuscation of functions containing byval (Probably?)

#### SplitBasicBlocks

Fixed possible heap corruption errors

#### StringEncryption

Support for strings in structs and arrays

Support for Rust strings

Support for arm64e

#### Substitution

Add more patterns

#### IndirectBranch

The order of the base blocks is rearranged

Stack-based jumping is enabled by default, which can make static analysis more difficult

###  Options of Obfuscation 

Only the modified parts will be covered here, for the original parts, please go to [https://github.com/HikariObfuscator/Hikari/wiki/](https://github.com/HikariObfuscator/Hikari/wiki/)

#### AntiClassDump

-acd-rename-methodimp

Rename the method function name shown in IDA (to ACDMethodIMP), not the method name. Default off

#### AntiHooking (Not recommended for use)

Enabling this function as a whole will cause the size of the generated binary file to swell dramatically.

Supports detection of Objective-C runtime hooks. If detected, the AHCallBack function (obtained from PreCompiled IR) will be called, and if AHCallBack does not exist, the program will exit.

InlineHook checks are currently only supported by arm64. Insert code into the function to check if the current function is hooked, and if it detects it, the AHCallBack function is called (obtained from PreCompiled IR). If AHCallBack does not exist, the program will exit.

-enable-antihook

Enable AntiHooking. Default off

-ah_inline

Detects whether the current function is inline hook. Default on

-ah_objcruntime

Detects whether the current function is caught by the runtime hook. Default on

-ah_antirebind

The symbols of generated file cannot be rebinded by fishhook. Default off

-adhexrirpath

The path of the AntiHooking PreCompiled IR file

#### AntiDebugging

Automatic anti-debugging in functions, if there are InitADB and ADBCallBack functions (obtained from PreCompiled IR), the ADBInit function will be called, if InitADB and ADBCallBack functions do not exist and are on Apple ARM64 platform, Inline assembly anti-debugging is automatically inserted into functions of void return type, otherwise nothing is done.

-enable-adb

Enable AntiDebugging. Default off

-adb_prob

The probability that anti-debug code is added to each function. Default is 40

-adbextirpath

The path of the AntiDebugging PreCompiled IR file

#### StringEncryption

-strcry_prob

The probability that each byte in each string is encrypted. Default is 100

#### BogusControlFlow

-bcf_onlyjunkasm

Insert only junk code in bogus blocks

-bcf_junkasm

Insert junk code in bogus blocks, interfering with IDA's recognition of functions. Default off

-bcf_junkasm_minnum

The minimum number of junk code to be added in a bogus block. Default is 2

-bcf_junkasm_maxnum

The maximum number of junk code to be added in a bogus block. Default is 4

-bcf_createfunc

Use functions to wrap opaque predicates. Default off

#### ConstantEncryption

Modified from https://iosre.com/t/llvm-llvm/11132

Xor encryption of constant integers used in instructions that can be processed

-enable-constenc

Enable ConstantEncryption. Default off

-constenc_times

How many time the ConstantEncryption pass loop on a function. Default is 1

-constenc_togv

Replace constant integers with global variables, and the result of a binary operator of type integer with global variables. Default off

-constenc_togv_prob

The probability that each ConstantInt wil be replaced with GlobalVariable. Default is 50

-constenc_subxor

Replace the xor operation of ConstantEncryption to make it more complex

-constenc_subxor_prob

The probability that each xor operation is replaced by a more complex operation. Default is 40

#### IndirectBranch

-indibran-use-stack

The address of the jump table is loaded into the stack in the Entry Block, and each basic block reads it from the stack. Default on

-indibran-enc-jump-target

Encrypts jump tables and the indexes. Default off

### Functions Annotations

#### Supported Options
##### C++/C functions
For example, if you have multiple functions, but you only want to obfuscate the function `int foo()` with `indibran-use-stack` enabled, you can declare it like this:
```
int foo() __attribute((__annotate__(("indibran_use_stack"))));
int foo() {
   return 2;
}
```
If you only want to obfuscate the function `int foo()` without using `indibran-use-stack`, you can declare it like this:
```
int foo() __attribute((__annotate__(("noindibran_use_stack"))));
int foo() {
   return 2;
}
```
If you only wanted the BogusControlFlow of function` int foo()` to be obfuscated with a probability of 100, you can declare it like this:
```
int foo() __attribute((__annotate__(("bcf_prob=100"))));
int foo() {
   return 2;
}
```
##### ObjC Methods
For example you want to pass `indibran-use-stack` like the C/C++ example:
```
#ifdef __cplusplus
extern "C" {
#endif
void hikari_indibran_use_stack(void);
#ifdef __cplusplus
}
#endif

@implementation foo2 : NSObject
+ (void)foo {
  hikari_indibran_use_stack();
  NSLog(@"FOOOO2");
}
@end
```
If you only wanted the BogusControlFlow of method `+ (void)foo` to be obfuscated with a probability of 100:
```
#ifdef __cplusplus
extern "C" {
#endif
void hikari_bcf_prob(uint32_t);
#ifdef __cplusplus
}
#endif

@implementation foo2 : NSObject
+ (void)foo {
  hikari_bcf_prob(100);
  NSLog(@"FOOOO2");
}
@end
```
##### Options
-   `ah_inline` 
-   `ah_objcruntime`
-   `ah_antirebind`  
-   `bcf_prob`
-   `bcf_loop`
-   `bcf_cond_compl`   
-   `bcf_onlyjunkasm`
-   `bcf_junkasm`
-   `bcf_junkasm_maxnum`
-   `bcf_junkasm_minnum`
-   `bcf_createfunc`
-   `constenc_subxor`
-   `constenc_togv`
-   `constenc_togv_prob`
-   `constenc_subxor_prob`
-   `constenc_times`
-   `fw_prob`
-   `indibran_use_stack`
-   `indibran_enc_jump_target`
-   `split_num`
-   `strcry_prob`
-   `sub_loop`
-   `sub_prob`

#### New Supported Flags

-   `adb` Anti Debugging
-   `antihook` Anti Hooking
-   `constenc` Constant Encryption

## License

See [https://github.com/HikariObfuscator/Hikari#license](https://github.com/HikariObfuscator/Hikari#license)

