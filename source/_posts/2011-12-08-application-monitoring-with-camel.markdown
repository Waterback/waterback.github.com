---
layout: post
title: "Camel: Application-Monitoring"
date: 2011-12-08 10:05
comments: true
categories: 
---                 
I've been working with Apache Camel for a year now and i am still scraping at the top of all
the possibilities you have with Camel. 

In this blog i wanna show something, that might be helpful when Camel is already used and someone want to keep
the statistics that your routes generate in a database for some sort of long time evaluations. 
I think it's quite easy to do so and definitely not limited to handling statistics of camel-routes.
If you have absolutely no idea what i could possibly mean with routes and statistics, i recommend some reading:
<a href="http://camel.apache.org">Camel WebSite</a> and for the jmx part itself: <a href="http://camel.apache.org/camel-jmx.html">Camel JMX</a>  
                                                           
Because this example here is build around camel-routes you'll see the statistics kept by 
a 'org.apache.camel.management.mbean.ManagedPerformanceCounter'  
I want to persist those stats in a database using JPA. So i need a @Entity for this:

                                                      
    @Entity(name = "CAMELSTATENTRY")
    @Table(name = "MyStats")
    public class CamelStatEntry {

        private Long id;
        private String nameIdentifier;
        private Date statisticTimeStamp;
        private Long totalExchanges;
        private Long exchangesCompleted;
        private Long exchangesFailed;
        private Long minProcessingTime;
        private Long maxProcessingTime;
        private Long totalProcessingTime;
        private Long lastProcessingTime;
        private Long meanProcessingTime;
        private Date firstExchangeCompletedTimestamp;
        private Date firstExchangeFailureTimestamp;
        private Date lastExchangeCompletedTimestamp;
        private Date lastExchangeFailureTimestamp;

        @Id
        @Column(name = "ID")
        public Long getId() {
            return id;
        }
    	...
 
I think it's not too complicated to see, what those values in the CamelStatEntry-Entity stand for.
<!--more -->
This values get collected by Camel while exchanges "run through" your routes.
The trick is now to gather those statistics in a time-interval. The idea behind the time-interval is to be able to see, how your routes "behave" over time.
For this time-interval we're going to use another route, a route defined in StatCollectionRouter-RouteBuilder.
This route is one, that get's triggered at defined points in time while using the
 <a href="http://camel.apache.org/quartz.html" >quartz component</a> (in my example every full quarter of an hour).
  
So there's the route:

	public class StatCollectionRouter extends RouteBuilder implements InitializingBean {

		/**
		 * Spring injected bean, that does all the collection work, and keeps the
		 * information which routes have to be checked.
         */
        private StatCollectorBean statCollBean;
        private String targetEndpoint = "jpa:?persistenceUnit=zifsdb&consumeDelete=false";

        @Override
        public void configure() throws Exception {
            from(quartz://zs3vsl/statctr?cron=0+15+*+*+*+?+*)
                .routeId("StatisticalRoute")
                .bean(statCollBean, "collect")
                .split(body())
                .to("jpa:?persistenceUnit=statsdb")
                .threads(1);
        }
    	...
    	
  
After the route got triggered it calls a bean, that does all the collection-work.
This bean returns a list of all CamelStatEntry-Objects. The split() divides the one trigger into so many message-exchanges
as you have CamelStatEntry-Objects in that List, returned by the collect-method of the statCollBean. Then every CamelStatEntry
gets stored, when sent to the jpa-component. Of course, the "statsdb"-persistenceUnit is defined in a persistence.xml-file.  
  
The last thing you need to know how to collect the stats from the Camel-Routes-MBeans.

Well the following assumes, that you have a jmx-connector started with your CamelContext and know a little bit about
connecting to them and requesting MBeans. Explaining all that would be to much at this. Look @ the recommended reading above
to see how this is done.

I cut my example-code a little bit, because there's a point in there that's rather boring: How to select the Objectnames to check. 
I need to connect to the MBeanServer of my "DefaultDomain". My Jmx-Agent is configured in the spring-config like this:        

     <camel:camelContext id="myCamelContext" >
        	<camel:jmxAgent id="com.innoq.jmxagent" createConnector="true"  
			mbeanObjectDomainName="com.innoq" mbeanServerDefaultDomain="WebSphere"
			xmlns:camel="http://camel.apache.org/schema/spring" connectorPort="39099" 
			onlyRegisterProcessorWithCustomId="true" />                          
	</camel:camelContext>                                                                                               
    
                                                                                                                             
The "DefaultDomain" mentioned above is "com.innoq".  
You need to know the default-domain, because when you search for your MBeans with the Objectname, you have to set the right Domain.
The following is a part of the initialization of the StatCollectorBean. I search for Route-Types in my MBeanServer. Note that
i have an extra-class "MyJMXHelper" that finds the right MBeanServer for me, which i left out for space reasons. And also allow me
to say that this is just an example for creating the list: "allObjectNames2Check" - the ObjectNames, for which i want to check my Statistics.  
Feel free to do this more elegantly.  
The things you need to know gather around the build-up of the searchstring s 
and that you maybe make a List of all MBeans that should be checked.
	  
    public class StatCollectorBean implements JMXConstants {

	    /**
 	    * Result-List of all Objectnames that have to be statistically checked.
 	    */
    	private Set<ObjectName> allObjectNames2Check;

	
     ...
   
	    @SuppressWarnings("unchecked")
        private void addRoutes2CheckSet(List<String> searchStringList) {
            MBeanServer mbs = MyJMXHelper.findDefaultDomainMBS();
            ObjectName ojnsearch;
            Set<ObjectName> routembeans = null;
            Iterator<String> it = searchStringList.iterator();
            while (it.hasNext()) {
                String routeIdName = it.next();
                try {
                    String s = "com.innoq:type=routes, name=\"" + routeIdName ""\", *";
                    ojnsearch = new ObjectName(s);
                    routembeans = mbs.queryNames(ojnsearch, null);
                    if (routembeans.size() != 0) {
                        allObjectNames2Check.addAll(routembeans);
                    }

                }
                catch (MalformedObjectNameException e) {
                    e.printStackTrace();
                }
                catch (NullPointerException e) {
                    e.printStackTrace();
                }
            }
        } 


The last piece of code i wanna show is the collect() method of the StatCollectorBean. It works with the Set "allObjectNames2Check"
initialized in the addRoutes2CheckSet-method

    public List<CamelStatEntry> collect() throws RuntimeException {
        if (!initialized) {
            synchronized (StatCollectorBean.class) {
                init();
            }
        }
        VirtualClock clock = new VirtualClock();
        List<CamelStatEntry> list = new ArrayList<CamelStatEntry>(allOjn2Check.size());
        try {
            for (ObjectName ojn : allOjn2Check) {
                list.add(createCSEfromMBean(ojn, clock.getDate()));
            }
        }
        catch (Exception e) {
            throw new RuntimeException(e);
        }
        return list;

    }

    private CamelStatEntry createCSEfromMBean(ObjectName ojn, Date statisticTimeStamp)
            throws Exception {            

   		final String[] emptySig = new String[] {};
		final Object[] emptyParms = new Object[] {};

        CamelStatEntry cse = new CamelStatEntry();
        MBeanServer mbs = MyJMXHelper.findDefaultDomainMBS();
        String nameId = (String) mbs.invoke(ojn, "getRouteId", emptyParms, emptySig);
        cse.setNameIdentifier(nameId);
        cse.setExchangesCompleted((Long) mbs.invoke(ojn,
                                                    "getExchangesCompleted",
                                                    emptyParms, emptySig));
        cse.setExchangesFailed((Long) mbs.invoke(ojn, 
													"getExchangesFailed", 
													emptyParms, emptySig));
        cse.setFirstExchangeCompletedTimestamp((Date) mbs.invoke(ojn, 
													"getFirstExchangeCompletedTimestamp", 
													emptyParms, emptySig));
        cse.setFirstExchangeFailureTimestamp((Date) mbs.invoke(ojn, 
													"getFirstExchangeFailureTimestamp", 
													emptyParms, emptySig));
        cse.setLastExchangeCompletedTimestamp((Date) mbs.invoke(ojn, 
													"getLastExchangeCompletedTimestamp", 
													emptyParms, emptySig));
        cse.setLastExchangeFailureTimestamp((Date) mbs.invoke(ojn,
                                                    "getLastExchangeFailureTimestamp",
                                                    emptyParms, emptySig));
        cse.setLastProcessingTime((Long) mbs.invoke(ojn, 
													"getLastProcessingTime",
                                                    emptyParms, emptySig));
        cse.setMaxProcessingTime((Long) mbs.invoke(ojn,
                                                   "getMaxProcessingTime",
                                                   emptyParms, emptySig));
        cse.setMinProcessingTime((Long) mbs.invoke(ojn,
                                                   "getMinProcessingTime",
                                                   emptyParms, emptySig));
        cse.setMeanProcessingTime((Long) mbs.invoke(ojn,
                                                    "getMeanProcessingTime",
                                                    emptyParms, emptySig));
        cse.setStatisticTimeStamp(statisticTimeStamp);
        cse.setTotalExchanges((Long) mbs.invoke(ojn, 
													"getExchangesTotal", 
													emptyParms, emptySig));
        cse.setId(statisticTimeStamp.getTime() + nameId.hashCode() + servername.hashCode());

        cse.setTotalProcessingTime((Long) mbs.invoke(ojn,
                                                     "getTotalProcessingTime",
                                                     emptyParms, emptySig));
        mbs.invoke(ojn, "reset", emptyParms, emptySig);
        return cse;
    }    

Note that it is important to reset the statistics, so i have for every time interval a fresh set of the statistics.
  
So that's it.  
Hope you'll be able to use some of this.    
   
Cheers, 
Martin





