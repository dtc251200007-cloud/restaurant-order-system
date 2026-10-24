FROM node:22-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm install --omit=dev && npm cache clean --force
COPY server.js ./
COPY public ./public
RUN mkdir -p /var/log/restaurant && chown -R node:node /usr/src/app /var/log/restaurant
USER node
ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "server.js"]
