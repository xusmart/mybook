# 7.1 gitbook配置使用
## 1 安装
- 环境:  
	Ubuntu 16.04.5 LTS
- 安装:   
	sudo apt-get install npm  
	sudo npm install -g gitbook-cli  
	sudo apt-get install node  
	sudo ln -s /usr/bin/nodejs /usr/bin/node
- 验证:  
	gitbook -V  
## 2 配置
- 创建仓库:  
	mkdir mybook  
- 编辑目录文件:  
	vim SUMMARY.md  
	参考配置如下:   

```markdown
# Summary

* [1.介绍](README.md)
* [2.inux驱动](linux/driver.md)
* [3.Linux系统配置](linux/config.md)
    * [3.1 ssh正向方向代理](linux/config_1.md)
* [4.Android App开发](android/app.md)
* [5.Android System开发](android/system.md)
* [6.人工智能](ai/ai.md)
* [7.常用工具配置](tools/tolls.md)
    * [7.1 gitbook配置使用](tools/tools_1.md)
```
- 生成相关文件:  
	gitbook init  

## 3 发布到github 和gitbook
git init  
git add *  
git commit -m "inital mybook"  
git remote add origin git@github.com:zhoushiqian/mybook.git  
git push -u origin master  

将gitbook 和github 链接在一起
gitbook->settings->integrations->github/mybook repositorie
