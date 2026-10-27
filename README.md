# BapakTunnel: RHB VPN Split Tunnel

Connect VPN bank (AnyConnect) di macOS. Hanya app internal yang disenaraikan lalu VPN, selebihnya guna internet biasa.

## Cara Kerja

1. `bapakTunnel` generate tokencode guna `stoken`.
2. Script jalankan `bapakTunnel.expect` guna `sudo`.
3. Expect launch `openconnect` dan isi PASSCODE/Password.
4. `vpn-slice` route hanya hostname dalam `VPN_ROUTES` dan tambah entri `/etc/hosts` sementara.

## Install

```bash
brew install openconnect stoken expect vpn-slice
```

Token SecurID mesti dah di-import ke `stoken`. Kalau tool bukan di `/opt/homebrew/bin` (Mac Intel guna `/usr/local/bin`), ubah path dalam script.

## Fail

```
bapakTunnel          # script utama (tiada extension)
bapakTunnel.expect   # mesti dalam folder sama
```

`bapakTunnel.expect`:

```tcl
set timeout -1

spawn $env(OPENCONNECT) \
    --protocol=anyconnect \
    --user=$env(VPN_USERNAME) \
    --script "$env(VPN_SCRIPT)" \
    $env(VPN_DOMAIN)

expect {
    -re {(?i)PASSCODE:} {
        send -- "$env(VPN_PASSCODE)\r"
        exp_continue
    }
    -re {(?i)Password:} {
        send -- "$env(VPN_PASSWORD)\r"
        exp_continue
    }
    eof {
        catch wait result
        exit [lindex $result 3]
    }
}
```

## Setting dalam bapakTunnel

Isi `USERNAME`, `PIN`, `VPN_PASSWORD` di bahagian atas, dan senaraikan app dalam `VPN_ROUTES`:

```bash
VPN_ROUTES="webmail.rhbgroup.com jira.intranet.rhbgroup.com bitbucket.intranet.rhbgroup.com confluence.intranet.rhbgroup.com myoneid.iam.intranet.rhbgroup.com rhbgroup.sharepoint.com smebankingsit.rhbgroup.com smebankinguat.rhbgroup.com smefinancingsit.rhbgroup.com smefinancinguat.rhbgroup.com smefinancingintrasit.rhbgroup.com smefinancingintrauat.rhbgroup.com"
VPN_SCRIPT="$VPN_SLICE -v $VPN_ROUTES"
```

Tukar baris pertama script kepada `#!/bin/bash`. Script guna `[[ ]]` dan `pipefail` yang ciri bash, bukan POSIX `sh`. Di macOS `sh` kebetulan bash jadi jalan, tapi tak boleh diharap.

## Guna

```bash
chmod +x bapakTunnel
./bapakTunnel
```

Masukkan password sudo. Biarkan terminal terbuka. `Ctrl+C` untuk putuskan, route dan entri `/etc/hosts` dibuang automatik.

## Tambah App Baru

1. Ambil hostname sebenar dari URL (jangan teka). Untuk git, guna `git remote -v`.
2. Tambah dalam `VPN_ROUTES`, lepas tu reconnect.
3. Kalau page separuh load atau stuck di login, buka DevTools > Network, cari domain lain (SSO/API) dan tambah juga.

## Troubleshooting

| Masalah | Penyelesaian |
|---|---|
| `Lookup ... returned nothing` | Hostname salah atau DNS VPN tak kenal, semak ejaan |
| App tak boleh dibuka | Hostname tiada dalam `VPN_ROUTES` |
| Browser abaikan `/etc/hosts` | Matikan Secure DNS (DoH) dalam browser |
| DNS lama tersangkut | `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| Passcode ditolak | Tunggu 60 saat, cuba semula |

## Keselamatan

- Jangan hardcode PIN/password. Guna Keychain:
  ```bash
  security add-generic-password -a "$USER" -s bapaktunnel -w
  VPN_PASSWORD="$(security find-generic-password -a "$USER" -s bapaktunnel -w)"
  ```
- Jangan letak script dalam repo atau folder sync. `chmod 700` fail script.
- Server bank push full tunnel. Pastikan IT/security benarkan split tunnel.
