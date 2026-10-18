# teleport-oss-build

Build di Teleport dal sorgente AGPL-3.0 di [gravitational/teleport](https://github.com/gravitational/teleport), per le appliance Guardian Edge e per il rendezvous centrale.

## Perché

Dalla versione 16 i binari e le immagini pubblicati da Gravitational sono distribuiti con la licenza «Teleport Community Edition», che vieta di rivenderli o incorporarli in prodotti e servizi. Il **sorgente** resta AGPL-3.0. Un binario compilato da noi dal tag di rilascio è quindi AGPL: lecito da incorporare nell'appliance, con gli obblighi AGPL (offerta del sorgente agli utenti via rete, note di copyright, nessun uso del marchio Teleport per il nostro prodotto).

Il Makefile upstream distingue da solo le build di terzi (`GITHUB_REPOSITORY_OWNER` ≠ `gravitational` → logo «agpl»). I passi Enterprise sono dietro `if [ -f e/Makefile ]`: il sottomodulo `e/` non viene clonato.

## Cosa fa il workflow

`build-teleport-oss` (`workflow_dispatch`, parametro `tag`):

1. checkout di `gravitational/teleport` al tag, senza sottomoduli; controllo che `LICENSE` sia AGPL e che `e/` non esista;
2. Go (da `go.mod`), Node 24 + pnpm (da `packageManager`), Rust (da `rust-toolchain.toml`) con target wasm32;
3. `make release-unix-preserving-webassets` dentro il buildbox CentOS 7 di upstream (come le loro release amd64: glibc vecchie, quindi i binari partono anche su Debian 12 / PVE 8), con il Go dichiarato dal tag; prova che `teleport`, `tctl`, `tsh`, `tbot` partano su Debian 12;
4. ricostruzione dell'archivio con la `LICENSE` AGPL al posto di `LICENSE-community`;
5. `teleport version` (deve non essere Enterprise);
6. **prova HA**: auth avviato con `storage.type: postgresql` e audit su Postgres, `tctl status`, tabelle nel DB;
7. release `vX.Y.Z-ss4i.N` con archivio, sha256 e firma cosign keyless;
8. **immagine** `ghcr.io/ss4i/teleport-oss:vX.Y.Z` (e `:vX.Y.Z-ss4i.N`): gli stessi binari su `distroless/cc-debian12`, con base, librerie ed `ENTRYPOINT` del Dockerfile distroless di upstream, quindi sostituisce `teleport-distroless` senza toccare il compose. Firmata con cosign per digest.

Verifica dell'immagine (nel compose conviene fissarla per digest):

```sh
cosign verify ghcr.io/ss4i/teleport-oss@sha256:<digest>   --certificate-identity-regexp 'github.com/ss4i/teleport-oss-build'   --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Verifica di una release:

```sh
cosign verify-blob --bundle teleport-vX.Y.Z-ss4i-linux-amd64-bin.tar.gz.sigstore.json \
  --certificate-identity-regexp 'github.com/ss4i/teleport-oss-build' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  teleport-vX.Y.Z-ss4i-linux-amd64-bin.tar.gz
```

## Non incluso

FIDO2 e PIV (chiavi hardware lato client): all'appliance servono il nodo e il servizio app, non i client.

`teleport-update` (aggiornamento automatico degli agenti): scarica i binari ufficiali dal CDN Gravitational, che hanno la licenza Community. Viene tolto dall'archivio; gli aggiornamenti passano dalle release di questo repo.

Anche il rendezvous centrale e i `tsh` degli operatori SS4I usano binari compilati da qui, non quelli ufficiali: servono a erogare un servizio, uso che la licenza Community non consente.
