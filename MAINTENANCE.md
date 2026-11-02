# Maintaining this fork

This repository is a real fork of [crimera/piko](https://github.com/crimera/piko): `main` is upstream
plus two commits of ours. That is what makes updates cheap.

```
main
 ├─ ... upstream history (a0ed9a5a = v3.10.0 when this fork was made)
 ├─ feat(Instagram): local view history for posts, Reels and Stories   <- 20 files, the feature
 └─ chore: build patches workflow + docs                                <- tooling only
```

The feature commit touches only these paths, so a rebase can never conflict anywhere else:

```
extensions/instagram/src/main/java/app/morphe/extension/instagram/db/PikoHistoryDb.java
extensions/instagram/src/main/java/app/morphe/extension/instagram/patches/history/
extensions/instagram/src/main/java/app/morphe/extension/instagram/settings/
extensions/instagram/src/main/java/app/morphe/extension/instagram/constants/Constants.java
extensions/instagram/src/main/java/app/morphe/extension/instagram/utils/Pref.java
extensions/instagram/stub/src/main/java/com/instagram/common/session/UserSession.java
patches/src/main/kotlin/app/crimera/patches/instagram/misc/history/
patches/src/main/kotlin/app/crimera/patches/instagram/entity/mediadata/Fingerprint.kt
patches/src/main/resources/addresources/values/instagram/strings.xml
```

## 1. Update to a new piko release

Once, in your clone:

```bash
git remote add upstream https://github.com/crimera/piko.git
```

Every time piko ships something new:

```bash
git fetch upstream
git switch main
git rebase upstream/main
git push --force-with-lease
```

- Conflicts, if any, will be in the files listed above — they are small edits, not rewrites.
  Take upstream's version of everything around them and keep our hunks.
- If a rebase goes sideways: `git rebase --abort` and try again, or
  `git merge upstream/main` instead (uglier history, same result).
- The push starts the **Build patches** workflow automatically. When it finishes, download the
  `piko-patches-view-history` artifact — that is your new `.mpp`.

## 2. Patch a new Instagram version

Two independent things can break when Instagram updates:

**a) The patch bundle itself** (only if piko changed): covered by step 1.

**b) The fingerprints** (if Instagram renamed/moved the hooked methods). Symptom, from the Manager
log or the CLI:

```
Feed view history: viewed_impression not found
Reels view history: clips item lookup not found
Stories view history: ReelItem media field not found
```

Each patch anchors on a trace string, so find the new location by searching the new APK's dex for
the same string. Decompile with jadx, or open the APK in MT Manager and search the string:

| Patch | Search for this string | File to edit |
|---|---|---|
| Feed | `viewed_impression` | `patches/.../misc/history/FeedViewHistoryPatch.kt` |
| Reels | `ClipsViewerViewPagerListener.onPageSelected` | `.../ReelsViewHistoryPatch.kt` |
| Stories | `ReelViewerFragment.onCurrentActiveItemBound` | `.../StoriesViewHistoryPatch.kt` |

Then update the `Fingerprint(...)` in that file to match what you found: `returnType`,
`parameters` (JVM descriptors like `"I"`, `"Ljava/lang/Object;"`, `"J"`), and `strings`. The
`execute { }` block below it usually needs no change — it locates the casts/fields itself.

For the Feed patch, the method must still cast an argument to `MediaData`'s media class and to the
carousel class, and the class must still have a `UserSession` field. For Stories, `ReelItem` must
still have exactly one `final` field of the media type.

Push, let Actions rebuild, re-patch.

> Do **not** rename `GetPostTypeExtensionFingerprint` back to `getPostType`. It must stay on
> `getPostTypeKey` (the method containing the `"A7Q"` literal); otherwise the patch overwrites the
> `"clips"` literal and Reels stop being recognised.

## 3. Build / patch locally instead of on Actions

Building needs a GitHub token with `read:packages` (the Morphe patcher library lives on GitHub
Packages) and JDK 17 + Android SDK 36:

```bash
export GITHUB_TOKEN=ghp_...        # read:packages
./gradlew buildAndroid
# -> patches/build/libs/patches-<version>.mpp
```

Patching a split APK (`.apkm`) is easiest with **Morphe Manager ≥ 1.33.0-dev.1** on the phone:
import the `.mpp`, pick the APKM, patch, install. Morphe CLI expects a single `.apk`; for `.apkm`
you would have to merge the splits first.

## 4. If the repository is ever lost

The feature is self-contained — 9 new files and 11 edited ones. To recreate it:

```bash
git clone https://github.com/crimera/piko.git
cd piko
git checkout -b view-history
# apply the saved diff
git apply view-history.patch
```

`view-history.patch` is the export of the feature commit. Regenerate it any time with:

```bash
git diff upstream/main..HEAD -- ':!VIEW_HISTORY.md' ':!MAINTENANCE.md' \
  ':!.github' > view-history.patch
```

The four Java classes under `patches/history/` plus `PikoHistoryDb`, and the five Kotlin patches,
are also readable standalone if you prefer to copy them file by file.

## 5. Version bump (optional)

`gradle.properties` holds `version = 3.10.0`, which is also the `.mpp` filename. Bump it when you
want to tell builds apart, e.g. `3.10.0-viewhistory.1`.
