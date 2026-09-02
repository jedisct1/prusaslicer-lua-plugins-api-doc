# Packaging and installation

Use an unpacked bundle while developing.
Create and sign a ZIP when you are ready to test the installer or distribute a release.

## Development installation

1. In PrusaSlicer, choose **Plugins > Show User Plugins Folder**.
2. Copy your whole bundle directory into that folder.
3. Choose **Plugins > Rescan Plugins**.
4. Run the command from the Plugins menu.

Directory scanning does not check signatures or checksums.
You can edit files in place.
Rescan after manifest or `info` changes and after adding or removing files.
PrusaSlicer rereads existing execution code when you click Run.

To remove a development bundle, delete its directory and rescan.
Alpha11 has no enable, disable, or uninstall command.

## Create a starter bundle

```sh
PrusaSlicer plugin init -d ./plugins com.example.my-plugin
```

The initializer asks for values that are not supplied as options, then creates:

```text
plugins/
  com.example.my-plugin/
    manifest.json
    hello.lua
```

Options:

| Option          | Meaning                                                                           |
| --------------- | --------------------------------------------------------------------------------- |
| `-d PATH`       | Parent directory where the bundle directory is created. Defaults to `.`.          |
| `-f`, `--force` | Remove an existing destination bundle directory and allow warned metadata values. |
| `bundle_id`     | Optional bundle ID. When omitted, the command asks for one.                       |

`--force` can delete an existing bundle directory at the destination.
Use it only when that exact directory is disposable or backed up.

## Generate an author key pair

```sh
PrusaSlicer plugin keygen \
  --public example-developer.pem \
  --private example-developer-private.pem
```

The default RSA key size is 2048 bits.
You may set a power-of-two size from 1024 through 8192:

```sh
PrusaSlicer plugin keygen \
  --keysize 4096 \
  --public example-developer.pem \
  --private example-developer-private.pem
```

The option names and generated files are correct.
In alpha11, the CLI help text accidentally describes `--public` as private and `--private` as public.

Keep the private key outside the bundle and user plugin directory.
Do not commit or distribute it.
The command restricts private-key file permissions to the current user where supported.

The public key may be distributed to testers.

`plugin keygen` overwrites both requested output paths without asking.
Use new filenames or back up existing key files before running it.

## Authorize the public key

The ZIP installer looks for:

```text
<PrusaSlicer user data>/authorized_authors/<manifest author>.pem
```

If the manifest contains:

```json
"author": "example-developer"
```

copy the public key as:

```text
authorized_authors/example-developer.pem
```

Alpha11 has no public-key import UI or CLI command.
Open the user plugin folder, move to its parent user data directory, and place the PEM in `authorized_authors` manually.

Never place the private key there.

## Sign and create the ZIP

Run this from the directory where you want the release ZIP to appear:

```sh
PrusaSlicer plugin sign \
  --private /secure/path/example-developer-private.pem \
  ./plugins/com.example.my-plugin
```

The command:

1. Parses and validates basic manifest metadata.
2. Hashes the bundle files with SHA-256.
3. Writes `manifest.txt` inside the bundle.
4. Signs `manifest.txt` and writes `manifest.sign` inside the bundle.
5. Creates `com.example.my-plugin.zip` in the current working directory.

`-f` or `--force` allows warned metadata values.
It does not make an invalid manifest or unreadable key valid.

Each signing run overwrites `manifest.txt` and `manifest.sign` inside the bundle.
It also replaces an existing `<bundle-id>.zip` in the current directory without asking, even when `--force` is absent.
Keep anything you need under different names or in another directory.

Release file and directory names inside the bundle may contain ASCII letters, digits, `.`, `-`, `_`, and spaces.
The signer rejects other path characters while creating the ZIP.

Sign only a clean release directory.
Every file becomes part of the strict checksum set, including editor backup files if you leave them there.

## ZIP layout

The archive must have `manifest.json` at its root:

```text
manifest.json
manifest.txt
manifest.sign
main.lua
assets/model.stl
```

Do not wrap these files in one extra directory inside the ZIP.
The `plugin sign` command creates the expected layout.

## Install through PrusaSlicer

1. Make sure the matching public key is in `authorized_authors`.
2. Choose **Plugins > Install Plugin Bundle**.
3. Select the signed ZIP.
4. Wait for the success or error notification.

The installer verifies:

- The author public key can be loaded.
- `manifest.sign` is a valid signature over `manifest.txt`.
- Every checksummed file matches.
- There are no unexpected unsigned files outside the checksum and signature whitelist.

On success, it installs the bundle at:

```text
<PrusaSlicer user data>/lua/<bundle id>
```

It then rescans plugins.

The success notification confirms signature verification and extraction.
It does not confirm that every Lua command registered successfully.
Check the Plugins menu and the application log after installation.

## Updating an installed bundle

Installing a ZIP with the same bundle ID replaces the existing directory.
The current replacement is not transactional:

1. PrusaSlicer removes the old installed directory.
2. It extracts the new bundle.

If extraction fails, the old copy is not restored automatically.
Keep release artifacts outside the managed plugin directory so you can reinstall a known-good version.

## What signing does and does not do

Signing protects the bundle contents handled by the ZIP installer.
It does not grant extra API permissions.
It also does not protect a directory after installation from local edits.
Normal directory rescans do not recheck signatures.

There is no current store submission, dependency, update feed, or automatic update API in alpha11.
