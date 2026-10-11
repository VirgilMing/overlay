# SourceGit 2026.21-r1 for gentoo-zh

[简体中文说明](README.zh_CN.md)

This package builds upstream tag `v2026.21`
(`a4ae633c51c922150ea113913c682a8213308560`) as Native AOT on amd64/glibc.
The official SDK 10.0.111 archive is a declared build input, extracted only into
`${WORKDIR}/dotnet-sdk`. Building does not depend on a system SDK package.

## Validation scope

Only amd64/glibc has been validated. The ebuild keeps `KEYWORDS="~amd64"`
and the `elibc_glibc` requirement; it does not offer verified musl support.
Musl adaptation and build/runtime validation remain for contributors using
that environment. The current AOT runtime packs and private native libraries
must be checked before relaxing the libc requirement.

The full pkgcheck scan still reports `RequiredUseDefaults` and
`RequiredUseUnsatisfiableInDev` for amd64 musl profiles because of this
requirement. These findings remain visible; no musl build was performed.

## USE flags

| Flags | Framework and behavior |
| --- | --- |
| `-wayland -kwin-iconfix` | Avalonia 11.3.22 and AvaloniaEdit `ddf00a91`; X11 backend |
| `wayland -kwin-iconfix` | Downstream Avalonia 12.1.3 migration and AvaloniaEdit `358f2c9e`; native Wayland when `WAYLAND_DISPLAY` is set, X11 otherwise |
| `wayland kwin-iconfix` | The same native build plus a launcher helper for KWin icons |

`kwin-iconfix` requires `wayland`. The `debug` flag retains native debug
information. Neither feature has a `+` default; desktop profiles can still
enable `wayland`.

The framework migration is downstream: upstream SourceGit has not accepted
<https://github.com/sourcegit-scm/sourcegit/pull/2768>. The compressed patch is
based on the release tag, not later `develop` changes. Both AvaloniaEdit sources
are immutable commits available from the configured public upstream fork.

## Dependencies and offline builds

Both variants use the same .NET 10.0.11 AOT compiler and runtime packs.
Their closures contain 26 common NuGets, 29 additional NuGets for Avalonia 11,
and 34 additional NuGets for Avalonia 12. Conditional `SRC_URI` selects only the
required set and submodule source. Restore uses declared distfiles and a fresh
cache. The SDK archive is also pinned through `SRC_URI` and `Manifest`, and all
SDK commands use the build directory. SourceGit therefore does not pull in
the .NET 6–10 runtime-NuGet packages through the system SDK's `PDEPEND`.

The SDK itself is still downloaded: the archive is 239,732,427 bytes, or
228.63 MiB. Its ICU, OpenSSL, Kerberos and LTTng libraries are declared build
dependencies. The SDK is not installed or included in the application image.
This is confined to SourceGit; no system SDK ebuild is changed. Future updates
must verify the SDK version, archive hashes and application NuGet list.

The installed application needs no .NET runtime. It consists of
`/usr/lib64/sourcegit/sourcegit` and three private native libraries, plus the
command, desktop entry and icon. Package-managed update detection is disabled.

The ebuild inherits `nuget` directly because the installed Native AOT output
is ELF, while `dotnet-pkg-base` restricts stripping to protect managed output.
There is no `RESTRICT=strip`; Portage follows the user's stripping and debug
settings. Publishing leaves stripping to Portage.

## KWin icon compatibility

With `kwin-iconfix`, the launcher invokes a compatibility helper as the launching
user. It only acts on Wayland when `org.kde.KWin` is already present on the
session bus, creates one fixed-ID rule matching `sourcegit`, and associates it
with `sourcegit.desktop`. It preserves other rules and repeated launches do
not create duplicate rules. Root and non-KWin sessions are skipped.

Portage does not edit user configuration. The generated rule is user-owned and
remains after disabling the USE flag or uninstalling. Remove only the helper's
rule as that user before removing the helper:

```sh
/usr/libexec/sourcegit/sourcegit-kwin-iconfix --remove
```

Existing manually created SourceGit rules have different IDs and are preserved.
No 1Password rule is included.

## Development and validation

Release tracking is configured in `.github/workflows/overlay.toml`. Automatic
bumps are disabled because the framework patch, AvaloniaEdit commit, SDK and NuGet
lists require verification for each release.

On amd64/glibc, isolated Portage builds with a fresh NuGet cache completed
through the install-image phase for X11, Wayland, and Wayland with
`kwin-iconfix`. Startup checks covered opening a repository on X11 and Wayland
and opening Blame on Wayland. These are brief startup checks, not a long-term
performance or exhaustive interaction test.

The regression checks use real KConfig tools with an isolated configuration.
Run from the overlay root:

```sh
python3 scripts/sourcegit-kwin-iconfix-test.py
```

Full dependency validation started with the SDK/distfile cache. The two
variants have 93 distinct inputs, all verified against Manifest sizes,
BLAKE2B and SHA512 hashes. X11 selects 58 inputs and Wayland selects 63;
each set contains one SDK archive.

Dependency previews against an empty installation root and normal
`emerge --onlydeps --with-bdeps=y` passed for both USE states. Neither plan
includes a system SDK or `dotnet-runtime-nugets` package. USE settings were
confined to SourceGit.

Normal source `emerge` builds and merges then passed without `--nodeps` or
user phase hooks. The observed SDK executables and CLR libraries came from
`${WORKDIR}/dotnet-sdk`. Validation set `SANDBOX_DENY` for the system SDK
directory and `/opt/dotnet-nugets`, and enabled `network-sandbox`. All inputs
were cached, each restore used a fresh NuGet cache, and Portage stripped the
installed native executable. Neither dependency checks nor either merge
produced `qa`, `warn` or `error` elog. Installed file lists contain no SDK.

| Tested USE | Native executable | Startup backend |
| --- | --- | --- |
| `-wayland -kwin-iconfix -debug` | 51.16 MiB, stripped | X11 |
| `wayland kwin-iconfix -debug` | 56.56 MiB, stripped | Native Wayland |

All installed native libraries resolved. Both installed variants stayed alive
through a 13-second startup check, with no CLR libraries or crash logs. These
checks do not establish long-term performance or exhaustive interaction
coverage. Architecture and libc coverage remain amd64/glibc only.

Keep AOT diagnostics visible. DataGrid emits an existing `IL3053` in both
framework versions. SourceGit uses explicit template columns with automatic
column generation and user sorting disabled.
