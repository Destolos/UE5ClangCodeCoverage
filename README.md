# UE5 Clang Code Coverage
This is a guide that explains how to use the clang code coverage for windows in UE5 (Version 5.5.3 was used).

## Prerequisites
* You will need the source version (https://dev.epicgames.com/documentation/en-us/unreal-engine/building-unreal-engine-from-source) of UE5 since this guide uses an engine modification.
* You will also have to install clang (https://dev.epicgames.com/documentation/de-de/unreal-engine/use-clang-to-build-microsoft-platforms-in-unreal-engine?application_version=5.4).
  * The preferred version of clang can be found in the release notes of your engine version (e.g. https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.5-release-notes#platformsdkupgrades). This guide is using LLVM clang 18.1.8.
* I had problems with the libpas.Build.cs when using the clang compiler and had to remove the check for the clang compiler in line 24.
 
## The Engine Modification
This is the most important part of this guide. It only affects the **VCToolChain.cs**.

### Pass the correct Compiler Arguments
You first have to extend the **AppendCLArguments_CPP** method to add the correct compiler arguments (which can be found here https://clang.llvm.org/docs/SourceBasedCodeCoverage.html). Just add this code block at the end of the function:
```cpp
// Enable code coverage flags
if(CompileEnvironment.bCodeCoverage)
{
	if (Target.WindowsPlatform.Compiler.IsClang())
	{
		// Disable compiler optimization.
		Arguments.Add("/Od");

		// Actual arguments to enable coverage
		Arguments.Add("-fprofile-instr-generate -fcoverage-mapping");
	}
}
```

### Link the relevant Libraries
Next you have to link the relevant libraries (explained here https://clang.llvm.org/docs/UsersManual.html#finding-clang-runtime-libraries). This is done inside **AppendLinkArguments**. Just add this code block at the end of the function:
```cpp
// Add code coverage libraries
if (LinkEnvironment.bCodeCoverage)
{
	string? LlvmPath = Environment.GetEnvironmentVariable("LLVM_PATH");
	if (Target.WindowsPlatform.Compiler.IsClang() && !String.IsNullOrEmpty(LlvmPath))
	{
		LinkEnvironment.SystemLibraries.Add("clang_rt.profile-x86_64.lib");

		DirectoryReference LlvmDirectory = new DirectoryReference(LlvmPath);

		DirectoryReference LlvmLib = DirectoryReference.Combine(LlvmDirectory, "lib", "clang", EnvVars.CompilerVersion.Components[0].ToString(), "lib", "windows");
		LinkEnvironment.SystemLibraryPaths.Add(LlvmLib);
	}
}
```

## Add the LLVM_PROFILE_FILE Environment Variable
The usage of this variable is explained here https://clang.llvm.org/docs/SourceBasedCodeCoverage.html#running-the-instrumented-program. This guide says that the variable is optional but for me it was necessary.
### Adding a Directory for the raw profile
Just add a directory on your machine where the default.profraw will be created (for me it's D:\LlvmProfile).
### Create the Environment Variable
Now create a new Environment Variable (named **LLVM_PROFILE_FILE**) pointing to the directory you created in the step before.

## Enable Code Coverage
You still have to enable code coverage. This can be done in various ways:
* In the BuildConfiguration.xml
* In the \<YourProjectName\>Editor.target.cs file
* Pass -CodeCoverage when compiling
  
## Use Clang
Don't forget to switch to the clang compiler (https://dev.epicgames.com/documentation/de-de/unreal-engine/use-clang-to-build-microsoft-platforms-in-unreal-engine?application_version=5.4).

## Run the Tests
Infos about the UE5 Test Framewrok can be found here https://dev.epicgames.com/documentation/en-us/unreal-engine/automation-test-framework-in-unreal-engine.
I recommend runing the tests via command line:

```
"<PathTOEngineBinaries>/UnrealEditor-cmd.exe" "<PathToProject>\MyProject.uproject" -nosplash -Unattended -nopause -nosound -nullrhi -nocontentbrowser -ExecCmds="Automation RunTests <TestsYouWantToRun>;**SoftQuit**" -TestExit="Automation Test Queue Empty" -log
```

The **SoftQuit** parameter is very important, otherwise the system quits before the relevant data for coverage are collected.

## Create the Coverage Report
### Indexing the Raw Profile
Following this https://clang.llvm.org/docs/SourceBasedCodeCoverage.html#creating-coverage-reports we just have to exectute the following commandline command:
```
"<PathToYourLLVMInstallation>\bin\llvm-profdata.exe" merge "<Directory where your LLVM_PROFILE_FILE variable is pointing to>\default.profraw" -o "<Directory where your LLVM_PROFILE_FILE variable is pointing to>\merged.profraw"
```
### Creating the HTML Coverage Report
Finally we have everything ready to create the report showing the code coverage. This is done by another commandline command:
```
"<PathToYourLLVMInstallation\>\bin\llvm-cov.exe" show -format=html -use-color -output-dir="<Path where the HTML is created>" -instr-profile "\<Directory where your LLVM_PROFILE_FILE variable is pointing to\>\merged.profraw" "\<Path to the *.dll that has been tested\>"
```
I recommend using the additional **-ignore-filename-regex** argument to remove all files from the report that should not be in there.
I am using the following
```
-ignore-filename-regex="UnrealEngine|Intermediate"
```

Now you should have a file similar to ![ExampleCoverageReport](https://github.com/user-attachments/assets/f21cd427-ebe0-490e-8102-1368929d4d01)
