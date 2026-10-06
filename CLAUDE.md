# CLAUDE.md — android-ci

## Cos'è

Reusable GitHub Actions workflow per build di app **Android native** (Gradle, Kotlin/Compose),
con l'eventuale libreria Rust compilata da Gradle con cargo-ndk. Quarto stack della famiglia
`flutter-ci` / `expo-ci` / `desktop-ci`, stesso schema: thin wrapper nel progetto → questo
reusable → `GabryXnLab/build-kit` (`setup`, `notify`). Nato per Riftgate (`android-native/`),
ma generico: nessun nome di progetto cablato.

## Stack

GitHub Actions (reusable `workflow_call`, bash con `set -euo pipefail`). Nessun codice da
compilare. Il modello da seguire è «Come si scrive una build» nel `CLAUDE.md` di `build-kit`.

## Struttura

```text
.github/workflows/android-build.yml   il reusable (un solo job: build)
.github/workflows/{copilot-setup-steps,jules_agent}.yml   uguali a quelli degli altri reusable
.github/readme/banner.svg             banner del README
```

## Comandi

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/android-build.yml'))"
actionlint .github/workflows/android-build.yml   # se installato
```

## Note architetturali importanti

- **`@main` è live per tutti**: input nuovi con `default`, mai rinominati né tolti senza
  aggiornare i wrapper. Il repo è pubblico e deve restarlo (un reusable pubblico usa solo
  azioni pubbliche).
- **Input comuni uguali agli altri stack**: `runner` (`self-hosted` | `github`),
  `max_workers`, `clear_cache`, con le descrizioni di `build-kit`. `runner: github` usa
  `github_runner_label` (default `ubuntu-24.04`, x86-64: l'NDK 27 ha toolchain solo x86-64 per
  Linux; `ubuntu-24.04-arm` serve un NDK con host aarch64).
- **Self-hosted (nexus-core, ARM64)**: JDK 17, `/opt/android-sdk` e NDK condivisi si usano, non
  si modificano; la toolchain NDK è quella patchata per host aarch64
  (`expo-ci/setup-ndk-aarch64-host`, idempotente). Daemon di Gradle spento; worker da
  `GRADLE_OPTS` di `build-kit/setup`. cargo-ndk sta in `~/ci/tools/cargo-ndk/<versione>`.
- **Su `runner: github` la cache è di GitHub, nel reusable** (regola di `build-kit`): `setup-gradle`,
  `rust-cache`, `actions/cache` per cargo-ndk, tutte con `if` sul runner e spente da `clear_cache`.
  SDK e NDK: quelli dell'immagine, l'NDK richiesto installato con `sdkmanager` se manca.
- **Firma (`signing_keystore`)**: stessa semantica di `expo-ci`: keystore in
  `/home/ubuntu/secrets/<nome>.{jks,properties}` del runner o nei secret
  `ANDROID_KEYSTORE_BASE64`/`ANDROID_KEYSTORE_PROPERTIES`; firma per `-Pandroid.injected.signing.*`;
  keystore introvabile = job fallito; l'impronta dell'APK è confrontata con quella del keystore con
  `apksigner`. Vuoto = firma del progetto (di norma debug).
- **`CARGO_TARGET_DIR` e gli script che cercano `target/` accanto al crate**: `setup` con `rust`
  mette la target di cargo in `~/ci/cache/cargo-target/<repo>`. Se lo script che Gradle lancia
  usa percorsi relativi `target/…`, `cargo_target_in_checkout: true` lancia Gradle senza quella
  variabile (la target resta nel checkout, che sul self-hosted non si pulisce).
- L'upload dell'artifact ha `continue-on-error` (quota dell'org); l'APK arriva comunque su Telegram.

## Convenzioni

Commenti, documentazione e commit in italiano; identificatori in inglese. Un cambio a un input o a
un comportamento si porta nei wrapper dei progetti (oggi: Riftgate `build-android-native.yml`) e,
se tocca la famiglia, nel `CLAUDE.md` di `build-kit`.
