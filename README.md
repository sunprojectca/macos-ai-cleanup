Apple Intelligence cleanup and storage audit for macOS

This guide is for people who want to disable Apple Intelligence and determine how much local storage its models, caches, and related services use—without weakening macOS security or blindly deleting protected operating-system files.

The included Zsh script is conservative by design:

• Its default mode is read-only.
• It never disables SIP or authenticated-root protection.
• It never deletes anything under /System, /Library, or /private.
• Its optional cleanup moves matching user cache folders to the Trash so they can be restored.
• It inventories known and discovered AI model-asset directories, including directories that macOS may protect or download again.

> [!IMPORTANT]
> Apple does not provide a supported command-line uninstaller for Apple Intelligence model assets. Turning features off and deleting their files are separate things. A macOS update or asset-management service can restore files that were manually removed.

What to do first

Use this order:

1. Turn off Siri AI and the Apple Intelligence features you do not want.
2. Restart the Mac.
3. Run the script in its default audit mode.
4. If relevant user caches are found, run its reversible cleanup mode.
5. Restart again, empty the Trash only after confirming the Mac works normally, and audit again.
6. Consider Recovery or a downgrade only if the audit proves protected assets are using enough space to justify the risk and work.

Disable Apple Intelligence in macOS 27

Apple’s current macOS 27 guide directs users to turn off Siri AI and individual AI features. Labels can move in point releases, so use System Settings and the affected apps rather than relying on undocumented defaults keys.

Siri and ChatGPT

1. Open System Settings → Siri.
2. Choose Turn Off Siri to disable Siri AI, or turn Siri back on and select Use Siri Classic if that option is available.
3. Under Siri → Extensions → ChatGPT, turn off Use Extension.
4. If shown, also turn off ChatGPT setup prompts.

Other AI-assisted features

• Messages: Messages → Settings → turn off Summarize Messages.
• Mail summaries: Mail → Settings → Viewing → turn off Summarize Message Previews.
• Mail replies: Mail → Settings → Composing → turn off Personalize Smart Replies.
• Notifications: System Settings → Notifications → Summarize Notifications → off.
• Phone: Phone → Settings → Calls → turn off Suggestions in Voicemail.
• Journal: Journal → Settings → General → turn off Get Writing Prompts.
• Screen Time: Use Siri restrictions if you want to block access rather than merely hide individual features.
• Review per-app Siri access and suggestions if you want to limit the personal context available to Siri.

On macOS 15 or 26, the settings page may instead include a broader Apple Intelligence switch. Use that visible switch when available.

Official instructions: Turn off and restrict Apple Intelligence features on Mac.

The script

Save the following as apple-intelligence-audit.zsh.

