# use this empty Dockerfile to build your assignment

# NOTE: The instructions are now in the README.md file in this directory.

FROM node:6-alpine

WORKDIR /usr/src/app

COPY package.json .

RUN apk add --no-cache tini && \
    npm install && \
    npm cache clean --force

COPY . .

EXPOSE 3000

CMD ["/sbin/tini", "node", "./bin/www"]