# Immagine di Teleport dai binari AGPL di questo repo. Ricalca
# build.assets/charts/Dockerfile-distroless di upstream (stessa base, stesse
# librerie, stesso ENTRYPOINT), così sostituisce teleport-distroless senza
# toccare il compose. Contesto: la cartella `teleport/` estratta dall'archivio.
ARG BASE_IMAGE=gcr.io/distroless/cc-debian12

FROM debian:12 AS staging
RUN apt-get update
COPY fetch-debs ./
RUN ./fetch-debs dumb-init libpam0g libaudit1 libcap-ng0

FROM debian:12 AS teleport
COPY teleport/ /tmp/teleport/
RUN mkdir -p /opt/staging/usr/local/bin /opt/staging/etc/teleport /opt/staging/usr/share/doc/teleport && \
    for b in teleport tctl tsh tbot fdpass-teleport; do install -m 0755 /tmp/teleport/$b /opt/staging/usr/local/bin/; done && \
    install -m 0644 /tmp/teleport/LICENSE /opt/staging/usr/share/doc/teleport/LICENSE && \
    test ! -e /opt/staging/usr/local/bin/teleport-update

FROM $BASE_IMAGE
COPY --from=teleport /opt/staging /
COPY --from=staging /opt/staging/root /
COPY --from=staging /opt/staging/status /var/lib/dpkg/status.d
ENV TELEPORT_TOOLS_VERSION=off
STOPSIGNAL SIGQUIT
ENTRYPOINT ["/usr/bin/dumb-init", "/usr/local/bin/teleport", "start", "-c", "/etc/teleport/teleport.yaml"]