```zsh
#!/bin/zsh

# Apple Intelligence storage audit and reversible user-cache cleanup.
# Default: read-only audit.
# No sudo. No SIP changes. No system-file deletion.

emulate -L zsh
setopt NO_UNSET
setopt PIPE_FAIL
setopt NULL_GLOB

MODE="audit"
DEEP_SCAN=0

usage() {
    cat <<'EOF'
Usage:
  zsh apple-intelligence-audit.zsh [options]

Options:
  --audit         Read-only audit (default)
  --clean-user    Move matching Apple user caches to the Trash
  --deep-scan     Also inspect common non-AI causes of large System Data
  --help          Show this help

Examples:
  zsh apple-intelligence-audit.zsh
  zsh apple-intelligence-audit.zsh --deep-scan
  zsh apple-intelligence-audit.zsh --clean-user
EOF
}

for arg in "$@"; do
    case "$arg" in
        --audit)
            MODE="audit"
            ;;
        --clean-user)
            MODE="clean-user"
            ;;
        --deep-scan)
            DEEP_SCAN=1
            ;;
        --help|-h)
            usage
            exit 0
            ;;
        *)
            print -u2 -- "Unknown option: $arg"
            usage >&2
            exit 2
            ;;
    esac
done

if [[ "$(uname -s)" != "Darwin" ]]; then
    print -u2 -- "This script is for macOS only."
    exit 1
fi

if (( EUID == 0 )); then
    print -u2 -- "Do not run this script with sudo or as root."
    print -u2 -- "Run it from your normal macOS account."
    exit 1
fi

typeset -r TIMESTAMP="$(date '+%Y%m%d-%H%M%S')"
typeset -a USER_CACHE_CANDIDATES
typeset -a SYSTEM_ASSET_CANDIDATES
typeset -A SEEN_USER
typeset -A SEEN_SYSTEM

heading() {
    print
    print -- "== $1 =="
}

show_size() {
    local path="$1"
    local size=""

    [[ -e "$path" ]] || return 0
    size="$(du -sh "$path" 2>/dev/null | awk '{print $1}')"
    [[ -n "$size" ]] || size="unreadable"
    printf '  %-12s %s\n' "$size" "$path"
}

add_user_candidate() {
    local path="$1"
    if [[ -z "${SEEN_USER[$path]-}" ]]; then
        SEEN_USER[$path]=1
        USER_CACHE_CANDIDATES+=("$path")
    fi
}

add_system_candidate() {
    local path="$1"
    if [[ -z "${SEEN_SYSTEM[$path]-}" ]]; then
        SEEN_SYSTEM[$path]=1
        SYSTEM_ASSET_CANDIDATES+=("$path")
    fi
}

looks_ai_related() {
    local name="${1:l}"
    [[ "$name" == *appleintelligence* ||
       "$name" == *intelligenceplatform* ||
       "$name" == *generative* ||
       "$name" == *imageplayground* ||
       "$name" == *siri* ||
       "$name" == *uaf* ]]
}

collect_user_caches() {
    local root path name
    local -a roots
    roots=(
        "$HOME/Library/Caches"
        "$HOME/Library/HTTPStorages"
    )

    for root in "${roots[@]}"; do
        [[ -d "$root" ]] || continue
        for path in "$root"/com.apple.*(N); do
            name="${path:t}"
            if looks_ai_related "$name"; then
                add_user_candidate "$path"
            fi
        done
    done
}

collect_system_assets() {
    local root path name
    local -a roots exact_paths

    roots=(
        "/System/Library/AssetsV2"
        "/Library/Apple/System/Library/AssetsV2"
        "/private/var/db/MobileAsset/AssetsV2"
    )

    exact_paths=(
        "/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"
        "/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"
        "/Library/Apple/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"
        "/Library/Apple/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"
        "/private/var/db/MobileAsset/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"
        "/private/var/db/MobileAsset/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"
        "/private/var/db/AppleIntelligencePlatform/AppModelAssets"
    )

    for path in "${exact_paths[@]}"; do
        [[ -e "$path" ]] && add_system_candidate "$path"
    done

    # Discover related asset families without assuming that names never change.
    for root in "${roots[@]}"; do
        [[ -d "$root" ]] || continue
        for path in "$root"/com_apple_MobileAsset_*(N); do
            name="${path:t}"
            if looks_ai_related "$name"; then
                add_system_candidate "$path"
            fi
        done
    done
}

print -- "Apple Intelligence storage audit"
print -- "Date:         $(date)"
print -- "macOS:       $(sw_vers -productVersion)"
print -- "Build:       $(sw_vers -buildVersion)"
print -- "Architecture: $(uname -m)"
print -- "Mode:         $MODE"

heading "macOS protection status"
csrutil status 2>/dev/null || print -- "  SIP status unavailable"
csrutil authenticated-root status 2>/dev/null || \
    print -- "  Authenticated-root status unavailable"
fdesetup status 2>/dev/null || print -- "  FileVault status unavailable"

collect_user_caches
collect_system_assets

heading "Matching user cache directories"
if (( ${#USER_CACHE_CANDIDATES[@]} == 0 )); then
    print -- "  None found in the selected cache roots."
else
    for path in "${USER_CACHE_CANDIDATES[@]}"; do
        show_size "$path"
    done
fi

heading "AI-related system and model assets (read-only)"
if (( ${#SYSTEM_ASSET_CANDIDATES[@]} == 0 )); then
    print -- "  No matching assets were visible in the known roots."
else
    for path in "${SYSTEM_ASSET_CANDIDATES[@]}"; do
        show_size "$path"
    done
fi

heading "Startup-volume free space"
df -h / | awk 'NR == 1 || NR == 2 {print "  " $0}'

if (( DEEP_SCAN == 1 )); then
    heading "Common non-AI storage consumers"
    typeset -a OTHER_PATHS
    OTHER_PATHS=(
        "$HOME/Library/Application Support/MobileSync/Backup"
        "$HOME/Library/Developer/Xcode/DerivedData"
        "$HOME/Library/Developer/Xcode/Archives"
        "$HOME/Library/Developer/CoreSimulator"
        "$HOME/Library/Containers/com.docker.docker"
        "$HOME/Library/Caches"
        "$HOME/Library/Logs"
        "/Library/Updates"
    )

    for path in "${OTHER_PATHS[@]}"; do
        show_size "$path"
    done

    heading "Local Time Machine snapshots"
    if command -v tmutil >/dev/null 2>&1; then
        tmutil listlocalsnapshots / 2>/dev/null | sed 's/^/  /'
    else
        print -- "  tmutil is unavailable."
    fi
fi

if [[ "$MODE" == "clean-user" ]]; then
    heading "Reversible user-cache cleanup"

    if (( ${#USER_CACHE_CANDIDATES[@]} == 0 )); then
        print -- "  Nothing to move."
    elif [[ ! -t 0 ]]; then
        print -u2 -- "Interactive confirmation is required."
        exit 1
    else
        print -- "The directories listed below will be moved to the Trash."
        print -- "No system or model-asset directory will be touched."
        print
        for path in "${USER_CACHE_CANDIDATES[@]}"; do
            print -- "  $path"
        done
        print
        read -r "REPLY?Continue? Type YES to proceed: "

        if [[ "$REPLY" != "YES" ]]; then
            print -- "Cleanup cancelled."
            exit 0
        fi

        typeset -r TRASH_ROOT="$HOME/.Trash/Apple-Intelligence-Cleanup-$TIMESTAMP"
        mkdir -p "$TRASH_ROOT"

        for path in "${USER_CACHE_CANDIDATES[@]}"; do
            [[ -e "$path" ]] || continue
            relative="${path#$HOME/}"
            target="$TRASH_ROOT/$relative"
            mkdir -p "${target:h}"
            if command mv "$path" "$target"; then
                print -- "Moved: $path"
            else
                print -u2 -- "Could not move: $path"
            fi
        done

        print
        print -- "Moved caches are in:"
        print -- "  $TRASH_ROOT"
        print -- "Restart the Mac and test it before emptying the Trash."
    fi
else
    heading "Audit complete"
    print -- "No files were changed."
    print -- "For reversible cache cleanup, run:"
    print -- "  zsh apple-intelligence-audit.zsh --clean-user"
fi
```

