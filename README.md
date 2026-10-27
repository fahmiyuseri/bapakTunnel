# BapakTunnel: RHB VPN Split Tunnel Guide

Panduan guna script `bapakTunnel` untuk connect VPN bank (AnyConnect) dengan **split tunnel**: hanya app internal yang dipilih lalu VPN, selebihnya guna internet biasa.

## 1. Gambaran Keseluruhan

Flow dia:

1. `bapakTunnel.sh` generate tokencode SecurID guna `stoken`.
2. Script jalankan `bapakTunnel.expect` sebagai root (`sudo`).
3. Expect launch `openconnect`, isi PASSCODE dan Password secara automatik.
4. `openconnect` panggil `vpn-slice` sebagai script, yang hanya route hostname/subnet dalam senarai dan tambah entri `/etc/hosts` sementara.

## 2. Dependencies

| Tool | Fungsi | Install |
|---|---|---|
| Homebrew | Package manager macOS | https://brew.sh |
| openconnect | Client VPN (protocol AnyConnect) | `brew install openconnect` |
| stoken | Generate tokencode RSA SecurID | `brew install stoken` |
| expect | Automasi input PASSCODE/Password | `brew install expect` |
| vpn-slice | Split tunnel (route + DNS + `/etc/hosts`) | `brew install vpn-slice` |
| python3 | Diperlukan oleh vpn-slice | Biasanya dah ada dengan Homebrew |

Install sekali gus:

```bash
brew install openconnect stoken expect vpn-slice
```

Kalau `vpn-slice` tiada dalam Homebrew versi kau, install guna pip:

```bash
pip3 install vpn-slice
```

Semak path semua tool (script guna `/opt/homebrew/bin/...`, untuk Apple Silicon):

```bash
which stoken openconnect vpn-slice expect
```

Kalau path lain (contoh Mac Intel guna `/usr/local/bin`), ubah pembolehubah `STOKEN`, `OPENCONNECT`, `VPN_SLICE` dalam script.

### Prasyarat lain

- Token SecurID dah di-import ke `stoken` (fail `~/.stokenrc`). Test: `stoken tokencode` patut keluarkan 8 digit.
- Akaun user kau ada kebenaran `sudo`.
- Kebenaran dari IT bank untuk guna split tunnel (lihat bahagian Keselamatan).

## 3. Struktur Fail

```
bapakTunnel.sh       # script utama (jalankan ini)
bapakTunnel.expect   # automasi login, mesti dalam folder yang sama
```

### bapakTunnel.expect

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

## 4. Konfigurasi dalam bapakTunnel.sh

Perubahan utama untuk split tunnel:

```bash
# Senarai hostname yang lalu VPN (asing dengan space)
VPN_ROUTES="webmail.rhbgroup.com jira.intranet.rhbgroup.com bitbucket.intranet.rhbgroup.com"

# VPN_SCRIPT sekarang command vpn-slice, bukan path fail
VPN_SCRIPT="$VPN_SLICE -v $VPN_ROUTES"
```

Semakan fail pun kena ubah, sebab `VPN_SCRIPT` bukan fail lagi:

```bash
if [[ ! -f "$EXPECT_SCRIPT" ]]; then
    echo "[ERROR] Missing Expect script." >&2
    exit 1
fi
```

Pastikan `VPN_SCRIPT` ada dalam baris `export` dan `unset` macam asal.

Kalau ada error `python3: not found` masa jalan sebagai root, tambah sebelum `sudo`:

```bash
export PATH="/opt/homebrew/bin:$PATH"
```

## 5. Cara Guna

Buat executable (sekali sahaja):

```bash
chmod +x bapakTunnel.sh
```

Connect:

```bash
./bapakTunnel.sh
```

Masukkan password macOS (sudo) bila diminta. Biarkan terminal terbuka, VPN putus bila tutup terminal atau tekan `Ctrl+C`.

### Tanda berjaya

- `CSTP connected` dalam log.
- Dapat IP VPN (contoh `10.8.41.x`).
- Tiada warning "Lookup ... returned nothing" untuk hostname kau.

## 6. Senarai App Semasa

