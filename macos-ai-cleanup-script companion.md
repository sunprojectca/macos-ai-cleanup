Apple Intelligence audit script: line-by-line explanation

Companion for macos-ai-cleanup • Source script: apple-intelligence-audit.zsh • Prepared September 17, 2026

This document explains all 280 lines of the audit script supplied in the original Apple-Intelligence-Cleanup-README.md, including comments, blank lines, options, functions, and the optional cache-moving branch.

Source note: The current saved README contained only #  when checked for this companion. The explanations below therefore use the earlier 280-line script supplied in this conversation and its retained original copy. The current README has not been overwritten or restored.

Correction to the earlier script: it has a confirmed Zsh variable-name bug. It passed a syntax check, but that did not establish correct runtime behavior. Treat the code below as a reference for explanation, and correct the issues described here before using its cleanup mode.

The confirmed runtime bug

Lowercase path is a special Zsh array linked to the PATH command-search setting. The script uses it for ordinary file paths in function declarations and loops. That changes where external programs are looked up. Declaring it local this way does not remove its special behavior. See the official Zsh parameter documentation.

In a separate local Zsh process, calling the original show_size function on /dev/null produced:

```text
show_size:5: command not found: awk
  unreadable   /dev/null
```

That is an actual reproduction using this function, not a report from your Mac. It explains why unreadable output can be caused by the script itself. Depending on the branch taken, later df, mkdir, mv, and other external commands can also fail to resolve.

The required naming correction is to use an ordinary name such as scan_path consistently in declarations, loops, expansions, and map keys. Merely changing one declaration is insufficient. The line table preserves the original text so it remains possible to compare it with the version already supplied.

What the script is intended to do

|Phase                                 |Intended result                                                              |Filesystem change                             |
|--------------------------------------|-----------------------------------------------------------------------------|----------------------------------------------|
|Parse arguments and check the platform|Select audit/cleanup and reject normal execution outside macOS or as root    |None                                          |
|Collect candidates                    |Build a user-item list and a separate system-asset list                      |None                                          |
|Print the report                      |Show candidate sizes, protection status, and startup-volume space            |None                                          |
|Optional `--deep-scan`                |Inspect a fixed list of other storage locations and list local snapshot names|None                                          |
|Optional `--clean-user` plus `YES`    |Move user candidates under a timestamped directory in `.Trash`               |Creates directories and moves selected entries|

The code does not disable Apple Intelligence, write preferences, remove system models, change SIP, restart the Mac, empty the Trash, or schedule later runs. Its system-asset list is used only for reporting. The optional move is intended to allow manual recovery while its destination still exists; it is not a backup or an automatic undo mechanism.

How to read the table

Line 1 is the shebang, #!/bin/zsh, in the script itself. Markdown fence lines and the surrounding README are excluded. Blank script lines are included. The source column omits indentation for readability; every physical source line has its own numbered row.

The explanation describes both the intended operation and any concrete limitation of the implementation. A message saying ‘complete’ or ‘moved caches are in’ is just output; it is not a separate verification step.

Lines 1–14: Interpreter, shell options, and defaults

|Line|Original source                                                        |Explanation                                                                                                                                                                      |
|---:|-----------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|1   |`#!/bin/zsh`                                                           |Selects `/bin/zsh` when the executable file is launched directly. Running `zsh filename` selects Zsh explicitly as well.                                                         |
|2   |*(blank)*                                                              |Blank separator; no operation.                                                                                                                                                   |
|3   |`# Apple Intelligence storage audit and reversible user-cache cleanup.`|Comment describing the intended audit and recoverable cache-moving behavior. A comment does not enforce these properties.                                                        |
|4   |`# Default: read-only audit.`                                          |Comment stating the default mode; line 12 implements that default.                                                                                                               |
|5   |`# No sudo. No SIP changes. No system-file deletion.`                  |Comment stating the intended limits. The code contains no `sudo`, protection-changing commands, or recursive delete.                                                             |
|6   |*(blank)*                                                              |Blank separator; no operation.                                                                                                                                                   |
|7   |`emulate -L zsh`                                                       |Uses Zsh emulation defaults. `-L` localizes option, pattern and trap changes to a surrounding function, if any; here the code is at script level. This does not create a sandbox.|
|8   |`setopt NO_UNSET`                                                      |Makes expansion of unset parameters an error. It helps catch misspelled variables; it does not validate filesystem targets.                                                      |
|9   |`setopt PIPE_FAIL`                                                     |Makes a failed component affect a pipeline’s exit status. The script does not enable automatic exit on command failure, so this alone will not stop it.                          |
|10  |`setopt NULL_GLOB`                                                     |Makes unmatched filename patterns expand to no arguments instead of raising a no-match error.                                                                                    |
|11  |*(blank)*                                                              |Blank separator; no operation.                                                                                                                                                   |
|12  |`MODE="audit"`                                                         |Sets the starting mode to `audit`. No command-line options are needed for that mode.                                                                                             |
|13  |`DEEP_SCAN=0`                                                          |Starts with the additional storage checks disabled.                                                                                                                              |
|14  |*(blank)*                                                              |Blank separator; no operation.                                                                                                                                                   |

Lines 15–32: Help text

