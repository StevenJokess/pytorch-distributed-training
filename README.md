# pytorch-distributed-training
Distribute Dataparallel (DDP) Training on Pytorch

## Features
* Easy to study DDP training
* You can directly copy this code for a quick start
* Learning Notes Sharing:
  - [Basic Theory]()
  - [Pytorch Gradient Accumulation](https://github.com/rentainhe/pytorch-distributed-training/blob/master/tutorials/1.%20Gradient%20Accumulation.md)
  - [More Details of DDP Training](https://github.com/rentainhe/pytorch-distributed-training/blob/master/tutorials/2.%20DDP%20Training%20Skills.md)
  - [Accelerate-on-Accelerate DDP Training Tricks]()

### Usage 单机多卡
#### 1. 获取当前进程的index
pytorch可以通过torch.distributed.lauch启动器，在命令行分布式地执行.py文件, 在执行的过程中会将当前进程的index通过参数传递给python
```python
import argparse
parser = argparse.ArgumentParser()
parser.add_argument('--local_rank', default=-1, type=int,
                    help='node rank for distributed training')
args = parser.parse_args()
print(args.local_rank)
```
#### 2. 定义 main_worker 函数 
主要的训练流程都写在main_worker函数中，main_worker需要接受三个参数（最后一个参数optional）: 
```python
def main_worker(local_rank, nprocs, args):
    training...
```
- local_rank: 接受当前进程的rank值，在一机多卡的情况下对应使用的GPU号
- nprocs: 进程数量
- args: 自己定义的额外参数

main_worker,相当于你每个进程需要运行的函数（每个进程执行的函数内容是一致的，只不过传入的local_rank不一样）

#### 3. main_worker函数中的整体流程
main_worker函数中完整的训练流程
```python
import torch
import torch.distributed as dist
import torch.backends.cudnn as cudnn
def main_worker(local_rank, nprocs, args):
    args.local_rank = local_rank
    # 分布式初始化，对于每个进程来说，都需要进行初始化
    cudnn.benchmark = True
    dist.init_process_group(backend='nccl', init_method='tcp://ip:port', world_size=nprocs, rank=local_rank)
    # 模型、损失函数、优化器定义
    model = ...
    criterion = ...
    optimizer = ...
    # 设置进程对应使用的GPU
    torch.cuda.set_device(local_rank)
    model.cuda(local_rank)
    # 使用分布式函数定义模型
    model = model = torch.nn.parallel.DistributedDataParallel(model, device_ids=[local_rank])
    
    # 数据集的定义，使用 DistributedSampler
    mini_batch_size = batch_size / nprocs # 手动划分 batch_size to mini-batch_size
    train_dataset = ...
    train_sampler = torch.utils.data.distributed.DistributedSampler(train_dataset)
    trainloader = torch.utils.data.DataLoader(train_dataset, batch_size=mini_batch_size, num_workers=..., pin_memory=..., 
                                              sampler=train_sampler)
    
    test_dataset = ...
    test_sampler = torch.utils.data.distributed.DistributedSampler(test_dataset)
    testloader = torch.utils.data.DataLoader(train_dataset, batch_size=mini_batch_size, num_workers=..., pin_memory=..., 
                                             sampler=test_sampler) 
    
    # 正常的 train 流程
    for epoch in range(300):
       model.train()
       for batch_idx, (images, target) in enumerate(trainloader):
          images = images.cuda(non_blocking=True)
          target = target.cuda(non_blocking=True)
          ...
          pred = model(images)
          loss = loss_function(pred, target)
          ...
          optimizer.zero_grad()
          loss.backward()
          optimizer.step()
```

#### 4. 定义main函数
```python
import argparse
import torch
parser = argparse.ArgumentParser(description='PyTorch ImageNet Training')
parser.add_argument('--local_rank', default=-1, type=int, help='node rank for distributed training')
parser.add_argument('--batch_size','--batch-size', default=256, type=int)
parser.add_argument('--lr', default=0.1, type=float)

def main_worker(local_rank, nprocs, args):
    ...

def main():
    args = parser.parse_args()
    args.nprocs = torch.cuda.device_count()
    # 执行 main_worker
    main_worker(args.local_rank, args.nprocs, args)

if __name__ == '__main__':
    main()
```

#### 5. Command Line 启动
```bash
CUDA_VISIBLE_DEVICES=0,1,2,3 python -m torch.distributed.launch --nproc_per_node=4 distributed.py
```

- `--ip=str`, e.g `--ip='10.24.82.10'` 来指定本机的ip地址
- `--port=int`, e.g `--port=23456` 来指定启动端口号

参数说明:
- --nnodes 表示机器的数量
- --node_rank 表示当前的机器
- --nproc_per_node 表示每台机器上的进程数量

参考 [distributed.py](https://github.com/rentainhe/pytorch-distributed-training/blob/master/distributed.py)

#### 6. torch.multiprocessing 
使用`torch.multiprocessing`来解决进程自发控制可能产生问题，这种方式比较稳定，推荐使用
```python
import argparse
import torch
import torch.multiprocessing as mp

parser = argparse.ArgumentParser(description='PyTorch ImageNet Training')
parser.add_argument('--local_rank', default=-1, type=int, help='node rank for distributed training')
parser.add_argument('--batch_size','--batch-size', default=256, type=int)
parser.add_argument('--lr', default=0.1, type=float)

def main_worker(local_rank, nprocs, args):
    ...

def main():
    args = parser.parse_args()
    args.nprocs = torch.cuda.device_count()
    # 将 main_worker 放入 mp.spawn 中
    mp.spawn(main_worker, nprocs=args.nprocs, args=(args.nprocs, args))

if __name__ == '__main__':
    main()
```

参考 [distributed_mp.py](https://github.com/rentainhe/pytorch-distributed-training/blob/master/distributed_mp.py) 启动方式如下:
```bash
$ CUDA_VISIBLE_DEVICES=0,1,2,3 python distributed_mp.py
```

- `--ip=str`, e.g `--ip='10.24.82.10'` 来指定本机的ip地址
- `--port=int`, e.g `--port=23456` 来指定启动端口号


## Implemented Work
参考的文章如下（如果有文章没有引用，但是内容差不多的，可以提issue给我，我会补上，实在抱歉）：
- [Pytorch: DDP系列](https://zhuanlan.zhihu.com/p/178402798)
- [分布式训练](https://zhuanlan.zhihu.com/p/98535650)
- [分布式训练（理论篇）](https://zhuanlan.zhihu.com/p/129912419)
- [DistributedSampler的问题](https://www.zhihu.com/question/67209417/answer/1017851899)
- I learned from this [repo](https://github.com/tczhangzhi/pytorch-distributed), and want to make it easier and cleaner.
