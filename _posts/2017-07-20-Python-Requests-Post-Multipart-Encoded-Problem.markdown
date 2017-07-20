---
layout: default
title: "Python Requests Post Multipart/form-data File问题排查"
categories: Python
---

#Python Requests Post Multipart/form-data File问题排查
***

近期有需求需要Post上传Multipart/form-data类型的数据，所以尝试使用Python的Requests库，在Requests库中，Post上传Multipart/form-data类型的数据需要使用`files`字段，见Requests库的[官方介绍](http://docs.python-requests.org/en/master/user/quickstart/#post-a-multipart-encoded-file):

	>>> url = 'http://httpbin.org/post'
	>>> files = {'file': ('report.csv', 'some,data,to,send\nanother,row,to,send\n')}
	
	>>> r = requests.post(url, files=files)
	>>> r.text
	{
	  ...
	  "files": {
	    "file": "some,data,to,send\\nanother,row,to,send\\n"
	  },
	  ...
	}

照猫画虎，写了如下代码：

	#!/usr/bin/env python
	
	import requests
	
	
	url = 'http://httpbin.org/post'
	
	
	def post_file(key1, value1, key2, value2):
		files = {
			key1: value1,
			key2: value2
		}
	
		r = requests.post(url, files=files)
	
	if __name__ == '__main__':
		post_file("a", 1, "b", 2)

尝试运行时报错：

	Traceback (most recent call last):
	  File "post_file.py", line 18, in <module>
	    post_file("a", 1, "b", 2)
	  File "post_file.py", line 15, in post_file
	    r = requests.post(url, files=files)
	  File "/usr/local/lib/python2.7/dist-packages/requests/api.py", line 111, in post
	    return request('post', url, data=data, json=json, **kwargs)
	  File "/usr/local/lib/python2.7/dist-packages/requests/api.py", line 57, in request
	    return session.request(method=method, url=url, **kwargs)
	  File "/usr/local/lib/python2.7/dist-packages/requests/sessions.py", line 461, in request
	    prep = self.prepare_request(req)
	  File "/usr/local/lib/python2.7/dist-packages/requests/sessions.py", line 394, in prepare_request
	    hooks=merge_hooks(request.hooks, self.hooks),
	  File "/usr/local/lib/python2.7/dist-packages/requests/models.py", line 298, in prepare
	    self.prepare_body(data, files, json)
	  File "/usr/local/lib/python2.7/dist-packages/requests/models.py", line 449, in prepare_body
	    (body, content_type) = self._encode_files(files, data)
	  File "/usr/local/lib/python2.7/dist-packages/requests/models.py", line 152, in _encode_files
	    fdata = fp.read()
	AttributeError: 'int' object has no attribute 'read'

看到`AttributeError: 'int' object has no attribute 'read'`这个错误一时想不到什么原因，直接找出Requests库的源码，在models.py文件，152行，在`_encode_files`函数中，如下所示：

	for (k, v) in files:
        # support for explicit filename
        ft = None
        fh = None
        if isinstance(v, (tuple, list)):
            if len(v) == 2:
                fn, fp = v
            elif len(v) == 3:
                fn, fp, ft = v
            else:
                fn, fp, ft, fh = v
        else:
            fn = guess_filename(v) or k
            fp = v

        if isinstance(fp, (str, bytes, bytearray)):
            fdata = fp
        else:
            fdata = fp.read()

其中最后一行`fdata = fp.read()`就是出问题报错的地方，这段代码的大致逻辑是处理我们传入的`files`内容，`fp`此时的值就是传入键值对的value，而这里对value有个判断，如果value是`str`或者`bytess`或者`bytearray`类型则，直接使用它的值，如果不是则认为`fp`是一个打开的文件对象，则调用read进行读取。

我的问题就出在传入键值对时没有注意value的类型，传入的value是`int`，所以才会出现最后的报错，将value修改为`str`类型即可。

在这里也顺便记录一下调试python库的做法：

- 首先找到相应库安装的位置，可以通过先import在查看__path__变量查看。
- 找到库的路径后，库中的内容都是.py源码和编译的.pyc中间码，修改源码后通过`python -m py_compile file.py`命令来编译生成pyc文件，并替换。
- 在使用时，重新import即可，在寻找问题时更加方便。
