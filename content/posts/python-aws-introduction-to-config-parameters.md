---
author: "Vittorio Mattei"
date: 2022-07-31
linktitle: Python configuration parameters with AWS
title: Python configuration parameters with AWS
type:
- post 
- posts
weight: 1
categories:
- Programming
tags:
- programming
- python
- beginner
- AWS
series:
- Python programming
description: Load configuration parameters in Python with boto3, AWS and SSM Parameter Store
draft: True
---

Loading configuration parameters in your program is an operation that every developer will probably implement at some point while writing code.

While the code can be as simple as `CONSTANT = value`, this is probably not the best idea of how to implement a configuration for your software.

In this article I want to analyze some concept that you can use for your implementation. The implementation used here will show how to apply those concept when using Python, AWS and boto3 as your tech stack.

# 1. Hard coded value
This is the easiest way to implement a configuration parameter when you're starting your project. You don't know if this parameter will change or if it depends on other features. You just want to use a constant value in multiple places of your code.

The implementation is as simple as this:

```
# constants.py

DB_USERNAME = "admin"
```

You then simply import constants.py and use the variable in your code.

There are multiple drawbacks to this method:
- Changing a value requires editing the source code and re-launching the program.
- If your repository is public, everyone will see the value. This is problematic when dealing with secrets (like passwords).
- If multiple programs use the same value, you need to edit all programs when changing your parameter.

# 2. Configuration file
This is especially effective when dealing with compiled languages, where changing a value will require a re-compilation of your code.

When loading parameters from a file, you only need tu re-run the program and the new value will be loaded in memory.

Let's improve the example written before:

```
# constants.json

{
    "DB_USERNAME": "admin"
}
```

We can use any file format we want. JSON, YAML and TOML are some common formats you can use to define your configuration file.

We then load the file into memory:

```
#constants.py
import json

file_name = "constants.json"
with open(file_name, "r") as f:
    json_config = json.load(f)

DB_USERNAME = json_config.get("DB_USERNAME") 
```

We don't need to edit the code when we want to change the value of our parameter, we just edit the configuration file.

Advantages:
- Can change value without editing code
- Multiple file formats supported and we can use default values when a parameter is not defined in the configuration file
- You can push in your repository an example file and other developer can define their own values without exposing secrets
  
Disadvantages:
- You still need to define all your common parameters for each project
- Reloading parameters still requires restarting the program
- Changing parameters requires editing the file on the host machine
- Working with Docker and serverless can be difficult, as it might not be that easy to edit the file locally, and you might need to push it in your repository

## Improvement: let's move the file on the cloud

There is no reason that we need to maintain  the file locally. In the era of cloud storage, it's just so easy to load the file on any cloud storage provider and then download. To do this we need a library to interact with the cloud provider: in this article we will use AWS as cloud provider and boto3 as the library.

Boto3 allows Python software to interact with AWS resources, mainly to dynamically operate on them.

***Note: for cloud infrastructure creation, it is better to use solutions like CDK or Terraform, as they allow to easily rollback and delete the defined infrastructure\*\**

We simply load the configuration on AWS S3. The path will be configurations/db_config.json. Remember that S3 resources follow the pattern BUCKET_NAME/KEY.

We now need to add code to read the configuration from S3.

```
# constants.py

import boto3
import json


bucket = "configurations"
key = "db_config.json"

s3_read_response = boto3.client("s3").get_object(Bucket=bucket, Key=key)
json_config = json.loads(s3_read_response['Body'].read())

DB_USERNAME = json_config.get("DB_USERNAME") 
```

Remember that in a real case you also want to handle exception for when you don't have access to a file or you can't find a file. You might also get a corrupt file or the file is not a JSON, so keep in mind these aspects when implementing this code.

You can find more information about boto3 and s3 in here.

Advantages:
- The configuration can now be edited only once and loaded on s3, possibly with CI/CD pipelines.
- You can create the file on a different project, edit it locally and push it on the cloud instead of operating on the host machine.

Disadvantages:
- Secrets are still exposed
- Reloading values still requires a restart

# Environment variables
There is a simple solution for when you don't want to expose a secret in a configuration file or hard code it in your program. We can use **environment variables** for this.

Environment variables can easily be defined and kept secret to the outside world (unless someone gains access to the machine and you might have even bigger problems.) You can define them on your local machine when testing your software, in Dockerfile and docker-compose for easy deployment, in your CI/CD pipeline or in cloud providers container solutions (see here for an example with AWS ECS).

