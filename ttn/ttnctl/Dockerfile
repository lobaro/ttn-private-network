FROM alpine

RUN apk --update add ca-certificates && rm -rf /var/cache/apk/*

RUN apk --update add curl tar \
  && mkdir /ttnsrc \
  && curl -sL -o /ttnsrc/ttnctl-linux-amd64.tar.gz "https://ttnreleases.blob.core.windows.net/release/master/ttnctl-linux-amd64.tar.gz" \
  && tar -xf /ttnsrc/ttnctl-linux-amd64.tar.gz -C /ttnsrc \
  && mv /ttnsrc/ttnctl-linux-amd64 /usr/local/bin/ttnctl \
  && chmod 755 /usr/local/bin/ttnctl \
  && rm -rf /ttnsrc \
  && apk del curl tar \
  && rm -rf /var/cache/apk/*

ENTRYPOINT ["/usr/local/bin/ttnctl"]
