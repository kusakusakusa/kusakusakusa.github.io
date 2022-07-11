---
title:  "AWS Lamba and API Gateway Integration With Terraform To Collect Emails"
date:   2020-03-07 09:00:00
permalink: aws-lamba-and-api-gateway-integration-with-terraform-to-collect-emails
---

This is a documentation on creating a service that collects emails. It runs on serverless technology utilizing AWS lambda and API Gateway. It is also made easy to deploy to the cloud with infrastructure as code via Terraform in the form of a plug-and-play methodology.

## Motivation

Often, I have to make static websites that are not exactly completely static because it requires a backend to collect the emails. While 3rd party services like mailchimp and sendgrid has their own SDKs to support easy integration for email collection, we have to be worried about hitting the limit in their packages and plans. This translate to stress for developers as we have to find a solution on it quickly and properly. If this happens on a weekend or a Friday, somehow this is always the case as more people are surfing the net then, the intensity is amplified.

For a new website, it is very hard to gauge the traffic and thus the plan required for the 3rd party service. this poses difficulties when budgeting for the project. Under utilizing the service also translate to unnecessary cost. The best kind of plan for such website is a pay as you go model, in my opinion, and that can be achieved by integrating with cloud providers like AWS.

## Technology Stack

### AWS Lambda

Enter AWS lambda where you only pay for what you use. You do not need to fork out money at the start of your project. Instead, you will just pay for how much you use, hence relieving you of the worry of wasting money on resources you are not using. In fact, this is only an issue if you are hitting 1 million potential users signing up with their email every month. The reason is because [AWS lambda has 1 million free request every month](https://aws.amazon.com/lambda/pricing/) before they start charging. This is highly unlikely for a new website, which means you now have a backend for your static website for free.

### API Gateway

For the Serverless fuction, that is AWS Lambda, to connect to the Internet via an API, we [need the API Gateway](https://aws.amazon.com/api-gateway/faqs/). This exposes the serverless function to be accessible by the World Wide Web with a HTTPS endpoint. It runs on the encrypted transport security layer protocol to uphold security by default. This allows your websites to use the serverless function via API calls.

### Terraform

To set up the infrastructures, the usual way is to navigate the AWS management console, deploy the required AWS resoures and link them. This can be a challenge if you are not familiar with the required configurations. Not only will this translate to loss of precious time to debug these issues, which otherwise developers could have spent it with your loved ones and challenge the meme below, but it will also lead to frustration.

While frustration is a part and parcel of life as a programmer, we can also avoid them with our knowledge of code. Here is where [Terraform](https://www.terraform.io/) enters the fray. It is an [Infrastructure As Code](https://en.wikipedia.org/wiki/Infrastructure_as_code) where you write the configurations of the infrastructure once and you can deploy it multiple time without having to go through the whole forest of the AWS console each time. This means you do not need to remember every single step and do not need to deal with surprise bugs because you forgot one of them, or worse, had a spelling error.

INSERT TWEET https://twitter.com/artemis_134/status/1010580495669891072?ref_src=twsrc%5Etfw

Since the blueprint infrastructure is in code, this means we can leverage version control features with git, and work together to improve the code base along the way without fear of not being able to rollback to the previous successful configuration.

## Terraform Files

I will start off with the terraform files required to setup the infrastructure to deploy the code. Let’s start off with the place to store our emails.

### The database – AWS DynamoDB

I will store the emails collected in AWS’s own noSQL database [DynamoDB](https://aws.amazon.com/dynamodb/faqs/). This is a fast, simply structured and schemaless storage which fits my use case very aptly.

It allows fast and simultneous writes at high speed, so there is no fear of race conditions from spike in the volume of signups during a PR event promoting the product and getting people to leave their emails at the website.

Since it is schemaless, we can easily add new details of the users that you would like to collect on top of their emails along the way without having to migrate and fiddle with the structure of the database. With proper metaprogramming, you do not need to touch the backend code as well, leaving only the frontend to work on adding the new text fields for data collection.

For the sake of argument, we can also use the [traditional relational database management system (RDBMS)](https://en.wikipedia.org/wiki/Relational_database) for this project. It is written in SQL, which is a langauge most, if not all, developers who every touched a database would have known. There is no need to use fancy noSQL for this simple project. In addition, the chances of leveraging the scaling advantage of noSQL over SQL databases are low, because you will need alot of traffic for that to become a worry. For a new website, that is highly unlikely to happen.

However, highly influenced by the cost, I am still sticking with DynamoDB in this case. To setup an AWS RDS to host a managed relational database, the cheapest MySQL database already goes for around 20 USD a month, as compared to the pay as you go model the DynamoDB employs. On top of that, it has a generous amount of free usage and storage under its [free tier](https://aws.amazon.com/dynamodb/pricing/provisioned/#DynamoDB_free_tier). This free tier does not last for the first 12 months after your signup but forever, unlike the RDS counterpart. We probably will NOT incur any cost using DynamoDB unless your marketing is brilliant for your new website.

```terraform
resource "aws_dynamodb_table" "main" {
  name = "${var.project_name}-dynamodb_table"
  billing_mode = "PROVISIONED"
  read_capacity = var.dynamodb-read_capacity
  write_capacity = var.dynamodb-write_capacity
  hash_key = "email"

  attribute {
    name = "email"
    type = "S"
  }
}
```

Provisioning the database is the simplest. I am using Terraform variables to substitute values to set the number of [reading and writing units](https://docs.amazonaws.cn/en_us/amazondynamodb/latest/developerguide/HowItWorks.ReadWriteCapacityMode.html) required, as well as the table name for robustness sake.

I have set the billing mode to “provisioned” for simplicity sake. Afterall I am not expecting any insane burst of traffic for a site that is not popular. Even if it does, maybe due to some incrediable promotion at some hugely popular event, I do not expect the load to require me to scale the reading and writing capacities of the database. It is going to be a quick write of a few bytes.

On top of that, provisioned capacity means less configurations needed for the permissions to autoscale of the capacities of the database. It can take some time to configure that, and since that is outside the topic of the article, I will stick to “provisioned” billing mode.

The hash_key, or “partition key” in other definitions, is analogous to the primary key in a SQL database table. It requires specific details under the attribute property. You can specify the [range_key, or “sort key”](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.CoreComponents.html) here if you require, and remember to add attribute to describe it as well.

Other attributes that are neither the partition key nor the sort key need not have a attribute property in this file. You can simply just write it in the database and it will register. Afterall, this is a schemaless database.

On top of that, it is a fully managed database, so it comes with all the goodies like backup and version maintenance to spare developers from all these chores.

### The backend – AWS Lambda

Next is the lambda function. It is written in Javascript using Nodejs. The file below is the configuration file to set the infrastructure required. Let’s dive into it.

```terraform
resource "aws_lambda_function" "main" {
  filename = var.zipfile_name
  function_name = "${var.project_name}"
  role = aws_iam_role.main.arn
  handler = "index.handler"

  source_code_hash = "${filebase64sha256("${var.zipfile_name}")}"

  runtime = "nodejs12.x"
}

resource "aws_iam_role" "main" {
  name = "${var.project_name}-iam_lambda"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Effect": "Allow",
      "Sid": ""
    }
  ]
}
EOF
}

resource "aws_iam_policy" "main" {
  name = "main"
  path = "/"
  description = "IAM policy for lambda to write to dynamodb table and logging"

  policy = templatefile("${path.module}/lambda_policy.tmpl", { dynamodb_arn = aws_dynamodb_table.main.arn })
}

resource "aws_iam_role_policy_attachment" "main" {
  role = "${aws_iam_role.main.name}"
  policy_arn = "${aws_iam_policy.main.arn}"
}

resource "aws_lambda_permission" "main" {
  statement_id = "AllowExecutionFromAPIGateway"
  action = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal = "apigateway.amazonaws.com"

  source_arn = "${aws_api_gateway_rest_api.main.execution_arn}/*/*/*"
}
```

Uploading of the backend code will be using the base64 hash of the zipfile of the code. The code will need to be first compressed and zipped before taking this action. We will see how we can automate this process later.

This lambda function will need the permissions to write to the dynamoDB table. This is done using


- `aws_iam_role` to establish trust between the 2 AWS services
- `aws_iam_policy` to give permission for the `lambda` function access the database resource and perform the `PutItem` action. Details of the policy is interpolated via a template file, which we will go through later
- `aws_iam_role_policy_attachment` to bind the aws_iam_role to the `aws_iam_policy` on the `lambda` function
- `aws_lambda_permission` to allow `API Gateway` to be able to integrate the lambda function and invoke it

The template file for the aws_iam_policy is shown below. It lists the actions that the lambda function is permitted to perform on the specified dynamodb table. It also contains the permissions for lambda function to push the logs to AWS Cloudwatch. By the way, these logging permissions are the default permissions for a lambda function, and this template adds on the DynamoDB permissions to them. Note the dynamodb_arn variable that is interpolated, which jusitifies the use of the template file instead of hardcoding the whole policy in the main terraform file for robustness sake.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "dynamodb:PutItem",
      "Resource": "${dynamodb_arn}",
      "Effect": "Allow"
    },
    {
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*",
      "Effect": "Allow"
    }
  ]
}
```

## The API layer – AWS API Gateway

The API Gateway is required to expose the lambda function to be consumed by servers and websites via a URL endpoint. The endpoint will be served over the HTTPS, which requires some extra configurations as documented below.

```terraform
resource "aws_api_gateway_rest_api" "main" {
  name = var.project_name
}

resource "aws_api_gateway_resource" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  parent_id = aws_api_gateway_rest_api.main.root_resource_id
  path_part = "email"
}

resource "aws_api_gateway_integration" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.main.id
  http_method = aws_api_gateway_method.main.http_method
  integration_http_method = aws_api_gateway_method.main.http_method
  type = "AWS_PROXY"
  uri = aws_lambda_function.main.invoke_arn
}

resource "aws_api_gateway_integration_response" "main" {
  depends_on = [aws_api_gateway_integration.main]

  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.main.id
  http_method = aws_api_gateway_method.main.http_method
  status_code = aws_api_gateway_method_response.main.status_code
}

resource "aws_api_gateway_method" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.main.id
  http_method = "POST"
  authorization = "NONE"
}

resource "aws_api_gateway_deployment" "main" {
  depends_on = [
    "aws_api_gateway_integration_response.main",
    "aws_api_gateway_method_response.main",
  ]
  rest_api_id = aws_api_gateway_rest_api.main.id
}

resource "aws_api_gateway_method_settings" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  stage_name = aws_api_gateway_stage.main.stage_name
  
  # settings not working when specifying the single method
  # refer to: https://github.com/hashicorp/terraform/issues/15119
  method_path = "*/*"

  settings {
    throttling_rate_limit = 5
    throttling_burst_limit = 10
  }
}

