---
title:  "ARCHITECTURE"
---

<b><font id="topic">PREFACE</font></b><br>

Sophia (<i>adj. sophisticated</i>) database and its architecture was born as a
result of research and rethinking primary alghorithmic constraints associated with a
getting popular <a href="http://en.wikipedia.org/wiki/Log-structured_file_system">Log-file based</a>
data structures such as <a href="http://en.wikipedia.org/wiki/Log-structured_merge-tree">LSM-tree</a>
it's variations based on <a href="http://en.wikipedia.org/wiki/Fractional_cascading">Fractional Cascading</a> ideas and a
<a href="http://en.wikipedia.org/wiki/B-tree">B-Tree</a>.

Those limitations are slow read's, meta-data bloat and unexpected
latency hops.

Most log-based databases (or LSM-alike specifically) trend to
organize their file storage as a collection of sorted files which are
periodically merged.

Thus, without using some sort of key filtering scheme (like <a href="http://en.wikipedia.org/wiki/Bloom_filter">bloom-filter</a>),
to find a key, it has to traverse all files which can take up
to <i>O(files_count * log(file_size))</i> in the worst case.

Sophia was specifically designed to improve the situation and get fast reads while
still benefit from append-only design.

<b><font id="topic">DESIGN</font></b>

Sophia's architecture combines a region in-memory index with a
in-memory key index.

A region index is represented as an ordered range of regions with their
min and max keys and a latest on-disk reference.
Regions never overlap.

These regions have the same semantical meaning as the B-Tree pages, but are
designed differently. They do not have a tree structure or any internal page-to-page
relationships and thus no meta-data overhead (specifically to append-only B-Tree).

A single region on-disk holds keys with values. And as
a B-tree page, region has it's maximum key count. Regions are uniquely
identified by region id number, by which they can be tracked in future.

A key index is very similar to LSM zero-level (memtable), but has a different
key lifecycle. All modifications first get into the index and hold until they will
be explicitly removed by merger.

The database update lifecycle is organized in terms of epochs.
Epoch lifetime is determined in terms of key updates. When the update counter
reaches an epoch's watermark number then the Rotation event happen.

Each epoch, depending to its state, is associated with a single log file
or database file. Before getting added to the in-memory index, modifications are
first written to the epoch's write-ahead log.

On each rotation event:<br>
<i>a</i>. current epoch, which is called 'live', is marked as 'transfer' and a <br>
new 'live' epoch is created (new log file)<br>
<i>b</i>. create new and swap current in-memory key index<br>
<i>c</i>. merger thread is being woken up

The merger thread is the core part that is responsible for region merging
and garbage collecting of a old regions and older epochs. On wakeup, the merger
thread iterates through list of epochs marked as 'transfer' and starts
the merge procedure.

The merge procedure has the following steps:

<i>1.</i> create new database file for the latest 'transfer' epoch<br>
<i>2.</i> fetch any keys from the in-memory index that associated with
a single<br> destination region<br>
<i>3.</i> for each fetched key and origin region start the merge and
write a new<br> region to the database file<br>
<i>4.</i> on each completed region
(current merged key count is less or equal to<br> max region key count):<br>
<i>a.</i> allocate new split region for region index, set min and max<br>
<i>b.</i> first region always has id of origin destination region<br>
<i>c.</i> link region and schedule for future commit<br>
<i>5.</i> on origin region update complete:<br>
<i>a.</i> update destination region index file reference to the current
epoch<br> and insert split regions<br>
<i>b.</i> remove keys from key index<br>
<i>6.</i> start step (2) until there is no updates left<br>
<i>7.</i> start garbage collector<br>
<i>8.</i> database synced with disk and if everything went well, remove all<br>
'transfer' epochs (log files) and gc'ed databases<br>
<i>9.</i> free index

The garbage collector has a simple design.

All that is needed is to track an epoch's total region count and a count of
transfered regions during merge procedure. Thus, if some older epoch database has fewer
than 70% (or any other changeable factor) live regions they just get copied to current
epoch database file and the old one is being deleted.

On database recovery, Sophia tracks and builds an index of pages from
the youngest epochs (biggest numbers) down to the oldest. Log files are
being replayed and epochs are marked as 'transfer'.

Sophia has been evaluated as having the following complexity
(in terms of disk accesses):

<i><b>set:</b></i> worst case is a <i>O(1)</i> append-only key write + in-memory index insert

<i><b>get:</b></i> worst case is a <i>O(1)</i> random region read, which itself do <i>amortized O(log region_size)</i> key compares +
      in-memory key index search + 
      in-memory region search

<i><b>range:</b></i> range queries are very fast due to the fact that each iteration
needs to compare no more that two keys without search them, and
access through mmaped database. Roughly complexity can be equally
evaluated as having to sequentially read a mmaped file.      



