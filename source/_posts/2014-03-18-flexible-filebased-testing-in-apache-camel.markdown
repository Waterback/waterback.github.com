---
layout: post
title: "Flexible Filebased-Testing in Apache Camel"
date: 2014-03-18 16:56
comments: true
categories: 
---
###Introduction
Well... "long time no see", but finally I get to cameleering through Software.        
	
For my Camel-Routes I used a Test-Setup, I came to think, it may be worth sharing.

The Camel-Route that is to be tested (from now on called: Working-Route) is actually not important. But you need to know, that the route is servlet that wraps 
SOAP-Service-Calls. It transforms schemes and the Transformation-Result at the output-endpoint has to be validated.

For the Unittest of the Camel-Route I have some Samples organized in Files. The idea is now to read the files with another route and send them to
the Files as new Exchanges to the servlet-input-Endpoint the Working-Route exposes. 
In the Test-Setup, the Working-Route sends it's (transformed) Exchanges to a instantiated Validation-Route (configured to test for the transformed scheme), 
filling in for the behind SOAP-Service.
The Response that goes through the initiating "file-reading"-Route wiretaps also to another instance of the Validation-Route testing for the backward-transformed Scheme. 

###File-Route
The File-Route looks like this: The "identifier" in the following code helps to the identify the Scheme to test for in this instance of the route.

		from("file:" + routeFolder + "?noop=true&readLock=rename&filter=#testfileFilter")    				// (*1)
       		   .autoStartup(false) 				                	                                        // (*2)
       		.routeId("fileread" + identifier)
       		.throttle(4)																					// (*3)
       		.convertBodyTo(String.class)
       		.wireTap("file://target/output/"+identifier)                                     				// (*4)
       		.choice()                                                                        				// (*5)
              	.when(simple("${in.body} not contains 'soap:Envelope'"))
               		.setBody().simple(TransformationConstants.soapStart + "${in.body}" +             
                                                                          TransformationConstants.soapEnd)
       		.end()
       		.to(transformationProxyIN)                                                       				// (*6)
       		.to(validateendpoint)                                                            				// (*7)
       		.setHeader("CamelFileName", simple("${file:name.noext}_TransformationResult.${file:name.ext}")) // (*8)
       		.to("file://target/output/"+identifier)                                                         // (*9) 
                                                                                                  
Explanation:    
(*1) The Fileroute is defined to read from a "routefolder" but doesn't change or remove Files (like the Camel-File-Consumer does in default)    
A filter makes sure only Testfiles are read. If noop is on, the file-component automatically uses a idempotent-repository to ensure the same file
is not read more than once.    
(*2) Any instance of this Route is not active from the get-go. Each Unit-Test uses its own folder, starts the correspondent Route, so only the folder with the associated 
Testcases are read                                                                                                  
(*3) For Performance-Reasons on the Test-Machine, the parallel-processing of files is throttled to 4 in parallel.     
(*4) A Requirement for the test is to direct the input & the output of the whole processing to a specific-folder in target. (Mainly to be able to compare them if not rightfully converted). 
This command-config directs a copy to the folder via the wiretap-EIP.   
(*5) If the sample-File doesn't contain a SOAP-Header (Well, some Files do, some don't, you'll never know ;-)) the sample gets embedded into
a soap-envelope     
(*6) Sends the file to the Working-Route    
(*7) Is feeded with the response of the Working-Route and sent to the Validation-Route    
(*8) Changes the CamelFileName-Header, which is used if the file-component is ever used again, which in fact is in *9.   
Isn't the "simple"-language a cute little thing. See how easy it is to change a file based on an old one.    
(*9) See *4.s.. Sends the result to the same folder like the original read file.    
    
###Validation-Route    
The Validation-Route has two interesting aspect - the Route that validates against a Scheme. And the errorhandling
for validation-failures. Validation failures get send to the "mock:failure" endpoint. From this MockEndpoint the
Unittest is able to evaluate the testresult. 

     /**
      *  The "validationroute" configured in this Method, strips the soap-parts off the message and validates it against the xsd
      *  if validation succeeds, the Message-Exchange is sent to MockEndpoint "success".
      *  If Validation fails, a ValidationException is thrown and handled by the configured ErrorHandling
      * @throws Exception
      */
     @Override 
     public void configure() throws Exception {
         Namespaces soapNS = new Namespaces("s", "http://schemas.xmlsoap.org/soap/envelope/");

         onException(ValidationException.class)
             .maximumRedeliveries(0)
             .handled(true)
             .to("mock:failedValidation");

         from(validateendpoint)
             .routeId("validate" + identifier)
             .setBody().xpath("/s:Envelope/s:Body/*[1]", soapNS)
             .to("validator:" + schemaLocation)
             .wireTap("mock:success")
             .process(new ResponseFileProcessor())
             .setBody().simple(TransformationConstants.soapStartTixBank + "${in.body}" + TransformationConstants.soapEnd);
         }
     } 

###Unit-Test
The test itself is relatively simple: 
 - Count the Files (so you know how many results to expect - another requirement: If there are new samples put into folder, tests
and assertions should work without changing code)
 - Start the file-reading route. 
 - Assert the results from mock:success and mock:failedValidation
 - and finally stop the file-reading route

		@Test
		public void testBasicTransformationFunctionalityIN_10() throws Exception {
        	int files = countFiles("filereadV10");
        	camelContext.startRoute("filereadV10");
        	validationAsserts(files, 0);
        	camelContext.stopRoute("filereadV10");
    	}

It may look a bit tricky and strange not to use java.io.File for this. 
It was just a try, but it worked, so i kept it, because it was the solution, that made the least work.
You can use a Endpoint-Definition (that I already had for the filereading-route) to look how many exchanges may wait there for me.
And then I used it to find out how many positive result have to be asserted for a successful test.
See the following piece of code:

        private int countFiles(String routeId) {
            BrowsableEndpoint browsableEndpoint = (BrowsableEndpoint) camelContext.getEndpoint(
                    camelContext.getRoute(routeId).getConsumer().getEndpoint().getEndpointUri());
            return browsableEndpoint.getExchanges().size();
        }
   
That's it folks, hope you find it useful. 