resource "aws_api_gateway_stage" "main" {
  stage_name = var.stage
  rest_api_id = aws_api_gateway_rest_api.main.id
  deployment_id = aws_api_gateway_deployment.main.id
}

resource "aws_api_gateway_method_response" "main" {
  rest_api_id = aws_api_gateway_rest_api.main.id
  resource_id = aws_api_gateway_resource.main.id
  http_method = aws_api_gateway_method.main.http_method
  status_code = "200"
}

output "endpoint" {
  value = "${aws_api_gateway_stage.main.invoke_url}${aws_api_gateway_resource.main.path}"
}
```

So let’s break it down.

The aws_api_gateway_rest_api represents the project in its entirety.

The aws_api_gateway_resource refers to each api route of this project, and there is only 1 in this case.

I have setup only 1 stage environment of aws_api_gateway_stage for this project using a Terraform variable. You can setup a different stages to differentiate the staging and production environments.

The aws_api_gateway_stage is associated to a aws_api_gateway_method_settings that sets the throttling rate of the API to prevent spams and overloading. For the method_path property, the wildcard route is used to apply to all routes instead of the only API route that was created. It is trivial in this case, but the explanation for picking this “easy” route is simply due to a bug. It I were to specify the exact route, which is in the [form of {resource_path}/{http_method}](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/api_gateway_method_settings.html), the settings on the throttling rate will not propagate. It was documented [here on github](https://github.com/hashicorp/terraform/issues/15119) but was not properly resolved. Leaving it here for now.

The aws_api_gateway_deployment configures the deployment of the API. Note the depends_on attribute that was assigned. This explicit dependency is critical to ensure the deployment is called into effect after all the necessary resources have been provisioned.

The aws_api_gateway_integration configuration sets the integration to lambda proxy using POST HTTP method without any authorization, as specified by the aws_api_gateway_method configuration. Lambda proxy allows us to handle the request from the server like how we would in a typical web application backend framework. The full request object is passed to lambda function and the API Gateway plays no part in mapping any of the request parameters. The API Gateway mapping has great potential to integrate interfaces properly, but for our use case, it is not necessary. I find [this article](https://www.alexdebrie.com/posts/api-gateway-elements/#roadmap-the-three-basic-parts) doing a great job in explaining the API Gateway features with easy to consume information and summary, like a gameshark guide book written by the half-blood prince. Do take a look to understand AWS API Gateway better.

The aws_api_gateway_integration_response is responsible for handling the response from the lambda function. This is where we can make changes to the headers returned from the lambda function using the response_parameters property, which is not used in this case. This is also the place to map and transform the response data from the backend to fit the desired data structure using the response_templates property.

The aws_api_gateway_method_response is where we can filter what response headers and data from aws_api_gateway_integration_response to pass on to the caller.

The transform and mapping of the headers and data from the backend (ie the lambda function) in aws_api_gateway_integration_response and the filter of headers and data before passing to the front end in aws_api_gateway_method_response is not needed in this sample application. It is just good knowledge to have. There are 2 reasons why we do not need them here.

First, in a bit, we will go through the front end that will make an API call that is a [simple request](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Simple_requests). A simple request does not require a preflight request, which is a API call made by browsers prior to the actual API call, as they are deemed safe since they are using [standard CORS-safelisted request headers](https://bugs.webkit.org/show_bug.cgi?id=165178). In the event that one does need a preflight request because one is not making a simple request, we will need to set up another API route that will transform the headers returned from the backend and allow the relevant headers to be passed on to the front end for this preflight request. This will allow the frontend website to overcome the [CORS policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) enabled by default in modern browsers. This will mean we need to configure a new set of aws_api_gateway_rest_api, aws_api_gateway_integration, aws_api_gateway_method, aws_api_gateway_integration_response, aws_api_gateway_method_response just for this preflight request. Things can get complicated here, so I will leave out of this article. If you still to implement CORS, [this gist](https://gist.github.com/keeth/6bf8b67c82f9a085e03ecbb289a859d6) is a good reference.

Second, we are using lambda proxy integration, so the full response from the lambda will be passed to the front end and mapped automatically, provided the response from the lambda code is properly formatted. Refer to [this documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-settings-method-response.html) for more details on it.

At last, the output resource will print the value of the enpoint of the api for us to integrate in our frontend.

## The Admin Stuff

This file contains the details that we will need to setup terraform and the variables we are using. The provider‘s region attribute here is hardcoded, which should ideally not be the case. I have yet to figure out how to make this dynamic and robust. The name with the todo- prefix should be changed to fit the project.

We are using an S3 bucket as the[ Terraform backend](https://www.terraform.io/language/settings/backends/configuration) to hold the state of the infrastructure provisioned by Terraform. ​Creation of the bucket will be automated via a script that we will go through during the section on deployment.

```terraform
provider "aws" {
  version = "~> 2.24"
  region = "eu-west-1"
}

