# 数据集基类（BaseDataset）

## 基本介绍

算法库中的数据集类负责在训练/测试过程中为模型提供输入数据，OpenMMLab 下各个算法库中的数据集有一些共同的特点和需求，比如需要高效的内部数据存储格式，需要支持数据集拼接、数据集重复采样等功能。

因此 **MMEngine** 实现了一个数据集基类（BaseDataset）并定义了一些基本接口，且基于这套接口实现了一些数据集包装（DatasetWrapper）。OpenMMLab 算法库中的大部分数据集都会满足这套数据集基类定义的接口，并使用统一的数据集包装。

数据集基类的基本功能是加载数据集信息，这里我们将数据集信息分成两类，一种是元信息 (meta information)，代表数据集自身相关的信息，有时需要被模型或其他外部组件获取，比如在图像分类任务中，数据集的元信息一般包含类别信息 `classes`，因为分类模型 `model` 一般需要记录数据集的类别信息；另一种为数据信息 (data information)，在数据信息中，定义了具体样本的文件路径、对应标签等的信息。除此之外，数据集基类的另一个功能为将数据送入数据流水线（data pipeline）中，进行数据预处理。

### 数据标注文件规范

为了统一不同任务的数据集接口，便于多任务的算法模型训练，OpenMMLab 制定了 **OpenMMLab 2.0 数据集格式规范**， 数据集标注文件需符合该规范，数据集基类基于该规范去读取与解析数据标注文件。如果用户提供的数据标注文件不符合规定格式，用户应该将其转化为规定格式才能使用 OpenMMLab 的算法库基于该数据标注文件进行算法训练和测试。

OpenMMLab 2.0 数据集格式规范规定，标注文件必须为 `json` 或 `yaml`，`yml` 或 `pickle`，`pkl` 格式；标注文件中存储的字典必须包含 `metadata` 和 `data_infos` 两个字段。其中 `metadata` 是一个字典，里面包含数据集的元信息；`data_infos` 是一个列表，列表中每个元素是一个字典，该字典定义了一个原始数据（raw data），每个原始数据包含一个或若干个训练/测试样本。

以下是一个 JSON 标注文件的例子（该例子中每个原始数据只包含一个训练/测试样本）:

```json

{
    'metadata':
        {
            'classes': ('cat', 'dog'),
            ...
        },
    'data_infos':
        [
            {
                'img_path': "xxx/xxx_0.jpg",
                'img_label': 0,
                ...
            },
            {
                'img_path': "xxx/xxx_1.jpg",
                'img_label': 1,
                ...
            },
            ...
        ]
}
```

同时假设数据存放路径如下：

```text
data
├── annotations
│   ├── train.json
├── train
│   ├── xxx/xxx_0.jpg
│   ├── xxx/xxx_1.jpg
│   ├── ...
```

### 数据集基类的初始化流程

数据集基类的初始化流程如下：

1. 获取数据集的元信息，元信息有三种来源，优先级从高到低为：

- `__init__()` 方法中用户传入的 `meta` 字典；改动频率最高，因为用户可以在实例化数据集时，传入该参数；

- 类属性 `BaseDataset.META` 字典；改动频率中等，因为用户可以改动自定义数据集类中的类属性 `BaseDataset.META`；

- 标注文件中包含的 `metadata` 字典；改动频率最低，因为标注文件一般不做改动。

    如果三种来源中有相同的字段，优先级最高的来源决定该字段的值；

2. 构建数据流水线（data pipeline），用于数据预处理与数据准备；

3. 读取与解析满足 OpenMMLab 2.0 数据集格式规范的标注文件，该步骤中会有 `parse_annotations()` 抽象方法，该抽象方法负责解析标注文件里的每个原始数据；

4. 过滤无用数据，比如不包含标注的样本等；

5. 采样数据，比如只取前 10 个样本参与训练/测试；

