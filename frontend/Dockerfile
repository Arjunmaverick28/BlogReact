FROM node:22-bookworm-slim AS build

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

ENV VITE_API_URL=/api

RUN npm run build


FROM nginx:1.29-alpine AS runtime

ENV BACKEND_HOST=blogapp-backend
ENV BACKEND_PORT=5000

COPY --from=build /app/dist /usr/share/nginx/html

COPY nginx.conf /etc/nginx/nginx.conf

COPY default.conf /etc/nginx/conf.d/default.conf

RUN mkdir -p \
    /tmp/nginx/client_temp \
    /tmp/nginx/proxy_temp \
    /tmp/nginx/fastcgi_temp \
    /tmp/nginx/uwsgi_temp \
    /tmp/nginx/scgi_temp \
    && chown -R nginx:nginx \
    /usr/share/nginx/html \
    /tmp/nginx

EXPOSE 8080

USER nginx

CMD ["nginx", "-g", "daemon off;"]
