---
layout: post
title: "Understanding Google&#8217;s File System"
tags: ["Distributed Systems"]
---

Today I read [the original paper](http://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf) about the Google File System (GFS), a system that provided the storage layer for many of Google's applications in the company's early days. The original implementation has reportedly been replaced by a newer version called Colossus, but reading about the original approach was still illuminating and I thought I'd do a quick write up about it.

### Why is/was GFS such a big deal?

The original paper was published in 2003 at SOSP (Symposium on Operating Systems Principles - one of, if not the, best conferences for operating systems research). 

GFS made it onto the program because of how revolutionary it was at the time - the accompanying paper detailed how Google had successfully implemented academic ideas of weak consistency and reliance on a single master controller (more on this later) at tremendous scale in an industry application.

The ultimate goal of GFS was to provide a replicated storage layer (redundant copies of data are kept across many machines) across the commodity level machines in a Google datacenter. The original motivation for developing such a system was to power batch jobs, although the system eventually powered other projects. 

Because GFS was designed for batch jobs, it primarily optimized for appending to, rather than modifying, files. Users of the program were generally writing large files out at once rather than making modifications to specific parts of a file.

### What are the abstractions that power GFS?

At the core of GFS is a concept called **chunks**. Chunks are used to split up files into fixed-size 64MB segments that are then replicated around the datacenter [&#8224;](#footnotes). Chunks are referred to by **chunk handles**, basically unique ids for a chunk. Splitting a large file into many chunks, then replicating those chunks across many machines accomplished two goals: improving performance (as there could now be many readers and writers of a single file), and allowing huge files to exist behind a simple abstraction. 

To make the idea of how this abstraction works more concrete, imagine using a library to open a file on a disk. Behind the scenes, that library now goes out and fetches all of the different pieces of the file you requested from computers all around your datacenter, then provides a transparent way to interact with the stitched together data [&#8224;](#footnotes).

The aforementioned library (called by your user program, the Client) performs fetching and writing operations by interacting with several components of GFS:

- **Master**: The master has a few responsibilties. To start, it is the first point of contact for a client when they want to interact with GFS. In addition to that function, the master is also responsible for communicating with a set of **chunk servers** that host chunks. To perform its functions, the master stores a few tables in RAM:
	- A mapping from filenames to **chunk handles** (chunk handles are basically IDs for chunks). 
	- A mapping from **chunk handles** to a list of the machines that the chunk is on, versioning information about the chunk (a piece of data to help with managing multiple writes to the same chunk), and two pieces of information related to managing writes to that chunk - the primary and the lease. I'll cover the primary and the lease in the next section.
- **Chunk Server**: Chunk servers handle work around writing to and reading from disk. A client starts talking to them after being told to do so by the master.

### How does writing and reading to GFS work?

#### Reading from GFS
To read a file to GFS, a client says to the master, "I would like to read this byte offset in this file", where the file looks like a regular file system path.

The master then receives the request from the client and calculates which chunk corresponds to the associated file and byte offset. Using the chunk handle of the calculated chunk, the master then gets the list of chunk servers that store the aforementioned chunk and provides it to the client. The client then chooses a chunk server, contacting it with the chunk and offset it wants, then is provided with the requested data.

Along the way, the client also caches information about the chunk and the chunkservers it can find that chunk on if it needs to rerequest the chunk.

#### Writing to GFS

Writing (in this case, appending) to files in GFS is significantly more complicated than reading from GFS. 

To start a client, asks the master for a specific file's last chunk (the end of the file is necessary because we are appending). The master then checks its tables for information on that chunk, using the returned chunk handle (the chunk handle is essentially the ID of the chunk). 

The master then inspects two pieces of information that it is storing about each chunk - the primary and lease fields. 

The **primary** is a reference to a chunk server that has been assigned to coordinate writes among chunk servers. This assignment is short lived, and is governed by the expiration of the **lease**. When the lease runs out, the master can assign a new chunk server to coordinate writes. 

If the chunk that a client requests does not have a **primary** assigned, the master assigns one, and increments the version of the data. Incrementing the version number allows the master to keep track of which data is the most recent. If the chunk already has a primary, this step is skipped.

The next step is to transmit information about the primary and secondaries (chunk servers that have the chunk, but aren't the primary) to the client. From there, the client sends the data it wants to write to the respective chunk servers. After all chunk servers have the data, the client tells the primary to write it. The primary chunk server chooses a byte offset in the chunk (whatever the end of the file is), and sends it to all of the secondaries, after which all of them perform the right. 

If the primary and all secondaries write, the client receives a success! If not all secondaries write, the client receives a failure, at which point it needs to recontact the master and repeat the process from the beginning. 
 
### Wrapping up

I [found an interview](https://queue.acm.org/detail.cfm?id=1594206) with one of the engineers who worked on GFS to be fairly interesting. GFS was very successful for the applications it was designed for and reached wide adoption within Google. 

Unfortunately, it didn't scale as well to new use cases for a few reasons. First off, the system used a single master process to store of chunk servers in addition to other metadata. Having all of this information in RAM on a single machine only went so far. 

Another issue that GFS ran into was in storing small files. For example, if a user wanted to store many files smaller than the chunk size, the master needed to store an entry for each file, and allocate the full chunk size on disk. Google ended up working on other systems and making tweaks to GFS to solve this problem (in particular, one of the systems that is discusses is BigTable).

### Footnotes: {#footnotes}

 - <a name="#1">[1]</a> Google's new storage system would try to decrease the chunk size for reasons that I talk about at the end of this post.
- <a name="#2">[2]</a> Whether the data is actually stitched together or not is somewhat of an implementation detail

### References:

- [1] [GFS paper](http://static.googleusercontent.com/media/research.google.com/en//archive/gfs-sosp2003.pdf)
- [2] [MIT Distributed Systems lecture on GFS](https://www.youtube.com/watch?v=EpIgvowZr00&feature=emb_title)
- [3] [Talk about GFS from Firas Abuzaid](https://cs.stanford.edu/~matei/courses/2015/6.S897/slides/gfs.pdf)
