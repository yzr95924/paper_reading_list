---
typora-copy-images-to: ../paper_figure
---
Sliding Look-Back Window Assisted Data Chunk Rewriting for Improving Deduplication Restore Performance
------------------------------------------
|           Venue            |       Category       |
| :------------------------: | :------------------: |
| FAST'19 | Deduplication Restore |
[TOC]

## 1. Summary
### Motivation of this paper
- Motivation
The data generated by deduplication is stored in data chunks or data containers, but the restore process is rather slow due to
> data fragmentation: data chunks are scattered
> read amplification: the size of data being read is larger than the size of data being restored

This paper focuses on reducing the data fragmentation and read amplification of container-based deduplication system.
> improve data chunk locality
> make a better trade-off between the deduplication ratio and the number of required container reads (compared with capping-FAST'13)
> reducing the number of container reads is a major task for restore performance improvement.


- Container-based deduplication system
Since the size of a data chunk is rather small
> storing individual data chunk directly cannot efficiently utilize the storage bandwidth for storage systems with low random read/write performance. (HDD)
> typically accumulates a number of data chunks in a *container* before writing them out together.


- The limitations of capping (FAST'13)
  - analyzes each segment in isolation without considering the contents of the previous or following segments.
  - the deduplication ration cannot be guaranteed via capping (not consider deduplication ratio as an optimization objective in capping scheme)
  - apply the same cap to each segment


- Main challenge
it is difficult to make decisions on which duplicate chunks need to be rewritten with the minimum reduction of the deduplication ratio.

### Sliding Look-back window
- Data rewrite
The decision to rewrite a duplicate data chunk has to be made during the deduplication process instead of being done at restore process like **caching**.
> avoiding the need to read these duplicate chunks from other old containers.

- Flexible container referenced count based design (FCRC)

1. Main idea: apply varied capping levels for different segments
> further reduce container reads 
> achieve even fewer data chunk rewrites

2. how to decide the "capping level" for different segments?
Instead of using a fixed capping level as the selection threshold, it uses a value of CNRC (*the number of data chunks in the segment that belongs to an old container*).
> rewrite duplicate data chunks from old containers that have CNRCs lower than the threshold.

The actual capping level is decided by
> the threshold $T_{cnrc}$
> the distribution of CNRCs of these segments

Two bounds for $T_{cnrc}$
1. bound for deduplication ratio reduction
a targeted deduplication ratio reduction limit $x%$ 
> bound the number of rewrite chunks 


2. bound for container reads:
a targeted number of container reads $Cap$ in one segment

Use those two bounds to determine the $T_{cnrc}$.

- Sliding look-back window (LWB)
In both the capping and FCRC schemes, the decision to rewrite duplicate data chunks near the segment partition boundaries may have issues.
> existing wasted rewrite chunks (since restore cache)

The rewrite decisions made with the statistics only from the current segment are less accurate.


The LBW acts as a recipe cache that maintains the metadata entries of data chunks in the order covered by the LBW in the byte stream.
> metadata entry:
> 1. chunk metadata
> 2. offset in the byte stream
> 3. container ID/address 
> 4. the offset in the container 

With both **past** and **future** information in the LBW, a more accurate decision can be made.

- Rewrite selection policy for LWB
  - use **two** criteria of restore process to adjust threshold and make more accurate rewrite decisions.
  - cache-effective range: how to maintain LBW with the size compatible with the cache-effective range
  - container read efficiency

The whole process of rewrite selection:
1. Step 1 (move LBW)
When LBW moves forward for one container size
> a container size of data chunks (added container) will be added to the front of the LBW 
> one container size of data chunks (evicted container) will be removed from the end of the LBW

2. Step 2 (process the added container)
classify the data chunks into three categories:
> unique chunks
> non-rewrite chunks (duplicate data chunks that will not be rewritten)
> candidate chunks (duplicate data chunks that may be rewritten)

3. Step 3 (update metadata entries)
update the metadata entries of data chunks in the added container
> identify candidate chunks and write them to rewrite candidate cache

4. Step 4 (recalculate $CNRC_{lbw}$)
recalculate the $CNRC_{lbw}$ of old containers that contain the data chunks in the rewrite candidate cache
> reclassify these data chunks 

5. Step 5 (rewrite)
rewrite the remaining candidate chunks and update metadata entries, write to the recipe persistently

6. Step 6 (adjust)
To make better trade-off between deduplication ratio reduction and the number of container reads
> adjust at the end of each cycle


### Implementation and Evaluation
- Evaluation
Speed factor: MB/container-read, the mean size data being restored (MB) per container read to indicate the amount of data that can be restored by one container read on average.
> the container I/O time dominates the whole restore time (in HDD)

Trace: FSL
select six types of traces in FSL

> each trace contains 10 full backup snapshots

1. Baseline method 
For rewrite:
> normal deduplication with no rewrite 
> capping scheme (Capping)
> flexible container referenced count based scheme (FCRC)
> sliding look-back windows scheme (LBW)

For restore cache:
> forward assembly area (FAA)
> adaptive look-ahead window chunk based caching (ALACC)


2. Metrics
> Deduplication ratio vs. Speed factor
> Restore performance 

## 2. Strength (Contributions of the paper)
- rewrite scheme
  - a flexible container-based referenced count based rewrite scheme
  - sliding look-back window based design

- Good experiment
  - compare with many state-of-art schemes


## 3. Weakness (Limitations of the paper)
- Does not show a complete system with proposed scheme
  - the real performance of LBW is not clear



## 4. Future Works
1. Use container or not use container?
- not use container
This paper mentions that some deduplication systems *directly store individual data chunks to the persistent storage*.
> HYDRAstor, iDedup, Dmdedup and ZFS

- use container
some backup products pack a number of data chunks (compression may be applied) in one I/O unit, and data in the same container are written out and read in together.
> Veritas and Data Domain
> benefit from the high sequential I/O performance and good chunk locality.


2. Adaptive to different workload characteristics
One start-point of this paper is the performance of previous schemes is closely related to the workload characteristics.
> Insight: the motivation to propose an adaptive method

3. Future work
how to design a scheme to adaptively change LBW size 
> more interlligent rewrite policies

how to combine garbage collection with the rewrite design