Fault Tolerance
===============

This document describes how Ray handles machine and process failures.

Tasks
-----

When a worker is executing a task, if the worker dies unexpectedly, either
because the process crashed or because the machine failed, Ray will rerun
the task (after a delay of several seconds) until either the task succeeds
or the maximum number of retries is exceeded. The default number of retries
is 4.

You can experiment with this behavior by running the following code.

.. code-block:: python

    import numpy as np
    import os
    import ray
    import time

    ray.init(ignore_reinit_error=True)

    @ray.remote(max_retries=1)
    def potentially_fail(failure_probability):
        time.sleep(0.2)
        if np.random.random() < failure_probability:
            os._exit(0)
        return 0

    for _ in range(3):
        try:
            # If this task crashes, Ray will retry it up to one additional
            # time. If either of the attempts succeeds, the call to ray.get
            # below will return normally. Otherwise, it will raise an
            # exception.
            ray.get(potentially_fail.remote(0.5))
            print('SUCCESS')
        except ray.exceptions.RayWorkerError:
            print('FAILURE')


Actors
------

Ray will automatically restart actors that crash unexpectedly.
This behavior is controlled using ``max_restarts``,
which sets the maximum number of times that an actor will be restarted.
If 0, the actor won't be restarted. If -1, it will be restarted infinitely.
When an actor is restarted, its state will be recreated by rerunning its
constructor.
After the specified number of restarts, subsequent actor methods will
raise a ``RayActorError``.
You can experiment with this behavior by running the following code.

.. code-block:: python

    import os
    import ray
    import time

    ray.init(ignore_reinit_error=True)

    @ray.remote(max_restarts=5)
    class Actor:
        def __init__(self):
            self.counter = 0

        def increment_and_possibly_fail(self):
            self.counter += 1
            time.sleep(0.2)
            if self.counter == 10:
                os._exit(0)
            return self.counter

    actor = Actor.remote()

    # The actor will be restarted up to 5 times. After that, methods will
    # raise exceptions. The actor is restarted by rerunning its
    # constructor. Methods that were executing when the actor died will also
    # raise exceptions.
    for _ in range(100):
        try:
            counter = ray.get(actor.increment_and_possibly_fail.remote())
            print(counter)
        except ray.exceptions.RayActorError:
            print('FAILURE')
