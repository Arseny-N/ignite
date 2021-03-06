Quickstart
==========

Welcome to Ignite quickstart guide that just gives essentials of getting a project up and running.

In several lines you can get your model training and validating:

Code
----

.. code-block:: python

    from ignite.engines import Events, create_supervised_trainer, create_supervised_evaluator

    model = Net()
    train_loader, val_loader = get_data_loaders(train_batch_size, val_batch_size)
    optimizer = torch.optim.SGD(model.parameters(), lr=0.01, momentum=0.8))
    loss = torch.nn.NLLLoss()

    trainer = create_supervised_trainer(model, optimizer, loss)
    evaluator = create_supervised_evaluator(model,
                                            metrics={
                                                'accuracy': CategoricalAccuracy(),
                                                'nll': Loss(loss)
                                            })

    @trainer.on(Events.ITERATION_COMPLETED)
    def log_training_loss(trainer):
        print("Epoch[{}] Loss: {:.2f}".format(trainer.state.epoch, len(train_loader), trainer.state.output))

    @trainer.on(Events.EPOCH_COMPLETED)
    def log_validation_results(trainer):
        evaluator.run(val_loader)
        metrics = evaluator.state.metrics
        print("Validation Results - Epoch: {}  Avg accuracy: {:.2f} Avg loss: {:.2f}"
              .format(trainer.state.epoch, metrics['accuracy'], metrics['nll']))

    trainer.run(train_loader, max_epochs=100)


Complete code can be found in the file `examples/mnist.py <https://github.com/pytorch/ignite/blob/master/examples/mnist.py>`_.

Explanation
-----------

Now let's break up the code and review it in details. In the first 4 lines we define our model, training and validation
datasets (as :class:`torch.utils.data.DataLoader`), optimizer and loss function:

.. code-block:: python

    model = Net()
    train_loader, val_loader = get_data_loaders(train_batch_size, val_batch_size)
    optimizer = torch.optim.SGD(model.parameters(), lr=0.01, momentum=0.8))
    loss = torch.nn.NLLLoss()

Next we define trainer and evaluator engines. The main component of Ignite is the :class:`Engine`, an abstraction over your
training loop. Getting started with the engine is easy, the constructor only requires one things:

- `update_function`: a function which is passed the engine and a batch and it passes data through and updates your model

Here we are using helper methods :meth:`create_supervised_trainer` and :meth:`create_supervised_evaluator`:

.. code-block:: python

    trainer = create_supervised_trainer(model, optimizer, loss)
    evaluator = create_supervised_evaluator(model,
                                            metrics={
                                                'accuracy': CategoricalAccuracy(),
                                                'nll': Loss(loss)
                                            })

However, we could also define trainer and evaluator using :class:`Engine`. If we look into the code of
:meth:`create_supervised_trainer` and :meth:`create_supervised_evaluator`, we can observe a pattern:

.. code-block:: python

    def create_engine(*args, **kwargs):

        def _update(engine, batch):
            # Update function logic
            pass

        return Engine(_update)

And update functions of the trainer and evaluator are simply:

.. code-block:: python

    def _update(engine, batch):
        model.train()
        optimizer.zero_grad()
        x, y = _prepare_batch(batch, cuda)
        y_pred = model(x)
        loss = loss_fn(y_pred, y)
        loss.backward()
        optimizer.step()
        return loss.data.cpu()[0]

    def _inference(engine, batch):
        model.eval()
        x, y = _prepare_batch(batch, cuda, volatile=True)
        y_pred = model(x)
        return to_tensor(y_pred, cpu=not cuda), to_tensor(y, cpu=not cuda)

Note that the helper function :meth:`create_supervised_evaluator` to create an evaluator accepts an
argument `metrics`:

.. code-block:: python

    metrics={
        'accuracy': CategoricalAccuracy(),
        'nll': Loss(loss)
    }

where we define two metrics: *categorical accuracy* and *loss* to compute on validation dataset. More information on
metrics can be found at :doc:`metrics`.


The most interesting part of the code snippet is adding event handlers. :class:`Engine` allows to add handlers on
various events that fired during the run. When an event is fired, attached handlers (functions) are executed. Thus, for
logging purposes we added a function to be executed after every iteration:

.. code-block:: python

    @trainer.on(Events.ITERATION_COMPLETED)
    def log_training_loss(engine):
        print("Epoch[{}] Loss: {:.2f}".format(engine.state.epoch, len(train_loader), engine.state.output))

or equivalently without the decorator

.. code-block:: python

    def log_training_loss(engine):
        print("Epoch[{}] Loss: {:.2f}".format(engine.state.epoch, len(train_loader), engine.state.output))

    trainer.add_event_handler(Events.ITERATION_COMPLETED, log_training_loss)

When an epoch ends we want to run model validation, therefore we attach another handler to the trainer on epoch complete
event:

.. code-block:: python

    @trainer.on(Events.EPOCH_COMPLETED)
    def log_validation_results(engine):
        evaluator.run(val_loader)
        metrics = evaluator.state.metrics
        print("Validation Results - Epoch: {}  Avg accuracy: {:.2f} Avg loss: {:.2f}"
              .format(engine.state.epoch, metrics['accuracy'], metrics['nll']))


.. Note ::

   Function :meth:`add_event_handler` (as well as :meth:`on` decorator) also accepts optional `args`, `kwargs` to be passed
   to the handler. For example:

   .. code-block:: python

      trainer.add_event_handler(Events.ITERATION_COMPLETED, log_training_loss, train_loader)


Finally, we start the engine on the training dataset and run it during 100 epochs:

.. code-block:: python

    trainer.run(train_loader, max_epochs=100)
