:orphan:

.. _model_init:

########################
Efficient initialization
########################

Here are common use cases where you should use :meth:`~lightning.fabric.fabric.Fabric.init_module` to avoid major speed and memory bottlenecks when initializing your model.


----


**************
Half-precision
**************

Instantiating a ``nn.Module`` in PyTorch creates all parameters on CPU in float32 precision by default.
To speed up initialization, you can force PyTorch to create the model directly on the target device and with the desired precision without changing your model code.

.. code-block:: python

    fabric = Fabric(accelerator="cuda", precision="16-true")

    with fabric.init_module():
        # models created here will be on GPU and in float16
        model = MyModel()

The larger the model, the more noticeable is the impact on

- **speed:** avoids redundant transfer of model parameters from CPU to device, avoids redundant casting from float32 to half precision
- **memory:** reduced peak memory usage since model parameters are never stored in float32


----


***********************************************
Loading checkpoints for inference or finetuning
***********************************************

When loading a model from a checkpoint, for example when fine-tuning, set ``empty_init=True`` to avoid expensive and redundant memory initialization:

.. code-block:: python

    with fabric.init_module(empty_init=True):
        # creation of the model is fast
        # and depending on the strategy allocates no memory, or uninitialized memory
        model = MyModel()

    # weights get loaded into the model
    model.load_state_dict(checkpoint["state_dict"])


.. warning::
    This is safe if you are loading a checkpoint that includes all parameters in the model.
    If you are loading a partial checkpoint (``strict=False``), you may end up with a subset of parameters that have uninitialized weights, unless you handle them accordingly.


----


********************************************
Model-parallel training (FSDP and DeepSpeed)
********************************************

When training sharded models with :doc:`FSDP <model_parallel/fsdp>` or DeepSpeed, using :meth:`~lightning.fabric.fabric.Fabric.init_module` is necessary in most cases because otherwise model initialization gets very slow (minutes) or (and that's more likely) you run out of CPU memory due to the size of the model.

.. code-block:: python

    # Recommended for FSDP and DeepSpeed
    with fabric.init_module(empty_init=True):
        model = GPT3()  # parameters are placed on the meta-device

    model = fabric.setup(model)  # parameters get sharded and initialized at once

.. note::
    Empty-init is experimental and the behavior may change in the future.
    For FSDP on PyTorch 2.1+, it is required that all user-defined modules that manage parameters implement a ``reset_parameters()`` method (all PyTorch built-in modules have this too).
