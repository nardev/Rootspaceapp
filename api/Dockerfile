FROM node:14

WORKDIR /opt/project

COPY yarn.lock .
COPY package.json .

RUN yarn install --frozen-lockfile

# project files to be added by mounting a volume

CMD ["yarn", "start"]