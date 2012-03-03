---
layout: post    
author: Martin "Waterback" Huber (innoq Deutschland GmbH) 
title: "Easy handmade SOAP-Webservice Versioning with Apache Camel"
date: 2012-03-02 22:38
comments: true  
published: true
categories:
 - Apache Camel
 - camel-http-component
 - camel-cxf-component
 - SOAP Versioning
---               
##Introduction
Have you ever wondered, how you could create a lightweight Versioning of your Webservices?
Or have you ever needed a simple way to create middle-tier for scheme-conversions
between a Webservice-Client and a Webservice-Server using different Schemes?

Then this post may be interesting to you.

A short while ago I was confronted with the exact problem I described as an Introduction. It was to create an integration help for a Business Process Management System. 
I mulled over that problem somewhat and came up with an example, how you could do this very easily (as i find) with my favorite Integration Framework. 

As you may have noticed I am talking about Apache Camel. In this Post I'd like to explain the solution i have found.

Before i get into the details, you could check and try (see readme) 
that example for yourself: It's located at github:           

[BPEL-Camel Example](https://github.com/Waterback/BPELCam)

Here in this post i will only repeat the interesting parts of it.

... Now let's begin ...

##Business Case

Lets suppose, we have a Webservice for an specific business-purpose. 
The business case is invented, so please don't give too much on the details of it.

I had to find a way to service webservice-clients that for example, know a Version 1-Schema of our Webservice, but need to be 
serviced by the Version 2-Webservice. 

And vice-versa, meaning a V2-Scheme-Call should be able to be serviced by the V1-Webservice.
 
##Configuration                                                          
I have two WSDLs in this example that differ only by the imported Xsd-Documents for V1 and V2 Schemas.

As Camel is well integrated with the CXF-Framework to provide Webservices, you only have to create some configuration. In my example
i use Spring for my configuration purposes.
Here's a part of the *camel-context.xml* -Spring config:

For this simple example i use the the Jetty-Component to create a Http-Endpoint as a servlet for both webservices: 
             
    <bean id="jetty" class="org.apache.camel.component.jetty.JettyHttpComponent" /> 

	<cxf:cxfEndpoint id="insuranceEndpoint-v1"
		address="http://localhost:9000/process1"
		wsdlURL="/wsdl/insuranceV1.wsdl">
		<cxf:properties>
			<entry key="dataFormat" value="MESSAGE" />
			<entry key="publishedEndpointUrl" value="http://localhost:9080/insuranceservice" />
		</cxf:properties>
	</cxf:cxfEndpoint>
	<cxf:cxfEndpoint id="insuranceEndpoint-v2"
		 address="http://localhost:9000/process2"
		wsdlURL="/wsdl/insuranceV2.wsdl">
		<cxf:properties>
			<entry key="dataFormat" value="MESSAGE" />
			<entry key="publishedEndpointUrl" value="http://localhost:9080/insuranceservice" />
		</cxf:properties>
	</cxf:cxfEndpoint>

The Endpoint-Address defined in the WSDL is overwritten with a configuration property: *publishedEndpointUrl* that points to that servlet i wanna use as a entrance to my camel-routes. 
This configuration takes care
for the Endpoint that is printed into the wsdl when you call: *http://localhost:9000/process1?wsdl*       

With this i can create a unified entrance to both webservices.

                                                              
##Camel-Routes: unified entrance for the webservices
But how is this handled in the camel-routes? See yourself:
    
	package com.innoq.bpelcam;

    public class SchemaEvalutionRouter extends RouteBuilder {

	    @Override
	    public void configure() throws Exception {

			Namespaces ns_v1 = new Namespaces("c1", "http://bpel.innoq.com/insurance/v1/types");
			Namespaces ns_v2 = new Namespaces("c2", "http://bpel.innoq.com/insurance/v2/types");

			from("jetty:http://localhost:9080/insuranceservice?minThreads=5")
				.choice()
				.when().xpath("//c1:CarInsuranceFindProcess", ns_v1)
					.setHeader("schemaVersion", constant("1"))
					.setHeader("targetVersion").xpath("//c1:targetVersion", String.class, ns_v1)	
					.to("seda:evaluateV1Scheme")
				.when().xpath("//c2:CarInsuranceFindProcess", ns_v2)
					.setHeader("schemaVersion", constant("2"))
					.setHeader("targetVersion").xpath("//c2:targetVersion", String.class, ns_v2)
					.to("seda:evaluateV2Scheme")
				.otherwise()
					.to("seda:fault");
                    
			...

So there's a Jetty-Endpoint which starts the jetty-container implicitly.        

If you'd like to use your average WebSphere-Web Container ;-) or a Tomcat you have to configure 
it a little bit different in the Spring-config, but this should be the topic for another post (or the Camel-documentation).

We have here a WS-Addressing-like (correct me, when i'm wrong) determination which Schema the Webservice-Caller uses with
a simple XPath-Expression. The corresponding namespaces have to be defined at the beginning of the configure method. 

For later evaluation this information is stored in the Camel-Exchange as Meta-Information (Header).
Also the **targetedVersion** is recognized and stored for later evalutaion as a Header.
Depending on the recognized Schema as V1 or V2, we incorporate a evaluation-route, which is build for each Webservice-Schema-Version:

		from("seda:evaluateV1Scheme")
			.choice()
				.when(header("targetVersion").isEqualTo("2"))
					.pipeline("direct:transformV1V2")
					.to("cxf:bean:orderEndpoint-v2")
				.otherwise()
					.to("cxf:bean:orderEndpoint-v1");           
					
		from("seda:evaluateV2Scheme")
			.choice()
			...
					
		from("direct:transformV1V2")
			.bean(new V1TransformerBean(), "transformV1toV2");  

There the targetVersion comes in handy. So you don't need no more XPath-evaluations to know where to go.
So in that matter, for each non-original schema-version (V2,V3 and so on), you need to expand this route 
with another when()-Block. 

The **pipeline("direct:transformV1V2")**-pattern, which enables to incorporate camel-routes (here: *"direct:transformV1V2"*) 
for definition is used in this example to show, 
that you can distribute your schema-conversion on different beans or/and routes as you wish.     
The V1TransformerBean takes care in my case for the whole V1 to V2-conversion of the sent document. 
This bean has grown in a not so easy component, so i'm gonna explain in one of my next posts.  

**Last trick...**     

The last trick is that the whole exchange is sent in the end to **.to("cxf:bean:insuranceEndpoint-v2") **. 
This is a cxf-endpoint.
This could be your actual implementation of the versioned Webservice:
     
	 package com.innoq.bpelcam;

     public class WebservicesRouter extends RouteBuilder { 
		@Override
		public void configure() throws Exception {
			DataFormat jaxb10 = new JaxbDataFormat("com.innoq.bpel.insurance.v1.types");
			DataFormat jaxb11 = new JaxbDataFormat("com.innoq.bpel.insurance.v2.types");

			Namespaces ns_v1 = new Namespaces("c1", "http://bpel.innoq.com/insurance/v1/types");
			Namespaces ns_v2 = new Namespaces("c2", "http://bpel.innoq.com/insurance/v2/types");

			from("cxf:bean:insuranceEndpoint-v1")
				.id("handleProcess_v1")
				.transform().xpath("//c1:CarInsuranceFindProcess", ns_v1)
				.unmarshal(jaxb10)
				.process(new OrderContentsProcessor())
				.marshal(jaxb10)
				.to("seda:save")
				.process(new ResponseBuilderProcessor(10));


But you could also easily redirect the whole soap-document to another system (i.e. BPEL).
Therefore it only needs to be configured in your camel-spring-configuration as Endpoint pointing to another system.
        
A side note here is: I always use the Java-Fluent-API, because with a little discipline in formatting, you could have your routes
configured in a very transparent and clear matter. I'd prefer that over xml- or even graphical-configuration every time.

##That's it
So that's it folks.
If something is not clear to you, i am happy for feedback/questions ;-)

Cheers,
Martin  


	







  


 
