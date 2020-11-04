# List of clues

These topics are not yet worked out to be put into the proposal. Thesee
are kept as collection of thoughts, ideas or todo points.

## Implementation considerations

* Single process versus multiprocessing
* communication layer
* latency
* Timeout in executed plan stubs
* yielding checkpoints during calculation
* propagating stop calls to objects if required
* Keeping thread alive until it returns so that missing traceback messages can be resolved?
* Presenting bluesky a status object with a watcher implemented intercepting other
  status objects?
* "Book keeping device" for recording solver progress ?

## Other implementation options

* Use bluesky server using zmq
    * subscribe to messages
    * insert new message in the queue as required

## Miscellaneous



### Questions to address

1. Range check:

    1. do not target support in first versions.
    2. At a later stage, encapsulate
      standard solvers and assume that these solvers could then be augmented with
      metadata (signals) that would reflect user's assumptions were the solution
      should be found. (Alternative: use constrained solvers)
