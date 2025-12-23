# YOPO 项目解析

本文对仓库的主要模块、数据流以及关键脚本进行梳理，便于快速理解和上手。

## 仓库结构与角色
- **YOPO/**：学习型规划器核心。包含训练/测试脚本、网络结构、损失函数以及轨迹/状态变换逻辑。
- **Simulator/**：传感器与环境仿真（深度图、点云），支持 CUDA 加速；`rosrun sensor_simulator sensor_simulator_cuda` 启动。
- **Controller/**：SO3 控制器与动力学仿真，提供姿态/位置控制模式；`roslaunch so3_quadrotor_simulator simulator_attitude_control.launch` 启动。
- **hardware/**：无人机硬件清单与结构件。

## 数据流与训练流程
1. **数据采集：** 在 `Simulator` 中执行 `rosrun sensor_simulator dataset_generator` 生成深度图与 `pose-*.csv`，默认保存至 `YOPO/dataset`。
2. **数据组织：** 每个子目录存放一组 `.png` 深度图和对应的 `pose-{idx}.csv`（位置+四元数），路径由 `YOPO/config/traj_opt.yaml` 的 `dataset_path` 键指定。
3. **训练：** `YOPO/train_yopo.py` 调用 `policy.YopoTrainer`，核心组件：
   - `policy/yopo_dataset.py` 随机采样速度、加速度、目标并读取深度图。
   - `policy/yopo_network.py`（ResNet 骨干 + 轨迹头）输出末端状态与评分。
   - `loss/loss_function.py` 计算平滑、碰撞、目标、加速度等成本。
   - 关键超参由 `config/traj_opt.yaml` 给出（轨迹数量、FOV、损失权重、速度/加速度范围等）。
4. **模型保存：** 日志与权重保存在 `YOPO/saved/YOPO_<trial>/epoch*.pth`。

## 推理与联调流程
1. 启动 `Controller`（姿态控制模式）和 `Simulator`（建议 CUDA 版本）。
2. 在 `YOPO` 环境下运行 `python test_yopo_ros.py --trial=<n> --epoch=<e>` 载入预训练模型，订阅仿真话题，输出轨迹与评分。
3. 通过 RViz（`rviz -d YOPO/yopo.rviz`）观察深度图、规划轨迹，可用 `2D Nav Goal` 设定目标。

## 关键文件速览
- `YOPO/config/traj_opt.yaml`：规划/训练超参（速度上限、轨迹数量、FOV、损失权重等）。
- `YOPO/policy/primitive.py`、`policy/state_transform.py`：运动原语与坐标变换。
- `YOPO/test_yopo_ros.py`：ROS 环境下的推理入口。
- `Controller/src/readme.md`、`Simulator/src/readme.md`：各自的构建与运行说明。

如需使用 AMP、调整轨迹速度或更换数据集，可直接修改上述配置和脚本。
