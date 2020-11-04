# Integration of callback-based solvers


## Abstract

Bluesky plans facilitate describing standard experiments: e.g. scanning a
Three-dimensional space etc. These plans are straightforward to implement if
the measurement steps to be taken are known in advance.

Some measurements require to execute solvers before a measurement starts or
during a measurement. It could be used for e.g. searching the minimum of
a signal before defining the scan range. Such solvers are typically
based on callbacks (e.g. the ones available in `scipy.optimize`) and can
not be integrated within bluesky plan stubs in a straightforward manner.

Thus it is proposed to develop a module that allows integrating such solvers
within bluesky plan stubs.


## Introduction

For many experiments it is sufficient to take data following a predefined
Plan of single measurement steps. These types of experiments are well
supported by Bluesky plans or plan stubs.

Some experiments require, however, to define or redefine the steps that need to
be executed for obtaining a proper data set. As examples one can consider:

1. Find the range, where a dependent variable changes the sign
2. Define the spacing of the independent variable in such a manner, that
   the spacing is made small in the range, where the independent variable
   changes in a fast manner.

Task (2) is available as adaptive plan ins bluesky. This solver was implemented
as a dedicated plan. Task (1) is typically solved by "scalar root solvers".

Many solvers exist today (e.g. see the solvers in `scipy.optimize`) that expect
a call back function, whose result it tries to optimise. Bluesky plans, however
work by emitting (yielding) messages. Thus, an integration of such solvers in
Bluesky plan stubs is not straight forward. Therefore, it is proposed to develop
a module that facilitates integration of callback-based solvers within bluesky
plan stubs.

Such a module should address the following issues:
1. provide a call back like interface to the solver.
2. provide a generator like interface, so that it can be used by the bluesky run
   engine.
3. Allow controlling, how data are taken during the

### Advantages of this approach

Many solvers exist for different applications and experience has been gained on
their numerical stability, best use cases etc. With such a module all these
different solvers can be integrated in a straightforward manner.


### Alternative approach.

Classical numerical solvers typically assume that the function to be optimised
returns data without noise: i.e. if a function is called twice with the same
input it is expected to return exactly the same value as was returned at the
last call. Thus these solvers
are used to evaluate experimental data, these can fail or get stuck too fast
in a local solution.

In the author's experience the problem describe above can be mitigated by:

* selecting start position, initial step size and stopping criteria in such a
 manner that the measurement error will be ignored by the solver
* using a cache for avoiding duplicate measurements
* using solvers that have been designed to handle measurement errors from the
  beginning.

#### Wrapping solver as IOC


At least the following modules are available for implementing a python based
IOC within an EPICS environment:

* [PyDevice](https://github.com/klemenv/PyDevice)
* [pyCASpy](https://pcaspy.readthedocs.io)
* [pydev](https://mdavidsaver.github.io/pyDevSup/)

Thus, the solver could be implemented in an IOC. The solver would be
integrated in the following manner:

1. The user needs to start the used solver IOC and assign EPICS variables to it.
2. The start variable needs to be set by the user
   (e.g. issuing a `bps.mv` messages).
3. A plan by the user needs to read the callback values from the device
   (e.g. issuing a `bps.trigger_and_read` message).
4. It is the user's responsibility to put the results to the EPICS variable that
   expects the return value.
5. Steps 3 and 4 needs be repeated until the solver signals that it is finished.

This choice is control system dependent. Furthermore, it would leave the user
responsible for making the IOC available. The IOC should take care, that is not
restarted while still evaluating a callback.

### Best practice for any approach

Assumption:

It is assumed that the solver will require to be based on at least 2
Tasks (or threads). One task will be used for providing the call back, while the
other task will be used to issue messages.

Disclaimer:
   Of course it is premature to describe best practixe given that none of the
   proposals have been tested "in the wild". The best practice described
   here is based on author's experience of wrapping
   [GSL](http://gnu.org/software/gsl/) (see [PyGSL](http://pygsl.sf.net))
   next to experiments with [bcib](https://github.com/hz-b/naus/)
   and [naus](https://github.com/hz-b/naus/) (please be warned these are
   not even alpha state software).

The recommendations listed here are based on the concept that consistent
error messages and reporting make code development more straightforward.


1. check that start parameters match the requirements of the solver
    * Reason: exception is raised as soon as the user calls the solver
	* Considering: Some solvers take start parameters and store them. The
	  execution is then delayed until some method is called (e.g. step()).
	  Only then the inconsistency is found out and reported. In a large
	  code base it can be tricky to realize where the offending command
	  has happened.

2. check that the values returned by the callback match the solver
   requirements.
   * Reason: the offending call will not (necessarily) be displayed on
     the traceback.
   * Considering: typical solvers expect float values or arrays of floats
     returned by the callback e.g. root solvers expect a single float while
	 fitting solvers expect an array of floats with shape [n, p].
	 Typical solvers then just report the issue, but no further information is
	 provided. The wrapper could check the return values and report e.g.
	 the plan used, the involved devices or associated metadata.

     This recommendation is based on author's experience that devices return
     "rather strange responses" occasionally when code is developed but rather
     "too often" when software is rolled out. Thus, extra information for the
     "strange responses" could help developers solving device specific issues
     early.


3. check that solver does not timeout before the call times out or deal with
   pending device evaluations when a timeout occurs
   * Reason: once more targeted to simplify user experience
   * Explanation: The solver should expect response within a given time. If not given,
     it could be difficult to find out which call to with device did not
     respond, as the device evaluation is happening in another thread than
     the callback handling.


## Implementation

This module is considered as an "infrastructure style" module. It facilitates the
use case describe above using different threads for processing the callback and
and executing the plan (stub).  Given python's flexible nature
it should simplify the integration of

1. any callback-based solver out there
2. providing an OpenAI compatible environment. Such solvers can be used by
   typical reinforcement learning type solver

### Proposed API


1. User supplied callback
    ```python

        def step_plan_stub(x):
            '''Example of a simple step plan being wrapped by

            This plan is expected to:
	        * yield the messages required for executing the measurement
		* return the value matching to the solver’s expectation
            '''
            yield from bps.mv(mot, x)
            r = yield from bps.trgger_and_read(list(det) + list(motors))
            # extract result
            return res
    ```

    Thus, the user writes a plan stub that serves as callback to the solver, but behaves like
    any other plan stub, as it is yielding messages to 'do something'


2. Plans stub to wrap the solver
    ```python
        def wrap_scale_solver(solver, step_plan, *, log):
            '''
            Args:
                solver :    a solver that evaluates a callback. It is assumed that it
                            sole argument is a callback that takes a single argument (x)
                step_plan : the step plan to execute at each step. It takes a single
                            argument and returns a value. See :func:`step_plan_stub`
                            for an example
                log:        the logger to be used. This logger must be supplied (not
		            compatible with standard bluesky plans		     .
            '''
    ```

    This plan stub hides all the details that are required to set up the different threads,
    handle exceptions, timeouts and so on.

### Simplified implementation

The implementation would then be based on a `bridge`. It "bridges" from one thread to the other.
(Another way to see it would be ADA like rendezvous.
Please correct me, which design pattern would be the correct one).

```python


    def cb_wrapper(bridge, solver, step_plan):
        '''Hides bridge handling details from the user
        '''
        def func(x):
            r = bridge.submit(partial(step_plan_stub, x))
            return r

        try:
            cb(func)
        finally:
            brigde.StopDelegation()


        assert(log is not None)

        def run_inner():
            bridge = setup_threaded_callback_iterator_bridge()

            # Calling for a context manager
            try:
                thread = Thread(target=cb_wrapper, args=[bridbe, cb],
                                name='run optimiser')
                thread.start()

		while True:
                    r = (yield from bridge_plan_stub(bridge, log=log))

            except Exception:
                log.error(f'run_environment: Failed to execute environment {env}')
                raise
            finally:
                log.info(f'run_environment: Finishing processing  {env}')
                # And there is no timeout here
                thread.join()
```

## Argument for yet another module

Such abridge was not found on pypi. It is assumed that its correct
implementation faces several challenges that
are considered not trivial to code properly. Therefore, it is assumed that it is
useful to have a clean implementation available.

These challenges are:

* ensuring that all exceptions that are raised in the plan_stub are caught
* ensuring that the exception is transferred to the callback and appropriately
  handled: e.g. reraising the exception
* ensuring that the traceback stack is reported, logged in a consistent fashion


## Conclusion

Callback based solvers are commonly available. These solvers can be useful to
bluesky users. It is not straightforward to use these solvers within bluesky
plan stubs or other methods. Thus it is proposed to implement a module which
facilitates the integration of such solvers within a separate module.
