![CI](https://github.com/aws-observability/aws-otel-lambda/workflows/CI/badge.svg)
# AWS Distro for OpenTelemetry Lambda Support (Metrics) 
The AWS Distro for OpenTelemetry(ADOT) [Lambda layer](https://docs.aws.amazon.com/lambda/latest/dg/configuration-layers.html) provides a simple user experience by attaching an additional layer to Lambda function. Users can onload and offload OpenTelemetry from their Lambda function with minimal code change.


## Sample
This sample will create AWS Distro for OpenTelemetry Lambda Extension and a sample application in region us-west-2 in your AWS account. By default the metrics are sent to AWS CloudWatch, you can see telemetry data generated by OpenTelemetry from AWS CloudWatch Console.
- Install [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) and [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html). For users have trouble in SAM CLI, see [documentation](../python-lambda/docs/misc/sam.md).
- Run `aws configure` to set aws credential([with administrator permissions](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install-mac.html#serverless-sam-cli-install-mac-iam-permissions)) and default region.
- Download a local copy of the [aws-otel-lambda repository from Github](https://github.com/aws-observability/aws-otel-lambda).
- Run command: `cd sample-apps/java-lambda && ./run.sh -r us-west-2`
- Open [Lambda console](https://us-west-2.console.aws.amazon.com/lambda/home?region=us-west-2#/functions) in us-west-2, find the new Lambda function `adot-java-sample-function-...`
the source code of Lambda function contains a metric emitter which forwards metrics to ADOT Lambda Extension
    <details><summary>source code</summary>

    ```
      @Override
      public String handleRequest(Void input, Context context) {
        long requestStartTime = System.currentTimeMillis();
        MetricEmitter metricEmitter = buildMetricEmitter();
        long latency = System.currentTimeMillis() - requestStartTime;
        metricEmitter.emitReturnTimeMetric(latency, "/lambda-sample-app", "200");
        String message = new String("200 OK");
        metricEmitter.shutDown();
        return message;
      }
    ```
    </details>

- Invoke Lambda function by clicking the API endpoint in API Gateway, Details
    <details>

    ![](./docs/images/sample1.png)

    </details>

- Open [CloudWatch console](https://us-west-2.console.aws.amazon.com/cloudwatch/home?region=us-west-2#metricsV2:graph=~();namespace=~'AWSOTel*2fAWSOTelSampleApp), click the latest metric, will see the `latency` metric sent by this sample application.
    <details>

    ![](./docs/images/sample2.png)

    </details>

- Open [CloudFormation console](https://us-west-2.console.aws.amazon.com/cloudformation/home?region=us-west-2#/stacks?filteringText=&filteringStatus=active&viewNested=true&hideStacks=false), clean the sample resources by **Delete** stack `adot-java-sample`.

***

## Getting started
To play ADOT in Lambda container image, the following steps are required: 1. Build the Lambda layer; 2. Include Lambda layer into your Lambda function container image.

### Build the Lambda layer
If you have went though Sample you already have ADOT Lambda layer, skip to the next step **Enable auto-instrumentation for your Lambda function**

1. Install [SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html) and [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html). For users have trouble in SAM CLI, see [documentation](docs/misc/sam.md).
2. Run `aws configure` to set aws credential([with administrator permissions](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install-mac.html#serverless-sam-cli-install-mac-iam-permissions)) and default region.
2. Download a local copy of the [aws-otel-lambda repository from Github](https://github.com/aws-observability/aws-otel-lambda).
3. `cd sample-apps/java-lambda && ./run.sh -b -l`

Tips:
- `run.sh -b -l` will compile and build ADOT layer in local under `sample-apps/java-lambda/aws_observability/build/`. If need both ADOT layer and Sample app, use `./run.sh` which will integrate the ADOT layer into Sample App's Dockerfile.
- Lambda layer is regionalized. To publish ADOT layer to other regions use `-r`, for example `./run.sh -r us-west-2`


### Include Lambda layer into your Lambda function container image

Now you have the layer built. To enable the AWS Distro for OpenTelemetry in your Lambda function, you need to [add layer into your application's Dockerfile](https://docs.aws.amazon.com/lambda/latest/dg/using-extensions.html#invocation-images-extensions).   
Example:
```
FROM gradle as build

WORKDIR /app
COPY ./build.gradle ./build.gradle
COPY ./src ./src
COPY ./settings.gradle ./settings.gradle

RUN gradle build
RUN tar -xvf build/distributions/spark-sample-app-1.0.tar
RUN gradle copyDependencies

FROM amazoncorretto:11 

WORKDIR /app
COPY --from=build /app/dependencies/*.jar ./
COPY --from=build /app/spark-sample-app-1.0 .

ENV HOME=/root
ENV OTEL_RESOURCE_ATTRIBUTES 'service.namespace=AWSOTel,service.name=AWSOTelSampleApp'
ENV OTEL_EXPORTER_OTLP_ENDPOINT 'localhost:55680'

ADD aws_observability/build/extensions /opt/extensions
ADD aws_observability/build/aoc /opt/aoc

ENTRYPOINT [ "/usr/bin/java", "-cp", "./*", "com.amazonaws.services.lambda.runtime.api.client.AWSLambda" ]
CMD ["com.amazon.sampleapp.App::handleRequest"]
```



***

## Configuration
The AWS Distro for OpenTelemetry Metrics Lambda layer contains the [AWS Distro for OpenTelemetry Collector](https://github.com/aws-observability/aws-otel-collector#overview). The configuration of Collector follows the OpenTelemetry standard.

By default, AWS Distro for OpenTelemetry in Lambda uses [config.yaml](https://github.com/aws-observability/aws-otel-lambda/blob/main/extensions/aoc-extension/config.yaml), which exports telemetry data to AWS X-Ray and AWS CloudWatch.

For debugging, you can turn on the logging exporter in the Collector by adding the environment variable `ADOT_DEBUG=true` in the Lambda function.

To customize the Collector config, there are two options:

* Bring custom config file (and ca/cert/key files) into the Lambda function by the Lambda layer, then set the Collector config by environment variable `ADOT_CONFIG=<your config file path>`
* Add the environment variable `ADOT_CONFIG_CONTENT=<Full content of your config file>`

For more information about AWS Distro for OpenTelemetry Collector configuration like adding ca/cert/key files, see the Github [README.md](../../extensions/sample-custom-config/README.md).

#### Environment variables

- `ADOT_DEBUG` - Set to `true` can turn on OpenTelemetry ConsoleExporter in SDK and logging exporter in Collector, to output more details in log for debug.
- `ADOT_CONFIG` - AWS Distro for OpenTelemetry Collector configuration file path.
- `ADOT_CONFIG_CONTENT` - AWS Distro for OpenTelemetry Collector configuration file content.
- `_ADOT_CI` - Reserved environment variable in ADOT CI.

    
***

## Security

See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.

## License

This project is licensed under the Apache-2.0 License.

