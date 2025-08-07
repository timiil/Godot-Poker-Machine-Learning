# Godot‑Poker‑Machine‑Learning – CI build notes

These notes document how the **Godot‑Poker‑Machine‑Learning** project was built on a Linux container and exported as a Windows executable.  The instructions are useful for a CI pipeline and can be adapted for local development.

## 1. Environment and prerequisites

* **Platform:** Ubuntu‑like container with basic build tools and `wget`.  No extra system packages were required for this project because Godot provides a portable editor binary.
* **Godot version:** The project targets **Godot 4.2.x** (see `config/features` and `addons/gut/utils.gd`).  To export, we used **Godot v4.2.2‑stable** for Linux x86_64.
* **Export templates:** Godot requires platform‑specific export templates to perform exports.  The official documentation notes that “apart from setting up the platform, the export templates must be installed to be able to export projects; they can be obtained as a TPZ file (a renamed ZIP archive) from the [download page of the website]【420661731508317†L7660-L7663】.”

## 2. Installing Godot and export templates

1. **Download the editor binary.**  Use `wget` to fetch the Linux editor zip and extract it into a convenient location (e.g. `$HOME/godot-linux`).

   ```sh
   # Download and extract the portable editor (change version if necessary)
   wget -q https://github.com/godotengine/godot/releases/download/4.2.2-stable/Godot_v4.2.2-stable_linux.x86_64.zip
   unzip -q Godot_v4.2.2-stable_linux.x86_64.zip -d ~/godot-linux
   ```

2. **Download export templates.**  Grab the `.tpz` archive corresponding to the same version of Godot (4.2.2).  Extract it and copy the Windows templates into your Godot templates directory.  In our environment this directory was `~/.local/share/godot/export_templates/4.2.2.stable`.  Create it if it doesn’t exist.

   ```sh
   wget -q https://github.com/godotengine/godot/releases/download/4.2.2-stable/Godot_v4.2.2-stable_export_templates.tpz
   unzip -q Godot_v4.2.2-stable_export_templates.tpz -d /tmp/godot_templates

   mkdir -p ~/.local/share/godot/export_templates/4.2.2.stable
   cp /tmp/godot_templates/templates/windows_release_x86_64*.exe \
      ~/.local/share/godot/export_templates/4.2.2.stable/
   # optional: copy other platform templates if needed
   ```

   Installing the export templates once is sufficient for repeated exports on the same machine.

## 3. Creating an export preset

Godot stores export configuration in the project file **`export_presets.cfg`**.  The docs explain that this file “contains the vast majority of the export configuration and can be safely committed to version control【420661731508317†L7702-L7707】.”  Sensitive credentials (e.g. keystore passwords) live in `.godot/export_credentials.cfg`【420661731508317†L7708-L7712】 and should **not** be committed.

For this project we added the following preset to `repo/Godot-Project/export_presets.cfg`:

```ini
[preset.0]
name="Windows Desktop"
platform="Windows Desktop"
export_format="exe"
include_filter_list=PackedStringArray("*")
exclude_filter_list=PackedStringArray(".git/*")
export_path="build/windows/poker_machine.exe"
binary_format/embed_pck=false
file_name="build/windows/poker_machine.exe"
```

Key fields are:

- `name` – the preset name.  This must match exactly when exporting.
- `platform` – `"Windows Desktop"` for 64‑bit Windows.
- `export_format` – `exe` exports an `.exe` binary plus a `.pck` data file.
- `file_name` – relative output path.  The target directory (here `build/windows`) must exist before exporting.

Commit this file to the repository so that CI can read the preset.

## 4. Exporting from the command line

Godot can export a project non‑interactively.  According to the Godot documentation, exporting via the command line still requires an export preset and is invoked with `godot --export <preset> <path>` or `godot --export-release`【420661731508317†L7721-L7724】.  The documentation clarifies that the preset name should be quoted if it contains spaces and that the output path is relative to the project directory or absolute and must include the filename【420661731508317†L7730-L7734】.  The man page for `godot` reiterates that the preset must match one defined in `export_presets.cfg` and that the target directory must exist【887755800066852†L130-L135】.

In our build we executed the following from inside `repo/Godot-Project`:

```sh
# Ensure the output directory exists
mkdir -p build/windows

# Run export; --headless avoids opening the editor window
~/godot-linux/Godot_v4.2.2-stable_linux.x86_64 \
  --headless \
  --export-release "Windows Desktop" \
  build/windows/poker_machine.exe
```

The export process prints warnings but should produce two files in `build/windows`: `poker_machine.exe` and `poker_machine.pck`.  Godot will also generate a temporary file (ending in random letters) which can be removed.

## 5. Packaging the Windows build

The exported `.exe` and `.pck` files must reside together when distributing.  In CI we packaged them into a single archive:

```sh
cd build/windows
zip poker_machine_windows.zip poker_machine.exe poker_machine.pck
```

The resulting `poker_machine_windows.zip` can be uploaded to the GitHub release.

## 6. Notes for CI integration

* Install the Godot editor and export templates once in the CI environment.  Cache the `~/.local/share/godot/export_templates/<version>` directory to avoid re‑downloading templates on every run.
* When cloning the repository on a new runner, copy any `.godot/export_credentials.cfg` if it contains signing keys (e.g. for Android).  This file is not included in version control【420661731508317†L7708-L7713】.
* Use the `--export-release` flag for release builds; use `--export-debug` for debug builds.
* On CI, you can combine `--export` or `--export-release` with `--path` to specify the project directory instead of changing directories【420661731508317†L7761-L7764】.
* Always create the output directory before exporting; Godot will not create it automatically【887755800066852†L130-L135】.

## 7. Repository deliverables

Following the steps above on a clean container, we produced the Windows build.  The files are located in `repo/Godot-Project/build/windows`:

- `poker_machine.exe` – the Windows executable.
- `poker_machine.pck` – data file required by the executable.
- `poker_machine_windows.zip` – zipped archive containing both files, ready for release.

These deliverables can be attached to a GitHub release.  The `ci.md` file (this document) should be committed to the project wiki to describe the build process.
