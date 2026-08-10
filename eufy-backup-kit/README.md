# Eufy Backup Kit Template

This is a sanitized starter layout for backing up and analyzing an Android application that you own or are authorized to test. It contains no APKs, credentials, device identifiers, account data, or captured memory.

## Intended workflow

1. Keep the original app bundle and every split APK in private storage.
2. Install only splits from the same app release and matching device ABI.
3. Use an isolated test Android environment and a dedicated account where possible.
4. Record app version, device ABI, Android version, and exact UI paths exercised.
5. Store extracted artifacts privately; publish only documentation and redacted metadata.
6. Analyze commands only for owned devices and accounts.

## Private artifact layout

Do not commit this layout. Keep it outside Git or in encrypted private storage.

```text
private-artifacts/
  input-bundle/
  runtime-captures/
  extracted-dex/
  reports/
```

## Capture report fields

Use these fields in a private report:

- app version and package name;
- Android version and ABI;
- whether the app was logged in;
- owned device flow exercised, such as Live View or event playback;
- capture method and result count;
- hashes of private artifacts, if needed for reproducibility.

## Publication boundary

Never publish APKs, split APKs, DEX/VDEX/OAT files, application logs, memory dumps, tokens, account data, device serials, public IP addresses, SSH material, or cloud-account identifiers.