|Line|Original source                                                         |Explanation                                                                                                    |
|---:|------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
|15  |`usage() {`                                                             |Defines the `usage` function. Its body runs when called at line 45 or 50.                                      |
|16  |`cat <<'EOF'`                                                           |Starts a literal here-document passed to `cat`. Quoting `EOF` prevents shell expansion of the help text.       |
|17  |`Usage:`                                                                |Help text: prints the usage heading.                                                                           |
|18  |`zsh apple-intelligence-audit.zsh [options]`                            |Help text: shows the filename and optional arguments. This line is printed, not executed.                      |
|19  |*(blank)*                                                               |Blank line inside the here-document; it becomes a blank line in the help output.                               |
|20  |`Options:`                                                              |Help text: prints the options heading.                                                                         |
|21  |`--audit         Read-only audit (default)`                             |Help text: describes the audit option.                                                                         |
|22  |`--clean-user    Move matching Apple user caches to the Trash`          |Help text: describes the intended user-cache move. Candidate selection is broad, as explained at lines 104–131.|
|23  |`--deep-scan     Also inspect common non-AI causes of large System Data`|Help text: describes additional storage checks. They are a fixed list, not a scan of the whole disk.           |
|24  |`--help          Show this help`                                        |Help text: describes the help option.                                                                          |
|25  |*(blank)*                                                               |Blank line printed in the help output.                                                                         |
|26  |`Examples:`                                                             |Help text: prints the examples heading.                                                                        |
|27  |`zsh apple-intelligence-audit.zsh`                                      |Help text: shows an audit invocation; does not run it.                                                         |
|28  |`zsh apple-intelligence-audit.zsh --deep-scan`                          |Help text: shows an audit with extra storage checks; does not run it.                                          |
|29  |`zsh apple-intelligence-audit.zsh --clean-user`                         |Help text: shows a cleanup invocation; does not run it.                                                        |
|30  |`EOF`                                                                   |Ends the here-document. This delimiter is not part of the displayed help.                                      |
|31  |`}`                                                                     |Ends the `usage` function definition.                                                                          |
|32  |*(blank)*                                                               |Blank separator; no operation.                                                                                 |

Lines 33–55: Command-line options

|Line|Original source                      |Explanation                                                                                                                |
|---:|-------------------------------------|---------------------------------------------------------------------------------------------------------------------------|
|33  |`for arg in "$@"; do`                |Loops over each command-line argument. Quoted `"$@"` keeps each original argument intact, including spaces.                |
|34  |`case "$arg" in`                     |Starts a `case` selection for the current argument.                                                                        |
|35  |`--audit)`                           |Matches the exact `--audit` option.                                                                                        |
|36  |`MODE="audit"`                       |Sets audit mode. When both mode flags are supplied, the last one encountered wins.                                         |
|37  |`;;`                                 |Ends this case branch.                                                                                                     |
|38  |`--clean-user)`                      |Matches the exact `--clean-user` option.                                                                                   |
|39  |`MODE="clean-user"`                  |Selects the cleanup branch that starts at line 232; this does not move files yet.                                          |
|40  |`;;`                                 |Ends this case branch.                                                                                                     |
|41  |`--deep-scan)`                       |Matches the exact `--deep-scan` option.                                                                                    |
|42  |`DEEP_SCAN=1`                        |Enables the extra checks at lines 206–230. It can accompany either mode.                                                   |
|43  |`;;`                                 |Ends this case branch.                                                                                                     |
|44  |`--help|-h)`                         |Matches either `--help` or `-h`; here the vertical bar separates alternative patterns.                                     |
|45  |`usage`                              |Calls `usage` to print the help text.                                                                                      |
|46  |`exit 0`                             |Exits successfully immediately. The macOS and root checks below are not reached when help is requested.                    |
|47  |`;;`                                 |Terminates the help branch syntactically; the preceding exit has already ended execution.                                  |
|48  |`*)`                                 |Matches any argument not handled by earlier branches.                                                                      |
|49  |`print -u2 -- "Unknown option: $arg"`|Prints the unknown-argument error to standard error. `-u2` selects file descriptor 2; `--` ends option parsing for `print`.|
|50  |`usage >&2`                          |Prints help to standard error through the `>&2` redirection.                                                               |
|51  |`exit 2`                             |Exits with status 2 to indicate an invalid command-line option.                                                            |
|52  |`;;`                                 |Ends the invalid-option branch syntactically.                                                                              |
|53  |`esac`                               |Ends the case selection.                                                                                                   |
|54  |`done`                               |Ends the argument loop.                                                                                                    |
|55  |*(blank)*                            |Blank separator; no operation.                                                                                             |

Lines 56–66: Platform and user checks

|Line|Original source                                              |Explanation                                                                                                                                |
|---:|-------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
|56  |`if [[ "$(uname -s)" != "Darwin" ]]; then`                   |Runs `uname -s` and checks that the result is `Darwin`, the expected kernel name on macOS. This does not detect a particular macOS release.|
|57  |`print -u2 -- "This script is for macOS only."`              |Prints the platform error to standard error.                                                                                               |
|58  |`exit 1`                                                     |Exits with failure status 1 on the platform-error path.                                                                                    |
|59  |`fi`                                                         |Ends the platform check.                                                                                                                   |
|60  |*(blank)*                                                    |Blank separator; no operation.                                                                                                             |
|61  |`if (( EUID == 0 )); then`                                   |Checks the effective user ID numerically. Zero means root; reading `EUID` does not change identity.                                        |
|62  |`print -u2 -- "Do not run this script with sudo or as root."`|Prints the instruction not to run as root or through `sudo`.                                                                               |
|63  |`print -u2 -- "Run it from your normal macOS account."`      |Prints the normal-account instruction.                                                                                                     |
|64  |`exit 1`                                                     |Exits with status 1 when root execution was detected.                                                                                      |
|65  |`fi`                                                         |Ends the root check. Normal execution is expected from a logged-in macOS user.                                                             |
|66  |*(blank)*                                                    |Blank separator; no operation.                                                                                                             |

Lines 67–77: Shared data and the heading helper