Regarding the previous example, there are two methods that we could use to hide DB_USERNAME from a public repository. We assume that we created a different repository that holds a Python script that creates "db_config.json" with the desired values and automatically pushes it to S3 using boto3 when making changes to it. The config file itself is only found on S3, where project contributors won't have direct access.

1. We can inject the environment variable into the main program and retrieve it without using db_config.json
2. We can inject it into the *config creater* program, that will then push the configuration file with the variable to S3.

The code to retrieve the variable is the same, so it is only a matter of use-case and preference for which one you choose. Nothing stops us from mixing them for different constants. We could save common variables in a .json and use environment variables for program specific values.

Let's see the second case:
```
# constants.py

import os
DB_USERNAME = os.environ.get("DB_USERNAME", "default")
```

This is pretty similar to getting values from a json: we can specify the **key** and a **default** value for when the key is not found.

Now, we don't need to write constants in any repository and we can allow external contributors to check out the source code of the program we're developing.

We can also leverage environment variables when developing deployment infrastructure with tools like Terraform. For example, we can deploy a database on a machine with a dynamic public IP and automatically retrieve that IP and pass it as an environment variable to the main program when deploying it on the cloud.

There are still a couple of issues with this approach:
- We can't change the parameters without a restart (or... a crash!).
- Somewhere, these parameters and secrets are still written. It's easy to save them as environment variables on our machine or create another file that is not committed to git, but what if there are multiple developers contributing to the same project and deployment? Passing those value around on Slack is not optimal nor advised.

# Store parameters on the cloud: SSM Parameter Store

Let's solve the second issue: where can we safely store these parameters so that they are available to any team member?

We can use cloud services that offer storing parameters as the main function. **AWS** offers **SSM Parameter Store**. It is possible to save parameters (secrets or not) there that can easily be retrieved by applications and users that have permission.

You can refer to AWS documentation for how to save parameters there. Let's see how we can then use them in our Python application.

```
# constants.py


try:
    client = boto3.client("ssm")
    DB_USERNAME = client.get_parameter(Name="common/db/username")
    DB_PASSWORD = client.get_parameter(Name="common/db/password", WithDecryption=True)
except Exception:
    # handle errors
```

In this case, you cannot give a default value to the parameters, so you should handle exceptions accordingly. Possible causes could be due to missing permissions, invalid parameter names, error in AWS connection.

You could either define a default value in the except block, retry or raise exceptions depending on your use case.

As you can see, you can specify a *WithDecryption* parameter that allows you to decode secret values stored on AWS, like passwords and keys.

There is only one issue left after this iteration: we still need to restart the program if we need to update the value.



# Caching: SSM Cache
We can solve this last issue by using a cache.

With a cache, we can save a value in memory for easy retrieval. Usually caches have a size, where frequent and recent elements are saved to cut access time to peripherals or external services.

In this case, we use a cache to save a value in memory for a certain amount of time.

This allows us to fetch the parameter value from AWS, keep it in memory for a set time and then refresh it automatically.

While it can be an interesting exercise to implement your own caching module for parameters, you can use the module [ssm-cache](https://pypi.org/project/ssm-cache/) for a fast implementation.

The above code would become this:

```
# constants.py

from ssm_cache import SSMParameterGroup
DB = SSMParameterGroup(max_age=300)  # Time in seconds
DB_USERNAME = DB.parameter("common/db/username")
DB_PASSWORD = DB.parameter("common/db/password")

value_1 = DB_USERNAME.value
value_2 = DB_USERNAME.value
```

After the set time, your parameter will be refreshed automatically, so you don't need to restart the program

# Additional ideas

There are still some issues left after all this.

For instance, what if you changed a parameter and your program uses the cached value before this is updated? You would get an exception.

We can still mitigate this issue by carefully handling exception. When an error arises due to an expired parameter, we can re-fetch the parameter and retry the operation or alert the calling function/user/service to retry.

We could also always fetch the parameter from SSM instead of caching it, but be careful as this won't scale well!

An excelent way to keep your parameters updated would be to automatically alert all the services that use one parameter that its value changed and they need to re-fetch it.  
You could do this with the event watcher of your cloud provider (if it allows checking for updated parameters), or you could create an additional service that allows you to update a parameter and alert all services (be careful, as an attacker gaining access to this service would mean losing all your secrets!).

Note that each service should implement an endpoint to receive this update event. If a service is down at that moment, it wouldn't be an issue, as they would fetch the new parameter anyway at the next restart.
