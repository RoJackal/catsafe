# catsafe

A `cat` replacement that prints files and stdin with secrets replaced by `[REDACTED]`, so output can be pasted into tickets, chats, logs or AI assistants without leaking credentials.

Single-file Perl script. No dependencies beyond a stock `perl` (5.10+).

## Features

- Reads files and/or stdin (`-` or no arguments), like `cat`
- Redacts by key name: any `key = value`, `key: value`, `"key": "value"`, `key => value`, `KEY=value`, `export KEY=value` whose name contains `secret`, `key`, `token`, `password`, `passwd`, `pwd`, `passphrase`, `auth`, `credential`, `private`, `api`, `salt`, `cookie`, `session`, `signature`, `dsn`, `webhook`, `bearer`, `cert`, `access`, `chatid` or ends in `_id` (case-insensitive, substring match)
- Redacts PEM private keys, both multi-line files and the single-line form produced by `jq` / JSON (`"-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----\n"`)
- Redacts passwords inside URLs: `scheme://user:PASSWORD@host`
- Redacts `Bearer`, `Basic` and `Token` credentials in headers
- Redacts well-known token formats: Stripe (`sk_live_`, `pk_test_`...), OpenAI-style `sk-...`, GitHub (`ghp_`, `gho_`...), Slack (`xoxb-`...), AWS access key IDs (`AKIA...`, `ASIA...`), JWTs, Telegram bot tokens
- Redacts long quoted random-looking strings (20+ chars with letters and digits) and bare hex strings of 32+ characters (hashes, session ids)
- Keeps everything else untouched, including line count and layout
- Optional syntax coloring (keys, values, comments, `[sections]`, redactions in bold red)
- Exit code 1 if any file could not be opened, otherwise 0

## Install

    git clone https://github.com/RoJackal/catsafe.git
    install -m 0755 catsafe/catsafe /usr/local/bin/catsafe

Or just download the single file and make it executable:

    curl -fsSL https://raw.githubusercontent.com/RoJackal/catsafe/main/catsafe -o /usr/local/bin/catsafe
    chmod 0755 /usr/local/bin/catsafe

## Usage

    catsafe [-C|--color] [-N|--no-color] [file ...]

| Option | Meaning |
| --- | --- |
| `-C`, `--color` | Force color, even when piped |
| `-N`, `--no-color` | Disable color |
| `file ...` | One or more files; `-` means stdin. No files means stdin |

Color is on by default only when stdout is a terminal, and is disabled when the `NO_COLOR` environment variable is set. `-C` and `-N` override both. With more than one file, each is preceded by a `== filename ==` header.

## Examples

### .env file

    $ catsafe .env
    APP_NAME=shop
    DB_HOST=10.0.0.5
    DB_PASSWORD=[REDACTED]
    API_KEY=[REDACTED]
    export AWS_SECRET_ACCESS_KEY=[REDACTED]
    DEBUG=true

### INI file (comments are scanned too)

    $ catsafe app.ini
    [database]
    host = db.local
    password = [REDACTED]
    # token = [REDACTED]
    user = admin

### YAML

    $ catsafe config.yml
    server:
      port: 8080
      api_key: "[REDACTED]"
      secret: [REDACTED]

### PHP array

    $ catsafe config.php
    $cfg = [
      "host" => "localhost",
      "db_pass" => "[REDACTED]",
      "salt" => "[REDACTED]",
    ];

### Google service account JSON (piped through jq)

    $ jq . service-account.json | catsafe
    {
      "type": "service_account",
      "project_id": "[REDACTED]",
      "private_key_id": "[REDACTED]",
      "private_key": "-----BEGIN PRIVATE KEY-----[REDACTED]-----END PRIVATE KEY-----\n",
      "client_email": "python-script@my-project.iam.gserviceaccount.com",
      "client_id": "[REDACTED]",
      "auth_uri": "[REDACTED]",
      "token_uri": "[REDACTED]",
      "auth_provider_x509_cert_url": "[REDACTED]",
      "client_x509_cert_url": "[REDACTED]",
      "universe_domain": "googleapis.com"
    }

### PEM private key file

    $ catsafe id_rsa
    -----BEGIN RSA PRIVATE KEY-----
    [REDACTED]
    -----END RSA PRIVATE KEY-----

### Passwords in URLs

    $ echo 'DATABASE_URL=postgres://app:pa55w0rd@db:5432/app' | catsafe
    DATABASE_URL=postgres://app:[REDACTED]@db:5432/app

### HTTP headers and known token formats

    $ echo 'curl -H "Authorization: Bearer abcdef1234567890xyz" https://api.example.com' | catsafe
    curl -H "Authorization: Bearer [REDACTED]" https://api.example.com

    $ printf 'stripe sk_live_abcdefghij1234567890\ngh ghp_abcdefghijklmnopqrstuvwxyz0123456789\n' | catsafe
    stripe [REDACTED]
    gh [REDACTED]

### Several files at once

    $ catsafe a.txt b.txt
    == a.txt ==
    a=1
    == b.txt ==
    token=[REDACTED]

### Common sysadmin one-liners

    catsafe /etc/mysql/my.cnf /root/.my.cnf
    catsafe /var/www/html/wp-config.php
    systemctl show myservice -p Environment | catsafe
    docker inspect mycontainer | catsafe
    env | catsafe
    kubectl get secret mysecret -o yaml | catsafe
    ansible-vault view group_vars/all.yml | catsafe
    journalctl -u myservice --since today | catsafe | head -50

### Missing files and exit code

    $ catsafe /nope; echo $?
    catsafe: /nope: No such file or directory
    1

### Color control

    catsafe -C config.ini | less -R     # keep color through a pager
    catsafe -N config.ini > clean.txt   # no escape codes
    NO_COLOR=1 catsafe config.ini       # same as -N

## Limitations

catsafe is pattern based. It is a safety net, not a guarantee.

- Secrets in free text are not recognized: `mysql -u root -pMyPass123`, or `Accepted password for root` in logs, pass through unchanged
- Redaction depends on the key name. A secret stored under a name such as `foo=hunter2` is only caught if the value looks like a known token or a long random string
- False positives are expected, because the match is a substring: `monkey=banana` is redacted (contains `key`), and every `*_id` value is redacted (`USER_ID=42`)
- Multi-line values other than PEM private keys are only redacted on the line where the key name appears
- Short passwords in unquoted free text and base64 blobs under non-secret names are not detected

Always review the output before sharing it.

## License

MIT
