Passing custom data from AWS Lambda Authorizer
==============================================

![captionless image](https://miro.medium.com/v2/resize:fit:1200/format:webp/1*FiUFb0hQX1JkeLNe2lZ1fw.png)

A lambda authorizer is used to validate incoming JWT Tokens in API Gateway.

However, sometime we would want to pass additional data after a successful validation so that the backend services can use them. This can be achieved by using the “**context**” field while we are building the auth response.

```json
{
	"principalId": "John",
	"policyDocument": {
		"Version": "2012-10-17",
		"Statement": [			{
				"Action": "execute-api:Invoke",
				"Effect": "Allow",
				"Resource": "arn:aws:execute-api:us-east-1:123456781345:0n1anivwva/test/GET/employees"
		]
	},
	"context": {
		"org": "UNSC",
		"role": "Spartan2"
	}
} 
```

Note that the context object only takes “key”:”value” pairs which means you cannot pass complex objects. Here we are passing the “org” and the “role” key values.

Now, how do we actually consume it ? This data is only available within the context of the lambda and not outside. This means it is not automatically passed to your backend. To use this data, the simplest way is to pass it using **integration headers** !

Simply add two headers to the headers section of **Integration Request** under the resource in api gateway,

![captionless image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*JEeJFHvZ2tj-Qqt4bcNusA.png)

The context from authorizer is available to be mapped using context.authorizer.{your key}

And there you go, you will now receive these headers as part of the request in your backend :)

Thanks for reading !
