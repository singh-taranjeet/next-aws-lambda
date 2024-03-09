Here I will be documenting how to deploy your nextJs application to AWS lambda. We will be building cloudformation stack using AWS SAM.
All code can be found in GitHub repository and it will be maintained and updated as per time: [github-repository](https://github.com/singh-taranjeet/next-aws-lambda)

So follow the steps as following:

## Initialise the NextJS application
Create directory named `app` where nextJS application will be installed.

Run following command to install next js application

```
npx create-next-app@latest
```

![next app installation](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ndgclevr1xka8g5eurck.png)


## Initialise the AWS SAM

Now we will be using code as infrastructure to build our cloudformation stack. So initialise the sam application using the following the guide as per your Operating system [aws-docs](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)


## Configure NextJS application

We need to export the application as standalone to deploy on lambda as a node server.

So in the next.config.js file add following code to deploy app on lambda:

```
/** @type {import('next').NextConfig} */
const nextConfig = {
  output: "standalone",
};

export default nextConfig;

```

Now we need to create a script to start our node js server.
So create a new file named `run.sh` in root of `app` directory and add following commands to run the server.

```
#!/bin/bash -x

[ ! -d '/tmp/cache' ] && mkdir -p /tmp/cache

exec node server.js
```

Now we need to configure docker image to which will be used by our lambda function.
So create a `Dockerfile` in root of `app` directory. Add following code to the `Dockerfile`.

```
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
```
In the above docker file we are using `aws-lambda-adapter`. It is a tool to run web applications on AWS Lambda.

AWS Lambda Web Adapter allows developers to build web apps (http api) with familiar frameworks (e.g. Express.js, Next.js, Flask, SpringBoot, ASP.NET and Laravel, anything speaks HTTP 1.1/1.0) and run it on AWS Lambda. The same docker image can run on AWS Lambda, Amazon EC2, AWS Fargate, and local computers.

# Configure SAM template
Now create a new file in the root of the application and outside the app directory.
`template.yaml`
Add following code to the `template.yaml` to configure AWS Lambda and AWS Api Gateway.

```
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  serverless app to deploy next app to aws lambda

Globals:
  Function:
    Timeout: 30

Resources:
  NextAppStackFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: app/
      PackageType: Image
      Architectures:
        - x86_64
      Events:
        HttpEvent:
          Type: HttpApi
    Metadata:
      DockerTag: v1
      DockerContext: ./app
      Dockerfile: Dockerfile
Outputs:
  NextAppStackUrl:
    Description: "API Gateway endpoint URL for servering the next app"
    Value: !Sub "https://${ServerlessHttpApi}.execute-api.${AWS::Region}.${AWS::URLSuffix}/"

```

Make sure docker is running on your machine.

Now we need to build sam application using following command.

```
sam build
```


After building the sam application, we can deploy the application on cloudformation stack using the following command.

```
sam deploy --guided
```

![Sam deploy --guided](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/to8fzedlgpghk6gjwizx.png)

After successfully deploying the output will look like following with the hosted url.

![deploy output](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/yjyl3tilxfjxd7ojw0a4.png)