terraform {
  required_version = "~> 0.12.0"
  backend "s3" {
    bucket = "todo-project-tfstate"
    key = "terraform.tfstate"
    region = "eu-west-1"
  }
}

variable "project_name" {
  type = string
  default = "todo-project"
}

variable "region" {
  type = string
  default = "eu-west-1"
}

variable "stage" {
  type = string
  default = "todo-stage"
}

variable "zipfile_name" {
  type = string
  default = "todo-project.zip"
}

variable "dynamodb-read_capacity" {
  type = number
  default = 1
}

variable "dynamodb-write_capacity" {
  type = number
  default = 1
}
```

## The Application

Here is the application code in written in nodejs. It is a simple write to the dynamodb with basic error handling. It takes in only 1 parameter, that is the email. This code can definitely be improved by allowing more parameters to be written to the database in a dynamic way, so that the same code base can be used for a site that collects the first and last name of the user, as well as another site that collects the date of birth of the user. I will leave that as a future personal quest.

```js
// Load the AWS SDK for Node.js
const AWS = require('aws-sdk');

// Set the region 
AWS.config.update({region: 'eu-west-1'});

// Create the DynamoDB service object
const ddb = new AWS.DynamoDB({apiVersion: '2012-08-10'});

exports.handler = async (event) => {
  console.log(JSON.stringify(event, null, 2));
  const params = {
    TableName: 'todo-project-dynamodb_table',
    Item: {
      'email' : {S: JSON.parse(event.body).email}
    }
  };

  // Call DynamoDB to add the item to the table
  ddb.putItem(params, function(err, data) {
    if (err) {
      console.log("Error", err);
    } else {
      console.log("Success", data);
    }
  });
  
  try {
    const result = await ddb.putItem(params).promise();
    console.log("Result", result);
    const response = {
      statusCode: 204,
      headers: {
        "Access-Control-Allow-Origin" : "*",
      },
    };
    return response;
  } catch(err) {
    console.log(err);
    const response = {
      statusCode: 500,
      headers: {
        "Access-Control-Allow-Origin" : "*",
      },
      body: JSON.stringify({ error: err.message }),
    };
    return response;
  }
};
```

A thing to note here is the need to return the Access-Control-Allow-Origin header in the response. The response also has to follow [a particular but straightforward and common format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format) in order for lambda proxy integration with API Gateway. This will map the response properly to the API Gateway method response and be returned to the frontend websites to overcome the CORS policy implemented by modern browsers.

## Deployment

I will be using 3 ruby scripts for deployment related tasks, namely init.rb, apply.rb and destroy.rb, and a helper service object, get_aws_profile.rb for the deployment process.

Let’s take a look at them.

### get_aws_profile.rb

```ruby
# get_aws_profile.rb

