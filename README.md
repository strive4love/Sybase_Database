# bcp
bcp provides a convenient and high-speed method for transferring data between a database table or view and an operating system file. bcp can read or write files in a wide variety of formats. When copying in from a file, bcp inserts data into an existing database table; when copying out to a file, bcp overwrites any previous contents of the file. The utility is located in:
(UNIX) $SYBASE/$SYBASE_OCS/bin/bcp  
(UNIX) $SYBASE/$SYBASE_OCS/bin/bcp_r(for parallel bcp)  
(Windows) %SYBASE%\%SYBASE_OCS%\bin\bcp.exe 

# parallel copy and locks
Starting many current parallel bcp sessions may cause Adaptive Server to run out of locks.
When you copy in to a table, bcp acquires an exclusive intent lock on the table, and either page or row locks, depending on the locking scheme. If you are copying in very large tables, and especially if you are performing simultaneous copies into a partitioned table, this can require a very large number of locks.To avoid running out of locks: Set the number of locks configuration parameter high enough, or Use the -b batchsize bcp flag to copy smaller batches. If you do not use the -b flag, the entire copy operation is treated as a single batch.
