![Logo](https://github.com/EastPoint/CroMagVersion/raw/master/logo-128.png)

# CroMagVersion

CroMag helps to version all of the projects within a solution via a single file convention, and will bump build and version numbers of your project automatically during the build based on a date and version number scheme AND a build number variable that can originate from a build server like [Jenkins](http://jenkins-ci.org/).  Furthermore, DVCS changeset hashes also make their way into built assemblies.

## Requirements

* Nuget
* Package restore via [PepitaGet](http://code.google.com/p/pepita/) (or Nuget) highly recommended
* MSBuild 4 or higher (uses static property functions)
* To have the current short SHA1 hash added to AssemblyConfiguration, you must be using Mercurial or Git


## What does it do?

CroMagVersion allows projects to share the following assembly attributes:

* [AssemblyCompany](http://msdn.microsoft.com/en-us/library/system.reflection.assemblycompanyattribute.aspx) - Uses $(VersionCompany) and $(VersionCompanyUrl) from ```version.props```
* [AssemblyCopyright](http://msdn.microsoft.com/en-us/library/system.reflection.assemblycopyrightattribute.aspx) - Copyright $(Year) $(VersionCompany) from ```version.props```
* [AssemblyConfiguration](http://msdn.microsoft.com/en-us/library/system.reflection.assemblyconfigurationattribute.aspx) - Annotated with build Configuration (i.e. Debug, Release) and the SHA1 hash of the current source version

* [AssemblyVersion](http://msdn.microsoft.com/en-us/library/system.reflection.assemblyversionattribute.aspx) - Calculates date convention based version
* [AssemblyFileVersion](http://msdn.microsoft.com/en-us/library/system.reflection.assemblyfileversionattribute.aspx) - Calculates date convention based version by default, but the layout can be customized (see below)
* [AssemblyInformationalVersion](http://msdn.microsoft.com/en-us/library/system.reflection.assemblyinformationalversionattribute.aspx) - Calculates date convention based version, but the layout can be customized (see below)

## How does it work?

* During the package installation ```SharedAssemblyInfo.cs``` is added to the project file as a **linked file** that is generated by a **linked** T4 template -
```CroMagVersion.tt```.  This file will be shared amongst all projects that install the package, and is not actually copied into any projects.

  **NOTE: This isn't quite a *standard* T4 template.  There is no dependency on
  Visual Studio or any special MSBuild tasks or other cruft that make this
  difficult to share with other developers or use on a build server.  The T4
  generator from MonoDevelop is called via an Exec task right before the compile.
  The T4 template results *may* be refreshed in Visual Studio by running the
  ```Run Custom Tool``` command.  However, since the T4 template has no access
  to the real MSBuild ```$(SolutionDir)``` or ```$(Configuration)``` variables,
  the generated file does not have the same contents that it would otherwise
  have during a build (and will further be regenerated on the next build /
  rebuild cycle).**

* The constant ```CROMAG``` is added to any project configurations that DOES NOT
have the ```DEBUG``` constant defined.  Use this constant to restrict the
generation of versioning information for local developer builds.  Given the
nature of metadata versioning it is impossible to support [incremental builds](http://msdn.microsoft.com/en-us/library/ms171483.aspx).
For large projects, its vital to define the ```CROMAG``` constant on only the
build configuration used by the build server.

* A new file called ```version.props``` is copied to the solution folder if it doesn't exist - this is where user modifications such as major and minor version should be made.  These values are configurable and should always be modified by hand as needed.  If following semantic guidelines, there's no easy way to identify what things are breaking changes - this requires a human of at least average intelligence.

```xml
<MajorVersion>0</MajorVersion>
<MinorVersion>1</MinorVersion>
<VersionCompany></VersionCompany>
<VersionCompanyUrl></VersionCompanyUrl>
<!-- Typically this value will be supplied by a build server like Jenkins -->
<!--
<BUILD_NUMBER>0</BUILD_NUMBER>
-->
```
**NOTE: While this looks like an MSBuild file, it is not.  It is a standard XML
file**

* ```version.props``` is shared at the solution level, but projects must opt-in.
That is, they must install the CroMagVersion NuGet package to take advantage of
automatic versioning.

* A ```CroMagVersion``` msbuild target is injected into the project and it is
set to run before CoreCompile (the CSharp target typically responsible for
running the csc compiler).  In prior versions, this was done with the [Import](http://msdn.microsoft.com/en-us/library/92x05xfs.aspx)
of a ```CroMagVersion.targets``` file, but this caused a chicken/egg scenario
in package restore scenarios, so the ```SharedAssemblyInfo.cs``` generation is
now performed by a T4 template instead, which doesn't prevent projects from
loading if it hasn't been restored from Nuget yet.

* Before the build goes down, ```SharedAssemblyInfo.cs``` file is updated with the major / minor version from ```version.props``` and has date based version information added in the following format:
$(MajorVersion).$(MinorVersion).$(YearMonth).$(DayNumber)$(Build)

    * ```YearMonth``` is a 2 digit year, 2 digit month - for instance 1203
    * ```DayNumber``` is a 2 digit day - for instance 03 or 31
    * ```$(Build)``` is 0 by default, or ```BUILD_NUMBER``` environment variable as supplied by Jenkins or via an override in ```version.props```.  Only the last 3 digits of this number can be used, as each fragment of the build number has a maximum of 65536.

Is this the best way to date tag a build?  Not necessarily, but it's a pretty reasonable solution that results in something human readable after the fact.

**NOTE: This system cannot accomodate multiple version changes in a day if there
is no build server providing a ```BUILD_NUMBER``` environment variable.**

## Customizing The Layout

```AssemblyFileVersion``` and ```AssemblyInformationalVersion``` can have customized layouts by enabling ```AssemblyFileVersionLayout``` or ```AssemblyInformationalVersionLayout``` in ```version.props``` .  This may be necessary to deal with wacky installers (like MSI) that don't consider a bump to the 4th number significant enough to upgrade an existing product install.  Sigh...

The following is pretty self-explanatory:

```xml
<!-- Available variables for a custom layout are:
  $(MajorVersion) - from this file
  $(MinorVersion) - from this file
  $(BUILD_NUMBER) - original number from build server / environment variable if available, otherwise 0, unless overriden in this file
  $(Build) - BUILD_NUMBER truncated to last 3 digits
  $(YearMonth) - yyMM
  $(DayNumber) - dd
  $(Year) - yyyy

  $(BuildVersion) - $(YearMonth)
  $(RevisionVersion) - $(DayNumber)$(Build)

  $(VersionNumber) - $(MajorVersion).$(MinorVersion).$(BuildVersion).$(RevisionVersion)

  Note that these are NOT proper MSBuild variables, and therefore
  arbitrary variables will not resolve.  Only these whitelisted
  variables will do anything.  All other strings, valid or not,
  will be treated as plaintext.
-->

<!--
<AssemblyFileVersionLayout>$(MajorVersion).$(MinorVersion).$(BuildVersion).$(RevisionVersion)</AssemblyFileVersionLayout>
<AssemblyInformationalVersionLayout>$(MajorVersion).$(MinorVersion).$(BuildVersion).$(RevisionVersion)</AssemblyInformationalVersionLayout>
-->
```

## A note about naming conventions for version pieces

This project follows the standard .NET style naming convention specified by [System.Version](http://msdn.microsoft.com/en-us/library/system.version.aspx) of Major.Minor.Build.Revision, where the 3rd and 4th numbers are Build and Revision.

Other systems tend to use different schemes:

* Major.Minor.Revision.Build - common many places, including, Java
* Major.Minor.Patch - for instance [SemVer](http://semver.org/)
* Major.Minor.Build, for instance MSI [ProductVersion](http://msdn.microsoft.com/en-us/library/windows/desktop/aa370859\(v=vs.85\).aspx).

The names for the slots aren't important so much as the semantics.

## Similar Projects

* [SemVerHarvester](https://github.com/jennings/SemVerHarvester) - MSBuild task library that harvests version numbers from tags in source control versions.  It appears to work with both Git and Mercurial.

## Future Improvements / Known Issues

* Until proper MSBuild inputs / outputs are figured out, there might be issues
in VS 2012 with parallel builds enabled.  File locking may occur, due to a race
condition with generation of SharedAssemblyInfo.cs.
* Ensure Mono works properly
* In Visual Studio, CroMagVersion.tt doesn't initially show under Properties,
even though it exists in the csproj on disk. This appears to be a VS bug.

## Release Notes

#### 0.3.4.0 - minor bugfix
    * Deleting README-CroMagVersion.txt during install
    * This file is used to ensure install.ps1 is kicked off, but thats it!

#### 0.3.2.0 - minor bugfix

* An initial DEBUG build would fail after the first package restore, until a RELEASE build
was executed first.  Now shipping an empty version file to work around this issue.

#### 0.3.1.0 - minor bugfixes

* If a project path contained spaces, TextTransform.exe would fail to handle the SolutionDir
properly.  There was an interesting issue with Mono.Options that made it more difficult than
it should have been to quote variables at the command line.
* Non-standard Git and Mercurial locations are OK, as long as git.exe or hg.exe is in the PATH.
* Uninstall no longer opens the AssemblyInfo.cs files modified during installation.  This is
because there is no way to differentiate a package update from a full on removal.


#### 0.3.0.0 - major rewrite of codegen procedure

* File generation is now provided through a T4 template instead of via an imported MSBuild target.
This was necessary to better support package restore, and should be a transparent under-the-hood change.
* Includes MonoDevelops TextTransform.exe to codegen instead of relying on anything that ships
with Visual Studio
* Adds a new CROMAG variable that must be defined to generate versioning code - by default this
is enabled for any non-DEBUG builds
* Breaking change - no longer emits MSBuild variables $(VersionNumber) $(Changeset) $(BuildVersion)
or $(RevisionVersion)

#### 0.2.5.0

* Removed MSBuild-y ness from version.props since its extraneous and confuses things (breaking change)
* Added ability to override FileVersion and AssemblyInformationalVersion versions with a custom layout given a few well known variables
* Able to find AssemblyInfo.cs in locations other than Properties folder
* Using BeforeTargets="CoreCompile" instead of BeforeTargets="Build"
* Renamed Lines ItemGroup internally to AssemblyInfoLines to prevent possible collisions with other global MSBuild variables
* Renamed Version target to CroMagVersion target to prevent possible MSBuild collisions
* CroMagVersion now registered as a dependency via BuildDependsOn and RebuildDependsOn

## Contributing

Fork the code, and submit a pull request!

Any useful changes are welcomed.  If you have an idea you'd like to see implemented that strays far from the simple spirit of the application, ping us first so that we're on the same page.

## Credits

* [MonoDevelop TextTemplating Add-In](https://github.com/mono/monodevelop/tree/master/main/src/addins/TextTemplating) - provides the
T4 templating engine used
* [Git-Last-Commit Gist](https://gist.github.com/966148) - inline msbuild task for grabbing Git command line output
* [StackOverlow - How can I auto increment the C# assembly version via our CI platform (Hudson)?](http://stackoverflow.com/questions/1126880/how-can-i-auto-increment-the-c-assembly-version-via-our-ci-platform-hudson)
* [Mercurial Revision No to Version your AssemblyInfo - MsBuild Series](http://markkemper1.blogspot.com/2010/10/mercurial-revision-no-to-version-your.html)
* The logo is from the [Dino Icon Pack](http://www.fasticon.com/freeware/dino-icons/) produced by FastIcon.
* [MSBuild.Mercurial](http://msbuildhg.codeplex.com/) - tasks for retrieving info from a hg repo - no longer used, but provided initial inspiration for changeset
retrieval