6. 序列化全部样本，以达到节省内存的效果，详情请参考[节省内存](#节省内存)。

数据集基类是一个抽象类，它有且只有一个抽象方法 `parse_annotations()` ，`parse_annotations()` 定义了将标注文件里的一个原始数据处理成一个或若干个训练/测试样本的方法。因此对于自定义数据集类，用户必须要实现 `parse_annotations()` 方法。

### 数据集基类提供的接口

与 `torch.utils.data.Dataset` 类似，数据集初始化后，支持 `__getitem__` 方法，用来索引数据，以及 `__len__` 操作获取数据集大小，除此之外，OpenMMLab 的数据集基类主要提供了以下接口来访问具体信息：

- `meta` 返回元信息，返回值为字典

- `get_data_info(idx)` 返回指定 `idx` 的样本全量信息，返回值为字典

- `__getitem__(idx)` ：返回指定 `idx` 的样本经过 pipeline 之后的结果（也就是送入模型的数据），返回值为字典

- `__len__()` 返回数据集长度，返回值为整数型

## 使用数据集基类自定义数据集类

在了解了数据集基类的初始化流程与提供的接口之后，就可以基于数据集基类自定义数据集类，如上所述，数据集基类是一个抽象类，它有且只有一个抽象方法 `parse_annotations()`，因此用户必须在自定义数据集类中实现该方法。以下是一个使用数据集基类来实现某一具体数据集的例子。

```python
import os.path as osp

from mmengine.data import BaseDataset


class ToyDataset(BaseDataset):

    # 以上面标注文件为例，在这里 raw_data_info 代表 `data_infos` 对应列表里的某个字典：
    # {
    #    'img_path': "xxx/xxx_0.jpg",
    #    'img_label': 0,
    #    ...
    # }
    def parse_annotations(self, raw_data_info):
        data_info = raw_data_info
        img_prefix = self.data_prefix.get('img', None)
        if img_prefix is not None:
            data_info['img_path'] = osp.join(
                img_prefix, data_info['img_path')
        return data_info

```

### 使用自定义数据集类

在定义了数据集类后，就可以通过如下配置实例化 `ToyDataset`：

```python
pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline)
```

同时可以使用数据集类提供的对外接口访问具体的样本信息：

```python
toy_dataset.meta
# dict(classes=('cat', 'dog'))

toy_dataset.get_data_info(0)
# {
#     'img_path': "data/train/xxx/xxx_0.jpg",
#     'img_label': 0,
#     ...
# }

len(toy_dataset)
# 2

toy_dataset[0]
# dict(img=xxx, label=0)
```

经过以上步骤，可以了解基于数据集基类如何自定义新的数据集类，以及如何使用自定义数据集类。

### 自定义视频的数据集类

在上面的例子中，标注文件的每个原始数据只包含一个训练/测试样本（通常是图像领域）。如果每个原始数据包含若干个训练/测试样本（通常是视频领域），则只需保证 `parse_annotations()` 的返回值为 `list[dict]` 即可：

```python
from mmengine.data import BaseDataset


class ToyVideoDataset(BaseDataset):

    # raw_data_info 仍为一个字典，但它包含了多个样本
    def parse_annotations(self, raw_data_info):
        data_infos = []

        ...

        for ... :

            data_info = dict()

            ...

            data_infos.append(data_info)

        return data_infos

```

`ToyVideoDataset` 使用方法与 `ToyDataset` 类似，在此不做赘述。

## 数据集基类的其它特性

数据集基类还包含以下特性：

### 懒加载（lazy init）

在数据集类实例化时，需要读取并解析标注文件，因此会消耗一定时间。然而在某些情况比如预测可视化时，往往只需要数据集类的元信息，可能并不需要读取与解析标注文件。为了节省这种情况下数据集类实例化的时间，数据集基类支持懒加载：

```python
pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline,
    # 在这里传入 lazy_init 变量
    lazy_init=True)
```

当 `lazy_init=True` 时，`ToyDataset` 的初始化方法只执行了[数据集基类的初始化流程](#数据集基类的初始化流程)中的 1、2 步骤，此时 `toy_dataset` 并未被完全初始化，因为 `toy_dataset` 并不会读取与解析标注文件，只会设置数据集类的元信息（`meta`）。

自然的，如果之后需要访问具体的数据信息，可以手动调用 `toy_dataset.full_init()` 接口来执行完整的初始化过程，在这个过程中数据标注文件将被读取与解析。调用 `get_data_info(idx)`, `__len__()`, `__getitem__()` 接口也会自动地调用 `full_init()` 接口来执行完整的初始化过程（仅在第一次调用时，之后调用不会重复地调用 `full_init()` 接口）：

```python
# 完整初始化
toy_dataset.full_init()

# 初始化完毕，现在可以访问具体数据
len(toy_dataset) # 2
toy_dataset[0] # dict(img=xxx, label=0)
```

**注意：**

通过直接调用 `__getitem__()` 接口来执行完整初始化会带来一定风险：如果一个数据集类首先通过设置 `lazy_init=True` 未进行完全初始化，然后直接送入数据加载器（dataloader）中，在后续读取数据的过程中，不同的 worker 会同时读取与解析标注文件，虽然这样可能可以正常运行，但是会消耗大量的时间与内存。**因此，建议在需要访问具体数据之前，提前手动调用 `full_init()` 接口来执行完整的初始化过程。**

以上通过设置 `lazy_init=True` 未进行完全初始化，之后根据需求再进行完整初始化的方式，称为懒加载。

### 节省内存

在具体的读取数据过程中，数据加载器（dataloader）通常会起多个 worker 来预取数据，多个 worker 都拥有完整的数据集类备份，因此内存中会存在多份相同的 `data_infos`，为了节省这部分内存消耗，数据集基类可以提前将 `data_infos` 序列化存入内存中，使得多个 worker 可以共享同一份 `data_infos`，以达到节省内存的目的。

数据集基类默认是将 `data_infos` 序列化存入内存，也可以通过 `serialize_data` 变量（默认为 `True`）来控制是否提前将 `data_infos` 序列化存入内存中：

```python
pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline,
    # 在这里传入 serialize_data 变量
    serialize_data=False)
```

上面例子不会提前将 `data_infos` 序列化存入内存中，因此不建议在使用数据加载器开多个 worker 加载数据的情况下，使用这种方式实例化数据集类。

## 数据集基类包装

除了数据集基类，MMEngine 也提供了若干个数据集基类包装：`ConcatDataset`, `RepeatDataset`, `ClassBalancedDataset`。这些数据集基类包装同样也支持懒加载与拥有节省内存的特性。

### ConcatDataset

MMEngine 提供了 `ConcatDataset` 包装来拼接多个数据集，使用方法如下：

```python
from mmengine.data import ConcatDataset

pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset_1 = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline)

toy_dataset_2 = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='val/'),
    ann_file='annotations/val.json',
    pipeline=pipeline)

toy_dataset_12 = ConcatDataset(datasets=[toy_dataset_1, toy_dataset_2])

```

上述例子将数据集的 `train` 部分与 `val` 部分合成一个大的数据集。

### RepeatDataset

MMEngine 提供了 `RepeatDataset` 包装来重复采样某个数据集若干次，使用方法如下：

```python
from mmengine.data import RepeatDataset

pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline)

toy_dataset_repeat = RepeatDataset(dataset=toy_dataset, times=5)

```

上述例子将数据集的 `train` 部分重复采样了 5 次。

### ClassBalancedDataset

MMEngine 提供了 `ClassBalancedDataset` 包装，来基于数据集中类别出现频率，重复采样相应样本。

**注意：**

`ClassBalancedDataset` 包装假设了被包装的数据集类支持 `get_cat_ids(idx)` 方法，`get_cat_ids(idx)` 方法返回一个列表，该列表包含了 `idx` 指定的 `data_info` 包含的样本类别，使用方法如下：

```python
from mmengine.data import BaseDataset, ClassBalancedDataset

class ToyDataset(BaseDataset):

    def parse_annotations(self, raw_data_info):
        data_info = raw_data_info
        img_prefix = self.data_prefix.get('img', None)
        if img_prefix is not None:
            data_info['img_path'] = osp.join(
                img_prefix, data_info['img_path')
        return data_info

    # 必须支持的方法，需要返回样本的类别
    def get_cat_ids(self, idx):
        data_info = self.get_data_info(idx)
        return [int(data_info['img_label'])]

pipeline = [
    dict(type='xxx', ...),
    dict(type='yyy', ...),
    ...
]

toy_dataset = ToyDataset(
    data_root='data/',
    data_prefix=dict(img='train/'),
    ann_file='annotations/train.json',
    pipeline=pipeline)

toy_dataset_repeat = ClassBalancedDataset(dataset=toy_dataset, oversample_thr=1e-3)

```

上述例子将数据集的 `train` 部分以 `oversample_thr=1e-3` 重新采样，具体地，对于数据集中出现频率低于 `1e-3` 的类别，会重复采样该类别对应的样本，否则不重复采样，具体采样策略请参考 `ClassBalancedDataset` API 文档。