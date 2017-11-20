---
layout: post
title: How to avoid Apex batch race conditions
date: 2017-09-12 00:00:00 +0300
description: How to avoid Apex batch race conditions using simple strategy. # Add post description (optional)
img: batch-race.jpg # Add image post (optional)
tags: [Salesforce, Apex] # add tag
categories: Blog
---

Time 3:00 am, it's a war-room at office. We are trying to figure out a weird (and costly) issue, solution looks miles away as we are yet to reach the root cause in the legacy code. Issue was quite simple though, customers had been charged on two instances twice for same transaction and attachment which was (supposed to) log payment attempts were blankly staring at us. So basically customers money just vanished in a black-hole with no footprints in Salesforce instance at least, gulp.

Everyone were up for almost 20 hours by now, only possible root cause we figured out so far was hiding behind two daemon (demon for us) Apex Batches overlapping within same time frame. The aha! moment slowly emerged over a cup of coffee. The batch processed the record if it satisfied a particular condition,

“What if another batch processes a particular record and first batch still sees the unprocessed record?”

The answer laid between start() and execute() methods, a lot (than obvious) was happening there!!!

The record was processed twice because force.com caches records queried in start() across start and execute invocations. This means Batch2 dirty reads record1 state as it was during the time Batch2.start() was executing and not the current database state after Batch1.execute(), which has already processed the record.

Solution: Never (ever) depend on the stale state of records in Batch.execute() method, better we query only Id’s based on where conditions, and pass the collection to execute.
In Batch.execute(), re-query the database on set of Id’s retrieved in Batch.start() including the where clauses. Including where clause again in Batch.execute() is very important because same records which passed the where clause in Batch.start() may fail in execute to satisfy the where condition(s), as some other batch transaction may have updated the records in the mean time.

To conclude, we programmers tend to think linearly and build batches without anticipating runtime interaction between multiple batch threads. Comprehensive testing is a key, though not all issues/bugs are guaranteed to be always uncovered.

So the best possible ways to tackle critical section issue is

Never assume the order of execution of the batches
Always follow defensive programming tactics, i.e. validating database state in execute.


Example Batch Template  :

    public class OpportunityBatchable implements Database.Batchable<sObject> {
    
	    private static final Integer TARGET_AMOUNT=10000;
	    public  static final String  OPPORTUNITY_STAGE_OPEN = 'Open';
	    
	    public Database.QueryLocator start(Database.BatchableContext batchableContext) {
		return Database.getQueryLocator(getQuery());
	    }
    
	    public void execute(  Database.BatchableContext batchableContext
		                , List<Opportunity> opportunityList) {
		//***re-query the database***
		opportunityList = Database.query(getQuery(opportunityList)); 
		
		if(!opportunityList.isEmpty()){
		    //Buisness logic which updates the opportunity status
		    //BusinessLogic.processOpportunities(opportunityList);
		    system.debug(opportunityList);
		}
	    }
	    
	    private String getQuery(){
		return getQuery(null);
	    }
	    
	    private String getQuery(List<Opportunity> opportunityList){
		
		String query =   ' Select Id'
		                +' From  '+  Schema.SobjectType.Opportunity.Name
		                +' Where '+  Opportunity.Amount    +' > :TARGET_AMOUNT'
		                +'   AND '+  Opportunity.StageName +' IN :OPPORTUNITY_STAGE_OPEN';
		
		if(null!=opportunityList && !opportunityList.isEmpty()){
		    query+= ' AND Id IN :opportunityList';
	    	}
        
		return query;
	    }
	    
	    public void finish(Database.BatchableContext batchableContext) {
	    }
    }


Thank you for visiting, if you have faced similar issues do mention in comments. Would love to hear your experiences too.
