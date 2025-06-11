==================
快速上手curator
==================

简介
=====
curator主要包括训练、模拟、选择和标注四个核心板块，一个完整的主动学习流程包括：从当前数据集中训练出一系列模型，之后利用这些模型探索新的样本空间（如MD，MC等方法），然后从模拟轨迹中用特定算法选择特定数量的样本，之后采用DFT进行标注，然后将标注完成的数据重新加入到原数据集中，进行下一轮迭代。

curator支持单独执行以上四种操作，分别由curator-train, curator-simulate, curator-select, curator-label实现；同时也支持全自动化的workflow，仅需定义好curator-workflow文件，即可进行自动的主动学习迭代。

训练
=====
进行训练前，首先需要将标注好的数据转换为 `traj <https://wiki.fysik.dtu.dk/ase/ase/io/trajectory.html>`_ 格式。若数据由VASP得到，可以执行以下命令方便地得到：

.. code-block:: bash

   ase convert OUTCAR dataset.traj

然后准备好训练所需的配置文件config.yaml，以下为实例及常用参数解释：

.. code-block:: bash



.. code-block:: bash

   pip install example-software

使用说明
=========
本软件的使用非常简单，只需运行以下命令：

.. code-block:: bash

   example-software --help

贡献
=====
如果你有兴趣为本软件贡献代码，请参考我们的 `贡献指南 <https://example.com/contributing>`_。

许可证
=======
本软件遵循 MIT 许可证。