class GetAwsProfile
  def self.call
    aws_profile = "todo-aws_profile"

    begin
      aws_access_key_id = `aws --profile #{aws_profile} configure get aws_access_key_id`.chomp
      abort('') if aws_access_key_id.empty?

      aws_secret_access_key = `aws --profile #{aws_profile} configure get aws_secret_access_key`.chomp
      abort('') if aws_secret_access_key.empty?
    rescue Errno::ENOENT => e
      abort("Make sure you have aws cli installed. Refer to https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html for more information.")
    end

    p "AWS_ACCESS_KEY_ID = #{aws_access_key_id}"
    p "AWS_SECRET_ACCESS_KEY = #{aws_secret_access_key}"

    [aws_profile, aws_access_key_id, aws_secret_access_key]
  end
end
```

This is a helper method that will get the aws_access_key_id and the aws_secret_access_key for usage in the scripts. Note that it uses the [aws cli command](https://aws.amazon.com/cli/) to attain the keys. Hence, it has to be installed on your local machine prior to running. It also assumes you are using [named profile](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html) to hold your credentials.

I don’t really like this setup since it requires these prerequisites. But well that can be solved again in the future.

### init.rb
The first script to run is init.rb.

The init.rb will create the S3 bucket to be used as the terraform backend. Line 20 checks for the presence of this bucket and throws an exception if the bucket does not exist. The rescue block, if triggered, will create the non-existent bucket.

The initialization process on [terraform is run via its docker image](https://vic-l.github.io/terraform-with-docker).

```ruby
require 'bundler/inline'

