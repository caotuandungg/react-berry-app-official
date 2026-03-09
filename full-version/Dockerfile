

# Stage 1: deps — Install dependencies
FROM node:18-alpine AS deps

# BẮT BUỘC CÓ: Tạo thư mục /app riêng để code không bị trộn lẫn rác vào hệ điều hành gốc của Container.
WORKDIR /app

COPY package.json package-lock.json ./

RUN npm ci --frozen-lockfile && \
    npm cache clean --force

# Stage 2: builder — Build production React bundle
FROM node:18-alpine AS builder

# BẮT BUỘC CÓ LẠI: Vì Stage 2 là một máy ảo mới tinh, phải khai báo lại thư mục làm việc để hứng code.
WORKDIR /app

# CỐT LÕI CỦA MULTI-STAGE: Bê nguyên cục thư viện (node_modules) đã tải xong từ Stage 1 sang Stage 2. Không có dòng này Stage 2 không build được.
# KHÔNG THỂ BỎ DÒNG NÀY ĐƯỢC VÀ CHẮC CHẮN SẼ BÁO LỖI. Vứt nó lên Tầng 1 sẽ khiến Tầng 2 mù mắt hoàn toàn.
COPY --from=deps /app/node_modules ./node_modules

COPY . .

ARG REACT_APP_VERSION
ARG REACT_APP_API_URL

ENV REACT_APP_VERSION=$REACT_APP_VERSION \
    REACT_APP_API_URL=$REACT_APP_API_URL

ENV GENERATE_SOURCEMAP=false
ENV NODE_ENV=production

RUN npm run build && \
    find /app/build -name '*.map' -delete && \
    # QUAN TRỌNG NHẤT VỀ HIỆU NĂNG: Nén sẵn các file tĩnh thành .gz ở mức cao nhất, giúp Nginx cân 100.000 user mà không nghẽn CPU.
    find /app/build/static -type f \( -name "*.js" -o -name "*.css" -o -name "*.svg" \) \
        -exec gzip -9 -k {} \; && \
    # XOÁ BẢN GỐC: Xoá các file tĩnh chưa nén đi để tối ưu hoá 500% dung lượng ổ cứng của Container.
    find /app/build/static -type f \( -name "*.js" -o -name "*.css" -o -name "*.svg" \) \
        ! -name "*.gz" -delete

# Stage 3: runtime — Production nginx (NO Node, npm , source code, build tools)
FROM nginx:1.27-alpine-slim AS runtime

LABEL maintainer="VNCloudOps" \
      description="Berry React Admin Dashboard - Production Container" \
      version="3.6.0"

RUN rm /etc/nginx/conf.d/default.conf

COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/templates/ /etc/nginx/templates/
COPY nginx/docker-entrypoint.sh /docker-entrypoint.sh
COPY --chown=nginx:nginx --from=builder /app/build /usr/share/nginx/html

RUN set -eux; \
    sed -i 's/\r$//' /docker-entrypoint.sh && chmod +x /docker-entrypoint.sh; \
    chown nginx:nginx /etc/nginx/conf.d; \
    chown -R nginx:nginx /var/cache/nginx; \
    chown -R nginx:nginx /var/log/nginx; \
    touch /var/run/nginx.pid; \
    chown nginx:nginx /var/run/nginx.pid; \
    mkdir -p /tmp/nginx/{client_temp,proxy_temp,fastcgi_temp,uwsgi_temp,scgi_temp}; \
    chown -R nginx:nginx /tmp/nginx; \
    find /usr/share/nginx/html -name '.env*' -delete 2>/dev/null || true; \
    rm -f /usr/share/nginx/html/asset-manifest.json; \
    apk --no-cache del apk-tools 2>/dev/null || true; \
    rm -rf /var/cache/apk/* /tmp/* /root/.cache

USER nginx

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -q --spider http://127.0.0.1:8080/health || exit 1

ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["nginx", "-g", "daemon off;"]
