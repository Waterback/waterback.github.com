---
layout: post
title: "Camel: Inherit Errorhandling"
date: 2011-12-09 00:14
comments: true
categories: 
---
Hello All,

today i want to write sth new from my ongoing understanding of my work with Apache Camel.
This time it's not that of a big story as last time, but quite useful:
Up to this point (actually a little bit in the past now) i thought when building Routes with 
Apache Camel in a org.apache.camel.RouteBuilder you have to write everything you want to configure
in that respective configure()-method and that's it. 

Well that's not true...
<!--more -->
You may as well as call any method you like, i.e. from a class your RouteBuilder inherits from 
and Camel interprets everything as a whole configuration.

Before some might ask: and why should that be useful - Well you could for example define some errorHandlers in that super class and make them accessible for every Camel-Route in which the errorhandling should be active, so you have to define only once but could use those errorhandling definitions in more than one RouteBuilder. 

	public class InboundRouter extends ErrorHandlingRouteBuilder {

	    @Override
        public void configure() throws Exception {
            super.inheritConfigure();
     
            from(fromEndpoint)
                  .noAutoStartup()
                  .transacted()
                  .policy(required)
                  .routeId(BasicRouteBuilder.stdfromRouteId(this.componentBaseName))
                  .process(neededHeadersProcessor)
                  .to(nextstepfromRoute);
        }

    }

And in ErrorhandlingRouteBuilder:       
  
    inheritConfigure() {

        errHdlRoute = "direct:errorfwd." + cpBase;
  
        errorHandler(deadLetterChannel(getContext().getEndpoint(errHdlRoute))
                        .maximumRedeliveries(errHandlingParameters.getMaxRedeliver())
                        .useOriginalMessage()
                        .logStackTrace(false)
                        .maximumRedeliveryDelay(errHandlingParameters.getMaxRedeliverDelay())
                        .redeliveryDelay(errHandlingParameters.getRedeliverDelay())
                        .backOffMultiplier(errHandlingParameters.getBackoffMultiplier())
                        .useExponentialBackOff());

        onException(PermanentError.class)
                        .maximumRedeliveries(0)
                        .useOriginalMessage()
                        .handled(true)
                        .to(errHdlRoute);

        from(errHdlRoute)
                        .routeId("errorforwarder")
                        .process(new ErrorhandlingHeadersProcessor())
                        .process(new ConvertErrorToJPAProcesor())
                        .to("jpa:com.innoq.errhandling.SaveMessageStore?persistenceUnit);

    }
      

Ahem, don't read too much into that errorhandling stuff itself, the configurations shown are parts, taken from my most recent project.<br>
I promise i will come back with another post to explain my solution for errorhandling.
I only wanted to show that every camel-configuration aspect in the above example is also considered from camel in it's configuration process.

As already mentioned you could build a big callstack in inherited classes or in others - it is only important to start your callstack inside the configure()-method of a RouteBuilder that gets loaded with the start of the CamelContext.

Now, if you might say: "Hello Mr. Obvious" - sorry that i wasted your precious time.

For me, from reading the available online docs and the Book "Camel in Action" 
(which is, by the way, a very good one and a great way to start/work with Apache Camel)
this wasn't so obvious to me.

Cheers...   
Martin
