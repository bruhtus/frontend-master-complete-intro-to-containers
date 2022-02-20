FROM node:12-stretch

USER node

RUN mkdir /home/node/src
WORKDIR /home/node/src

COPY --chown=node:node package.json package.json
RUN yarn

COPY --chown=node:node . .

CMD ["node", "index.js"]
