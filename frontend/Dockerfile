# Build step
FROM node:18 AS build

WORKDIR /app
COPY . .

# تمرير متغير API وقت الـ build
ARG VITE_API_URL
ENV VITE_API_URL=$VITE_API_URL

RUN npm install && npm run build

# Serve step
FROM nginx:alpine

# نسخ ملفات Nginx config مع دعم تحويل /api إلى backend
COPY nginx.conf /etc/nginx/conf.d/default.conf

# نسخ ملفات البناء إلى مجلد nginx
COPY --from=build /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]