gemfile do
  source 'https://rubygems.org'
  gem 'pry'
  gem 'aws-sdk-s3', '~> 1'
end

require './get_aws_profile.rb'

aws_profile, aws_access_key_id, aws_secret_access_key = GetAwsProfile.call

s3_client = Aws::S3::Client.new(
  access_key_id: aws_access_key_id,
  secret_access_key: aws_secret_access_key,
  region: 'eu-west-1'
)

begin
  s3_client.head_bucket({
    bucket: 'todo-project-tfstate',
    use_accelerate_endpoint: false
  })
rescue StandardError
  s3_client.create_bucket(
    bucket: 'todo-project-tfstate',
    create_bucket_configuration: {
      location_constraint: 'eu-west-1'
    }
  )
end

response = `docker run \
  --rm \
  --env AWS_ACCESS_KEY_ID=#{aws_access_key_id} \
  --env AWS_SECRET_ACCESS_KEY=#{aws_secret_access_key} \
  -v #{Dir.pwd}:/workspace \
  -w /workspace \
  -it \
  hashicorp/terraform:0.12.12 \
  init`

puts response
```

### apply.rb

Once initialized, the next script to run is apply.rb.

Prior to applying the Terraform instructure, the backend code is packaged into a zip file. After application, the zip file is deleted for housekeeping.

```ruby
require 'bundler/inline'

