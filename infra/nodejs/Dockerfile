# ========== Build Stage ==========
FROM node:22-alpine AS builder

WORKDIR /usr/src/app

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile

COPY tsconfig.json ./
COPY prisma ./prisma
COPY src ./src

RUN npx prisma generate
RUN npm run build_prod

# ========== Runtime Stage ==========
FROM node:22-alpine

RUN mkdir -p /usr/src/app/uploads
WORKDIR /usr/src/app

COPY package.json ./
COPY --from=builder /usr/src/app/node_modules ./node_modules
COPY --from=builder /usr/src/app/dist ./dist
COPY --from=builder /usr/src/app/prisma ./prisma

EXPOSE 3003
CMD ["node", "./dist/server.js"]
