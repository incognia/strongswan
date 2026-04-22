# FortiGate Email Code / PASSCODE support for xauth-generic

This fork adds a one-line change to the `xauth-generic` plugin so that
strongSwan's IKEv1 + XAuth peer can respond to servers that issue a
`CPRQ(X_TYPE X_CODE X_MSG)` challenge — the one FortiGate sends when it
asks for a dynamic *Email Code*, *SMS Code* or FortiToken value after
validating the permanent password.

Upstream's `xauth_generic_t::process_peer()` only handles
`XAUTH_USER_PASSWORD` (16521) and `XAUTH_NEXT_PIN` (16527); any other
attribute falls through to `default: break;` and strongSwan silently
sends back an empty CFG_REPLY, which the FortiGate interprets as "no
answer" and tears the tunnel down.

With the patch, `XAUTH_PASSCODE` (16522) is treated like
`XAUTH_NEXT_PIN`, which means:

- The shared-key lookup uses `SHARED_PIN` instead of `SHARED_EAP`,
  so `charon-cmd`'s credential callback prompts the user for a *fresh*
  value (visible as `PIN:` on the terminal) rather than re-sending the
  cached XAuth password.
- The reply is a proper `CPRP(X_CODE)` with the user-supplied code,
  which the FortiGate accepts and completes phase 2.

See commit `feat/xauth-generic-passcode` for the actual diff.

## Scope and limitations

- Only the peer side is patched (what a dialup client needs). The
  server side is unchanged.
- The change is intentionally minimal — adding one `case` label — to
  make it trivial to forward-port or cherry-pick into upstream if the
  strongSwan team wants to adopt it.
- This does **not** alter how the plugin handles `USER_NAME`,
  `USER_PASSWORD` or `NEXT_PIN`. Existing deployments keep working
  identically.

## Quick build (tarball route — any distro)

Requires GCC, autotools and `gmp-devel` (or your distro's equivalent).

```bash
git clone git@github.com:incognia/strongswan.git
cd strongswan
git checkout feat/xauth-generic-passcode
./autogen.sh           # only if building from a fresh clone
./configure --prefix=/usr --sysconfdir=/etc --libdir=/usr/lib64 \
            --disable-defaults --enable-xauth-generic
make -C src/libcharon/plugins/xauth_generic
```

The artifact is
`src/libcharon/plugins/xauth_generic/.libs/libstrongswan-xauth-generic.so`.
Back up the distro's version and drop the patched `.so` in place:

```bash
sudo cp /usr/lib64/strongswan/plugins/libstrongswan-xauth-generic.so \
        /root/libstrongswan-xauth-generic.so.dist
sudo install -m 0755 \
    src/libcharon/plugins/xauth_generic/.libs/libstrongswan-xauth-generic.so \
    /usr/lib64/strongswan/plugins/libstrongswan-xauth-generic.so
sudo restorecon /usr/lib64/strongswan/plugins/libstrongswan-xauth-generic.so
sudo systemctl restart strongswan
```

Rollback is just copying the `.so.dist` back and restarting the service.

### Per-distro build dependencies

- **Fedora / RHEL / derivatives**

  ```bash
  sudo dnf install -y gcc make autoconf automake libtool pkgconfig \
                      gmp-devel gperf flex bison
  ```

  The stock `strongswan` RPM from Fedora ships under `/usr/lib64/`.

- **Debian / Ubuntu**

  ```bash
  sudo apt install -y build-essential autoconf automake libtool \
                      pkg-config libgmp-dev gperf flex bison
  ```

  On Debian and Ubuntu the plugin lives at
  `/usr/lib/x86_64-linux-gnu/strongswan/plugins/libstrongswan-xauth-generic.so`.
  Adjust the `install`/`cp` target accordingly.

## Using the patched plugin with charon-cmd

Once installed, `charon-cmd` (which registers its own credential set
that prompts on the TTY) is the easiest way to test. Against a
FortiGate dialup with `psk + xauth + email code`:

```bash
sudo systemctl stop strongswan
sudo charon-cmd \
    --host <fortigate-ip> \
    --identity <vpn-user> \
    --xauth-username <vpn-user> \
    --remote-ts <internal-subnet> \
    --ike-proposal aes128-sha1-modp1536 \
    --esp-proposal aes128-sha1-modp1536 \
    --profile ikev1-xauth-psk-am
```

Expected sequence on the terminal:

```
Preshared Key:             ← permanent PSK
EAP password:              ← permanent XAuth password
PIN:                       ← the Email Code that arrives by mail
IKE_SA cmd[1] established ...
CHILD_SA cmd{1} established with SPIs ... and TS VIP/32 === <internal-subnet>
```

`Ctrl+C` tears the tunnel down cleanly.

You can drive this from an `expect` wrapper to pre-fill the PSK and the
permanent password so the user only has to paste the *Email Code* when
it arrives — see the examples shipped with my Tamaulipas deployment
(`vpn-tamaulipas-ipsec`, `vpn-reynosa-ipsec`).

## Why a one-line change fixes the whole flow

- FortiGate's *Email Code* challenge uses IANA attribute 16522
  (`XAUTH_PASSCODE`), not the older 16521 (`XAUTH_USER_PASSWORD`) or
  16527 (`XAUTH_NEXT_PIN`) that `xauth-generic` already knows.
- `SHARED_EAP` is used for the first-round password prompt, so any
  later round that asks for the same `SHARED_EAP` would return the
  cached permanent password — which is *not* what we want in the
  Email Code round.
- `SHARED_PIN` is a separate cache namespace, so reusing it forces a
  fresh prompt. That's exactly how upstream already handles
  `XAUTH_NEXT_PIN`, and it's the only behaviour change needed for the
  FortiGate case.

## Upstreaming

This is intended to be proposed upstream once the author signs the
strongSwan Contributor License Agreement. In the meantime, pull from
`incognia/strongswan` on the `feat/xauth-generic-passcode` branch.

## Report back

Issues or questions about this fork's FortiGate behaviour: open an
issue on <https://github.com/incognia/strongswan/issues>. Upstream
strongSwan issues unrelated to this patch should go to
<https://github.com/strongswan/strongswan/issues>.
