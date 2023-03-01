FROM node:14

WORKDIR /opt/project

COPY yarn.lock .
COPY package.json .

RUN yarn install --frozen-lockfile
RUN apt-get update && apt-get install -y rsync

# project files to be added by mounting a volume

CMD ["yarn", "build"]
