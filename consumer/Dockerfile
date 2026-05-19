FROM node:20-alpine
RUN apk update && apk upgrade --no-cache
WORKDIR /app
COPY package*.json ./
RUN npm install --omit=dev
COPY . .
USER node
CMD ["node", "tracker-consumer.js"]
