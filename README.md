# 开发网盘中碰到的问题
## 首先介绍一下项目：土星云企业网盘(http://www.saturncloud.com.cn/netdisk),一款基于vue+element ui面向企业和用户的人员管理和存储网盘。主要功能包含：文件、文件夹并行上传下载，复合文件（超过4G）的上传、下载。文件、文件夹分享功能，文件夹共享功能。文件、文件夹重命名、复制、移动功能。文件、文件夹批量删除、下载、移动功能。已删除文件、文件夹批量还原、彻底删除功能。上传下载列表展示功能。内部共享功能，公共资源功能。人员管理链接邀请成员功能。企业信息和产品套餐功能等。基本功能类似百度企业网盘。
###  本项目，上传采用的是ipfs分布式存储上传。感兴趣的童鞋可以了解一下。因为和传统的http上传下载不同，所以在前端处理上会用到许多知识点。首先，所有的上传下载都依赖以'Saturn Agent '命名的桌面客户端。不同的系统对应不同的客户端。![在这里插入图片描述](https://img-blog.csdnimg.cn/20210324155949883.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
### 本项目文件上传可以概括为：
#### 1.file_upload 检查内存是否充足
#### 2.random 生成密钥(45位字符串,文件加密/解密密钥，multibase-Base58BTC编码)
#### 3.pack(POST multipart/form-data) 将文件size，random返回值等作为参数，对原始文件进行处理，完成压缩、加密、切片。返回Multihash等值
#### 4./ws/up 文件上传接口，将数据块、链接块 分发给不同的存储节点冗余编码计算（并行），生成 校验块 分发给不同的存储节点使用websocket协议反馈数据上传进度、速度信息。将pack返回的Multihash作为文件标识，socket更新的percent同步上传列表文件进度。为防止因网速等因素导致token失效，在同步进度同时要在socket推送过程中触发/combo/123请求来延长token
#### 5.Add_file 服务器上传(普通文件is_dir传-1，文件夹传1，复合文件传2)，ipfs上传完毕后将文件上传到服务器上。
### 由此，是一个普通文件的上传过程。如果选择多个文件的话，则循环重复上述操作。如果选择文件夹的话，则通过我自己封装的一个方法（后面会介绍）来判断如果是文件夹的话则is_dir传1，如果是文件的话依然重复上述操作。这里主要需要注意一个问题，选择上传的文件夹里面的文件、文件夹路径要跟最终页面上展示的一模一样。
##### Q1:浏览器解析文件夹files属性后返回的文件信息不满足或不好处理文件上传业务
###### 处理方法：封装我需要的文件信息结构
```
let TREE = function (name, is_dir) {
                                this.name = name;
                                this.is_dir = is_dir;
                                this.path = "";
                                this.index;
                                this.size;
                                this.children = [];
                                this.add = function (child) {
                                    this.children.push(child);
                                };
                            };
                            var now_dir;
                            var children;
                            let root = new TREE(
                                folder[0].webkitRelativePath.split("/")[0],
                                1
                            );
                            for (var i = 0; i < folder.length; i++) {
                                var folderPath = folder[i].webkitRelativePath.split("/");
                                now_dir = root;
                                for (var j = 1; j < folderPath.length - 1; j++) {
                                    var find_flag = 0;
                                    for (var k = 0; k < now_dir.children.length; k++) {
                                        if (now_dir.children[k].is_dir == 1)
                                            children = now_dir.children[k];
                                        else continue;

                                        if (children.name == folderPath[j]) {
                                            find_flag = 1;
                                            now_dir = children;
                                            break;
                                        }
                                    }

                                    if (find_flag == 0) {
                                        let new_dir = new TREE(folderPath[j], 1);
                                        now_dir.add(new_dir);
                                        now_dir = new_dir;
                                    }
                                }

                                let file = new TREE(folderPath[j], 0);
                                file.path = folder[i].webkitRelativePath;
                                file.index = folder[i];
                                file.size = folder[i].size
                                now_dir.add(file);
                            }
                            console.log(root)
```
###### 分别打印选择文件、多文件、层级复杂文件夹封装前后的浏览器解析结构
单个文件![在这里插入图片描述](https://img-blog.csdnimg.cn/20210324165655790.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
多个文件![在这里插入图片描述](https://img-blog.csdnimg.cn/2021032416583269.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
文件夹封装前后对比![在这里插入图片描述](https://img-blog.csdnimg.cn/20210324170250586.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)




### 本项目文件下载可以概括为：
#### 单个普通文件下载：1./ws/down 文件下载接口，文件下载接口，收集数据块、链接块用以恢复文件;若出现异常（数据块缺失等）触发冗余解码机制，完成数据块恢复;使用websocket协议反馈数据下载进度、速度信息。将文件Hash作为参数，socekt更新的percent同步下载列表文件进度。为防止因网速等因素导致token失效，在同步进度同时要在socket推送过程中触发/combo/123请求来延长token
#### 2.unpack ipfs下载完毕后，需要将文件通过浏览器下载器弹出。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622145524686.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
#### 文件夹下载：1.file_dir_info 
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622150356751.png)
#### Hash的数量就是该文件夹中文件的数量。文件夹解析完后，每个文件开启socket更新下载进度。考虑到用户体验问题，将socket请求与下载器同步。用户不需要等待所有文件下载完毕后才能看到下载器弹出。
#### 2.tar 类似上述unpack
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622150840754.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
#### 复合文件下载：1.union_file_meta ![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622151443737.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
#### 此处我选择的是一个大小为9.07G的复合文件，我们按照1G为一片来将复合文件分片。所以会返回10个Hash值。复合文件解析完后，每个文件片开启socket更新下载进度。
#### 2.merge 类似上述unpack
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622152140771.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
##### Q2:window.open受制于不同浏览器弹窗权限问题，无法弹出下载器下载
###### 处理方法：模拟表单提交的方法弹出下载器。（上述unpack、tar、merge截图）
#### 文件、文件夹、复合文件批量下载  1.
![在这里插入图片描述](https://img-blog.csdnimg.cn/2021062215284619.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
####  将文件、文件夹、复合文件构造并糅合成新的组合。
#### 2.每个文件片开启socket更新下载进度。
#### 2.tar (将新组合的数据交给下载器)
##### Q3:如果并行上传下载文件、文件夹、复合文件任务多时，如何给浏览器缓解压力
###### 处理方法：设置limit限制数，页面保留所有任务，自定义限制同时发送的请求数量。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622160929448.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
###### limit：限制同时上传/下载的数量、comdoing：当前上传/下载中的任务数、comtodo：准备中的任务数。eg：limit限制为3，如果文件为4，则同时处理3个任务，只要当一个任务处理完毕，队列中的一个跻身任务中。当任务中存在多个文件和多个复合文件时，文件和复合文件同样计算任务数，而一个复合文件分片后同样记录任务数。eg：同时上传两个大小为4G的复合文件，首先被分为八个1G的分片，此时总任务数为8，同时处理3个分片，如果有一个分片完成，队列中的分片跻身任务中。
##### Q3:并行上传、下载的同时加进任务数，会造成进度紊乱
###### 处理方法：FastMutex 使用 LocalStorage 实现互斥锁。使用 promise 使其更具可读性，尤其是与 async/await 一起使用时。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210622162638772.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80Mjg0MTkzNw==,size_16,color_FFFFFF,t_70)
###### 在所有存和取上传、下载列表相关信息的地方加锁
