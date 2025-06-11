==================
快速上手curator
==================

简介
=====
curator主要包括训练、模拟、选择和标注四个核心板块，一个完整的主动学习流程包括：从当前数据集中训练出一系列模型，之后利用这些模型探索新的样本空间（如MD，MC等方法），然后从模拟轨迹中用特定算法选择特定数量的样本，之后采用DFT进行标注，然后将标注完成的数据重新加入到原数据集中，进行下一轮迭代。

curator支持单独执行以上四种操作，分别由curator-train, curator-simulate, curator-select, curator-label实现；同时也支持全自动化的workflow，仅需定义好curator-workflow文件，即可进行自动的主动学习迭代。

当前curator支持的MLP模型包括PaiNN, Nequip和MACE, 支持的模拟软件包括LAMMPS和ASE，workflow中支持的DFT计算软件仅包含VASP（CP2K接口正在开发中）。

训练
=====
进行训练前，首先需要将标注好的数据转换为 `traj <https://wiki.fysik.dtu.dk/ase/ase/io/trajectory.html>`_ 格式。若数据由VASP得到，可以执行以下命令方便地得到：

.. code-block:: bash

   ase convert OUTCAR dataset.traj

然后准备好训练所需的配置文件config.yaml， `这里 <https://github.com/cxtt0000/curator/blob/main/docs/source/tutorials/config.yaml>`_  提供了一份样例，以下为常用参数解释：

.. code-block:: bash

   devices: 训练使用GPU数量
   max_epochs: 最大epoch数量
   lr: 学习率
   patience: 收敛判据，在该值的步数内全部满足收敛判据才停止训练
   loss_weight: 目标函数的权重，常用为energy:1, force:200, virial:1
   energy_weight/forces_weight/virial_weight: 同上
   model: representation: _target_: 训练所用模型，后为模型对应的超参数。对于painn模型而言，一般仅需修改num_features，默认为128
   datapath：训练所用数据集路径（绝对路径）
   batch_size：一般经过多次尝试，取刚好能跑满显存的值
   model_path：是否加载现有模型，null为不加载，若需加载则填写模型所在目录的绝对路径。

准备好配置文件后，在当前目录下执行：

.. code-block:: bash

   mq submit "shell:curator-train cfg=config.yaml" -X--gres=gpu:2 -R 96:hgpu2:5d

即可提交训练任务。curator使用myqueue进行任务提交和管理，其中gpu:2代表使用2块GPU（该值需和配置文件中的devices一致），96:hgpu2:5d表示使用96个cpu核，提交到hgpu2队列中，最大时长为5天。

训练期间会产生training.log文件，其中记载了训练的详细信息。同时会产生model_path文件夹，其中储存了训练期间loss最小的文件，会随着训练动态更新。如果配置文件中指定deploy_model: true，则最终还会产生编译好的模型compiled_model.pt。

模拟
=========
curator提供多种方式进行模拟，包括使用定义好的配置文件提交ase、lammps任务，或是使用编译好的模型，通过编写lammps脚本的方式进行模拟。推荐使用最后一种方法，这样具有最高的自由度，能够与一般的lammps任务使用完全一致的参数，仅需修改pair_style和pair_coeff。

首先对上述训练得到的模型进行编译：

.. code-block:: bash

   curator-deploy a.ckpt b.ckpt --target_path compiled_model.pt

其中a.ckpt, b.ckpt为训练产生的model_path下的模型，若指定了多个，则最终编译得到的模型为ensemble model。

然后将编译后的模型放置于lammps任务所在的文件夹中，编写lammps提交脚本in.lammps。若模型为ensemble model，需要指定的有：

.. code-block:: bash

   newton          off # 不进行跨核传输
  
   # Ensemble model使用以下参数
   pair_style      curator uncertainty force_sd
   pair_coeff      * * compiled_model.pt 1 8 # 这里1 8修改为当前初始结构中元素顺序对应的原子序号，需要和in.data中一致。compiled_model.pt替换为实际所用的模型文件名称
   
   neighbor        0.0 bin #不使用skin方法
   neigh_modify    delay 0 every 1 check no

   variable        uncertainty_low  equal 0.05       # low uncertainty threshold
   variable        uncertainty_high equal 0.5        # high uncertainty threshold
   variable        check_interval   equal 1         # interval for checking uncertainty
   variable        curr_uncertainty equal c_f_sd     # variable for fix halt
   variable        check_dump       equal "v_curr_uncertainty < v_uncertainty_low"

   # 如果uncertainty大于uncertainty_high，则停止模拟
   compute         f_sd water uncertainty force_sd
   fix             check_uncertainty all halt 1 v_curr_uncertainty > ${uncertainty_high} error hard message yes 

   # 将uncertainty大于uncertainty_low的结构储存于dump_uncertain.lammpstrj中
   dump            uncertain all custom 1 dump_uncertain.lammpstrj id type x y z vx vy vz fx fy fz
   dump_modify     uncertain sort id append yes every ${check_interval} skip v_check_dump

若模型为单个模型，则无法计算uncertainty，in.lammps文件需按如下方式编写：

.. code-block:: bash

   newton off

   pair_style      curator
   pair_coeff      * * compiled_model.pt 1 8 # 这里1 8修改为当前初始结构中元素顺序对应的原子序号，需要和in.data中一致。compiled_model.pt替换为实际所用的模型文件名称

   neighbor        0.0 bin #不使用skin方法
   neigh_modify    delay 0 every 1 check no

准备好in.lammps, in.data和compiled_model.pt后，使用如下脚本提交任务：

.. code-block:: bash

   #!/bin/bash -l
   #SBATCH --job-name=test
   #SBATCH -p hgpu2
   #SBATCH --nodes=1
   #SBATCH --ntasks=1
   #SBATCH --gpus-per-node=1
   #SBATCH --output=%j.out
   #SBATCH --error=%j.err
   
   # Load necessary modules (if any)
   lammps_modules
   
   # Run nvidia-smi to get GPU information
   
   # Additional GPU queries can be added here
   
   lmp -in in.lammps

选择
=========
curator集成了多种 `选择算法 <https://arxiv.org/abs/2203.09410>`_，通常而言选择默认的Largest cluster maximum distance即可。需要准备好配置文件config.yaml，示例如下：

.. code-block:: yaml

   _convert_: all
   cfg: select.yaml
   seed: 123
   run_path: .
   model_path: # 替换为实际模型所在路径
   - test_train/112/model_path
   - test_train/120/model_path
   - test_train/128/model_path
   - test_train/136/model_path
   - test_train/144/model_path
   device: cuda
   dataset: null
   split_file: null
   train_set: null
   pool_set: # 用于选择的轨迹文件，如有多个则换行依次写
   - ./warning_struct.traj
   batch_size: 500 # 选择后得到的样本数量
   kernel: full-g # 选择算法
   method: lcmd_greedy # 选择算法
   n_random_features: 500
   save_features: false
   save_images: false
   debug: false
   transforms: []
   trainset: ./select.traj # 储存选择得到的结构的文件，如为已有的traj，则会在末尾追加写入


标注
=========


workflow
=========