Run the script

Open Terminal, change to the folder containing the file, and run:

```bash
chmod +x apple-intelligence-audit.zsh
zsh apple-intelligence-audit.zsh
```

The explicit zsh command works even if another shell is currently active.

For the additional System Data checks:

```bash
zsh apple-intelligence-audit.zsh --deep-scan
```

For reversible user-cache cleanup:

```bash
zsh apple-intelligence-audit.zsh --clean-user
```

The cleanup requires you to type uppercase YES. It moves matching Apple cache directories into a timestamped folder in the Trash. It does not immediately reclaim that space; restart and test first, then empty the Trash when satisfied.

Full Disk Access

The audit can run without Full Disk Access, but macOS may report some directories as unreadable.

If you want a more complete report:

1. Open System Settings → Privacy & Security → Full Disk Access.
2. Enable the terminal application you are using.
3. Quit and reopen Terminal.
4. Run the audit again.

Do not run the script with sudo. Root execution can point $HOME at the wrong account and defeats the script’s user-only safety boundary.

Reading the results

Large paths under AssetsV2

These are managed assets rather than ordinary application caches. The audit reports them but deliberately does not delete them. They may be protected by SIP, the Signed System Volume, or asset-management rules, and macOS may redownload them.

Commonly reported families include:

```text
com_apple_MobileAsset_UAF_FM_GenerativeModels
com_apple_MobileAsset_UAF_FM_Visual
```

Names and locations are implementation details and can change in any update. The script therefore checks a small known list and discovers related names inside narrowly scoped asset roots.

“Unreadable” or no result

This does not prove the directory is absent. It can mean Terminal lacks Full Disk Access or macOS denied traversal. Granting Full Disk Access improves inspection; it does not override SIP.

Storage does not update immediately

Finder and System Settings can take time to recalculate categories. After cleanup:

1. Restart the Mac.
2. Empty the Trash only after testing.
3. Wait for Storage settings to recalculate.
4. Run the same audit again and compare exact paths.

Recovery: what it means

Recovery is a separate maintenance environment. Entering Recovery does not reinstall macOS. Reinstallation occurs only if you deliberately select Reinstall macOS and continue through it.

On Apple silicon:

1. Shut down the Mac.
2. Press and hold the power button until startup options appear.
3. Choose Options → Continue.
4. Authenticate if asked.
5. Open Utilities → Terminal from the menu bar.

Apple’s official instructions: Start up from macOS Recovery.

Read-only Recovery inspection

Volume names vary, especially when FileVault is enabled. Inspect them first:

```bash
diskutil apfs list
ls -la /Volumes
```

After identifying the correct mounted Data volume, inspection commands can look like this:

```bash
ls -la "/Volumes/Macintosh HD - Data/System/Library/AssetsV2"
du -sh "/Volumes/Macintosh HD - Data/System/Library/AssetsV2"/*UAF* 2>/dev/null
```

