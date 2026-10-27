# catsafe

`cat` replacement that prints files and stdin with secrets redacted (`[REDACTED]`). Single-file Perl, no dependencies.

## Usage

    catsafe file1 file2
    cat config.json | jq | catsafe
    catsafe -C file    # force color
    catsafe -N file    # no color

Redacts values of keys matching secret/key/token/password/etc, PEM private keys (multi-line and single-line JSON), URL credentials, Bearer/Basic tokens, JWTs, and common API key formats.

## Install

    install -m 0755 catsafe /usr/local/bin/catsafe

## Warning

Pattern based. Review output before sharing it.

## License

MIT
