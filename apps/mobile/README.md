# T3 Code Mobile

> [!WARNING]
> T3 Code Mobile is currently in development and is not distributed yet. If you want to try it out, you can build it from source.

## Quickstart

> [!NOTE]
> Uses native modules so using Expo Go is not supported. You need to use the Expo Dev Client.

This app has three variants:

- `development`: Expo dev client, installable side-by-side as `T3 Code Dev`
- `preview`: persistent internal preview build, installable side-by-side as `T3 Code Preview`
- `production`: store/release build as `T3 Code`

Run commands from `apps/mobile`.

T3 Connect is optional and disabled in a fresh clone. Public configuration belongs in the
repository-root `.env` or `.env.local`, not an `apps/mobile/.env` file. See
[`../../.env.example`](../../.env.example).

## Development

Start Metro for the dev client:

```bash
vp run dev:client
```

Build and run the local iOS dev client:

```bash
vp run ios:dev
```

If your Xcode account only has a Personal Team, use a bundle identifier you control and opt into the
reduced-capability local build. Personal Team builds omit the widget and share extensions, push
entitlement, and native Sign in with Apple entitlement; builds without this opt-in are unchanged.

```bash
T3CODE_IOS_PERSONAL_TEAM=1 \
T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.example.t3code.dev \
vp run ios:dev
```

Build and install a self-contained Release app that does not need Metro:

```bash
vp run ios:release
```

The Personal Team equivalent also needs a unique bundle identifier:

```bash
T3CODE_IOS_PERSONAL_TEAM=1 \
T3CODE_IOS_PERSONAL_TEAM_BUNDLE_ID=com.example.t3code \
vp run ios:release
```

Build and run the local iOS preview app:

```bash
vp run ios:preview
```

Force the review diff highlighter engine:

```bash
EXPO_PUBLIC_REVIEW_HIGHLIGHTER_ENGINE=javascript vp run ios:dev
```

`javascript` is the default and recommended setting for the review diff screen. Set `EXPO_PUBLIC_REVIEW_HIGHLIGHTER_ENGINE=native` only when you explicitly want to test the native Shiki engine.

Inspect the resolved Expo config for a variant:

```bash
vp run config:dev
vp run config:preview
```

Run static checks for mobile native code:

```bash
node ../../scripts/mobile-native-static-check.ts
```

The native lint task runs SwiftLint for Swift plus ktlint and detekt for Kotlin. Missing native tools are reported as warnings and skipped locally. CI installs the default toolset from `apps/mobile/Brewfile` before running the native checks.

## EAS Builds

CI uses Expo fingerprinting with the `preview:dev` profile to reuse an existing compatible build when possible, or start a new internal EAS build when native runtime inputs change. Production and default local builds continue to use the `appVersion` runtime policy.

For preview or production EAS environments, set `T3CODE_CLERK_PUBLISHABLE_KEY`,
`T3CODE_CLERK_JWT_TEMPLATE`, and `T3CODE_RELAY_URL`
as EAS environment variables. Expo config maps the canonical values into the mobile build.

Create a PR preview dev-client build manually:

```bash
vp run eas:ios:preview:dev
```

Create a cloud dev-client build:

```bash
vp run eas:ios:dev
```

Create a persistent preview build:

```bash
vp run eas:ios:preview
```

Android equivalents:

```bash
vp run eas:android:dev
vp run eas:android:preview:dev
vp run eas:android:preview
```

## Personal TestFlight builds (artieeg fork)

> [!NOTE]
> This section describes a fork-only setup. It does not apply to `pingdotgg/t3code`,
> and the identifiers below are not T3 Tools'.

This fork can build the iOS app and ship it to TestFlight under a personal Apple
account, so a self-built app can talk to a personal T3 Connect server without
needing access to the upstream Apple team or EAS project.

|                           | Upstream                           | This fork                                |
| ------------------------- | ---------------------------------- | ---------------------------------------- |
| EAS project               | `@pingdotgg/t3-code` (`d763fcb8…`) | `@artieeg/t3-code` (`d207b086…`)         |
| iOS bundle id             | `com.t3tools.t3code`               | `com.artieeg.t3code`                     |
| Apple team                | `ARK85ZXQ4Z` (T3 Tools)            | `ZMC8WZHB36` (Artem Griukov, Individual) |
| ASC record                | `6787819824`                       | `6804201379` (`T3 Code (Artem)`)         |
| Production runtime policy | `fingerprint`                      | `appVersion`                             |

These values are committed in `app.config.ts` and `eas.json` rather than read from
`.env`, because `.env` and `.env.local` are gitignored and never reach the EAS build
server — which re-evaluates the config from a git archive. An env-only override would
silently fall back to the upstream project instead of failing loudly.

### Building and submitting

Builds must run on EAS; the iOS SDK required by Expo SDK 56 is newer than some local
Xcode installs. `MOBILE_VERSION_POLICY=appVersion` is set on the production profile so
`eas build` can be invoked from macOS — the `fingerprint` policy requires the invoking
machine to match the build environment, and it buys nothing here because this fork
ships no OTA updates.

```bash
eas build --profile production --platform ios --auto-submit
```

First-time setup, in order:

1. `eas credentials -p ios` → Build Credentials. Interactive only; it needs an Apple
   login with 2FA to create the provisioning profile.
2. Set `T3CODE_CLERK_PUBLISHABLE_KEY`, `T3CODE_CLERK_JWT_TEMPLATE`, and
   `T3CODE_RELAY_URL` on the EAS `production` environment. Without them
   `hasCloudPublicConfig()` is false and T3 Connect is disabled entirely.
3. The first submit had to be interactive (`eas submit -p ios --latest`) so App Store
   Connect could create the app record. That is done: the record is `6804201379`, now
   pinned as `submit.production.ios.ascAppId`, so `--auto-submit` runs non-interactively
   from here on. TestFlight internal access is granted via the `Team (Expo)` group.

### Reduced capabilities

The build defaults to the reduced-capability path that `T3CODE_IOS_PERSONAL_TEAM`
originally gated for Personal Teams. A paid Individual team could sign these, but each
extension is another provisioning profile to maintain, and the widget cannot work
regardless. Set `T3CODE_IOS_PERSONAL_TEAM=0` to restore the upstream full build.

What does not work under a personal bundle id, and why:

- **Push notifications and the Agent Activity widget.** The relay is configured with a
  single APNs team/key/bundle (`APNS_BUNDLE_ID`, `infra/relay/src/worker.ts`) targeting
  `com.t3tools.t3code`. APNs will not deliver to another bundle id. Self-hosting
  `infra/relay` with your own APNs key is the only fix.
- **Native Google and Apple sign-in.** Those client ids are bound to the `com.t3tools.*`
  bundle ids. Sign in with email instead; it reaches the same Clerk account.
- **Universal links and passkeys** for `clerk.t3.codes`. The entitlement signs fine, but
  the domain's AASA file cannot list a third-party app id. Sign-in callbacks still
  complete over the custom URL scheme.

Everything foreground — threads, terminal, diffs, review — works normally, because
T3 Connect authorizes on Clerk account plus a locally generated DPoP keypair
(`src/features/cloud/dpop.ts`), never on app identity.
