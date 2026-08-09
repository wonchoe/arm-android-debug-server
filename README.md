# ARM Android Debug Server

Reusable, short-lived ARM64 Android environment for debugging applications you own or are authorized to test. It uses an ARM EC2 VM, a root-capable redroid container, and an SSH-only ADB tunnel.

## Safety and scope

- Use this only for your own applications, devices, and accounts, or with written authorization.
- Do not expose ADB, Android debugging ports, or container ports to the public internet.
- Use a new security group with SSH restricted to one operator IP.
- Treat the VM as temporary. Configure automatic shutdown/termination and `DeleteOnTermination` for its EBS volume.

## Architecture

```text
Local ADB ─ SSH tunnel ─ EC2 ARM64 host ─ localhost:5555 ─ redroid Android
```

The Android ADB endpoint stays private on the EC2 host. Only SSH is reachable, from a single approved IP address.

## Recommended temporary EC2 shape

For a modest debug session, use a Graviton instance with 4 GiB RAM (for example `t4g.medium`), Ubuntu Server ARM64, and a 30 GiB gp3 root volume. Check current pricing in your AWS region before launch. Set a short lifetime appropriate for the work.

## AWS setup

1. Create a new security group.
2. Add exactly one inbound rule: TCP 22 from `YOUR_PUBLIC_IP/32`.
3. Launch an Ubuntu ARM64 instance in a subnet that assigns a public IPv4 address.
4. Use the `cloud-init.yaml.example` template as user data. Replace the SSH public key placeholder.
5. Set instance-initiated shutdown behavior to `terminate` and enable EBS `DeleteOnTermination`.

Example read-only discovery commands:

```bash
aws ec2 describe-vpcs --filters Name=isDefault,Values=true
aws ec2 describe-subnets --filters Name=default-for-az,Values=true
aws ssm get-parameter \
  --name /aws/service/canonical/ubuntu/server/24.04/stable/current/arm64/hvm/ebs-gp3/ami-id
```

## Host validation

Connect with your own key:

```bash
ssh -i ~/.ssh/YOUR_KEY ubuntu@YOUR_SERVER_IP
```

Verify required kernel support and Docker:

```bash
uname -m
docker --version
lsmod | grep binder || true
ls -ld /dev/binderfs
```

Expected architecture is `aarch64` and the binder driver must be available. If binder cannot be enabled, do not try to force it on a production server; use a dedicated VM.

## Start redroid

On the VM:

```bash
sudo mkdir -p /opt/android-debug/data
sudo docker run --privileged -d \
  --name android-debug \
  --restart unless-stopped \
  -v /opt/android-debug/data:/data \
  -p 127.0.0.1:5555:5555 \
  redroid/redroid:12.0.0-latest \
  androidboot.redroid_gpu_mode=guest
```

Do not change `127.0.0.1:5555` to `0.0.0.0:5555`.

## Connect local ADB through SSH

On the workstation, keep this command running:

```bash
ssh -i ~/.ssh/YOUR_KEY -N \
  -L 127.0.0.1:5556:127.0.0.1:5555 \
  ubuntu@YOUR_SERVER_IP
```

Then, in another terminal:

```bash
adb connect 127.0.0.1:5556
adb -s 127.0.0.1:5556 shell getprop ro.product.cpu.abi
adb -s 127.0.0.1:5556 root
adb -s 127.0.0.1:5556 shell id
```

The expected ABI is `arm64-v8a`. Use application bundles that contain ARM64 splits.

## Installing app bundles

Do not install a lone `base.apk` if Android reports a missing split. Extract a complete app bundle and install only the required architecture, language, density, and dynamic-feature splits from the same version.

```bash
adb -s 127.0.0.1:5556 install-multiple -r -g \
  base.apk split_config.arm64_v8a.apk split_config.en.apk split_config.xhdpi.apk
```

Add required dynamic-feature APKs when the bundle declares them. Never mix split APKs from different releases.

## Memory capture for authorized unpacking work

When a protected app rejects instrumentation re-attachment, root-level memory capture can be less invasive than repeatedly injecting agents. Capture only the target application's readable DEX mappings and store the result locally.

High-level workflow:

1. Start the authorized app normally.
2. Exercise the owned-device paths needed to materialize the code under investigation.
3. Identify `anon:dalvik-DEX data` mappings in `/proc/<pid>/maps`.
4. Read those mappings from `/proc/<pid>/mem` as root.
5. Carve only valid DEX records by checking the DEX magic and declared file size.
6. Preserve the original partial dumps and record exactly which app flows were exercised.

Avoid exposing dumps or customer/account content. Keep results in private storage unless they have been sanitized for sharing.

## Teardown checklist

1. Pull only the artifacts you need.
2. Stop the Android container.
3. Terminate the EC2 instance.
4. Confirm the EBS volume was deleted.
5. Delete the temporary security group after confirming no instance uses it.

## Repository hygiene

This repository intentionally contains no credentials, IP addresses, AWS account IDs, package identifiers, APKs, DEX files, logs, or captured memory.
