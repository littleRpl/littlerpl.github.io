---
layout:       post
title:        "How can I bypass disk I/O in h5py?"
subtitle:     "Bypass disk I/O and read h5 files in memory"
date:         2019-06-13 12:00:00
author:       "Rpl"
header-style:   "text"
tags:
    - python
    - hdfs
    - 技术
---


>工作需求， 大批量从hdfs数据库中读取h5文件。为了加快读取速度，需要将hdfs数据库中的h5文件直接从内存读取出来，不再经过磁盘I/O读写文件。
>此脚本转载自stackoverflow，点击[此处](https://stackoverflow.com/questions/11588630/pass-hdf5-file-to-h5py-as-binary-blob-string/45900556)查看原文

传统方式下需要先将文件从h5数据库中拉下来，写入文件。然后在从文件中读取数据。有了磁盘I/O的读写操作，速度既慢，流程又麻烦。
本文的思路是直接从数据库中获取二进制的文件数据，然后创建一个虚假的临时文件，绕过磁盘I/O，直接从内存中读取h5文件。

代码如下，可以直接复制使用。
```python
"""HDF5 in memory file reading example."""

try:
    import contextlib
    import os
    import tempfile

    import h5py

    hdf5_data = (
        b'\x89HDF\r\n\x1a\n\x00\x00\x00\x00\x00\x08\x08\x00\x04\x00\x10\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xff\xff\xff\xff\xff'
        b'\xff\xff\xff|\x05\x00\x00\x00\x00\x00\x00\xff\xff\xff\xff\xff\xff'
        b'\xff\xff\x00\x00\x00\x00\x00\x00\x00\x00`\x00\x00\x00\x00\x00\x00'
        b'\x00\x01\x00\x00\x00\x00\x00\x00\x00\x88\x00\x00\x00\x00\x00\x00\x00'
        b'\xa8\x02\x00\x00\x00\x00\x00\x00\x01\x00\x01\x00\x01\x00\x00\x00\x18'
        b'\x00\x00\x00\x00\x00\x00\x00\x11\x00\x10\x00\x00\x00\x00\x00\x88\x00'
        b'\x00\x00\x00\x00\x00\x00\xa8\x02\x00\x00\x00\x00\x00\x00TREE\x00\x00'
        b'\x01\x00\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff\xff'
        b'\xff\x00\x00\x00\x00\x00\x00\x00\x000\x04\x00\x00\x00\x00\x00\x00'
        b'\x08\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00HEAP\x00\x00\x00\x00X'
        b'\x00\x00\x00\x00\x00\x00\x00\x10\x00\x00\x00\x00\x00\x00\x00\xc8\x02'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00test\x00\x00'
        b'\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00H\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x01\x00\x06\x00\x01\x00\x00\x00\x00\x01\x00'
        b'\x00\x00\x00\x00\x00\x01\x00(\x00\x00\x00\x00\x00\x01\x02\x01\x00'
        b'\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00'
        b'\x00\x00\x00\x01\x00\x00\x00\x00\x00\x00\x00\x01\x00\x00\x00\x00\x00'
        b'\x00\x00\x03\x00\x10\x00\x01\x00\x00\x00\x10\x08\x00\x00\x04\x00\x00'
        b'\x00\x00\x00 \x00\x00\x00\x00\x00\x05\x00\x08\x00\x01\x00\x00\x00'
        b'\x02\x02\x02\x01\x00\x00\x00\x00\x08\x00\x18\x00\x01\x00\x00\x00\x03'
        b'\x01x\x05\x00\x00\x00\x00\x00\x00\x04\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x12\x00\x08\x00\x00\x00\x00\x00\x01\x00\x00'
        b'\x00\xed\xf3\xa1Y\x00\x00p\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00SNOD\x01\x00\x01\x00\x08\x00\x00\x00\x00\x00\x00'
        b'\x00 \x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
        b'\x00\x00\x00\x00\x00\x00\x0090\x00\x00'
    )

    file_access_property_list = h5py.h5p.create(h5py.h5p.FILE_ACCESS)
    file_access_property_list.set_fapl_core(backing_store=False)
    file_access_property_list.set_file_image(hdf5_data)

    file_id_args = {
        'fapl': file_access_property_list,
        'flags': h5py.h5f.ACC_RDONLY,
        'name': next(tempfile._get_candidate_names()).encode(),
    }

    h5_file_args = {'backing_store': False, 'driver': 'core', 'mode': 'r'}

    with contextlib.closing(h5py.h5f.open(**file_id_args)) as file_id:
        with h5py.File(file_id, **h5_file_args) as h5_file:
            # h5文件已经跳过磁盘I/O打开了，可在此处填入自己需要的代码
            pass

except:
    print('Something went wrong!')
    raise

else:
    print('It works!! You can read HDF5 in memory!')
```

此脚本是Stack Overflow上的一位作者通过h5py包实现的，虽然传递了一个文件名，但该文件并没有被真正创建，于是绕过了磁盘，从内存中直接读取了文件。

读者可以通过以下链接深入了解脚本代码。
* [https://github.com/h5py/h5py/blob/c2ad0b91f074b5b62d5c10b3970d39ae55b8ec1f/h5py/h5p.pyx#L1171](https://github.com/h5py/h5py/blob/c2ad0b91f074b5b62d5c10b3970d39ae55b8ec1f/h5py/h5p.pyx#L1171)

* [https://github.com/h5py/h5py/pull/680#issue-135939862](https://github.com/h5py/h5py/pull/680#issue-135939862)

* [http://docs.h5py.org/en/latest/high/file.html#file-driver](http://docs.h5py.org/en/latest/high/file.html#file-driver)


***
以下代码是从hdfs数据库拉取文件，并从内存读取,转成dataframe数据。
```python
import os
import tempfile
import contextlib

import h5py
from hdfs import *

# 建立hdfs数据库，客户端连接
client = Client('http://192.168.0.1:50070')

def hdfs_read(hdfs_file='/h5file/2018/1012/001.h5', ):

    with client.read(hdfs_file) as f:
        hdf5_data = f.read()

    file_access_property_list = h5py.h5p.create(h5py.h5p.FILE_ACCESS)
    file_access_property_list.set_fapl_core(backing_store=False)
    file_access_property_list.set_file_image(hdf5_data)

    file_id_args = {
        'fapl': file_access_property_list,
        'flags': h5py.h5f.ACC_RDONLY,
        'name': next(tempfile._get_candidate_names()).encode(),
    }

    h5_file_args = {'backing_store': False, 'driver': 'core', 'mode': 'r'}

    with contextlib.closing(h5py.h5f.open(**file_id_args)) as file_id:
        with h5py.File(file_id, **h5_file_args) as h5_file:
           df = turn_to_df(h5_data)  # 将其转成我需要的dataframe数据

    return df
    
    
# h5数据转成dataframe数据
def turn_to__df(h5_data):
     # if grab == 'data':
            data = h5_file['data_f'].value
            data = data[:, :]
            # elif grab == 'featurelist':
            columns = h5_file['f_axis0'].value
            columns = columns[:]
            columns = columns.tolist()
            columns = [s.decode('utf-8') for s in columns]

            df = pd.DataFrame(data, columns=columns)
                  
```