|Line|Original source                                  |Explanation                                                                                                                                                            |
|---:|-------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|67  |`typeset -r TIMESTAMP="$(date '+%Y%m%d-%H%M%S')"`|Captures the current local date and time once and marks it read-only. The format is year, month, day, hyphen, hour, minute, second; it will name the Trash destination.|
|68  |`typeset -a USER_CACHE_CANDIDATES`               |Declares an indexed array for the selected user items. In normal Zsh emulation it starts empty.                                                                        |
|69  |`typeset -a SYSTEM_ASSET_CANDIDATES`             |Declares an indexed array for system asset candidates. The cleanup branch never iterates this array for moves.                                                         |
|70  |`typeset -A SEEN_USER`                           |Declares an associative array used as a set of user paths already added.                                                                                               |
|71  |`typeset -A SEEN_SYSTEM`                         |Declares a separate associative array used as a set of system paths already added.                                                                                     |
|72  |*(blank)*                                        |Blank separator; no operation.                                                                                                                                         |
|73  |`heading() {`                                    |Defines the report-heading helper.                                                                                                                                     |
|74  |`print`                                          |Prints a blank line when the helper runs.                                                                                                                              |
|75  |`print -- "== $1 =="`                            |Prints its first argument between `==` markers.                                                                                                                        |
|76  |`}`                                              |Ends the heading helper.                                                                                                                                               |
|77  |*(blank)*                                        |Blank separator; no operation.                                                                                                                                         |

Lines 78–103: Size reporting and duplicate suppression

|Line|Original source                                          |Explanation                                                                                                                                                                 |
|---:|---------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|78  |`show_size() {`                                          |Defines the helper intended to measure and print one item’s disk usage.                                                                                                     |
|79  |`local path="$1"`                                        |BUG: tries to store the argument in `path`, but that name is Zsh’s special command-search array tied to `PATH`. The assignment disrupts command lookup within this function.|
|80  |`local size=""`                                          |Initializes a local string to hold the size result.                                                                                                                         |
|81  |*(blank)*                                                |Blank separator; no operation.                                                                                                                                              |
|82  |`[[ -e "$path" ]] || return 0`                           |Skips an item if the existence test fails, returning success. Permission-denied traversal and dangling links can also fail this test; no warning is produced.               |
|83  |`size="$(du -sh "$path" 2>/dev/null | awk '{print $1}')"`|Intends to run `du -sh` for a readable size and use `awk` to keep its first field. It suppresses `du` errors. The `path` bug can prevent both programs from being located.  |
|84  |`[[ -n "$size" ]] || size="unreadable"`                  |Uses `unreadable` only when no size text was captured. A partial size from a failing `du` can still be printed as if complete.                                              |
|85  |`printf '  %-12s %s\n' "$size" "$path"`                  |Prints a left-aligned size field at least 12 characters wide, followed by the path and a newline.                                                                           |
|86  |`}`                                                      |Ends the size helper; the status of its last command can hide earlier measurement failures.                                                                                 |
|87  |*(blank)*                                                |Blank separator; no operation.                                                                                                                                              |
|88  |`add_user_candidate() {`                                 |Defines a helper to add a user candidate without repeating exactly the same path string.                                                                                    |
|89  |`local path="$1"`                                        |BUG: again uses the special Zsh `path` parameter as a local argument variable.                                                                                              |
|90  |`if [[ -z "${SEEN_USER[$path]-}" ]]; then`               |Checks whether the map entry is empty or missing; the trailing `-` supplies an empty fallback for an unset key.                                                             |
|91  |`SEEN_USER[$path]=1`                                     |Marks this exact string as already seen.                                                                                                                                    |
|92  |`USER_CACHE_CANDIDATES+=("$path")`                       |Appends the path to the user candidate array without splitting a path that contains spaces.                                                                                 |
|93  |`fi`                                                     |Ends the not-seen check.                                                                                                                                                    |
|94  |`}`                                                      |Ends the user-candidate helper.                                                                                                                                             |
|95  |*(blank)*                                                |Blank separator; no operation.                                                                                                                                              |
|96  |`add_system_candidate() {`                               |Defines the equivalent helper for system asset candidates.                                                                                                                  |
|97  |`local path="$1"`                                        |BUG: again stores an ordinary value in the special `path` parameter.                                                                                                        |
|98  |`if [[ -z "${SEEN_SYSTEM[$path]-}" ]]; then`             |Checks whether this exact system-path string has already been recorded.                                                                                                     |
|99  |`SEEN_SYSTEM[$path]=1`                                   |Marks the system path as seen.                                                                                                                                              |
|100 |`SYSTEM_ASSET_CANDIDATES+=("$path")`                     |Appends it to the system asset array.                                                                                                                                       |
|101 |`fi`                                                     |Ends the not-seen check.                                                                                                                                                    |
|102 |`}`                                                      |Ends the system-candidate helper.                                                                                                                                           |
|103 |*(blank)*                                                |Blank separator; no operation.                                                                                                                                              |

Lines 104–113: Filename matching

|Line|Original source                       |Explanation                                                                                                                 |
|---:|--------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
|104 |`looks_ai_related() {`                |Defines a filename-matching predicate. Its result is an exit status used by `if`; it prints nothing.                        |
|105 |`local name="${1:l}"`                 |Lowercases its first argument using the Zsh `:l` modifier so the following checks ignore letter case.                       |
|106 |`[[ "$name" == *appleintelligence* ||`|Starts a condition looking for `appleintelligence` anywhere in the name. Each `||` means another alternative can also match.|
|107 |`"$name" == *intelligenceplatform* ||`|Also accepts names containing `intelligenceplatform`.                                                                       |
|108 |`"$name" == *generative* ||`          |Also accepts names containing `generative`.                                                                                 |
|109 |`"$name" == *imageplayground* ||`     |Also accepts names containing `imageplayground`.                                                                            |
|110 |`"$name" == *siri* ||`                |Also accepts names containing `siri`, which does not establish that the contents are Apple Intelligence-only caches.        |
|111 |`"$name" == *uaf* ]]`                 |Also accepts names containing `uaf`, then closes the condition. These are name heuristics, not content or ownership checks. |
|112 |`}`                                   |Ends the predicate; its last condition supplies success for a match and failure otherwise.                                  |
|113 |*(blank)*                             |Blank separator; no operation.                                                                                              |

