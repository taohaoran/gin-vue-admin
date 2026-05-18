# 如果需要用 cicd ，请设置环境变量：
# variables:
#     DOCKER_BUILDKIT: 1

# 升级至Node.js 22 LTS，pnpm 11.x依赖node:sqlite内置模块，Node 20不支持
FROM node:22-slim AS base
ENV PNPM_HOME="/pnpm"
ENV PATH="$PNPM_HOME:$PATH"
RUN corepack enable
COPY . /app
WORKDIR /app


FROM base AS prod-deps
# pnpm 11.x默认拒绝未声明的构建脚本，--ignore-scripts跳过构建以避免ERR_PNPM_IGNORED_BUILDS错误
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --prod --ignore-scripts

FROM base AS build
COPY --from=prod-deps /app/node_modules /app/node_modules
# pnpm 11.x默认拒绝未声明的构建脚本，--ignore-scripts跳过构建以避免ERR_PNPM_IGNORED_BUILDS错误
RUN --mount=type=cache,id=pnpm,target=/pnpm/store pnpm install --ignore-scripts && pnpm run build


FROM nginx:alpine
LABEL MAINTAINER="bypanghu@163.com"
COPY --from=base  /app/.docker-compose/nginx/conf.d/my.conf /etc/nginx/conf.d/my.conf
COPY --from=build  /app/dist /usr/share/nginx/html
RUN ls -al /usr/share/nginx/html
