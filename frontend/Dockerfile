FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

FROM base AS test
RUN npm run test

FROM base AS build
RUN npm run build

FROM nginxinc/nginx-unprivileged:alpine AS production
COPY --from=build --chown=101:101 /app/dist /usr/share/nginx/html
EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]