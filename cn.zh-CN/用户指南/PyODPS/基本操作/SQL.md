# SQL {#concept_hqh_5vf_cfb .concept}

PyODPS支持ODPS SQL的查询，并可以读取执行的结果。

## SQL {#section_aqj_rb1_hfb .section}

`execute_sql` 或者 `run_sql` 方法的返回值是 [运行实例](cn.zh-CN/用户指南/PyODPS/基本操作/任务实例.md#) 。

**说明：** 并非所有在 ODPS Console 中可以执行的命令都是 ODPS 可以接受的 SQL 语句。 在调用非 DDL / DML 语句时，请使用其他方法，例如 GRANT / REVOKE 等语句请使用`run_security_query`方法，PAI 命令请使用 `run_xflow` 或 `execute_xflow`方法。

## 执行SQL {#section_ntm_54m_cfb .section}

```
>>> o.execute_sql('select * from dual')  #  同步的方式执行，会阻塞直到SQL执行完成
>>>
>>> instance = o.run_sql('select * from dual')  # 异步的方式执行
>>> print(instance.get_logview_address())  # 获取logview地址
>>> instance.wait_for_success()  # 阻塞直到完成
```

## 设置运行参数 {#section_jr3_w4m_cfb .section}

有时，我们在运行时，需要设置运行时参数，我们可以通过设置 `hints`参数，参数类型是dict。

```
>>> o.execute_sql('select * from pyodps_iris', hints={'odps.sql.mapper.split.size': 16})
```

我们可以对于全局配置设置sql.settings后，每次运行时则都会添加相关的运行时参数。

```
>>> from odps import options
>>> options.sql.settings = {'odps.sql.mapper.split.size': 16}
>>> o.execute_sql('select * from pyodps_iris')  # 会根据全局配置添加hints
```

## 读取SQL执行结果 {#section_uv2_1pm_cfb .section}

运行 SQL 的 instance 能够直接执行 `open_reader`的操作，一种情况是SQL返回了结构化的数据。

```
>>> with o.execute_sql('select * from dual').open_reader() as reader:
>>>     for record in reader:
>>>         # 处理每一个record
```

另一种情况是 SQL 可能执行的比如 `desc`，这时通过 `reader.raw`属性取到原始的SQL执行结果。

```
>>> with o.execute_sql('desc dual').open_reader() as reader:
>>>     print(reader.raw)
```

如果 `options.tunnel.use_instance_tunnel == True`，在调用 open\_reader 时，PyODPS 会默认调用 Instance Tunnel， 否则会调用旧的 Result 接口。如果你使用了版本较低的 MaxCompute 服务，或者调用 Instance Tunnel 出现了问题，PyODPS 会给出警告并自动降级到旧的 Result 接口，可根据警告信息判断导致降级的原因。如果 Instance Tunnel 的结果不合预期， 请将该选项设为 False。在调用 open\_reader 时，也可以使用 `tunnel`参数来指定使用何种结果接口，例如

```
>>> # 使用 Instance Tunnel
>>> with o.execute_sql('select * from dual').open_reader(tunnel=True) as reader:
>>>     for record in reader:
>>>         # 处理每一个record
>>> # 使用 Results 接口
>>> with o.execute_sql('select * from dual').open_reader(tunnel=False) as reader:
>>>     for record in reader:
>>>         # 处理每一个record
```

PyODPS 默认不限制能够从 Instance 读取的数据规模。对于受保护的 Project，通过 Tunnel 下载数据受限。此时， 如果 options.tunnel.limit\_instance\_tunnel 未设置，会自动打开数据量限制。此时，可下载的数据条数受到 Project 配置限制， 通常该限制为 10000 条。如果你想要手动限制下载数据的规模，可以为 open\_reader 方法增加 limit 选项， 或者设置 options.tunnel.limit\_instance\_tunnel = True 。

如果你所使用的 MaxCompute 只能支持旧 Result 接口，同时你需要读取所有数据，可将 SQL 结果写入另一张表后用读表接口读取 （可能受到 Project 安全设置的限制）。

## 设置alias {#section_h2h_qpm_cfb .section}

有时在运行时，比如某个UDF引用的资源是动态变化的，我们可以alias旧的资源名到新的资源，这样免去了重新删除并重新创建UDF的麻烦。

```
from odps.models import Schema

myfunc = '''\
from odps.udf import annotate
from odps.distcache import get_cache_file

@annotate('bigint->bigint')
class Example(object):
    def __init__(self):
        self.n = int(get_cache_file('test_alias_res1').read())

    def evaluate(self, arg):
        return arg + self.n
'''
res1 = o.create_resource('test_alias_res1', 'file', file_obj='1')
o.create_resource('test_alias.py', 'py', file_obj=myfunc)
o.create_function('test_alias_func',
                  class_type='test_alias.Example',
                  resources=['test_alias.py', 'test_alias_res1'])

table = o.create_table(
    'test_table',
    schema=Schema.from_lists(['size'], ['bigint']),
    if_not_exists=True
)

data = [[1, ], ]
# 写入一行数据，只包含一个值1
o.write_table(table, 0, [table.new_record(it) for it in data])

with o.execute_sql(
    'select test_alias_func(size) from test_table').open_reader() as reader:
    print(reader[0][0])
```

```
2
```

```
res2 = o.create_resource('test_alias_res2', 'file', file_obj='2')
# 把内容为1的资源alias成内容为2的资源，我们不需要修改UDF或资源
with o.execute_sql(
    'select test_alias_func(size) from test_table',
    aliases={'test_alias_res1': 'test_alias_res2'}).open_reader() as reader:
    print(reader[0][0])
```

```
3
```

## 在交互式环境执行 SQL {#section_xgv_vpm_cfb .section}

在 ipython 和 jupyter 里支持[使用 SQL 插件的方式运行 SQL](cn.zh-CN/用户指南/PyODPS/交互体验增强/命令行增强.md#) ，且支持 [参数化查询](cn.zh-CN/用户指南/PyODPS/交互体验增强/IPython增强.md#)。

## 设置 biz\_id {#section_p3z_1sm_cfb .section}

在少数情形下，可能在提交 SQL 时，需要同时提交 biz\_id，否则执行会报错。此时，你可以设置全局 options 里的 biz\_id。

```
from odps import options

options.biz_id = 'my_biz_id'
o.execute_sql('select * from pyodps_iris')
```

