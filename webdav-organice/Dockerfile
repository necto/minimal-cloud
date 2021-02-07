FROM node:12.13.1

RUN apt-get update && apt-get install -yq --no-install-recommends nginx nginx-extras gosu apache2-utils && rm -rf /var/lib/apt/lists/*

RUN ln -sf /dev/stderr /var/log/nginx/error.log
RUN chmod go+rwX -R /var /run
VOLUME /media

COPY organice /opt/organice
WORKDIR /opt/organice

RUN yarn install \
    && yarn global add serve \
    && yarn build \
    && yarn cache clean \
    && rm -rf node_modules

COPY entrypoint.sh /
COPY nginx.conf /etc/nginx/

ENV NODE_ENV=production
EXPOSE 8080

ENTRYPOINT ["/entrypoint.sh"]
CMD ["nginx"]
