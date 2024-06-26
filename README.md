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
```  
    Indexers                        S3 Glacier               Standalone      
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
```
ColdToFrozenDir Configuration - [indexes.conf](https://docs.splunk.com/Documentation/Splunk/9.2.1/Admin/Indexesconf#indexes.conf.spec) configuration within Splunk. This configuration is used instead of coldToFrozenScript to prevent IO wait during S3 communication from impacting Splunk indexer performance. This should follow the format /path/to/some/directory/frozenCache/new/indexname. Where frozenCache is a root directory for frozen files, 'new' is a subdirectory to hold recently frozen files. And indexname should match the index the data is coming from. Such a naming scheme is important when thawing to search. The indexes.conf settings would be something like:  
[main]
coldToFrozenDir = /path/to/directory/frozenCache/new/main

Copy To S3 Routine - Copy and verify receipt of complete data in S3. This routine can be used with a Snowball, or with a direct copy to S3 over the network. Note that if a snowball is used, the local copy will not be removed until an additional confirmation of data being received in S3 located in AWS.

Confirm and Delete - Only delete the local bucket copy when data is verified and available in S3 Glacier. (Snowball does not quality for deletion.)

Thaw Routine - Specify the indexes and date range to be thawed. Poll Glacier for qualifying buckets based on name. Calculate indexer requirements for thaw. Deploy instances, place data in thaw locations, and peer to search heads.

Frozen Purge Routine - Routine to remove aged out buckets based on index name and time range based on bucket name. Permanently deletes data.

Decomm Routine - Destroy instances used to search thawed data.
