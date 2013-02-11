# XProtect Packager

This is a simple tool that builds OS X installer packages out of Apple's XProtect meta and data plists directly from Apple's client configuration plist. It can also import them into a Munki repository with appropriate OS version restrictions for each XProtect package.

In the near future I'd like the tool to cache all versions and allow you to diff the changes across versions, and provide more feedback about what has changed when new definitions are available.

## Background

XProtect is a mechanism used in OS X, starting in version 10.6, to maintain a list of known malware, as well as minimum versions for web plugins such as Flash and Java. The definitions are updated by the `XProtectUpdater` utility at startup and every 24 hours. In managed environments, it can be problematic when Apple decides to blacklist a version of a plugin within hours of a known security issue, and the latest version has only just become available (or, isn't available _at all_).

The definitions live in two files, `XProtect.meta.plist`, which contains metadata about the definitions and minimum plugin version requirements, and `XProtect.plist`, which contains malware signatures. The contents of these two files are made from a single configuration plist fetched by `XProtectUpdater`. Each version of OS X fetches its own plist with its own metadata version number. XProtect Packager generates these two plists. In my testing, there are zero differences between the plists written by `XProtectUpdater` and this tool.

It is useless to manage these files on clients if the updater mechanism will just overwrite them every day, so you will still need to disable the `com.apple.xprotectupdater` LaunchDaemon . Greg Neagle has a [pre-built package](http://managingosx.wordpress.com/2013/01/31/disabled-java-plugins-xprotect-updater) available to do that.

The package built by XProtect Packager contains only these two files, no postinstall scripts, identified as `com.github.ManagedXProtect` and is versioned in the form:

`METAVERSION.YYYY.MM.DD`, where:

* METAVERSION is the definition version listed in `XProtect.meta.plist`.
* YYYY.MM.DD is the date the definition plist was last modified by Apple (this is used by `XProtectUpdater` to determine whether its definitions are up to date).

The identifier 'com.github.ManagedXProtect' can be overridden by defaults (see below). The versioning scheme was chosen for the following reasons:

* METAVERSION should always increase with each metadata version, as `XProtectUpdater` checks this version for an incrementing value
* METAVERSION increases by 1000 with each major OS version, so a client whose OS has been upgraded will see the next meta version as a logical version upgrade
* the date of the last updated definition on a client can be easily audited with a command like: `pkgutil --pkg-info com.github.ManagedXProtect`

## Requirements

OS X 10.6-10.8, and the `pkgbuild` tool to be available at its standard location at `/usr/bin/pkgbuild`. This is standard on OS X 10.7/10.8, and part of the Xcode 3.2.6 or 4.x download for 10.6.

## Usage

All is done using the `xpp` script. Currently `--build` and `--munkiimport` are the two available "actions". Here are some examples:

`./xpp --build`

Builds packages for all OS X versions and records that these versions were built.

`./xpp --build --os 10.7 --os 10.8`

Builds for just 10.7 and 10.8.

`./xpp --force --output-dir /some/other/path`

Force building the most recent available version(s), using a specified output directory. By default, `xpp` outputs to the current working directory.

`./xpp --build --munkiimport`

Builds packages and imports into Munki in one step.

`./xpp --munkiimport`

Imports packages into Munki, assuming they were just built and are accessible at the output path.

`./xpp --help`

Get a list of all available options. Most options have a short form switch available, ie: `./xpp -bf`.

## Munki support

Packages imported into Munki automatically have their `minimum_os_version` and `maximum_os_version` keys set to restrict the package to the OS X version that matches the XProtect definition. With this restriction, you can use the same package name in all client manifests and not require conditional items, and Munki will install the correct version for the client OS. The name of the pkginfo item will be the last component of the package identifier. Extra options to `munkiimport` can be defined as well (see below).

Another approach to importing into Munki would be an option to build a `copy_from_dmg`-type item instead, which uses a `file`-type installs item (and thus an md5 checksum) on the plist files. This would allow the admin to manually roll back to any given version by simply making the desired version the highest version available in the Munki catalog, rather than being "stuck" with receipts and adding check scripts or fake higher versions in order to roll back a file. Feedback is welcome on such functionality, and how such behavior should be defined (command option, defaults pref, etc.).

## Customization

Besides the available command options, there are settings that can be defined in the defaults domain: `com.github.XProtectPackager`. Defining these in `/Library/Preferences/com.github.XProtectPackager` should also work if you'd like to define it at a system level.

Generally, these are options that you should only need to set once to suit your configuration (and you should _never_ change the package identifier once you've pushed versions to clients unless you have a very good reason).

### Package Identifier

Define your own package identifier with the `PackageIdentifier` key:

`defaults write com.github.XProtectPackager PackageIdentifier my.org.EXProtect`

### Options for munkiimport

Define any custom options to munkiimport by defining the `MunkiimportOptions` key as an array of strings, like so:

<pre>&lt;key&gt;MunkiimportOptions&lt;/key&gt;
&lt;array&gt;
    &lt;string&gt;--subdirectory&lt;/string&gt;
    &lt;string>my/own/place&lt;/string&gt;
&lt;/array&gt;</pre>

Options you do not need to override:

* `--minimum/maximum_os_version` (derived based the XProtect meta version)
* `--name` (derived from the last component of the package identifier)

The default options are: `--subdirectory support/ManagedXProtect --unattended_install --displayname [same as name]`
