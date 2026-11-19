# Export Authy tokens

Authy has no export button. This repo is how I got my 2FA codes out of it and into Bitwarden, written up step by step because the guides I found were harder to follow than they needed to be.

It works like this: you capture the request the Authy iPhone app makes for your tokens, save the response (your tokens, still encrypted with your Authy backup password), and run `authy_exporter.py`. The script decrypts everything on your own computer and writes a file you can import into Bitwarden, plus `otpauth://` links that work with any other authenticator app.

You don't need a jailbroken phone. You need:

- A computer (Windows, macOS or Linux) with Python 3.7 or newer
- An iPhone with Authy on it
- About 20 minutes

Nothing leaves your computer. The script makes no network calls.

## 1. Install Python and mitmproxy

mitmproxy is the tool that lets you see the request the Authy app makes. `cryptography` is what the script uses to decrypt.

Windows (PowerShell):

```powershell
winget install --id Python.Python.3 -e --source winget
```

Or use the installer from https://www.python.org/downloads/ and tick "Add Python to PATH". Then, in a new PowerShell window:

```powershell
python -m venv venv
venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

macOS:

```bash
brew install python mitmproxy
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Linux (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install -y python3 python3-venv python3-pip
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

If something here fails, see [Install problems](#install-problems) at the bottom.

## 2. Set an Authy backup password

In Authy on your phone, go to Settings, then Accounts, and create a backup password if you don't already have one. Your tokens come back encrypted with it, and the script asks for it at the end.

## 3. Start mitmweb and point your phone at it

Run:

```bash
mitmweb
```

Leave that terminal open. The web interface is at http://127.0.0.1:8081 and the proxy listens on port 8080.

On your iPhone:

1. Go to Settings, Wi-Fi, tap the (i) next to your network, and set HTTP Proxy to Manual. Use your computer's IP address (`ipconfig` on Windows, `ip addr` on Linux, System Settings on a Mac) and port `8080`.
2. Open Safari, go to http://mitm.it and install the iOS profile.
3. Go to Settings, General, VPN & Device Management, and install the mitmproxy profile.
4. Go to Settings, General, About, Certificate Trust Settings, and turn on full trust for mitmproxy.

If no traffic shows up in mitmweb, see [Proxy problems](#proxy-problems).

## 4. Capture the token request

1. In mitmweb, type this into the search bar:

   ```
   ~d "api.authy.com" token
   ```

2. In Authy, go to Settings, Accounts, and turn Backups off. When it asks, tap "Don't Disable". That triggers the request without actually turning your backups off.
3. In mitmweb, click the GET request to `api.authy.com`. Its URL looks like this:

   ```
   https://api.authy.com/json/users/12345678/authenticator_tokens?locale=en&api_key=...&otp2=...&otp3=...&device_id=...&apps=123,456
   ```

4. Copy the whole URL and delete everything after `apps=` so it ends in `apps=`:

   ```
   https://api.authy.com/json/users/12345678/authenticator_tokens?locale=en&api_key=...&otp2=...&otp3=...&device_id=...&apps=
   ```

The `otp2` and `otp3` values in that URL are one-time codes, so if the link stops working later, capture it again.

## 5. Save the response

Open the trimmed URL in a browser on your computer, copy the raw JSON it returns, and save it as `authenticator_tokens.json` in this folder. Or download it directly:

```bash
curl -L "<your trimmed URL>" -o authenticator_tokens.json
```

The file should start with `"message": "success"` and contain an `authenticator_tokens` list, where each entry has an `encrypted_seed`, `salt`, `key_derivation_iterations` and `unique_iv`. If you get HTML or a login page instead, the URL wasn't trimmed right or the codes expired: capture it again.

## 6. Run the script

```bash
python authy_exporter.py
```

Type your Authy backup password when it asks (it isn't shown as you type). The script decrypts each token, prints which ones worked, and writes `authenticator_tokens_export_bitwarden.json`.

Other formats:

```bash
python authy_exporter.py --list-formats       # show all formats
python authy_exporter.py --format text        # otpauth:// links, one per account
python authy_exporter.py -i other.json -f generic
```

| Format | File | What it's for |
|---|---|---|
| `bitwarden` (default) | `.json` | Bitwarden's own import format |
| `text` | `.txt` | Each account's secret and its `otpauth://` link. Paste a link into any authenticator app that accepts one, or turn it into a QR code and scan it |
| `generic` | `.json` | A plain list of names, secrets and links, for your own scripts or backups |
| `lastpass` | `.csv` | A CSV of names and secrets |
| `google`, `microsoft` | `.json` | Simple JSON lists of the secrets. Google Authenticator and Microsoft Authenticator can't import JSON files, so for those apps use the `text` links instead |

Output files are named `<input>_export_<format>.<ext>` and land next to your input file.

## 7. Import, then clean up

For Bitwarden: log in at https://vault.bitwarden.com, go to Tools, Import data, pick "Bitwarden (json)" and upload the export. You can also do it one at a time: open a login item and paste its secret into "Authenticator key (TOTP)".

When you're done:

- Delete the export file. It has your 2FA secrets in plain text. (`authenticator_tokens.json` is still encrypted with your backup password, so keeping it is less risky, but I can't promise this method keeps working if you need it again.)
- Stop mitmweb with Ctrl+C.
- On the iPhone, set the Wi-Fi HTTP Proxy back to Off.
- Delete the mitmproxy profile (Settings, General, VPN & Device Management) and turn off its trust in Certificate Trust Settings. Leaving a trusted proxy certificate on your phone is a real security risk.

The `.gitignore` already keeps the token and export files out of git, in case you're working inside a clone.

## Troubleshooting

### Install problems

- Windows says `python` isn't recognized: try `py -3 --version`. If that works, use `py -3 -m venv venv`. Otherwise rerun the Python installer, pick Modify, and tick "Add Python to PATH".
- PowerShell won't run `Activate.ps1`: run `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser`.
- `pip install mitmproxy` fails with build or SSL errors: run `python -m pip install --upgrade pip setuptools wheel` and try again. On macOS, also run the "Install Certificates.command" that comes with the python.org installer.

### Proxy problems

- No requests show up: check the phone and computer are on the same Wi-Fi, the proxy IP and port `8080` are right, and VPNs are off on both. Your computer's firewall may be blocking port 8080. Load http://example.com in Safari: it should appear in mitmweb.
- Certificate errors, or HTTPS requests don't show: make sure the trust toggle in Certificate Trust Settings is on. If it still fails, delete the profile, restart the phone, and install it again from http://mitm.it. The phone's date and time need to be correct.
- The Authy request never appears: clear the filter and look for anything to `api.authy.com`, then toggle Backups off again and tap "Don't Disable".

### Decryption problems

- Every token fails: the backup password is wrong. Retype it carefully; extra spaces at the start or end count.
- Some tokens fail with base64 or padding errors: the JSON was probably damaged when you copied it. Download it again with `curl` and check it's valid at https://jsonlint.com.

### Bitwarden won't import

- Make sure you picked "Bitwarden (json)" and uploaded the `_export_bitwarden.json` file.
- If one entry is the problem, delete entries from a copy of the file until it imports, to find which one.

## Credits

The capture method comes from the community. This gist has more fixes and discussion: https://gist.github.com/gboudreau/94bb0c11a6209c82418d01a59d958c93. Also see [BrenoFariasdaSilva/Authy-iOS-MiTM](https://github.com/BrenoFariasdaSilva/Authy-iOS-MiTM). Built on [mitmproxy](https://mitmproxy.org/) and [cryptography](https://cryptography.io/).

Use this only on your own accounts. MIT license, see [LICENSE](LICENSE).
