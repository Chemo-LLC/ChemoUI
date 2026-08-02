# Bundle verifier sandbox

Advanced Safety loads every incoming avatar bundle in a throwaway process before the game touches
it. That process is this player. The client downloads it on demand and installs it to
`<game>/UserData/BundleVerifier`, so nobody has to build it by hand.

## Files

| File | What it is |
|------|------------|
| `manifest.txt` | `key=value` lines the client reads first: payload version, file name, sha256, size, and the Unity version it was built with |
| `BundleVerifier.zip` | The player itself, unzipped straight into `UserData/BundleVerifier` |

The client compares `version` against `UserData/BundleVerifier/version.txt` and only downloads when
they differ or nothing is installed. The download is rejected unless its sha256 matches, so a
truncated or swapped payload never reaches disk as a runnable player.

## Updating the payload

Build it from the client repo (`Tools/BundleVerifierApp/build.ps1`, needs Unity 2022.3.22f1 with
IL2CPP support and the MSVC toolchain), then zip the install directory without the
`BundleVerifier_BackUpThisFolder_ButDontShipItWithYourGame` symbols folder and without `scratch`:

```powershell
$src = "C:\Program Files (x86)\Steam\steamapps\common\VRChat 1865\UserData\BundleVerifier"
$stage = "$env:TEMP\bvstage"
New-Item -ItemType Directory -Force $stage | Out-Null
Get-ChildItem $src |
    Where-Object { $_.Name -ne "BundleVerifier_BackUpThisFolder_ButDontShipItWithYourGame" -and $_.Name -ne "scratch" } |
    ForEach-Object { Copy-Item $_.FullName -Destination $stage -Recurse -Force }
Compress-Archive -Path "$stage\*" -DestinationPath BundleVerifier.zip -CompressionLevel Optimal -Force
(Get-FileHash BundleVerifier.zip -Algorithm SHA256).Hash.ToLower()
```

Then bump `version` in both `manifest.txt` and the `version.txt` inside the zip, and put the new
sha256 and size in `manifest.txt`. Clients on the old version pick the new one up on their next
launch.

The player keeps the stock Unity engine it was built against rather than VRChat's own
`UnityPlayer.dll`. VRChat's custom build resolves its IL2CPP bindings through obfuscated export
names and cannot host a `GameAssembly.dll` produced by a stock editor. Bundle serialization is a
Unity version level format, so the stock engine of the same version reads bundles the same way.
