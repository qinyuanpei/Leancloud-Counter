# LeanCloud-Counter

## 项目简介
基于LeanCloud的访问量统计工具，理论上可以接入任意站点。在 [Jekyll使用LeanCloud记录文章的访问次数](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/06/jekyll%E4%BD%BF%E7%94%A8LeanCloud%E8%AE%B0%E5%BD%95%E6%96%87%E7%AB%A0%E7%9A%84%E8%AE%BF%E9%97%AE%E6%AC%A1%E6%95%B0/) 这篇文章的基础上，增加针对站点PV(**访问量**)、UV(**访客数**)的统计，以及访客地理位置([https://api.ip.sb](https://api.ip.sb))、UserAgent([UAParser.js](http://faisalman.github.io/ua-parser-js/))等信息的统计功能。相比[原作者](<https://priesttomb.github.io/about/>)的方案主要有以下优点：

1、使用`querySelectAll()`替换jQuery的`$`

2、实现基于[REST API](https://leancloud.cn/docs/rest_api.html), 不再依赖`leancloud-storage`

##  快速开始
1、在页面中引入以下JavaScript代码片段：
```JavaScript
<script src="/leancloud-counter.min.js"></script>
<script>
        (function () {
            new VisitorCounter().init({
                appId: <Your appId>,       //应用ID, 必输
                appKey: <Your appKey>,      //应用Key, 必输
                region: '华北',      //华东, 华北, 国际, 默认华北
                domain: '',      //自定义域名, 选填
                collectIP: true, //是否收集IP及位置信息, 默认开启
                collectUA: true  //是否收集UserAgent, 默认开启
            });
        })();
</script>
```

2、在页面中加入以下HTML片段：
```HTML
/* 页面PV */
<span id="lc_counter_value_page_pv"></span>
/* 页面UV */
<span id="lc_counter_value_page_uv"></span>
/* 站点PV */
<span id="lc_counter_value_site_pv"></span>
/* 站点UV */
<span id="lc_counter_value_site_uv"></span>

```

3、在LeanCloud后台`云引擎`->`部署`模块内增加下面两个`Hooks`函数，并完成部署：

```JavaScript
AV.Cloud.afterSave('VisitorRecord', async function(request) {
  var query = new AV.Query('VisitorCounter');
  var page_url = request.object.get('page_url');
  console.log('query page_url: ' + page_url);
  query.equalTo('page_url', page_url);
  return query.find().then(function (counters) {
  	if (counters.length > 0){
    	counters[0].increment('page_uv');
    	console.log('increment UV of page_url: ' + page_url + ", " + counters[0].get('page_pv'));
    	return counters[0].save()
    }
  });
});

AV.Cloud.afterUpdate('VisitorCounter', async function(request) {
  var page_url = request.object.get('page_url');
  if(page_url.indexOf('//') == -1){
    return;
  }
  var site_url = page_url.split('//')[1];
  site_url = site_url.substring(0, site_url.indexOf('/'));
  console.log('now to update site PV/UV with: ' + site_url);
  if (request.object.updatedKeys.indexOf('page_pv') != -1) {
    var query = new AV.Query('VisitorCounter');
    query.equalTo('page_url',site_url);
    query.find().then(function(counters){
        if(counters.length>0){
            counters[0].increment('page_pv');
            console.log('update site PV of ' + site_url + ", " + counters[0].get('page_pv'));
            return counters[0].save();
        }
    });
  } else if (request.object.updatedKeys.indexOf('page_uv') != -1) {
    var query = new AV.Query('VisitorCounter');
    query.equalTo('page_url',site_url);
    query.find().then(function(counters){
        if(counters.length>0){
            counters[0].increment('page_uv');
            console.log('update site PV of ' + site_url + ", " + counters[0].get('page_uv'));
            return counters[0].save();
      }
   });
 }
});
```

## Q & A

Q：这可能是我见过最麻烦的Quick Start，尤其是第三点

A：如果不需要统计站点的PV/UV，其实是不需要第三点的，这样是不是相当地Quick啦



Q：为什么这里需要使用`Hooks`这种对使用者不太友好的东西

A：因为LeanCloud里使用的是Mongodb这种非关系型数据库，它无法使用`Sum()`这类聚合方法，而离线分析等高级用法只有商业版支持，况且我不想在客户端处理数据。



Q：这样统计出来的`UV`和`PV`感觉不太准确啊，每次刷新都视为一次点击吗？

A：也许加入时间的判断逻辑会更好一点，可如果想做一个通用型的方案，这样子算是无功无过吧！


