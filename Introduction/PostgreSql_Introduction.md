### What is PostgreSQL 
- PostgreSQL is a free and open source object-relational database management system(ORDBMS)
- PostgreSQL began its journey in 1986 as POSTGRES, a research project of the University of California at Berkeley. 
- PostgreSQL is cross platform and runs on many operation systems such as Linux, Windows, OS X , Solaris, FreeBSD.
- PostgreSQL features transaction with atomicity, Consistency, Isolation, Durability (ACID).
- PostgreSQL manages concurrency through multiversion concurrency contril (MVCC).

### What is Page
- Page is a smallest unit of data storage.
- Every Table and Index is stored as an array of pages of fixed size. 
- By Default In PostgreSQL the page size is 8kb.
- We can configure different page size during compiling the server.
- All pages are logically equivalent and any row can stored in any page. 
- Page storage data on the disk, when this page is loaded in the memory its call buffer. 