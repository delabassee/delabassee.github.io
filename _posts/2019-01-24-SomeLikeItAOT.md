---
layout: post
title: Serverless Functions — Some Like It AOT!
excerpt: This article explains how Fn users can use GraalVM and the benefits its AOT compiler brings to Serverless Java functions....
---


<p align="center">
<img alt="Photo by Markus Spiske" src="https://delabassee.com/images/blog/aot0.jpg" width="75%"/>
</p>

This article explains how Fn users can use GraalVM and the benefits GraalVM and its Ahead-of-Time (AOT) compiler bring to Serverless Java functions.

## Introduction

[Fn Project](http://github.com/fnproject/fn) is an open-source, container-native, polyglot FaaS (Function as a Service) platform.


Fn is *open-source*, one can run Fn on-premises and/or in the cloud; running Fn on a laptop is also convenient for experimentation and development.


Fn is *container-native* as it leverages Docker. In a nutshell, serverless functions are automatically wrapped into Docker container images (but advanced users can also provide their own Dockerfile!). Fn will take care of all the plumbing, from the creation of the function Docker image to the interaction between the function and the Fn platform to the scaling of this same function, etc.


Finally, Fn is *polyglot* as it offers multiple FDKs (Function Development Kit) to easily write serverless functions using popular languages such as Java, Go, Node, etc. And given that Fn uses Docker under the hood, it is also trivial to add support for additional languages.


[GraalVM](https://www.graalvm.org/) is an open source high-performance embeddable polyglot virtual machine that recently sparked a lot of interests in the Java community as it supports Java and other JVM languages such as Groovy, Kotlin, Scala, etc. In addition, GraalVM also supports JavaScript (including the ability to run Node.js applications), Ruby, R, Python and even native languages that have an LLVM backend such as C or C++. GraalVM is incredibly powerful and versatile, it can help in many ways from boosting the performance of Java applications to enabling polyglot applications development that combine different languages in order to to get the best tools and features from different ecosystems. For example, using GraalVM, it is possible to use R for data visualization, Python for machine learning and JavaScript to combine those two functionalities together.


This article will focus on a specific GraalVM capability, i.e. GraalVM Ahead-of-time Compilation (AOT) and more specifically on GraalVM native-image feature with Java functions.


## Your First GraalVM Serverless Function

Building GraalVM based Java functions is very similar to building and deploying “regular” Java functions so it is a good idea to familiarize yourself with the [Java FDK tutorial](https://github.com/fnproject/tutorials/tree/master/JavaFDKIntroduction).


The only requirement to run Fn locally is to have Docker (17.10+) installed and running. You then only need to install the [Fn CLI](https://github.com/fnproject/cli) which will in turn install Fn in your local Docker.


To bootstrap a GraalVM based Java function, simply use the following `init` command.


`fn init --init-image fnproject/fn-java-native-init myfunc`

If you compare this to the approach used for generating a “regular” Java function, the key difference is that we instruct the Fn CLI to rely the *fnproject/fn-java-native-init* Docker init-image (see [here](https://medium.com/fnproject/even-wider-language-support-in-fn-with-init-images-a7a1b3135a6e) for more details on init-image) to generate a boilerplate GraalVM based Java function (instead of relying on the regular *java* runtime option).


You should see the following output…


<p align="center">
<img alt="fn init" src="https://delabassee.com/images/blog/aot1.png" />
</p>


You can now check the different files that have been generated by looking into the *myfunc* directory. And unless you are in the business of writing “Hello World”, you will at least tweak the sources of the Java function located in *src/main/* and its corresponding tests in *src/test*.


The *func.yaml* contains some meta-data related to the function (its version, its name, etc.). It is very similar to a regular Java *func.yaml*, the only difference being the runtime entry. The Java function uses the *java* runtime while the GraalVM *native-image* function rely on the default Docker runtime which also explains the presence of a *Dockerfile*.


Assuming that Fn is running (`fn start), you can now deploy the function that you have just created. Fn uses the notion of “application” to logically group related functions. You simply specify this application (*myapp* in this case) during the deployment of the function. We don’t specify the function given that we are in the function directory.

`fn deploy --local --app myapp`


The `—-local argument is used to instruct Fn to not push the newly generated function Docker image to a registry.


Given that the function wasn’t built before (eg. via `fn build`), the Fn CLI will build it and deploy it to the Fn Server. It should be mentioned that the function itself is built within a Docker container so there’s no need to have any toolchain installed locally.


To see in details what is going on, you can add the `—-verbose` flag.


`fn --verbose deploy --local --app myapp myfunc`


<p align="center">
<img alt="fn deploy" src="https://delabassee.com/images/blog/aot2.png" />
</p>



To generate the Docker image of the function, the Fn CLI is relying on the Dockerfile that was generated during the previous init phase. If you inspect this Dockerfile or if you look at the verbose output of the depoyment, you will notice that one of the step is using GraalVM’s *native-image* utility to compile the Java function ahead-of-time into a native executable. Depending on the machine it runs on, this particular step will take some time.


<p align="center">
<img alt="Step 15/22" src="https://delabassee.com/images/blog/aot3.png" />
</p>




The resulting function does not run on a “regular” Java virtual machine but uses the necessary components like memory management, thread scheduling from a different virtual machine called Substrate VM (SVM). SVM, which is a part of the GraalVM project, is written in Java and is embedded into the generated native executable of the function. Given it is a small native executable, this native function has a faster startup time and a lower runtime memory overhead compared to the same function compiled to Java bytecode running on top of a “regular” JVM.


That native function executable is finally added to a base lightweight image (`busybox:glibc`) with some related dependencies. This will constitute the function Docker image that the Fn infrastructure will use when the function is invoked.


<p align="center">
<img alt="fn deploy output" src="https://delabassee.com/images/blog/aot4.png" />
</p>


You can check the size of the resulting function image using Docker.


<p align="center">
<img alt="docker images | size" src="https://delabassee.com/images/blog/aot5.png" />
</p>


As you can see, the function image that includes everything required to run, including the operating system and the function native executable, is only weighing around 21 MB! Since all necessary components of a virtual machine (ex. GC) are embedded into the function executable, there is no need to have a separate Java runtime (JRE) in the function Docker image! When a function is invoked through Fn, Fn will instruct Docker to pull and launch the corresponding container image and hence the smaller this container image is, the better it is to reduce the cold-startup time of this function.


Finally, to invoke the deployed function, simply use the `invoke` command as you would do with any function, ex. `fn invoke myapp myfunc. The first argument is the name of the application and the second argument is the name of the function itself. Obviously, we could also add an HTTP trigger to invoke the function using HTTP.


To give us an idea, we can quickly measure the cold startup of the function.


<p align="center">
<img alt="time fn invoke svm" src="https://delabassee.com/images/blog/aot6.png" />
</p>


We can compare those results to the results of the same “Hello World” function that uses HotSpot.


<p align="center">
<img alt="time fn invoke java" src="https://delabassee.com/images/blog/aot7.png" />
</p>


We can see that the cold startup time of a GraalVM *native-image* function is improved.


Those numbers will vary depending on the machine you run the tests on but this basic benchmark shows that the cold startup of the GraalVM *native-image* function is faster. In this particular example, the cold startup of the GraalVM *native-image* function is ~70% (604ms) of the cold startup time (847ms) of the same Java function that uses a regular JVM (HotSpot in this case).


This “Hello World” function is obviously not reflective of any real-world scenario, we plan to share more “real-life” measures in a follow-up article.


## Conclusions

As you can see with this article, using GraalVM in Fn for Serverless Java function is simple and straightforward. The Fn GraalVM integration relies on GraalVM’s *native-image* feature to compile ahead-of-time a Java function into a native executable that embeds the function itself and the necessary components of a runtime like the garbage collector; resulting in a lightweight function Docker image with faster startup-time. Using GraalVM in Fn gives us Java function that are on par with functions written in languages that are compiled natively such as Go! In Fn, GraalVM offers all the benefits of Java functions, including support of the Java FDK, with the benefits of native functions!


PS: Currently, GraalVM supports Java 8 but the team is working on adding support for more recent Java versions. You might also need to provide additional configuration to the *native-image* utility to successfully compile your code, see here for more details on the [current limitations](https://github.com/oracle/graal/blob/master/substratevm/LIMITATIONS.md).


PPS: Thanks tulo GraalVM Team for bringing GraalVM support into Fn!


### Additional resources

* [GraalVM Ahead-of-time Compilation](https://www.graalvm.org/docs/reference-manual/aot-compilation/)
* [GraalVM GitHub repository](https://github.com/oracle/graal)
* [Fn Project GitHub organisation](http://github.com/fnproject/fn)


*Originaly posted on [Medium](https://medium.com/fnproject/serverless-functions-some-like-it-aot-ea8b46951335).*