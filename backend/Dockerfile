# Stage 1: Install production deps
FROM node:20-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Run
FROM node:20-alpine

WORKDIR /app

COPY --from=builder /app/node_modules ./node_modules

COPY . .

ENV NODE_ENV=production
ENV PORT=5001

EXPOSE 5001

CMD ["node", "src/server.js"]