Lines 114–132: User candidate discovery

|Line|Original source                         |Explanation                                                                                                                                                                                          |
|---:|----------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|114 |`collect_user_caches() {`               |Defines collection of user items from two directory roots.                                                                                                                                           |
|115 |`local root path name`                  |Declares locals for this function. BUG: `path` still retains its special Zsh meaning when localized this way.                                                                                        |
|116 |`local -a roots`                        |Declares the local array of roots.                                                                                                                                                                   |
|117 |`roots=(`                               |Starts assigning the roots to inspect.                                                                                                                                                               |
|118 |`"$HOME/Library/Caches"`                |Selects the current user’s top-level Library cache directory.                                                                                                                                        |
|119 |`"$HOME/Library/HTTPStorages"`          |Also selects the user’s HTTP storage directory. The code does not establish that every matching item here is a disposable cache.                                                                     |
|120 |`)`                                     |Ends the root array assignment.                                                                                                                                                                      |
|121 |*(blank)*                               |Blank separator; no operation.                                                                                                                                                                       |
|122 |`for root in "${roots[@]}"; do`         |Visits each of the two roots, preserving whitespace in each path.                                                                                                                                    |
|123 |`[[ -d "$root" ]] || continue`          |Skips a root when it is not visible as a directory; there is no separate missing-versus-inaccessible report.                                                                                         |
|124 |`for path in "$root"/com.apple.*(N); do`|Expands immediate children whose names start with `com.apple.`. `(N)` permits no matches. There is no directory-only or symlink-rejection qualifier. BUG: assigning the loop variable changes `path`.|
|125 |`name="${path:t}"`                      |Extracts the last pathname component using `:t`; it does not inspect the item’s contents.                                                                                                            |
|126 |`if looks_ai_related "$name"; then`     |Applies the name heuristic to that component.                                                                                                                                                        |
|127 |`add_user_candidate "$path"`            |Adds a matching entry to the user list. This same list is later used for moves, without a separate cleanup allowlist.                                                                                |
|128 |`fi`                                    |Ends the name-match check.                                                                                                                                                                           |
|129 |`done`                                  |Ends the immediate-child loop; this enumeration itself is not recursive.                                                                                                                             |
|130 |`done`                                  |Ends the root loop.                                                                                                                                                                                  |
|131 |`}`                                     |Ends the user collector.                                                                                                                                                                             |
|132 |*(blank)*                               |Blank separator; no operation.                                                                                                                                                                       |

Lines 133–168: System asset discovery

|Line|Original source                                                                         |Explanation                                                                                                                                  |
|---:|----------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
|133 |`collect_system_assets() {`                                                             |Defines collection of system asset candidates for reporting.                                                                                 |
|134 |`local root path name`                                                                  |Declares locals. BUG: localizing `path` does not turn it into an ordinary safe variable name.                                                |
|135 |`local -a roots exact_paths`                                                            |Declares a root array and a separate list of exact candidate paths.                                                                          |
|136 |*(blank)*                                                                               |Blank separator; no operation.                                                                                                               |
|137 |`roots=(`                                                                               |Begins the asset-root list.                                                                                                                  |
|138 |`"/System/Library/AssetsV2"`                                                            |Adds `/System/Library/AssetsV2` as a discovery root.                                                                                         |
|139 |`"/Library/Apple/System/Library/AssetsV2"`                                              |Adds `/Library/Apple/System/Library/AssetsV2` as a second root.                                                                              |
|140 |`"/private/var/db/MobileAsset/AssetsV2"`                                                |Adds `/private/var/db/MobileAsset/AssetsV2` as a third root.                                                                                 |
|141 |`)`                                                                                     |Ends the root list. These strings describe search locations; they do not establish which physical volume stores the data.                    |
|142 |*(blank)*                                                                               |Blank separator; no operation.                                                                                                               |
|143 |`exact_paths=(`                                                                         |Begins the exact-path candidate list. Each entry is tested for visibility before being recorded.                                             |
|144 |`"/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"`              |Checks the generative-model family beneath the first asset root. The literal path is a candidate, not a guarantee it exists on a given build.|
|145 |`"/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"`                        |Checks the visual-model family beneath that first root.                                                                                      |
|146 |`"/Library/Apple/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"`|Checks the generative-model family beneath the second root.                                                                                  |
|147 |`"/Library/Apple/System/Library/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"`          |Checks the visual-model family beneath the second root.                                                                                      |
|148 |`"/private/var/db/MobileAsset/AssetsV2/com_apple_MobileAsset_UAF_FM_GenerativeModels"`  |Checks the generative-model family beneath the third root.                                                                                   |
|149 |`"/private/var/db/MobileAsset/AssetsV2/com_apple_MobileAsset_UAF_FM_Visual"`            |Checks the visual-model family beneath the third root.                                                                                       |
|150 |`"/private/var/db/AppleIntelligencePlatform/AppModelAssets"`                            |Adds a further AppModelAssets candidate. The script does not verify what its contents represent on a particular macOS build.                 |
|151 |`)`                                                                                     |Ends the exact-path list.                                                                                                                    |
|152 |*(blank)*                                                                               |Blank separator; no operation.                                                                                                               |
|153 |`for path in "${exact_paths[@]}"; do`                                                   |Visits every exact candidate. BUG: the loop reassigns Zsh’s special `path` parameter.                                                        |
|154 |`[[ -e "$path" ]] && add_system_candidate "$path"`                                      |Records an entry only when `-e` succeeds; `&&` runs the helper only on success.                                                              |
|155 |`done`                                                                                  |Ends the exact-candidate loop.                                                                                                               |
|156 |*(blank)*                                                                               |Blank separator; no operation.                                                                                                               |
|157 |`# Discover related asset families without assuming that names never change.`           |Comment explaining that name-based discovery supplements the exact-path list.                                                                |
|158 |`for root in "${roots[@]}"; do`                                                         |Visits each asset root.                                                                                                                      |
|159 |`[[ -d "$root" ]] || continue`                                                          |Skips roots not visible as directories.                                                                                                      |
|160 |`for path in "$root"/com_apple_MobileAsset_*(N); do`                                    |Enumerates immediate children starting with `com_apple_MobileAsset_`; `(N)` permits none. BUG: the loop uses `path` again.                   |
|161 |`name="${path:t}"`                                                                      |Extracts the entry’s last component for matching.                                                                                            |
|162 |`if looks_ai_related "$name"; then`                                                     |Applies the same name heuristic used for user items.                                                                                         |
|163 |`add_system_candidate "$path"`                                                          |Records a match in the system-report list. Exact duplicates from the prior loop are suppressed.                                              |
|164 |`fi`                                                                                    |Ends the match check.                                                                                                                        |
|165 |`done`                                                                                  |Ends the asset-child loop.                                                                                                                   |
|166 |`done`                                                                                  |Ends the asset-root loop.                                                                                                                    |
|167 |`}`                                                                                     |Ends the system collector.                                                                                                                   |
|168 |*(blank)*                                                                               |Blank separator; no operation.                                                                                                               |

