Train Person-ReID on a Custom Dataset
-------------------------------------

Here we describe how to finetune Hailo's person-reid network with your own custom dataset.

Prerequisites
^^^^^^^^^^^^^


* docker (\ `installation instructions <https://docs.docker.com/engine/install/ubuntu/>`_\ )
* nvidia-docker2 (\ `installation instructions <https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html>`_\ )

**NOTE:**\  In case you are using the Hailo Software Suite docker, make sure to run all of the following instructions outside of that docker.

Environment Preparations
^^^^^^^^^^^^^^^^^^^^^^^^

#. 
   **Build the docker image:**

   .. code-block::

      
      cd hailo_model_zoo/hailo_models/reid/
      docker build  --build-arg timezone=`cat /etc/timezone` -t person_reid:v0 .
      

   | the following optional arguments can be passed via --build-arg:


   * | ``timezone`` - a string for setting up timezone. E.g. "Asia/Jerusalem"
   * | ``user`` - username for a local non-root user. Defaults to 'hailo'.
   * | ``group`` - default group for a local non-root user. Defaults to 'hailo'.
   * | ``uid`` - user id for a local non-root user.
   * | ``gid`` - group id for a local non-root user.

     | This command will build the docker image with the necessary requirements using the Dockerfile that exists in this directory.

#. 
   **Start your docker:**

   .. code-block::

      
      docker run --name "your_docker_name" -it --gpus all --ipc=host -v /path/to/local/drive:<span
      val="docker_vol_path">/path/to/docker/dir person_reid:v0
      


   * ``docker run`` create a new docker container.
   * ``--name <your_docker_name>`` name for your container.
   * ``-it`` runs the command interactively.
   * ``--gpus all`` allows access to all GPUs.
   * ``--ipc=host`` sets the IPC mode for the container.
   * ``-v /path/to/local/data/dir:/path/to/docker/data/dir`` maps ``/path/to/local/data/dir`` from the host to the container. You can use this command multiple times to mount multiple directories.
   * ``personface_detection:v0`` the name of the docker image.

Finetuning and exporting to ONNX
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^


#. | **Train the network on your dataset**
   | Once the docker is started, you can train the person-reid model on your custom dataset. We recommend following the instructions from the torchreid repo that can be found in `here <https://kaiyangzhou.github.io/deep-person-reid/user_guide.html#use-your-own-dataset>`_.


   * 
     Insert your dataset as described in `use-your-own-dataset <https://kaiyangzhou.github.io/deep-person-reid/user_guide.html#use-your-own-dataset>`_.

   * 
     Start training on your dataset starting from our pre-trained weights in ``models/repvgg_a0_person_reid_512.pth`` or ``models/repvgg_a0_person_reid_2048.pth`` (you can also download it from `512-dim <https://hailo-model-zoo.s3.eu-west-2.amazonaws.com/HailoNets/MCPReID/reid/repvgg_a0_person_reid_512/2022-04-18/repvgg_a0_person_reid_512.pth>`_ & `2048-dim <https://hailo-model-zoo.s3.eu-west-2.amazonaws.com/HailoNets/MCPReID/reid/repvgg_a0_person_reid_2048/2022-04-18/repvgg_a0_person_reid_2048.pth>`_\ ) - to do so, you can edit the added yaml ``configs/repvgg_a0_hailo_pre_train.yaml`` and take a look at the examples in `torchreid <https://github.com/KaiyangZhou/deep-person-reid>`_.

     .. code-block::

         
         python scripts/main.py  --config-file configs/repvgg_a0_hailo_pre_train.yaml
         

#. | **Export to ONNX**
   | Export the model to ONNX using the following command:

   .. code-block::

      
      python scripts/export.py --model_name <model_name> --weights /path/to/model/pth
      

----

Compile the Model using Hailo Model Zoo
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

| In case you exported to onnx based on one of our provided RepVGG models, you can generate an HEF file for inference on Hailo-8 from your trained ONNX model. In order to do so you need a working model-zoo environment.
| Choose the model YAML from our networks configuration directory, i.e. ``hailo_model_zoo/cfg/networks/repvgg_a0_person_reid_512.yaml`` (or 2048), and run compilation using the model zoo:

.. code-block::

   
   hailomz compile --ckpt repvgg_a0_person_reid_512.onnx --calib-path /path/to/calibration/imgs/dir/ --yaml path/to/repvgg_a0_person_reid_512.yaml --start-node-names name1 name2 --end-node-names name1
   


* | ``--ckpt`` - path to  your ONNX file.
* | ``--calib-path`` - path to a directory with your calibration images in JPEG/png format
* | ``--yaml`` - path to your configuration YAML file.
* | ``--start-node-names`` and ``--end-node-names`` - node names for customizing parsing behavior (optional).
* | The model zoo will take care of adding the input normalization to be part of the model.

.. note::
  - Since it’s an Hailo model, calibration set must be manually supplied. 
  - On `market1501.yaml <https://github.com/hailo-ai/hailo_model_zoo/blob/master/hailo_model_zoo/cfg/base/market1501.yaml>`_,
    change ``preprocessing.input_shape`` if changed on retraining
  
  More details about YAML files are presented `here <../../../docs/YAML.rst>`_.