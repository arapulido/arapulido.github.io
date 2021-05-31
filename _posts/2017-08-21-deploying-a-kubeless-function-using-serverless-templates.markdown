---
layout:	post
title:	"Deploying a Kubeless function using Serverless templates"
date:	2017-08-21
---

![](/img/1*_BZYlfdv85zsJpkg9jjgug.png)[Serverless Framework](https://serverless.com/framework/) is a framework that helps building serverless applications and deploying them in a consistent manner to different clouds (Google, AWS, Azure, etc.).

At Bitnami we have been working alongside the Serverless team to include [Kubeless](http://kubeless.io/), the serverless framework for Kubernetes, in that list, helping people, who are already familiar with the Serverless Framework, to deploy their applications on-premise on top of Kubernetes.

In [a previous blog post](https://medium.com/bitnami-perspectives/a-serverless-plugin-for-kubeless-64cd0f7e4f62), we introduced the Kubeless plugin for Serverless and explained how we could deploy a function using the framework, which included having to create a serverless.yml file for your particular function.

From Serverless Framework version 1.20 onwards, building and deploying functions to Kubeless has become even easier, with the introduction of Kubeless templates. In this blog post, we will show you how to build and deploy a Python function using templates.

This posts assumes you have Kubeless installed in your Kubernetes cluster. You can follow the installation process for Kubeless in its [README.md file](https://github.com/kubeless/kubeless/blob/master/README.md).

The first thing we will need to do is to install the Serverless CLI globally:

{% highlight sh %}
$ npm install serverless -g
{% endhighlight %}

Once installed, we will create the needed scaffolding for our Python function, using the right template and specifying an optional path for your service:

{% highlight sh %}
$ serverless create --template kubeless-python --path new-project
{% endhighlight %}

This will create the needed files in the new-project folder.

Inside the folder you will find the following files: serverless.yml, handler.py and package.json.

serverless.yml is the Serverless Framework description file for your function.

{% highlight yaml %}
# This is a serverless framework way to group  
# several functions. Not to be confused with K8s services  
service: new-project  
provider:  
 name: kubeless  
 runtime: python2.7plugins:  
 - serverless-kubelessfunctions:  
 # The top name will be the name of the Function object  
 # and the K8s service object to get a request to call the function  
 hello:  
 # The function to call as a response to the HTTP event  
 handler: handler.hello
{% endhighlight %}
 
handler.py is the file where you define the Python function to call as a response to the HTTP event. The template creates for you an example hello function for you:

{% highlight python %}
import json  
  
def hello(request):  
 body = {  
 "message": "Go Serverless v1.0! Your function executed successfully!",  
 "input": request.json  
 }  
  
 response = {  
 "statusCode": 200,  
 "body": json.dumps(body)  
 }  
  
 print("hello world!")  
  
 return response
{% endhighlight %}
 
package.json is the npm package definition of our functions with all their dependencies, including the kubeless-serverless plugin.

Let’s try deploying the example function to Kubeless and run it:

{% highlight sh %}
$ cd new-project  
# Install npm dependencies  
$ npm install  
# Deploy the service using serverless  
$ serverless deploy
{% endhighlight %}

We can see that there is a new Service in our cluster with the same name as our function and a Function object:

{% highlight sh %}
$ kubectl get svc  
NAME CLUSTER-IP EXTERNAL-IP PORT(S) AGE  
hello 10.0.0.91 <none> 8080/TCP 2h$ kubectl get functions  
NAME KIND  
hello Function.v1.k8s.io
{% endhighlight %}

Now, let’s run the function and get the logs:

{% highlight sh %}
$ serverless invoke --function hello --data '{"Kubeless": "Welcome!"}' -l

Serverless: Calling function: hello...  
--------------------------------------------------------------------  
{ body: '{"input": {"Kubeless": "Welcome!"}, "message": "Go Serverless v1.0! Your function executed successfully!"}',  
 statusCode: 200 }
{% endhighlight %}
 
You can open up a separate tab in your console and stream all logs for a specific Function using this command:

{% highlight sh %}
$ serverless logs -f hello -t
{% endhighlight %}

That’s it! You can see how easy and straightforward is to deploy a Python function in Kubeless using the Serverless Framework templates.

#### Also for Nodejs functions

There is also a Nodejs template for Kubeless. You can create your scaffolding for your Nodejs application specifying the kubeless-nodejs template:

{% highlight sh %}
$ serverless create --template kubeless-nodejs --path my-node-project
{% endhighlight %}

In version 1.20 of the Serverless Framework, there is an open bug in the kubeless template for node and the runtime name was wrong. Check your serverless.yml file to make sure the name of the runtime version is nodejs6 or modify it to that before deploying the function. This will be fixed in the next version of the Serverless Framework.