These commands only list and measure files. This README intentionally does not provide an automated Recovery deletion command because the correct volume, protection state, and asset paths must be verified on that exact Mac. A broad rm -rf mistake in Recovery can make the system unbootable.

Should you disable SIP?

Not for this cleanup. SIP and authenticated-root protection are core macOS security controls. Saving several gigabytes is generally not worth permanently weakening them, and disabling them still does not guarantee that macOS will not redownload managed assets.

The script reports their status so an accidental disabled state is visible. It never changes either control.

Downgrading from macOS 27 to Sequoia

Downgrading to an older major macOS release should be treated as an erase-and-migrate operation, not as a normal in-place update. Recovery normally reinstalls the current version of the most recently installed macOS; it is not automatically a downgrade. Apple documents this behavior in How to reinstall macOS.

Before a downgrade:

• Make a fresh Time Machine backup.
• Make an independent copy of irreplaceable files.
• Copy the entire Photos library while Photos is closed.
• If iCloud Photos uses Optimize Mac Storage, choose Download Originals to this Mac and allow it to finish before relying on the local library copy.
• Export irreplaceable originals from Photos as an additional safeguard.
• Verify that the target macOS supports the Mac model.
• Confirm that critical applications, plug-ins, audio drivers, and project formats work on the older release.
• Record licenses, recovery keys, VPN settings, and other configuration that may not migrate cleanly.
• Expect a Photos library upgraded by a newer Photos version to be incompatible with an older Photos version. Preserve the pre-upgrade library or exported originals.

Apple’s bootable-installer instructions are here: Create a bootable installer for macOS.

Other causes of large “System Data”

Apple Intelligence may be visible because of the timing of an upgrade, but it is not always the largest consumer. Run --deep-scan and check these separately:

• Time Machine local snapshots.
• iPhone and iPad device backups.
• macOS update downloads and incomplete installers.
• Xcode DerivedData, archives, simulators, and device-support files.
• Docker or virtual-machine disk images.
• Mail attachments and message downloads.
• Browser profiles and caches.
• Audio sample libraries, DAW render caches, and plug-in installers.
• Photos originals, derivatives, and local iCloud downloads.
• Cloud-storage files marked to remain downloaded.
• Large log files after a crashing or looping service.
• APFS snapshots created by updates or backup software.

Also inspect System Settings → General → Storage before deleting anything. Avoid generic “cleaner” commands that erase all of /Library/Caches, /private/var, or AssetsV2; those locations contain unrelated operating-system and application data.

Restore caches moved by the script

Normally macOS recreates caches automatically. If an affected app behaves incorrectly, open the timestamped folder in the Trash and move its contents back to the original hierarchy under your home folder. Do this while the affected apps are closed, then restart.

Example Trash folder:

```text
~/.Trash/Apple-Intelligence-Cleanup-20260917-143000/
```

If everything works after a restart, emptying the Trash permanently removes those cache copies.

Troubleshooting

The model assets returned after reboot

This generally means macOS still considers them required. Recheck that Siri AI and the individual AI features are off. A script cannot promise permanent removal while macOS continues to manage those assets.

The script finds little, but System Data is huge

Run --deep-scan, inspect local snapshots, and review Storage settings. “System Data” is a broad accounting category, not a directory named System Data.

The script says SIP is disabled

Do not continue with protected-file deletion. If you did not intentionally disable SIP, restart into Recovery, open Terminal, run csrutil enable, and restart. Recheck the status from normal macOS.

Photos appears smaller after changing settings

Do not assume files were safely backed up. With optimized iCloud storage, the Mac may hold only selected local originals. Confirm iCloud synchronization status and make a separate verified backup before downgrading, erasing, or changing libraries.

Safety summary

|Operation                                 |Risk            |Included here                      |
|------------------------------------------|---------------:|-----------------------------------|
|Turn off AI features in supported settings|Low             |Instructions                       |
|Audit model and cache sizes               |Low             |Yes; default mode                  |
|Move user cache directories to Trash      |Low to moderate |Yes; explicit mode and confirmation|
|Delete managed `AssetsV2` content         |High/unsupported|No                                 |
|Disable SIP or authenticated root         |High            |No                                 |
|Enter Recovery for inspection             |Low if read-only|Instructions                       |
|Erase and downgrade macOS                 |High            |Planning notes only                |

Official references

• Turn off and restrict access to Apple Intelligence features on Mac
• Turn off ChatGPT on Mac
• Start up from macOS Recovery
• How to reinstall macOS
• Create a bootable installer for macOS
• Back up the Photos library on Mac

Last reviewed for macOS 27 Golden Gate on September 17, 2026. Apple can change settings, asset names, and protection behavior in point releases; rerun the audit after major updates and prefer Apple’s visible controls over undocumented preference keys.
