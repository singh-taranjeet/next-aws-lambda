FROM public.ecr.aws/docker/library/node:20.9.0-slim AS builder
WORKDIR /app
COPY . .
# declare the sharp path to be /tmp/node_modules/sharp
ENV NEXT_SHARP_PATH=/tmp/node_modules/sharp
# install the dependencies and build the app
RUN npm ci && npm run build

FROM public.ecr.aws/docker/library/node:20.9.0-slim AS runner
# install aws-lambda-adapter
COPY --from=public.ecr.aws/awsguru/aws-lambda-adapter:0.7.2 /lambda-adapter /opt/extensions/lambda-adapter
# expose port 3000 and set env variables
ENV PORT=3000 NODE_ENV=production
ENV AWS_LWA_ENABLE_COMPRESSION=true
WORKDIR /app

# copy static files and images from build
COPY --from=builder /app/public ./public
COPY --from=builder /app/package.json ./package.json
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/run.sh ./run.sh
RUN ln -s /tmp/cache ./.next/cache

# configure the run command to start the server
RUN ["chmod", "+x", "./run.sh"]
CMD exec ./run.sh