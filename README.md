# Overview
This directory contains a modified version of the 'ezca' library
(version 20020802):

 - thread-safeness added (mutex allows only one thread into ezca at
   a time)
 - enable preemptive callbacks with EPICS 3.14
 - bugfix: GETNELEM should set *wp->inp
 - fixed some signedness warnings
 - added 'ezcaAbort()' to abort polling loops from a signal handler.
 - added 'ezcaPollCbInstall()' to let the user install a callback
   to poll for an abort condition while pending for CA activity.
 - added channel management (i.e. channels that are not found are 
   removed from the cache; added ezcaClearChannel() and ezcaPurge()
   to allow for explicitely removing cache entries and disconnect
   the respective channels).

MEMORY MANAGEMENT NOTE:

Ezca uses two dyamically managed objects 'struct channel' and
'struct work'. The former are associated with a CA cid and are
accessible via a hash table entry.

```
hash_table
(PV name)
__________
__________        |--------------|
__________------->|struct channel|
__________        |              |
                  |          CID |---------> CA internal object
                  |--------------|
```

channel nodes are allocated after successful ca_search_and_connect
and are released after successful ca_clear_channel(). Only if
ca_clear_channel() fails, the corresponding channel node is leaked.
When a channel node is released/deallocated, its hash table entry
is removed.

Management of channel nodes is a little more involved, however, 
since multiple work nodes (see below) can reference the same
channel node. A channel node may only go away after the last
referring work node releasing its reference.

The latter, i.e., 'work nodes' are (among other purposes) used
to pass information to CA callback routines and to hold information
about grouped EZCA work requests. While processing grouped work
(during execution of ezcaEndGroupWithReport ONLY, the list of 
work nodes holds pointers to channel nodes needed by the work
requests. These references are acquired at entry of
ezcaEndGroupWithReport and are released prior to exiting that routine.

```
|---------|
|work node|
|---------|           ------------------
|channel  | --------> | struct channel |
|---------|     ----> |  REFCNT: 2     |
|work type|     |     ------------------
|---------|     |
|next  ------   |
|---------| |   |
            |   |
     -------|   |
     |          |
     V          |
|---------|     |
|work node|     |
|---------|     |     ------------------
|channel  | ----+---> | struct channel |
|---------|     |     |  REFCNT: 1     |
|work type|     |     ------------------
|---------|     |
|next  ------   |
|---------| |   |
            |   |
     -------|   |
     |          |
     V          |
|---------|     |
|work node|     |
|---------|     |
|channel  | -----
|---------| 
|work type|
|---------|
|next    NULL
|---------|
```

The reference count is needed because ezcaEndGroupWithReport
can discover the need to destroy a channel node. It may only
do so when processing the last work node referencing the
channel in question.

Pointers to work nodes are passed to CA callback routines.
Such nodes must not be reused or deallocated while the
callback is outstanding. Instead, when EZCA detects that
a callback has not executed timely, its work node is marked
'trashed' and is put on a 'Discarded' list. Should the
callback still run eventually, it will recycle the work
node.

However, there are cases where CA effectively cancels
outstanding callbacks (e.g. 'virtual channel disconnect'
/ server disconnect). The current EZCA implementation
currently LEAKS the corresponding work nodes.

Possible recovery algorithm (NOT IMPLEMENTED YET):

Trashed work nodes could have their 'cp' field set
to the channel node corresponding to the outstanding
callback. A garbage collector or the connection event
handler could gather all trashed work nodes of a channel,
destroy the channel and recycle the work nodes.

Due to the asynchronous nature of this scenario, the
implementation is likely to be complex and error-prone.
Since trashed work nodes are not considered to be a
very frequent phenomenon, we feel that the suffered
memory loss is acceptable.

Till Straumann, 2002-2004

# Build 
This project can be build as follows:

```bash
cat > configure/RELEASE <<EOF
EPICS_BASE=/usr/local/epics/base
EOF

make
```

(Replace `/usr/local/epics/base` with the location of your epics installation)