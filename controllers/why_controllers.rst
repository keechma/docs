Why Controllers?
================

Every frontend framework tries to answer the same question - what is the
best way to manage the application state. There are many approaches -
MVC, MVVM, Flux, Redux… - and Keechma also has it’s own.

Each of this approaches is giving the answers to the following
questions:

-  How to communicate with the rest of the world (calling an API)?
-  How to respond to user interactions (mouse clicks, key presses…)?
-  How to mutate the application state?

Keechma apps implement this kind of code in controllers. Controllers are
a place for all the dirty, impure parts of your app code, and they act
as a bridge between your (pure) domain code and code that has side
effects (storing a user on the server).

How are controllers different?
------------------------------

While philosophically close to Redux actions and reducers, Keechma
controllers differ significantly in the implementation.

-  Keechma controllers have enforced lifecycle
-  Keechma controllers are route driven
-  Keechma controllers can implement a long-running process that can
   react to commands

Drivers of change
-----------------

Changes in the application state happen for a few different reasons:

1. Page reload
2. Route change
3. User action

Keechma treats page reload and route change as tectonic changes - a lot
(or all) of the data in the application state will probably change when
one of these happen. User actions are more of an incremental change,
they will probably affect a small amount of data (for instance, the user
might favorite a post - this is a small, incremental change to the
application state).

Keechma controllers have their lifecycles controlled and enforced by the
URL. Each controller implements a ``param`` function which tells the
controller manager if that controller should be running or not.
Controller Manager (internal part of Keechma) has a set of rules which
determine what should happen when the route changes. Whenever the URL
changes, Controller Manager will do the following:

1. It will call the params function of all registered controllers
2. It will compare the returned value to the previous value (returned on
   the previous URL change)
3. Based on the returned value it will do the

   1. If the previous value was nil and the current value is nil it
      won’t do a thing
   2. If the previous value was nil and the current value is not nil it
      will start the controller
   3. If the previous value was not nil and the current value is nil it
      will stop the controller
   4. If the previous value was not nil and the current value is not nil
      but those values are same, it won’t do a thing
   5. If the previous value was not nil and the current value is not nil
      but those values are different it will restart the controller

Controller manager ensures that the same controllers will always run for
the same URL - it doesn’t matter if it’s a route change or a full page
reload. This makes reasoning about the application state easier, you can
treat every route change as if it was a full page reload. This was
inspired by the React’s way of reasoning where you don’t care how the
DOM is changed, you can mentally treat it as a full re-render.

Minimal layer of abstraction
----------------------------

Redux and similar frameworks model state changes as a combination of
actions and reducers.

    The only way to change the state tree is to emit an action, an
    object describing what happened. To specify how the actions
    transform the state tree, you write pure reducers.

    `Redux documentation <http://redux.js.org/>`__

While I like the simplicity of this approach, I feel that it’s pushing
the state management complexity to the application layer. If you model
every state change as a (synchronous) action, every interaction that
talks to the outer world will require multiple actions. This makes the
flow hard to follow.

Instead of abstracting that kind of code, Keechma gives you complete
control of **how** and **when** you change the application state.
Controllers get full (read / write) access to the application state, and
you can use any approach that fits your application.

Here’s an example of a non-standard controller:

.. code-block:: clojure

    (defrecord Controller []
      controller/IController
      (params [_ route]
        ;; This controller is active only on the order-history page
        (when (= (get-in route [:data :page]) "order-history")
          true))
      (start [this params app-db]
        ;; When the controller is started, load the order history
        (controller/execute this :load-order-history)
        app-db)
      (handler [this app-db-atom in-chan out-chan]
        ;; When the controller is started connect to the websocket.
        ;; This way we can receive the messages when something changes
        ;; and update the application state accordingly.
        ;;
        ;; connect-socketio function returns the function that can be
        ;; used to disconnect from the websocket.
        (let [disconnect (connect-socketio in-chan)]
          (go (loop []
                (let [[command args] (<! in-chan)]
                  (case command
                    ;; When the controller is started, load the order-history
                    :load-order-history (load-order-history app-db-atom)
                    ;; When we get the order-created command from the websocket,
                    ;; create a new order
                    :order-created (order-created app-db-atom args)
                    ;; When we get the order-updated command from the websocket,
                    ;; update the order in the entity-db
                    :order-updated (order-updated app-db-atom args)
                    ;; When we get the order-removed command from the websocket,
                    ;; remove the item from the entity-db. This will automatically
                    ;; remove it from any list that references it
                    :order-removed (order-removed app-db-atom args)
                    ;; Disconnect from the websocket
                    :disconnect (disconnect)
                    nil)
                  (when command (recur)))))))
      (stop [this params app-db]
        ;; When the controller is stopped, send the command to disconnect from
        ;; the websocket and remove any data this controller has loaded.
        (controller/execute this :disconnect)
        (edb/remove-collection app-db :orders :history)))

`Source
Code <https://github.com/keechma/example-place-my-order/blob/66cd3138897f72f9e683dae2e562610c55cd8984/client/src/client/controllers/order.cljs#L43-L73>`__

This controller contains logic for a data source that receives updates
over a websocket. Here’s what’s going on:

1. Controller will be started when the route’s page attribute is
   ``order-history``
2. On controller start, it will load the order history from server
3. On controller start, it will connect to a websocket and listen to
   events (``connect-socketio`` function returns a function that
   disconnects a websocket connection).
4. On controller stop, it will disconnect itself from the websocket and
   remove any loaded data from the application state

Important thing is that all of this functionality lives in the same
place, and you can easily figure out how it works. There is no need to
jump around and play the event ping pong.

Abstractions on top of controllers
----------------------------------

Low level of abstraction is great because it doesn’t force you to fit
your problem into the approach that is implemented by the framework. The
bad thing about the low level of abstraction is that you have a lot of
boilerplate code to solve the simple stuff.

This is why the pipelines were introduced. You can read the full blog
post about them
`here <https://keechma.com/news/introducing-keechma-toolbox-part-1-pipelines/>`__,
but in a nutshell - they exist to make the simple problems easy to
solve.

Pipelines embrace the asynchronous nature of frontend development while
allowing you to keep the related code grouped together. Let’s take a
look at an example that is familiar: You want to load some data from the
server, and you want to let the user know what is the status of the
request. You also want to handle any errors that might happen:

.. code-block:: clojure

    (pipeline! [value app-db]
        (pp/commit! (assoc app-db :articles-status :loading))
        (load-articles-from-server)
        (pp/commit! (-> app-db
                      (assoc :articles-status :loaded)
                      (assoc :articles value)))
       (rescue! [error]
         (pp/commit! (assoc app-db :articles-status :error))))

This approach was inspired by the `Railway Oriented
Programming <https://fsharpforfunandprofit.com/rop/>`__ talk, and the
nice thing about it is that it was possible to implement a system like
this because controllers give you a full access to application state.
Pipelines are not a core Keechma feature, they are implemented in a
separate library.

Conclusion
----------

Controllers give you a full control over your application. They don’t
presume that you can fit your problem into any pattern or way of
thinking. Their abstraction level is intentionally low, and you have a
complete access to the application state. This makes it possible to
solve non standard, specific problems with them. When you need an easy
way to handle standard problems (like data loading, or user interaction)
- use pipelines.
