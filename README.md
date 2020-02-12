# postbuild-tools-macos

A set of automated codesign/notarization tools for distributing executables to the newer versions of macOS.

## Summary

These are the tools that makes the notarize process rather simple when `xcodebuild` is used in your project:

```sh
$ xcodebuild
$ postbuild-codesign
$ postbuild-notarize
```

As code signing and notarizaton are done according to the information in `.xcodeproj`, no options are needed to get the whole process done, unlike the other similar notarization-related scripts.

## postbuild-codesign

Automates codesign and make the binary ready for notarization.

```sh
$ postbuild-codesign --help
```

```sh
usage: postbuild-codesign [--list] [--info] [--help] [--sign <identity>] [--no-deep] [--no-runtime] [package-path]

options:

  --list                 list available identities
  --info, --info-full    display codesign info for given binary

  --sign <identity>      identity for codesigning
  --no-deep              don't codesign recursively
  --no-runtime           disable hardened runtime opt-in
```

In case you run this command without options, it extracts the product path from `.xcodeproj` in the current directory, performs a codesign with *Developer ID Application Certificate* found in the keychain, which is required to get successfully notarized by the Notary Service, and enables hardened runtime if needed. Another bonus of this command is that it contains a workaround for the `--deep` option of `codesign` – it fails to sign sub-packages contained in folders such as `*.app/Contents/Resources`.

Note that the whole process is completely optional if the Xcode project is configured to sign your products with proper configuration.

If you are working with the project that no `.xcodeproj` is involved, just give the path to the package you want to notarize at the end of the command-line options.

## postbuild-notarize

Automates notarization and waits until the stapling gets done.

```sh
$ postbuild-notarize --help
```

```sh
usage: postbuild-notarize [--store-password] [--list-providers] [--help] [package-path]

options:

  --username <name>    user name
  --provider <name>    short provider name

  --store-password     store password in keychain
  --list-providers     list available teams
  -h, --help           show this help
```

If you are going to notarize apps on your local machine, it is recommended to run the following command to store the password for your Apple ID in the keychain:

```sh
$ postbuild-notarize --store-password
```

Enter the username and the password that you use for authentication, and the credential will be stored in the keychain. Then, try running the command without options:

```sh
$ postbuild-notarize
```

Again, it automatically extracts the product path and the bundle identifier from `.xcodeproj` in the current directory, compresses the built product as a zip archive and uploads the payload onto the Notary Service by kicking `altool` in background. After the payload gets accepted by the server, start polling to the Notary Service until it gets notarized, and then staple the binary if the notarization is succeeded. The process usually takes a few minutes or so, and the notification banner will be popped up when finished if you are running the command within a GUI session.

In case you want to notarize apps on a CI environment, you can pass the credential via the environmental variables `$AC_USERNAME` and `$AC_PASSWORD` instead. Make sure to explicitly give which *App Store Connect Provider* to use via `$AC_PROVIDER` or `--provider` option if your account belongs to multiple teams. To find out the possible values, run `$ postbuild-notarize --list-providers` and check the column of `ProviderShortname`.

If you are working with the project that no `.xcodeproj` is involved, just give the path to the package you want to notarize at the end of the command-line options.

## Prerequisites

Tested on macOS 10.14.6. Make sure to install Command Line Tools for Xcode before running the scripts.

## License

TBD
