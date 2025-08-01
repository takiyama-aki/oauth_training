FROM node:16-alpine
WORKDIR /app

# 依存インストール
COPY package.json package-lock.json ./
RUN npm ci

# ソースをコピー（ホットリロード用にマウント）
COPY . ./

EXPOSE 3000

# Vite 開発サーバー起動
CMD ["npm", "run", "dev"]