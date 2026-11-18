# burpshell

Spawn an interactive shell where HTTP(S) traffic from the tools you run is routed through Burp Suite.

```bash
./burpshell
```

The shell gets the usual proxy environment variables plus a Node.js bootstrap that patches `http`, `https`, and `undici`, so CLI tools, scripts, and Node apps show up in Burp's Proxy history without per-tool configuration.

## Requirements

- Python 3.8+
- Burp Suite running with the Proxy listener enabled (default `127.0.0.1:8080`)
- Burp's CA certificate exported as **PEM**
- Optional: Node.js (the `undici` patch only applies when Node is present)
- `zsh` or `bash`

## Certificate setup

Burp exports the proxy CA in DER by default. Convert it to PEM once:

```bash
openssl x509 -inform der -in cacert.der -out ~/.burp.pem
```

The script looks for the certificate in this order:

1. `--cert PATH`
2. `~/.burp.pem`
3. `~/burp.pem`
4. `/usr/local/share/ca-certificates/burp.crt`

If none exists, the shell still starts but TLS verification for proxied HTTPS will fail.

## Usage

```bash
./burpshell                              # 127.0.0.1:8080, auto-detected cert
./burpshell --host 127.0.0.1 --port 8081
./burpshell --cert /path/to/burp.pem
```

Inside the shell, the prompt is prefixed with a red `[burp]` marker:

```
[burp] ~ $
```

Run any command normally and check Burp's Proxy history:

```bash
curl -sS https://example.com >/dev/null
python3 -c 'import requests; requests.get("https://example.com")'
node -e 'fetch("https://example.com")'
```

Type `exit` to leave the shell.

> **Legal notice:** Use only on systems and targets you are authorised to test, and respect the rules of any bug bounty program or engagement.

## How it works

- Sets `http_proxy`, `https_proxy`, and `all_proxy` (plus uppercase variants) for the child shell.
- Sets `REQUESTS_CA_BUNDLE`, `SSL_CERT_FILE`, `NODE_EXTRA_CA_CERTS`, and `GIT_SSL_CAINFO` to the Burp CA.
- Writes a temporary Node bootstrap and adds it to `NODE_OPTIONS` via `--require`.
- The bootstrap:
  - replaces `http.globalAgent` / `https.globalAgent` with Burp-aware agents;
  - uses HTTP `CONNECT` tunnels for HTTPS;
  - hooks `Module._load` to patch `undici` every time it is required, including after startup.
- Prints `[burp]` status lines to stderr so you can confirm which patches loaded.

## Caveats

- Only processes launched from the spawned shell inherit the proxy settings.
- Programs that ignore proxy environment variables, use raw sockets, or pin certificates will not be captured.
- The `undici` hook targets the common Debian/Ubuntu path `/usr/share/nodejs/undici`; a locally installed or bundled copy may not be patched.
- The temporary bootstrap file is removed when the shell exits.