| App | Hostname |
|---|---|
| Webmail | `webmail.rhbgroup.com` |
| Jira | `jira.intranet.rhbgroup.com` |
| Bitbucket | `bitbucket.intranet.rhbgroup.com` |

## 7. Tambah App Baru

1. Buka app dalam browser, ambil hostname dari URL.
2. Tambah dalam `VPN_ROUTES`.
3. Disconnect dan connect semula.
4. Kalau page separuh load atau stuck di login, buka DevTools (F12), tab **Network**, cari domain lain yang dipanggil (SSO, API, CDN), dan tambah juga.
5. Untuk `git clone` guna SSH, semak hostname dalam URL clone. Kalau berbeza dari web, tambah sekali.

Jangan teka hostname. Guna yang sebenar dari URL atau `git remote -v`.

### Pilihan lanjutan

- Route satu subnet: tambah contoh `172.26.0.0/16` dalam `VPN_ROUTES`.
- DNS ikut suffix: tambah `-d intranet.rhbgroup.com` supaya semua hostname bawah suffix tu guna DNS VPN. Route masih perlu ditambah untuk host/subnet berkaitan.
- Elak `-d rhbgroup.com` sebab boleh kacau site public bank.

## 8. Test Selepas Connect

```bash
grep rhbgroup /etc/hosts
```

Patut ada entri bertanda `vpn-slice-utunX AUTOCREATED`.

```bash
netstat -rn | grep utun
```

Patut ada route untuk IP app kau.

```bash
curl -v --connect-timeout 10 https://jira.intranet.rhbgroup.com 2>&1 | head -30
```

`Connected to ...` bermaksud route OK.

```bash
curl -s ifconfig.me
```

Patut tunjuk IP ISP kau, bukan IP bank. Itu bukti split tunnel jalan.

## 9. Troubleshooting

| Masalah | Punca | Penyelesaian |
|---|---|---|
| `Lookup ... returned nothing` | Hostname salah atau DNS VPN tak kenal | Semak ejaan hostname, tanya IT |
| Connect berjaya tapi app tak boleh dibuka | Hostname tak ada dalam `VPN_ROUTES` | Tambah, reconnect |
| Page separuh load / login stuck | Domain SSO/API tertinggal | Cari dalam DevTools Network, tambah |
| Browser tak guna `/etc/hosts` | Secure DNS (DoH) aktif | Matikan Secure DNS dalam browser |
| `python3: not found` | PATH root tak ada Homebrew | `export PATH="/opt/homebrew/bin:$PATH"` |
| `Executable not found` | Path tool salah | Semak `which`, ubah path dalam script |
| `networksetup ... not a recognized network service` | Bug vpnc-script biasa (full tunnel) | Guna vpn-slice, tak kena masalah ni |
| DNS lama tersangkut | Cache macOS | `sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder` |
| Passcode ditolak | PIN salah atau token belum berubah | Tunggu 60 saat, cuba semula |

Untuk debug, vpn-slice sudah dalam mod `-v`. Tambah `--dump-http-traffic` pada `openconnect` kalau perlu tengok apa server push.

## 10. Keselamatan dan Compliance

- **Jangan hardcode PIN dan password dalam script.** Letak dalam macOS Keychain:
  ```bash
  security add-generic-password -a "$USER" -s bapaktunnel -w
  VPN_PASSWORD="$(security find-generic-password -a "$USER" -s bapaktunnel -w)"
  ```
- Jangan commit script ke repo atau letak dalam folder sync (iCloud/Dropbox) dengan credentials di dalamnya.
- Fail script patut `chmod 700`.
- Server VPN bank push default route, maksudnya polisi asal ialah full tunnel. Split tunnel dari sisi client memang boleh dibuat secara teknikal, tapi **pastikan IT atau security bank bagi kebenaran** supaya tak melanggar polisi.
- Banner VPN nyatakan semua akses dipantau dan direkod. Guna hanya untuk keperluan kerja.

## 11. Putuskan Sambungan

Tekan `Ctrl+C` dalam terminal. Vpn-slice akan buang route dan entri `/etc/hosts` secara automatik. Semak:

```bash
grep vpn-slice /etc/hosts
```

Kalau ada entri tertinggal selepas crash, buang baris bertanda `vpn-slice` secara manual.
