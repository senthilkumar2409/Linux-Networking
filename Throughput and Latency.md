## Database I/O issue - 
 https://aws.amazon.com/blogs/database/capture-and-diagnose-i-o-bottlenecks-on-amazon-rds-for-sql-server/

## Throughput
 
 **Server** 
   - The total number of request processed at a time, Like how many client requests, transactions, or megabytes the server handles per second or minute.
   - How efficiently the server application handles concurrent requests.
   - **Requests per second (RPS):** Counts web or API calls.**Transactions per second (TPS):** Counts database or payment actions.**Megabytes per second (MBps):** Counts data sent or received
    
 **Network** 
   - Throughput is the actual amount of data successfully transferred between two points (like a client and server, or across a network link) in a given amount of time — usually measured in bits per second (bps), or larger units like Kbps, Mbps, Gbps.

## Latency
   - A time taken to process a request
   - Latency is the time delay between sending a request and receiving the first response — essentially, how long it takes for data to travel from source to destination (and often back again). It's usually measured in milliseconds (ms).


## IOPS

 - IOPS (Input/Output Operations Per Second) is a basic unit of measure. It tells you how many read or write tasks a storage device or hard drive can do in one second.
 - A higher IOPS score means the drive is faster and responds quicker.
 
 Key ConceptsRead and Write:
 - Read means pulling data off the drive. Write means saving data to the drive.
 - Why it matters: High IOPS helps databases and virtual machines run fast. Traditional hard drives (HDDs) have low IOPS, while solid-state drives (SSDs) and NVMe drives have very high IOPS

<img width="812" height="235" alt="image" src="https://github.com/user-attachments/assets/159412b0-0b22-4c89-8926-cf0d94c13c6f" />

## indexing in database:

 - A database index is a supplementary data structure used to **speed up data retrieval operations** by minimizing the number of disk accesses required to locate records.
 - Think of it exactly like an index at the back of a textbook; instead of flipping through every single page to find a specific topic, you look up the topic alphabetically in the index and jump directly to the correct page number.
   
### what happen if we don't have index?
 - Without an index, the database must perform a Full Table Scan for every single query.
 - This means the database engine has to look at every single row, from the very first to the very last, to see if it matches your search criteria.

## What Happens to the Database

* Severe Slowness: A search that takes milliseconds with an index can take minutes without one as your data grows.
* High CPU Usage: The database server wastes massive amounts of processor power reading endless rows of data.
* Disk Bottlenecks: The system is forced to constantly read data from the hard drive, slowing down the entire server.
* Application Crashes: Queries may take so long that they hit a timeout limit and fail completely.
* User Frustration: Websites or apps loading data will spin endlessly, causing a terrible user experience.

 ### The Ultimate Trade-Off:

 <img width="882" height="465" alt="image" src="https://github.com/user-attachments/assets/f22db202-d661-4573-9564-ba4a2d870f53" />

