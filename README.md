# SplunkS3FreezeThawSystem
Description and design for Splunk freeze routine. Smartstore of classic indexers, utilizing S3 and glacier for long term storage.

This project makes several assumptions about the Splunk environment.  
1. Retention period is beyond the requirements for immediately searchable time range.  
2. Finite resources, as in their is motive to reduce S3 costs.

Background  
SmartStore is the best indexer mode to manage petabyte scale data sets in Splunk. It reduces the cost of storage immensely, boosts search performance of almost all queries, and allows for sane management of management of indexers at scale. One could go so far as maintenance on indexers housing 30+ terabytes of data each become more and more unmanagement as the volume of data on each increases.  

While S3 remote storage has huge benefits for Splunk, it still has performance and monetary costs that must be optimized.  Only make available to SmartStore indexers data that is required. If normal users only need to search 6 months, only keep that time range available.  
1. Create a more performant system by reduce the dataset Splunk must handle. Specifically, reduce Smartstore cache churn and 'bad' searches, such as a cybersecurity search for a single IP across all data sources for an entire year.  
2. Reduce the cost of S3 storage by moving freeze buckets (which are inherently smaller) to S3 Glacier.

To complete the system, a routine to rapidly load thawed buckets to Splunk indexers in classic mode is required.  This would be need for less common, but critical, investigations.  
  
    Indexers                        S3 Storage               Standalone      
                                                             Instance        
┌──────────────────┐                                                         
│                  │                                                         
│  Aged Out Bucket │                                                         
│                  │                                                         
└────────┬─────────┘                                                         
         │                                                                   
  ColdToFrozenDir                                                            
   configuration                                                             
         │                                                                   
┌────────┴─────────┐                                                         
│                  │                                                         
│ Frozen Bucket    │                                                         
│                  │                                                         
└────────┬─────────┘                                                         
         │                    ┌─────────────────┐                            
         │      Copy to       │                 │                            
         └─────────S3─────────┤ Frozen Bucket   │                            
                 Routine      │                 │                            
                              └────────┬────────┘                            
                                       │                                     
                                       │                                     
                         Confirm       │                                     
  Delete from Local─────────&──────────┤                                     
                         Delete        │                                     
                                       │                                     
                                       │                   ┌────────────────┐
                                       │      Thaw         │   Searchable   │
                                       ├──────Routine──────┤     Data       │
                                       │                   │                │
                                       │                   └───────┬────────┘
                                       │                           │         
                                  Frozen Purge                     │         
                                     Routine                    Decomm       
                                       │                        Routine      
                                       │                           │         
                                       │                           │         
                                  Deleted from                     │         
                                      S3                                     
                                                               Instance      
                                                               & Thawed data 
                                                               Deleted       
