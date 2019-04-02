[:abc: English](./README.md) | :mahjong: 简体中文

# 如何免费创建云端爬虫集群


## 在线体验
[scrapydweb.herokuapp.com](https://scrapydweb.herokuapp.com)


## 网路拓扑图
![network topo](https://raw.githubusercontent.com/my8100/files/master/scrapyd-cluster-on-heroku/screenshots/network_topology.png)


## 注册帐号
1. Heroku

访问 [heroku.com](https://signup.heroku.com) 注册免费账号（注册页面需要调用 google recaptcha 人机验证，登录页面也需要**科学上网**，访问 APP 运行页面则没有该问题），免费账号最多可以**创建和运行5个 APP**。

![heroku register](https://raw.githubusercontent.com/my8100/files/master/scrapyd-cluster-on-heroku/screenshots/heroku_register.png)

2. Redis Labs（可选）

访问 [redislabs.com](https://redislabs.com) 注册免费账号，提供**30MB 存储空间**，用于下文通过 [scrapy-redis](https://github.com/rmax/scrapy-redis) 实现**分布式爬虫**。

![redislabs register](https://raw.githubusercontent.com/my8100/files/master/scrapyd-cluster-on-heroku/screenshots/redislabs_register.png)


## 安装工具
1. [Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
2. [Heroku CLI](https://devcenter.heroku.com/articles/heroku-cli)
3. [Python client for Redis](https://pypi.org/project/redis/)：运行 `pip install redis` 命令即可。


## 下载配置文件
新开一个命令行提示符：
```
git clone https://github.com/my8100/scrapyd-cluster-on-heroku
cd scrapyd-cluster-on-heroku
```


## 登录 Heroku
```
heroku login
# outputs:
# heroku: Press any key to open up the browser to login or q to exit:
# Opening browser to https://cli-auth.heroku.com/auth/browser/12345-abcde
# Logging in... done
# Logged in as username@gmail.com
```


## 创建 Scrapyd 集群
1. 新建 Git 仓库
```
cd scrapyd
git init
git status
git add .
git commit -a -m "first commit"
git status
```

2. 部署 Scrapyd APP
```
heroku apps:create svr-1
heroku git:remote -a svr-1
git remote -v
git push heroku master
heroku logs --tail
# Press ctrl+c to stop logs outputting
# Visit https://svr-1.herokuapp.com
```

3. 添加环境变量
    - 设置时区
    ```
    # python -c "import tzlocal; print(tzlocal.get_localzone())"
    heroku config:set TZ=Asia/Shanghai
    # heroku config:get TZ
    ```
    - 添加 Redis 账号（可选，详见 *scrapy_redis_demo_project.zip* 中的 *settings.py*）
    ```
    heroku config:set REDIS_HOST=your-redis-host
    heroku config:set REDIS_PORT=your-redis-port
    heroku config:set REDIS_PASSWORD=your-redis-password
    ```

4. 重复上述第2和第3步完成余下三个 Scrapyd APP 的部署和配置：`svr-2`，`svr-3` 和 `svr-4`


## 创建 ScrapydWeb APP
1. 新建 Git 仓库
```
cd ..
cd scrapydweb
git init
git status
git add .
git commit -a -m "first commit"
git status
```

2. 部署 ScrapydWeb APP
```
heroku apps:create myscrapydweb
heroku git:remote -a myscrapydweb
git remote -v
git push heroku master
```

3. 添加环境变量
    - 设置时区
    ```
    heroku config:set TZ=Asia/Shanghai
    ```
    - 添加 Scrapyd server（详见 *scrapydweb_settings_v8.py* in the *scrapydweb* directory）
    ```
    heroku config:set SCRAPYD_SERVER_1=svr-1.herokuapp.com:80
    heroku config:set SCRAPYD_SERVER_2=svr-2.herokuapp.com:80#group1
    heroku config:set SCRAPYD_SERVER_3=svr-3.herokuapp.com:80#group1
    heroku config:set SCRAPYD_SERVER_4=svr-4.herokuapp.com:80#group2
    ````

4. 访问 [myscrapydweb.herokuapp.com](https://myscrapydweb.herokuapp.com)
![scrapydweb](https://raw.githubusercontent.com/my8100/files/master/scrapyd-cluster-on-heroku/screenshots/scrapydweb.png)


## 部署和运行分布式爬虫
1. 上传 demo 项目，即 *scrapyd-cluster-on-heroku* 目录下的压缩文档 *scrapy_redis_demo_project.zip*
2. 将种子请求推入 `mycrawler:start_urls` 触发爬虫并查看结果
```
In [1]: import redis

In [2]: r = redis.Redis(host='your-redis-host', port=your-redis-port, password='your-redis-password')

In [3]: r.delete('mycrawler_redis:requests', 'mycrawler_redis:dupefilter', 'mycrawler_redis:items')
Out[3]: 0

In [4]: r.lpush('mycrawler:start_urls', 'http://books.toscrape.com', 'http://quotes.toscrape.com')
Out[4]: 2

# wait for a minute
In [5]: r.lrange('mycrawler_redis:items', 0, 1)
Out[5]:
[b'{"url": "http://quotes.toscrape.com/", "title": "Quotes to Scrape", "hostname": "d6cf94d5-324e-4def-a1ab-e7ee2aaca45a", "crawled": "2019-04-02 03:42:37", "spider": "mycrawler_redis"}',
 b'{"url": "http://books.toscrape.com/index.html", "title": "All products | Books to Scrape - Sandbox", "hostname": "d6cf94d5-324e-4def-a1ab-e7ee2aaca45a", "crawled": "2019-04-02 03:42:37", "spider": "mycrawler_redis"}']
```

![scrapyd cluster on heroku](https://raw.githubusercontent.com/my8100/files/master/scrapyd-cluster-on-heroku/screenshots/scrapyd_cluster_on_heroku.gif)


## 总结
 - 优点
    - 免费
    - 可以爬 Google 等外网
    - 可扩展（借助于 [ScrapydWeb](https://github.com/my8100/scrapydweb)）
 - 缺点
    - 注册和登录需要科学上网
    - **Heroku APP 每天至少自动重启一次并且重置所有文件**，因此需要外接数据库保存数据，详见 [devcenter.heroku.com](https://devcenter.heroku.com/articles/dynos#restarting)