Lines 169–184: Report header and collection calls

|Line|Original source                                                           |Explanation                                                                                                                     |
|---:|--------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
|169 |`print -- "Apple Intelligence storage audit"`                             |Starts the main report by printing its title. The helpers above were definitions; the next lines perform the actual audit.      |
|170 |`print -- "Date:         $(date)"`                                        |Runs `date` to display the current local date and time.                                                                         |
|171 |`print -- "macOS:       $(sw_vers -productVersion)"`                      |Runs `sw_vers -productVersion` to display the installed macOS version.                                                          |
|172 |`print -- "Build:       $(sw_vers -buildVersion)"`                        |Runs `sw_vers -buildVersion` to display the macOS build identifier.                                                             |
|173 |`print -- "Architecture: $(uname -m)"`                                    |Runs `uname -m` to report the execution architecture; this field alone is not a definitive hardware inventory under translation.|
|174 |`print -- "Mode:         $MODE"`                                          |Displays the selected mode after all options were processed.                                                                    |
|175 |*(blank)*                                                                 |Blank separator; no operation.                                                                                                  |
|176 |`heading "macOS protection status"`                                       |Prints a heading for the protection-status checks.                                                                              |
|177 |`csrutil status 2>/dev/null || print -- "  SIP status unavailable"`       |Queries SIP status, hiding command errors and printing a fallback if the command fails. It does not enable or disable SIP.      |
|178 |`csrutil authenticated-root status 2>/dev/null || \`                      |Queries authenticated-root status. The final backslash continues the shell statement onto the next physical line.               |
|179 |`print -- "  Authenticated-root status unavailable"`                      |Prints a fallback if the preceding query fails. It does not change authenticated-root protection.                               |
|180 |`fdesetup status 2>/dev/null || print -- "  FileVault status unavailable"`|Queries FileVault status, with a fallback on failure. It does not unlock or reconfigure encryption.                             |
|181 |*(blank)*                                                                 |Blank separator; no operation.                                                                                                  |
|182 |`collect_user_caches`                                                     |Calls the user collector to populate its candidate array.                                                                       |
|183 |`collect_system_assets`                                                   |Calls the system collector to populate the separate report-only asset array.                                                    |
|184 |*(blank)*                                                                 |Blank separator; no operation.                                                                                                  |

Lines 185–205: Candidate reports and free space

|Line|Original source                                                   |Explanation                                                                                                                                                                       |
|---:|------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|185 |`heading "Matching user cache directories"`                       |Prints the heading for user candidates. Its use of the word ‘directories’ is stronger than the selection logic, which can include other entry types.                              |
|186 |`if (( ${#USER_CACHE_CANDIDATES[@]} == 0 )); then`                |Counts the user array elements and checks whether the count is zero.                                                                                                              |
|187 |`print -- "  None found in the selected cache roots."`            |Prints the no-candidates message. This is not proof that no related files exist elsewhere or behind an access restriction.                                                        |
|188 |`else`                                                            |Starts the branch used when user candidates exist.                                                                                                                                |
|189 |`for path in "${USER_CACHE_CANDIDATES[@]}"; do`                   |Visits every selected user entry. BUG: this script-level `path` loop can also leave command lookup broken for later external commands.                                            |
|190 |`show_size "$path"`                                               |Calls the size helper on this entry; that helper has its own `path` naming bug.                                                                                                   |
|191 |`done`                                                            |Ends the user-report loop.                                                                                                                                                        |
|192 |`fi`                                                              |Ends the user-list conditional.                                                                                                                                                   |
|193 |*(blank)*                                                         |Blank separator; no operation.                                                                                                                                                    |
|194 |`heading "AI-related system and model assets (read-only)"`        |Prints the report-only system-assets heading.                                                                                                                                     |
|195 |`if (( ${#SYSTEM_ASSET_CANDIDATES[@]} == 0 )); then`              |Counts the system array elements and checks for zero.                                                                                                                             |
|196 |`print -- "  No matching assets were visible in the known roots."`|Reports no visible matches. It does not establish that all Apple Intelligence data is absent.                                                                                     |
|197 |`else`                                                            |Starts the branch used when system candidates exist.                                                                                                                              |
|198 |`for path in "${SYSTEM_ASSET_CANDIDATES[@]}"; do`                 |Visits each system candidate. BUG: the script-level loop also assigns to Zsh’s special `path`.                                                                                    |
|199 |`show_size "$path"`                                               |Attempts to print a size for that candidate; no move or delete is performed here.                                                                                                 |
|200 |`done`                                                            |Ends the system-report loop.                                                                                                                                                      |
|201 |`fi`                                                              |Ends the system-list conditional.                                                                                                                                                 |
|202 |*(blank)*                                                         |Blank separator; no operation.                                                                                                                                                    |
|203 |`heading "Startup-volume free space"`                             |Prints the free-space heading.                                                                                                                                                    |
|204 |`df -h / | awk 'NR == 1 || NR == 2 {print "  " $0}'`              |Intends to show `df -h /`’s heading and first data row using `awk`. It is not a before/after savings calculation. Prior `path` assignments may stop the commands from being found.|
|205 |*(blank)*                                                         |Blank separator; no operation.                                                                                                                                                    |

Lines 206–231: Additional storage checks

|Line|Original source                                          |Explanation                                                                                                                                          |
|---:|---------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------|
|206 |`if (( DEEP_SCAN == 1 )); then`                          |Enters the extra checks only if `--deep-scan` set the flag to 1.                                                                                     |
|207 |`heading "Common non-AI storage consumers"`              |Prints the additional-storage heading. Some listed roots are broad and can overlap AI-related data already reported.                                 |
|208 |`typeset -a OTHER_PATHS`                                 |Declares an array for extra inspection targets.                                                                                                      |
|209 |`OTHER_PATHS=(`                                          |Begins the extra-target list.                                                                                                                        |
|210 |`"$HOME/Library/Application Support/MobileSync/Backup"`  |Adds the conventional user location for Finder/iTunes-style iPhone and iPad backups.                                                                 |
|211 |`"$HOME/Library/Developer/Xcode/DerivedData"`            |Adds Xcode DerivedData as a possible storage consumer.                                                                                               |
|212 |`"$HOME/Library/Developer/Xcode/Archives"`               |Adds Xcode archives.                                                                                                                                 |
|213 |`"$HOME/Library/Developer/CoreSimulator"`                |Adds user CoreSimulator data.                                                                                                                        |
|214 |`"$HOME/Library/Containers/com.docker.docker"`           |Adds the Docker Desktop container location as a candidate; custom or changed storage locations will be missed.                                       |
|215 |`"$HOME/Library/Caches"`                                 |Adds the entire user cache root. Its size can overlap selected user entries and should not be added to them as separate usage.                       |
|216 |`"$HOME/Library/Logs"`                                   |Adds the user log directory.                                                                                                                         |
|217 |`"/Library/Updates"`                                     |Adds `/Library/Updates` as another possible storage consumer.                                                                                        |
|218 |`)`                                                      |Ends the extra-target list.                                                                                                                          |
|219 |*(blank)*                                                |Blank separator; no operation.                                                                                                                       |
|220 |`for path in "${OTHER_PATHS[@]}"; do`                    |Visits each extra path. BUG: the loop again uses the special `path` variable.                                                                        |
|221 |`show_size "$path"`                                      |Attempts to print its size using `show_size`; missing entries are silently skipped by that helper.                                                   |
|222 |`done`                                                   |Ends the extra-target size loop.                                                                                                                     |
|223 |*(blank)*                                                |Blank separator; no operation.                                                                                                                       |
|224 |`heading "Local Time Machine snapshots"`                 |Prints the local Time Machine snapshot heading.                                                                                                      |
|225 |`if command -v tmutil >/dev/null 2>&1; then`             |Checks whether `tmutil` can be located, suppressing both output streams. Broken command lookup can cause a false ‘unavailable’ result.               |
|226 |`tmutil listlocalsnapshots / 2>/dev/null | sed 's/^/  /'`|Lists local snapshot names for `/` and indents the output with `sed`. This neither deletes snapshots nor calculates their uniquely reclaimable space.|
|227 |`else`                                                   |Starts the fallback branch if `tmutil` was not located.                                                                                              |
|228 |`print -- "  tmutil is unavailable."`                    |Prints the unavailable message.                                                                                                                      |
|229 |`fi`                                                     |Ends the command-availability check.                                                                                                                 |
|230 |`fi`                                                     |Ends the additional checks.                                                                                                                          |
|231 |*(blank)*                                                |Blank separator; no operation.                                                                                                                       |

Lines 232–254: Cleanup selection and confirmation

|Line|Original source                                                      |Explanation                                                                                                                                      |
|---:|---------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|
|232 |`if [[ "$MODE" == "clean-user" ]]; then`                             |Selects the cleanup branch only when `MODE` equals `clean-user`.                                                                                 |
|233 |`heading "Reversible user-cache cleanup"`                            |Prints the cleanup heading; the word ‘reversible’ expresses intent, not a transactional rollback guarantee.                                      |
|234 |*(blank)*                                                            |Blank separator; no operation.                                                                                                                   |
|235 |`if (( ${#USER_CACHE_CANDIDATES[@]} == 0 )); then`                   |Checks whether there are any selected user entries.                                                                                              |
|236 |`print -- "  Nothing to move."`                                      |Prints ‘Nothing to move’ for an empty user list.                                                                                                 |
|237 |`elif [[ ! -t 0 ]]; then`                                            |Otherwise, tests whether standard input, descriptor 0, is not a terminal.                                                                        |
|238 |`print -u2 -- "Interactive confirmation is required."`               |Reports that confirmation requires interactive terminal input.                                                                                   |
|239 |`exit 1`                                                             |Exits with status 1 on noninteractive cleanup input.                                                                                             |
|240 |`else`                                                               |Starts the branch for a nonempty candidate list with terminal input.                                                                             |
|241 |`print -- "The directories listed below will be moved to the Trash."`|Prints the planned-operation statement. Candidates have not been reclassified or validated as disposable caches.                                 |
|242 |`print -- "No system or model-asset directory will be touched."`     |Prints the intended system-file boundary. Only the user list is used below, but the code lacks canonical-path and symlink-boundary validation.   |
|243 |`print`                                                              |Prints a blank line.                                                                                                                             |
|244 |`for path in "${USER_CACHE_CANDIDATES[@]}"; do`                      |Lists the user candidates before asking for confirmation. BUG: this loop again changes the special `path` parameter.                             |
|245 |`print -- "  $path"`                                                 |Prints the current candidate’s name.                                                                                                             |
|246 |`done`                                                               |Ends the candidate-display loop.                                                                                                                 |
|247 |`print`                                                              |Prints a blank line.                                                                                                                             |
|248 |`read -r "REPLY?Continue? Type YES to proceed: "`                    |Prompts for input and stores it in `REPLY`; `-r` keeps backslashes literal. The code does not explicitly check whether the read itself succeeded.|
|249 |*(blank)*                                                            |Blank separator; no operation.                                                                                                                   |
|250 |`if [[ "$REPLY" != "YES" ]]; then`                                   |Tests for anything other than uppercase `YES`; there is no case-insensitive acceptance.                                                          |
|251 |`print -- "Cleanup cancelled."`                                      |Prints a cancellation message when the exact confirmation was not received.                                                                      |
|252 |`exit 0`                                                             |Exits successfully on that cancellation path. Cancelling an optional action is not treated as an error.                                          |
|253 |`fi`                                                                 |Ends the confirmation check.                                                                                                                     |
|254 |*(blank)*                                                            |Blank separator; no operation.                                                                                                                   |

Lines 255–280: Destination creation, moves, and completion

|Line|Original source                                                             |Explanation                                                                                                                                                                                   |
|---:|----------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
|255 |`typeset -r TRASH_ROOT="$HOME/.Trash/Apple-Intelligence-Cleanup-$TIMESTAMP"`|Builds a read-only destination string under the current user’s `.Trash` with the timestamp. A second run within the same second can produce the same name.                                    |
|256 |`mkdir -p "$TRASH_ROOT"`                                                    |Attempts to create the destination and any missing parents. Its status is not checked; the `path` bug can prevent `mkdir` from being found.                                                   |
|257 |*(blank)*                                                                   |Blank separator; no operation.                                                                                                                                                                |
|258 |`for path in "${USER_CACHE_CANDIDATES[@]}"; do`                             |Visits every original user candidate for the move. BUG: it again assigns to the special `path` variable.                                                                                      |
|259 |`[[ -e "$path" ]] || continue`                                              |Skips a candidate no longer visible as existing. This is only an existence check; it does not verify ownership, type, or a canonical containment boundary.                                    |
|260 |`relative="${path#$HOME/}"`                                                 |Intends to strip the home-directory prefix and preserve the remaining hierarchy, such as `Library/Caches/...`. Prefix removal is not a security validation.                                   |
|261 |`target="$TRASH_ROOT/$relative"`                                            |Appends that relative string to the Trash destination.                                                                                                                                        |
|262 |`mkdir -p "${target:h}"`                                                    |Attempts to create the destination’s parent using `:h` to remove the last path component. Errors are not checked and command lookup may already be broken.                                    |
|263 |`if command mv "$path" "$target"; then`                                     |Attempts the actual move and branches on its status. `command` bypasses a same-named shell function, but still relies on command lookup for `mv`. No no-clobber or collision check is present.|
|264 |`print -- "Moved: $path"`                                                   |Prints success for this individual move when `mv` returned zero.                                                                                                                              |
|265 |`else`                                                                      |Starts the per-item failure branch.                                                                                                                                                           |
|266 |`print -u2 -- "Could not move: $path"`                                      |Prints failure for this item to standard error.                                                                                                                                               |
|267 |`fi`                                                                        |Ends the move-result check. There is no cumulative failure counter.                                                                                                                           |
|268 |`done`                                                                      |Ends the move loop; there is no rollback if a later item fails.                                                                                                                               |
|269 |*(blank)*                                                                   |Blank separator; no operation.                                                                                                                                                                |
|270 |`print`                                                                     |Prints a blank line.                                                                                                                                                                          |
|271 |`print -- "Moved caches are in:"`                                           |Prints the final destination heading even if all moves failed.                                                                                                                                |
|272 |`print -- "  $TRASH_ROOT"`                                                  |Prints the intended destination string. This is not verification that it exists or contains all items.                                                                                        |
|273 |`print -- "Restart the Mac and test it before emptying the Trash."`         |Prints the restart-and-test suggestion. Restarting and emptying the Trash are separate actions the script does not perform.                                                                   |
|274 |`fi`                                                                        |Ends the interactive cleanup branch.                                                                                                                                                          |
|275 |`else`                                                                      |Starts the audit-only finish when the selected mode is not cleanup.                                                                                                                           |
|276 |`heading "Audit complete"`                                                  |Prints the audit-complete heading, regardless of whether some measurements failed.                                                                                                            |
|277 |`print -- "No files were changed."`                                         |States that the audit has not intentionally changed files. It is not evidence that every check succeeded.                                                                                     |
|278 |`print -- "For reversible cache cleanup, run:"`                             |Prints an instruction introducing the cleanup command.                                                                                                                                        |
|279 |`print -- "  zsh apple-intelligence-audit.zsh --clean-user"`                |Prints the cleanup invocation as text; does not execute it.                                                                                                                                   |
|280 |`fi`                                                                        |Ends the mode conditional. There is no explicit overall failure-status calculation at the end.                                                                                                |

Practical limitations exposed by the code

These points concern this exact script. They do not establish that any particular Mac has missing assets, damaged files, or disabled protections.

A matching name does not establish a safe cleanup target

Lines 104–131 select entries by words in their names, and the same list feeds cleanup. There is no independent cleanup allowlist. The patterns can include traditional Siri data and other related components. The script also includes ~/Library/HTTPStorages, whose entries are not verified here as disposable caches.

The globs select immediate children regardless of whether they are directories, regular files, or symlinks. Before distributing a cleanup version, keep broad matching for reporting and restrict file moves to specifically reviewed cache targets.

The filesystem boundary is not fully validated

The script constructs paths below $HOME, but does not resolve their real location or reject symlinked parent directories. The string operation on line 260 only removes a prefix; it does not enforce containment. Lines 259 and 263 are separate checks and actions, so the target can also change between them.

Running as an ordinary user reduces its privileges. It does not turn unvalidated targets into safe ones. A repaired version should check the actual candidate and destination paths immediately before each move.

Size errors are hidden or incomplete

Line 82 silently skips entries that fail its existence check. Line 83 discards du error output, then line 84 substitutes unreadable only if no output was captured. A permission-limited traversal may produce a partial size. After the naming bug is fixed, that error handling still needs improvement.

PIPE_FAIL helps expose pipeline failures through an exit status, but this script never inspects the size pipeline’s result. It also does not enable an automatic stop on ordinary command failures. The report should explicitly distinguish missing, inaccessible, partial, failed, and successfully measured items. See Zsh pipeline-status behavior.

Moving to the Trash does not immediately free disk space

When the destination is on the same filesystem, the files still occupy space after a move. The script does not empty the Trash. It also does not calculate bytes recovered or compare free space before and after the operation.

The folder is created with mkdir and populated with plain mv; the script does not invoke Finder’s Trash API or record Finder restoration metadata. Do not assume Finder’s Put Back action will restore these entries automatically. The retained Library/... hierarchy tells you their intended original locations, but restoration still needs care if the application has already recreated files there.

Cleanup is not transactional

Directory creation failures are not checked at lines 256 and 262. The timestamp has only one-second resolution and is not a collision-resistant destination allocator. mv has no explicit no-clobber option here; a destination that already exists can cause nesting or replacement behavior.

The script does not stop on the first failed move, roll back earlier moves, keep a persistent restore manifest, or calculate an overall failure result. The final location message can appear even if no files were moved.

The report is not a complete inventory

--deep-scan measures only the listed roots. It does not search the entire disk. Other accounts, app containers, changed model locations, and custom storage paths can be missed. Matching AssetsV2 names does not identify the owning volume or prove SIP/SSV protection for that individual directory.

Lists can overlap: the entire user cache root includes selected subfolders already measured. APFS aliases, clones and snapshots can also make summed directory sizes differ from unique reclaimable physical storage. Do not add every printed value and label the sum ‘Apple Intelligence space recoverable.’

Small Zsh syntax reference

This table decodes recurring notation from the actual script. References for the underlying syntax are Zsh grammar, conditional expressions, expansion, and built-in commands.

|Notation                                |Meaning in this script                                                                        |
|----------------------------------------|----------------------------------------------------------------------------------------------|
|`"$@"`                                  |All command-line arguments, kept as separate values.                                          |
|`"$1"`                                  |The first argument to the current function or script.                                         |
|`$(...)`                                |Runs a command and captures its standard output as text.                                      |
|`[[ ... ]]`                             |Tests a condition involving strings, paths, or terminal state.                                |
|`(( ... ))`                             |Evaluates an arithmetic condition.                                                            |
|`-e`, `-d`                              |Tests existence or directory type, respectively.                                              |
|`-z`, `-n`                              |Tests for an empty or nonempty string.                                                        |
|`-t 0`                                  |Tests whether standard input is connected to a terminal.                                      |
|`&&`, `||`                              |Runs the next command after success or failure, respectively; also used to combine conditions.|
|`|`                                     |Sends one command’s standard output to another command’s input.                               |
|`2>/dev/null`                           |Discards standard error.                                                                      |
|`>/dev/null 2>&1`                       |Discards standard output, then directs standard error to the same destination.                |
|`return 0`, `exit 0`                    |Returns successfully from a function, or ends the script successfully.                        |
|`${items[@]}`                           |Expands an array’s elements; quotes preserve the individual elements.                         |
|`${#items[@]}`                          |Counts array entries.                                                                         |
|`${value:l}`, `${value:t}`, `${value:h}`|Lowercase, last pathname component, or parent pathname.                                       |
|`*(N)`                                  |A filename wildcard where no matches produce an empty result.                                 |
|`<<'EOF'`                               |Starts literal multiline input ending at the matching delimiter.                              |

Correction checklist for the repo

This companion documents the supplied version; it does not silently change that version or claim it is repaired.

1. Rename every ordinary use of path to a nonspecial name such as scan_path.
2. Use a reviewed cleanup allowlist separate from broad audit discovery, and review whether HTTP storage should be eligible at all.
3. Validate source and destination containment, symlinks, types, ownership and existing destinations before moving.
4. Check directory-creation and move results, report partial failures, and return a meaningful final exit status.
5. Make missing/inaccessible/partial measurements explicit instead of hiding all du errors.
6. Use a unique destination and record original-to-destination mappings for manual restore.
7. Keep syntax validation and runtime validation separate. Exercise audit and cleanup behavior with disposable fixture directories before trying real user data.

Renaming fixes the confirmed command-lookup defect; it does not, by itself, implement the other safeguards.

Validation performed for this explanation

• Matched the explanations to all 280 physical lines of the retained original script, with no missing line numbers.
• Confirmed the Zsh path behavior in the official manual and reproduced the original size helper’s failure in a separate local Zsh process.
• Confirmed that a freshly declared indexed array reports zero elements in the local Zsh environment used for inspection; it was not incorrectly classified here as an unset-array bug.
• Did not run the full cleanup script or move/delete any user caches.
• Did not test against a live macOS installation; platform-specific reporting and recovery behavior still require testing there.

To show line numbers in a real script file without executing it, use:

```bash
nl -ba apple-intelligence-audit.zsh
```

To check parsing without executing the body, use:

```bash
zsh -n apple-intelligence-audit.zsh
```

A successful parsing check does not catch the path command-lookup issue or prove that cleanup is safe.