gemfile do
  source 'https://rubygems.org'
  gem 'pry'
  gem 'rubyzip', '>= 1.0.0'
end

require './get_aws_profile.rb'
require 'zip'

aws_profile, aws_access_key_id, aws_secret_access_key = GetAwsProfile.call

folder = Dir.pwd
input_filenames = ['index.js']
zipfile_name = File.join(Dir.pwd, 'todo-project.zip')

File.delete(zipfile_name) if File.exist?(zipfile_name)

Zip::File.open(zipfile_name, Zip::File::CREATE) do |zipfile|
  input_filenames.each do |filename|
    zipfile.add(filename, File.join(folder, filename))
  end
end

response = `docker run \
  --rm \
  --env AWS_ACCESS_KEY_ID=#{aws_access_key_id} \
  --env AWS_SECRET_ACCESS_KEY=#{aws_secret_access_key} \
  -v #{Dir.pwd}:/workspace \
  -w /workspace \
  -it \
  hashicorp/terraform:0.12.12 \
  apply -auto-approve`

puts response

File.delete(zipfile_name) if File.exist?(zipfile_name)
```

With this, the api is now deployed and can be called from any website. We will go through a sample front end integration in a bit.

### destroy.rb

Once you are done with the project or are in the process of debugging, the destroy script will remove all the resources deployed. It will also remove the S3 backend that was created outside of Terraform.

```ruby
require 'bundler/inline'

gemfile do
  source 'https://rubygems.org'
  gem 'pry'
  gem 'aws-sdk-s3', '~> 1'
end

require './get_aws_profile.rb'

aws_profile, aws_access_key_id, aws_secret_access_key = GetAwsProfile.call

response = `docker run \
  --rm \
  --env AWS_ACCESS_KEY_ID=#{aws_access_key_id} \
  --env AWS_SECRET_ACCESS_KEY=#{aws_secret_access_key} \
  -v #{Dir.pwd}:/workspace \
  -w /workspace \
  -it \
  hashicorp/terraform:0.12.12 \
  destroy -auto-approve`

puts response

s3_client = Aws::S3::Client.new(
  access_key_id: aws_access_key_id,
  secret_access_key: aws_secret_access_key,
  region: 'eu-west-1'
)

begin
  s3_client.head_bucket({
    bucket: 'todo-project-tfstate',
    use_accelerate_endpoint: false
  })

  s3_client.delete_object({
    bucket:  'todo-project-tfstate',
    key: 'terraform.tfstate', 
  })
  s3_client.delete_bucket(bucket: 'todo-project-tfstate')
rescue StandardError
  puts "todo-project-tfstate S3 bucket already destroyed."
end
```

## Sample Frontend Integration

```html
<!DOCTYPE html>
<html>
<head>
  <script
  src="https://code.jquery.com/jquery-3.4.1.min.js"
  integrity="sha256-CSXorXvZcTkaix6Yvo6HppcZGetbYMGWSFlBw8HfCJo="
  crossorigin="anonymous"></script>
</head>
<body>

  <h2>HTML Forms</h2>

  <form id="form">
    <label for="email">First name:</label><br>
    <input type="text" id="email" name="email" value="test@test.com"><br>
    <input type="submit" value="Submit">
  </form>

  <script type="text/javascript">
    $( "#form" ).submit(function(event) {
      event.preventDefault();

      $.ajax({
        type: "POST",
        url: "https://todo-endpoint.execute-api.eu-west-1.amazonaws.com/todo-stage/email",
        data: JSON.stringify({
          email: $('#email').val()
        }),
        success: function(data, textStatus, jqXHR) {
          debugger
        },
        error: function(jqXHR, textStatus, errorThrown) {
          debugger
        }
      });
    });
  </script>

</body>
</html>
```

Above is a simple html web page that will has the email prefilled for demonstration purpose. The form will submit via jquery.ajax() using default settings so as not to trigger the need for preflight request.
You will see that the email will be added to the DynamoDB table, and the logs of the lambda funciton will be recorded in AWS Cloudwatch.

## Conclusion

This exercise helped me understand how lambda is integrated with API Gateway, as well as the immense potential as a robust middleware the latter can be. In addition, I got to understand preflight request and CORS better, as well as the jquery.ajax() function.

The project is saved in [this repository](https://github.com/Vic-L/email-collector) for future reference.

