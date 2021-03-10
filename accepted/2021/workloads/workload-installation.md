In .NET 6, we will be delivering iOS and Android support as optional SDK workloads.  In the future, additional SDK components (possibly including ASP.NET and Windows Forms and WPF) will become optional SDK workloads.  This proposal describes how we will support installing these optional workloads.
 
Related links:
 
- [.NET Optional SDK Workloads](../../2020/workloads/workloads.md) - Overview
- [MSBuild SDK Resolvers and optional workloads in .NET 5](../../2020/workloads/workload-resolvers) - Describes where workload files are laid out on disk and how they are resolved
- [Workload Manifests](https://github.com/dotnet/designs/pull/120)
 
# Installer Abstraction Layer (IAL).

The .NET SDK can be installed via a variety of different installers.  Workloads will typically be installed using the same type of installer as the .NET SDK, so the workload installation (as well as update and uninstall) process needs to support various types of installers.

In order to support these different installers, the .NET SDK will have an "installer abstraction layer" (IAL) which will allow common installation code to work with the different types of installers.  (NOTE: We've previously called this a Platform Abstraction Layer (PAL), so some documents may continue to refer to it as such.)

The IAL is an abstraction that is internal to the .NET SDK.  It is not intended as a public extensibility mechanism to allow supporting arbitrary installers, and we may make breaking changes to its API as needed.

# Supported installation types

We plan to (eventually) support the following installation types.  Some of them may not be supported in .NET 6 if we don't have workloads in .NET 6 that are supported on the corresponding platforms.

## .NET SDK managed installation

In this implementation of the IAL, the .NET SDK itself manages workload installations directly by writing files to the dotnet folder.  This is generally for copies of the .NET SDK which were not installed via an OS installer, for example those that are installed via the dotnet-install scripts, or via unzipping archives of the .NET SDK.

Other terms we have at times used for this installation type: xcopy, universal installer, files on disk, NuGet installer

## Visual Studio managed installation

This is an installation of the .NET SDK installed and managed by the Visual Studio installer.  The .NET SDK workloads will have corresponding Visual Studio workloads or components, which will install workload packs via Windows MSIs.  Note that the Visual Studio workloads or components should also include the necessary Visual Studio components (designers, project system, etc) to support the workload, in addition to the .NET SDK workload packs.

## Standalone Windows installation

This is an installation of the .NET SDK installed on Windows via the .NET SDK installer bundle, which is the .exe which can be downloaded on the .NET SDK download pages.  The bundle installs the same MSIs for the .NET SDK as Visual Studio does, and the workload installer will likewise install the same MSIs for workload packs as the Visual Studio installer.

The same dotnet folder may have different versions of the SDK installed by both the Visual Studio installer and the Standalone installer, which may share the same versions of some components.  For the .NET SDK, the underlying MSIs are ref-counted so that they are uninstalled when there are no more installations that depend on them.  We will need to make sure that workload installation and management also works well when there are different install types coexisting.

## Package manager installation

This is a copy of the .NET SDK installed as a package from a package manager such as APT or RPM.  In this case, the workloads and / or workload packs will also be represented as packages which will be installed via the package manager.

# Choosing the installer abstraction

Before any installation operation is started, the .NET SDK will need to choose which installer abstraction will be used and instantiate it.  To do so, it will look for marker files in the `<DOTNET ROOT>/sdk-manifests/<FEATURE VERSION BAND>/.installertype/` folder.  If a `visualstudio` file is present in that folder, the Visual Studio installer abtraction will be used.  Otherwise, if an `msi` file is present, the standalone Windows installation abstraction will be used.  Otherwise, the .NET SDK managed installer abstraction will be used.

This means that the installer packages that we create for the .NET SDK will need to include these files.  For Windows, we will include the `visualstudio` file in the placeholder MSI which we create for Visual Studio, and the `msi` file will go in the main SDK MSI.

On Mac OS, we will always use the .NET SDK managed installer.  There will be packages for the initial installation of the .NET SDK, but whenever the SDK is installing or updating workloads it will simply do so by copying files around on disk.

# Installation operations

## Workload pack tracking and installation cleanup

Workload artifacts should not be left over after uninstalling or updating workloads.  This is complicated by the fact that the same workload pack may be used by different workloads, and even different .NET SDK bands.  It will be the responsibility of the installer abstraction implementation to manage this.  

For installer abstraction implementations that wrap installers where the workload will appear in a user-visible fashion (for example as a checkbox in the Visual Studio installer, an entry in Windows Add/Remove programs, or as a package installed on a Linux OS), the installer abstraction implementation will generally rely on the underlying installer technology to manage the cleanup of artifacts which may be shared with other workload installations.

Installer abstraction implementations that install workload packs individually will also need to remove those packs when they are no longer needed.  After a workload installation operation (which includes updating or uninstalling workloads), the installer abstraction implementation should remove workload packs which are no longer needed.  To support this, they will generally need to keep a record of which workload packs have been installed for which SDK version band, along with whether the pack is still in use for that SDK band.  This information should be kept in a form that is likely to be forward and backwards compatible across different SDK versions.  First, the workload pack installation records for any SDK bands which are no longer installed can be removed.  Then any workload pack installation records which are not currently used by an installed workload for the current band can be marked as not in use.  Then, any workload packs that don't have any installation records marked as in use can be uninstalled.

## SDK update process
When the .NET SDK is updated to a new version, developers may expect for the same workloads to be installed for the new version as were installed for the old version.  We probably won't be able to do this for all installer technologies.  Nevertheless, the ideal behavior would be this:

- For in-place updates to the .NET SDK, we should also update the workloads when we update the SDK
  - For SDK installations managed by Visual Studio, in-place updates can include updating to a new SDK feature band.  In that case the workload installation is managed by Visual Studio, so the workloads will be updated as we want with the plans we have today
- When installing a new version of the .NET SDK side-by-side with an existing version, we should default to installing the same workloads that were installed for the previous version.  The developer should be able to override this, for example by unchecking the corresponding checkboxes in an installer UI

Given that for some installers we won't be able to automatically, as part of the first run experience we should check to see if there were workloads installed for a previous version, and if there were print a message telling the developer how to install them.  The process for this will be:

- Determine highest previously installed version band
- Via IAL, read workload installation records for that (NOT the current one) band to find out which workloads were installed
- If no workloads were installed, then end process and don't print a message
- Otherwise, ask IAL if it supports installing workloads or just giving instructions
  - If it supports installing, print message suggesting command similar to `dotnet workload install workload1 workload2 workload3` (but listing the actual workloads)
  - If it doesn't support installing, then call IAL to get the command(s) needed to install the workloads, and suggest that command

## Updating the workload manifests

The workload manifests describe available workloads and what workload packs make up each workload.

A baseline version of each manifest will be included in the .NET SDK, which will allow the SDK to notify the user if a workload is needed to build a project.  However, these baseline manifest versions will not necessarily have the latest workload pack versions.  This is so that workloads don't have to insert each build into the .NET SDK, effectively linking the build graphs of all the workloads with the .NET SDK build graph.

Because the manifests may be out of date (especially the baseline manifest), the workload manifests will by default be updated to the latest version when installing or updating workloads.

In order to support rolling back a failed install, as well as to support notifying about available workload updates, there will be separate "advertising manifests", which represent a version of the workload manifest that is available but may not have been installed.  The advertising manifests will be acquired via a NuGet package and stored in the NuGet [global-packages](https://docs.microsoft.com/en-us/nuget/Consume-Packages/managing-the-global-packages-and-cache-folders) folder.  The installed manifests will be in the .NET SDK installation folder, and may be installed via the native installation technology that is in use for the SDK install (for example, there will be an MSI that installs each manifest in the Program Files folder on Windows).

The advertising manifest will be updated as part of other commands (such as workload install or workload update).  It will use the following process for each workload manifest:

- Each workload manifest should be delivered in a NuGet package
  - The package ID should combine the workload manifest ID with the SDK version band, separated with a period.  For example, `Microsoft.NET.Android.WorkloadManifest.6.0.100`.
  - The manifest payload (`WorkloadManifest.json`, `WorkloadManifest.targets`, and any other files to be included with the manifest) should be in a `data/` folder in the NuGet package.  This ensures that there's not confusion about whether a file in the root is part of the workload manifest or a file owned by NuGet.
- Download the latest version of the workload manifest NuGet package using NuGet
  - The package should be extracted into the NuGet [global-packages](https://docs.microsoft.com/en-us/nuget/Consume-Packages/managing-the-global-packages-and-cache-folders) folder.

*Possible optimization: For some installers, it will be necessary to have a native installation package that includes the workload manifest.  For example, there will need to be an MSI which installs the manifest files to the `dotnet` folder in `Program Files`.  To avoid downloading the same manifest twice, we could have the IAL be involved in downloading the workload manifest package, and extracting the information in it as the advertising manifest.  However, this could add significant complexity, and the manifests should be small, so we won't plan to do this optimization initially.*

To install updated manifests, the following process will be used (for each workload manifest):

- Check if advertising manifest is newer than the currently installed version
  - This requires a way to get the NuGet package version of the workload manifest package corresponding to the currently installed version.  Either we need to support semantic versions in the version field of the workload manifest schema, or we need to store the NuGet package version separately (ie in a text file).
  - If the installed version of the workload manifest is greater than or equal to the advertised version, then it is already up-to-date, and we are done with the update process for this manifest
- Tell IAL: Install workload manifest (given ID and version)

## Workload installation process
 
- Input: list of workloads to install
- With rollback:
  - Unless `--skip-manifest-update` parameter has been specified
    - From IAL: Get list of installed workloads
    - Add list of currently installed workloads to the list of workloads to install (since with updated manifest, there may be updated packs to install)
    - Update workload advertising manifests
    - Install updated workload manifests
  - Ask installation IAL: is the unit of installation workloads or workload packs?
    - If unit of installation is workloads:
      - Tell IAL: Install the workloads
    - If unit of installation is workload pack:
      - Read workload manifest, find list of packs that make up workloads to be installed
      - Tell IAL: Install workload packs
  - Tell IAL: Record which workloads have been installed (per sdk band)
    - This is for upgrades and garbage collection
- Tell IAL: Run garbage collection (if unit of installation is packs)

## Workload uninstallation process
- Input: list of workloads to uninstall
- Ask IAL: Is the unit of installation workloads or workload packs?
  - If unit of installation is workloads:
    - Tell IAL: Uninstall workload list
  - If unit of installation is workload pack:
    - Tell IAL: Remove specified workloads from installation record
    - Tell IAL: Run garbage collection to remove leftover workload packs

## Workload update process
- Update workload advertising manifests
- From IAL: Get list of workloads installed from workload installation record
- With rollback:
  - Install updated workload manifests
  - Run workload installation process for list of workloads that were installed
- If unit of installation is workload pack:
  - Run garbage collection to remove leftover workload packs
- If unit of installation is workload:
  - The IAL should handle uninstalling previous workloads from the same band (or treating the install of a newer version as a patch of the previous ones)

# Installer Abstraction API

- Get unit of installation: workloads or packs
- If unit of installation is workloads
  - Install workload
    - Optionally using offline installation cache folder
    - Should support transaction / rollback
  - Download workload artifacts to offline cache folder
  - Uninstall workload
  - List installed workloads
  - Not part of the API, but important for implementation: How to map from SDK workload ID to underlying installation ID and version?
- If unit of installation is workload pack
  - Install workload pack
    - Optionally using offline installation cache folder
    - Should support transaction / rollback
  - Download workload pack artifacts to offline cache folder
  - Garbage collect installed workload packs
- Install workload manifest (given a workload manifest ID and version)
  - Should support transaction / rollback
  - This may involve downloading a package or installer for the workload manifest
  - Should optionally support using offline installation cache folder
  - End result should be that the manifest under `<DOTNET ROOT>/sdk-manifests/<FEATURE VERSION BAND>/` is installed / updated
- Does IAL support installing or just giving instructions on how to install?
  - Initial installer implementations may all support installation directly, so this may not need to be included in the first version of the installer abstraction API
- Workload installation records
  - List installed workloads (SDK band)
  - Write installation record (workload ID, SDK band)
  - Delete installation record (workload ID, SDK band)

# Installer implementations

## Files on Disk IAL Implementation
 
- Unit of installation: Workload packs
- Install workload pack
  - Check if workload pack is already in the dotnet folder
    - Depending on package type, this may be a check for a folder or a nupkg file on disk
    - Note that we don't check the workload pack installation record here
    - If the workload pack is already installed, then just write the workload pack installation record, and skip the rest
  - Download NuGet package (eventually via new NuGet API)
    - NuGet package type should be DotnetPlatform (will this prevent projects / packages from incorrectly taking a dependency on them?)
    - NuGet package ID and version will be exactly the same as the workload pack ID and version
    - If an offline installation cache folder was specified, we'll look for the NuGet package there first before trying to download it
    - Otherwise, we'll use the NuGet APIs to download the NuGet package to a temporary folder (`%TEMP%/dotnetsdk`)
  - Determine target folder for NuGet package.  This depends on the pack type, and can be determined via the workload resolver
  - For Library and Template packs, copy nupkg to target folder
  - For other pack types, copy contents of `data/` folder in .nupkg archive to target folder
  - Write a workload pack installation record (see below)
  - To roll back an installation action, we simply delete the folder or nupkg file that we copied into the dotnet folder, and delete the workload pack installation record file
  - Cleanup: whether an install action is committed or rolled back, we will delete the nupkg file from the TEMP folder (if it wasn't already there when we started)
- Download to offline cache
  - This will use the same download mechanics as when installing a workload pack.  However, the target folder will be different (passed in via an argument) and the pack won't be installed to the dotnet folder
- Garbage collect: See below
- Install workload manifest
  - This implementation doesn't need to support downloading the .nupkg manifest, as the advertised manifest from the NuGet global packages folder suffices
  - Calculate workload manifest package ID: `<ManifestID>.<SdkFeatureBand>`
  - Look for specified version of workload manifest package in global packages folder
  - Copy contents of `data` folder from package to `<DOTNET ROOT>/sdk-manifests/<SdkFeatureBand>/<ManifestID>/`
    - To support rollback: first copy existing contents to temporary folder
    - To rollback: delete folder in SDK and copy contents back from temporary folder
- Read / write workload installation record
  - Installation records should be empty files with workload ID name under `sdk-manifests/<SdkFeatureBand>/.installedworkloads/`
  - Read: List files in band directory
  - Write: Create file
  - Delete: Delete file

### Workload Pack installation records and garbage collection

A workload pack installation record for a workload pack that is in use will be an empty file with the path `<DOTNET ROOT>/sdk-manifests/.installedpacks/v1/<Pack ID>/<Pack Version>/<SdkFeatureBand>/.active`.  The `v1` element in the path gives us some flexibility to change the format in a later version if we need to.  If the folder exists but the `.active` file doesn't, that indicates that the SDK Band no longer needs the workload pack, but that the files may be on disk and may be used by another band.  When there are no SDK bands with an `.active` file for the installation record, then the workload pack can be deleted from disk, and then the whole workload pack installation record folder can be deleted.

Thus, the garbage collection process will be as follows:

- Get installed SDK versions and map them to SDK feature bands (ie 6.0.200-preview6 or 6.0.205 would map to 6.0.200)
- Get installed workloads for current SDK feature band
- Using current workload manifest, map from currently installed workloads to all pack IDs and versions that should be installed for the current band
- For each pack / version combination where there is an installation record folder (ie `<DOTNET ROOT>/sdk-manifests/.installedpacks/v1/<Pack ID>/<Pack Version>/`):
  - Delete any folders that correspond to a band that is not an installed SDK
  - If there is a folder that corresponds to the current SDK band where the pack ID and version is not in the list that should be installed for the current band, delete the `.active` file (if it exists) from that folder
  - If none of the SDK feature band folders now contain a `.active` file, then the workload pack can be garbage collected
    - Delete the workload pack file or folder from disk
    - Delete the workload pack installation record folder: `<DOTNET ROOT>/sdk-manifests/.installedpacks/v1/<Pack ID>/<Pack Version>/`
 
## MSI (Standalone) IAL Implementation

Notes: This needs to be able to work well together with VS installations.  To do this, it should keep a record of the MSIs it installed (ie a workload pack installation record).  If it needs to install a pack that VS has already installed but that the Standalone Installer implementation hasn't, then it still needs to "install" that MSI or otherwise increment the MSI installation ref-count.

- Unit of installation: Workload packs
- Install workload pack
  - The MSI will be wrapped in a NuGet package
    - NuGet package ID will be `<WorkloadPackID>.Msi.<InstallerPlatform>`, version will match the workload pack version
    - The InstallerPlatform will be x64 or x86 (and possibly eventually arm64 and more).  It refers to which Program Files directory the assets will be installed to, not what architecture the assets inside the MSI are for (they will usually be architecture-neutral, and if they differ there will be different WorkloadPackIDs and the manifest RID aliases will handle that).
    - There should be a single MSI to install in the `data/` folder of the NuGet package.  The IAL implementation doesn't need to know the exact filename.
  - If MSI NuGet package is not in an offline cache, then download it (to temporary folder) via NuGet API
  - If MSI isn't installed, then 
    - Extract MSI from .nupkg to the MSI package cache
    - Install MSI
  - Add ref count for workload pack in registry.  Create `Dependents\Microsoft.NET.Sdk,<SDK Band>` under MSI ref counting key
  - Rollback:
    - Delete registry key for ref count
    - If MSI was installed as part of this operation
      - Uninstall MSI
      - Delete MSI from MSI package cache (this only happens for rollback, the MSI is left in the package cache if the install is successful)
    - If MSI NuGet package was downloaded, then delete it from the temporary folder
- Download to offline cache
  - Download the same MSI NuGet package that would be downloaded for workload pack install
- Garbage collect: See below
- Install workload manifest
  - Acquire workload manifest MSI
    - MSI will be wrapped in a NuGet package
      - NuGet package ID will be `<ManifestID>.<SdkFeatureBand>.Msi.<InstallerPlatform>`
      - There should be a single MSI to install in the `data/` folder of the NuGet package
    - Download workload manifest MSI NuGet package (to temporary folder)
  - Extract MSI to the package cache
  - Install MSI
  - Add ref count to workload manifest MSI: `HKLM\Software\Classes\Installer\Dependencies\<DependencyProviderKey>\Dependents\Microsoft.NET.Sdk,<SDK Band>`
  - ...
- Read / write workload installation record
- Workload installation record: Registry
  - `HKLM\SOFTWARE\Microsoft\dotnet\InstalledWorkloads\<SdkFeatureBand>\<WorkloadID>`

Garbage collection notes:
- Uninstall workload pack
  - Uninstall MSI
  - What do we need to do this?  Actual MSI on disk, or a Product GUID, or what?
  - Delete MSI from package cache

How do we track "active" workload packs?

### Workload pack MSIs
- MSI should create the following keys in the registry:
  - MSI Ref counting key: `HKLM\Software\Classes\Installer\Dependencies\<DependencyProviderKey>`
    - DependencyProviderKey should include the Pack ID, full semantic version of the workload pack, and InstallerPlatform.  The full semantic version should be used since multiple packs with the same ID but different versions can be installed side-by-side and should have separate ref counts
    - `InstallerPlatform` won't be part of the path, but 32-bit processes will end up reading / writing under the WOW6432Node key
  - Workload pack installation record: `HKLM\SOFTWARE\Microsoft\dotnet\InstalledPacks\<Pack ID>\<Pack Version>`
    - Create a value under the workload pack installation record with the name `DependencyProviderKey` and the DependencyProviderKey name from the ref counting key as the value, in order to link the workload pack installation record with the MSI ref counting key

### Workload manifest MSIs
- MSI should upgrade previous versions of the workload manifest MSI for the same workload manifest ID and SDK Feature Band
  - This is different than most MSIs for the .NET SDK, such as the workload packs, which install side-by-side
- MSI should create a Ref counting key: `HKLM\Software\Classes\Installer\Dependencies\<DependencyProviderKey>`
  - DependencyProviderKey should include the Manifest ID, SDK Feature Band, and the InstallerPlatform.  It should not include the workload manifest version, as updated versions of the manifest are installed in-place and should not be ref-counted separately

Ref counting for the workload manifest MSIs will work as follows:

- The baseline workload manifest MSI will be included in the standalone bundle.  Thus, when the bundle is installed, a ref from the bundle will be written under the Dependencies key
- When an updated manifest is installed by the MSI (Standalone) IAL implementation, it will write another ref under the Dependencies key (of the form `Microsoft.NET.Sdk,<SDK Band>`)
- If an updated manifest is installed by VS, it will also write a different ref under the dependencies key (of the form `VS.{GUID}`)
- When the standalone bundle is uninstalled, it will remove its ref count key, and then uninstall the MSI if a newer MSI hasn't been installed and there are no other ref counts
- The custom bundle finalizer will remove the ref created by the MSI (Standalone) IAL implementation, and uninstall the MSI if there are no other refs


## VS (Willow) IAL Implementation
- What's the priority on this?  Do we really need it (especially at first)?
 
## VS Installation
 
- Won't be going through normal workload flow, will just be installing the MSIs
- Needs to interact correctly with checking whether packs are installed, garbage collecting

# Notes

## Resolved 

- Update in separate spec or same? - Same
- How does the workload manifest get updated? - See update process
- For Standalone MSI installer implementation, we should ref-count per SDK band, which means we can handle garbage collection / cleanup even if we aren't the highest installed version
- TODO: How do we list all installed workload packs for .NET SDK managed install?  We may not have the manifests available for previously installed versions which left workload packs hanging around.

## Open / TODO

- How do we differentiate between 32 and 64 bit MSI installers?
- TODO: Spec out package names and versions for each installer abstraction
  - Integers for manifest versions?
- TODO: What do we call the files on disk / xcopy / universal installer?
- What happens when you have an standalone installation of the SDK that's being used in Visual Studio and you try to use VS's in product acquisition
- How to handle versioning of workload manifests versus workloads represented as packages (e.g. apt)
- How do we represent workload updates / versions to users?
- TODO: Figure out how aliases and RID-specific workload packs work with installer abstractions that operate on workloads
- IAL-specific mappings / data should go in manifest package installed by IAL, but not generic .nupkg
  - This probably doesn't work for all cases, for example the VS installer
- How to handle elevation for standalone installer
- Registry path for MSI workload pack installation record
- How to handle workload installation record for VS / Willow PAL?  We can't rely on writing / deleting the records in our installation process, as that process won't be used if installing with Willow.
- PM question: After update to SDK, when we realize that there are workloads you previously had installed, is it OK to suggest an install command that lists out all the workloads to install, or do we need a new command to "sync" to previously installed workloads
- Add option to allow creating offline cache for future SDK release band.  This is to support VS for Mac, and will be best effort (because the workload manifest format may change)
- Standardize on `SdkFeatureBand` for terminology
- Should we put "Manifest" in the NuGet package IDs for manifests to help disambiguate from workload packs?
- How do we get the SemVer version of a workload manifest NuGet package from just the installed version of the workload manifest?  It seems we either need to update the manifest schema to support a semantic version for the version, or we need to store the semantic version separately (ie in a text file).
- Need to figure out a way to generate an MSI version number for workload manifest MSI.  As much as possible, the MSI version number for manifest version A should be greater than the MSI version number for manifest version B if and only if the NuGet semantic version for version A of the manifest is greater than the NuGet semantic version for version B of the manifest
- How does MSI IAL know what the DependencyProviderKey for a workload manifest MSI is?  Convention?  File in workload manifest folder when installed by MSI?  File that is always in the workload manifest (including NuGet version)?
- How does VS / Willow IAL know which version of Visual Studio to use?


# Updates to existing behavior / specs
- TODO: We think yes: Should the SDK resolver look at the workload installation record before resolving a workload pack?  This would mean the workloads that are available to an SDK would match the user intent (recorded as the installation record) of what should be installed.  Otherwise workload functionality may light up because a different SDK installed the same workload packs, or because we couldn't actually garbage collect workload packs that were uninstalled in a lower SDK version.
- TODO: How do we handle baseline manifests when we update to a new SDK band?  We want the existing workloads to continue to work without requiring a workload to update the baseline manifest immediately.  However, for installers such as MSIs, the package owned by the workload will lay out the workload manifest in the feature band folder.  So when we update the SDK to feature band 6.0.200, the installers for the workload manifests will still lay them out in the 6.0.100 sdk-manifests folder.
  - Proposal: Have file listing included manifests IDs in `<DOTNET ROOT>/sdk-manifests/<FEATURE VERSION BAND>/`.  If workload resolver doesn't find a manifest for one of those IDs in the current feature band folder, it will look at previous feature band folders until it finds it, and use that.  (Should it error if not found?  Not sure)
- TODO: Workload resolver should ignore folders in the manifests folder that start with `.`, under both `<DOTNET ROOT>/sdk-manifests/` and `<DOTNET ROOT>/sdk-manifests/<FEATURE VERSION BAND>/`.  That way installer abstractions or other code can store information there, such as the workload installation records
- Version in workload manifest json should be a string to store semantic versions.  Workload manifest reader will need to be able to compare semantic versions when validating manifest dependency versions.


# Updates since review

- How to choose installer abstraction
- Details about how workload manifests are updated and installed
- Details about how to update workloads when SDK is updated (SDK update process)
- Updated installer abstraction API - offline installation cache support, rollback, installing workload manifest
- More details for IAL implementations
- IAL now in charge of garbage collection for workload packs.  Also files on disk implementation should keep a ref count per SDK band so cross-version cleanup works
- Updated workload installation process to update manifest automatically unless `--skip-manifest-update` is specified
- Added garbage collection process for .NET SDK managed installer
- Updated description of how to choose installer abstraction
- Added section on workload manifest update