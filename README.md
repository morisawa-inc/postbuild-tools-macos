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
usage: postbuild-codesign [--list] [--info] [--help] [--sign <identity>] [--entitlements <plist>] [--no-deep] [--no-runtime] [--no-preserve-metadata] [package-path]

options:

  --list                    list available identities
  --info, --info-full       display codesign info for given binary

  --sign <identity>         identity for codesigning
  --entitlements <plist>    entitlements plist for codesigning
  --no-deep                 don't codesign recursively
  --no-runtime              disable hardened runtime opt-in
  --no-preserve-metadata    don't preserve metadata
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

### Migrating to `notarytool`

The following steps are required to prefer `notarytool` instead of `altool` for notarization:

- Make a directory at `$HOME/.private_keys`
- Generate a API key on App Store Connect
- Put the private key to the file `$HOME/.private_keys/AuthKey_<your_api_key_goes_here>.p8`
- Make sure to export the following environment variables:
  - `AC_API_KEY`: API Key
  - `AC_API_ISSUER`: Issuer ID

To get the configuration right, I recommend to put a [Fastlane API Key JSON file](https://docs.fastlane.tools/app-store-connect-api#using-fastlane-api-key-json-file) under the name `$HOME/<your_api_key_goes_here>.json`, and run the folloing script at the shell startup:

```sh
export FASTLANE_JSON_KEY_FILE="$HOME/.<your_api_key_goes_here>.json"
export FASTLANE_JSON_KEY_DATA_RAW="$(cat "$FASTLANE_JSON_KEY_FILE")"
export AC_API_KEY="$(jq -r .key_id $FASTLANE_JSON_KEY_FILE)"
export AC_API_ISSUER="$(jq -r .issuer_id $FASTLANE_JSON_KEY_FILE)"
if [ ! -f "$HOME/.private_keys/AuthKey_${AC_API_KEY}.p8" ]; then
  mkdir -p "$HOME/.private_keys" && jq -r .key $FASTLANE_JSON_KEY_FILE > "$HOME/.private_keys/AuthKey_${AC_API_KEY}.p8"
fi
```

Note that the `jq` command must be installed.

The current state of the `notarytool` support is quite ad-hoc; the codebase of `postbuild-notarize` contains a recipe for `altool` but `altool` will get deprecated soon. The good news is that the notarization process using `notarytool` finishes much faster than before, and the command line interface is much more human-friendly. The workflow should be simplified when the `altool` support gets dropped in the script.

## Prerequisites

Tested on macOS 10.14.6. Make sure to install Command Line Tools for Xcode before running the scripts.

## License

TBD
