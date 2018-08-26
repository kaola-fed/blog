<!-- 为了更方便归档，请先完善以上信息，正文贴下面 -->
<!--
注意点：
0. 文章中的资源（主要是图片）引用请使用 HTTPS
1. 文章末可以加上自己的署名，如： by [Kaola](http://www.kaola.com)
2. 最好不要用 NOS 图床，感觉加防盗链是迟早的事
3. 文章会定期归档到 https://blog.kaolafed.com/
-->
### 前言
之前有接触到一些做node爬虫的工程师，在获取淘宝卖家授权后，通过nodejs爬取该卖家的淘宝店各项信息，获得每月销售额、好评率等各项信息，对此较感兴趣，自己实现了一个简单的爬虫，爬取计量一新闻库，将文章内容、图片保存至本地。 

### 实现方式
实现这样的效果其实很简单，通过request模块请求某一个新闻页的html，然后通过cheerio实现类似于JQuery操作dom的效果，将你所需要的信息从dom中获取出来。

### 主要实现步骤
1. 通过分析http://www.cjlu.edu.cn/do.jsp?dotype=newsmm&columnsid=13&currentPage=1 该新闻库可以发现，该类目下新闻共有64页，每页30条新闻，url参数currentPage对应分页页码。
2. 使用request发送get请求，获取第一分页的html，然后用cheerio分析html，获取当前页内每一条新闻的链接，存入数组urlList。通过network可以发现，该http请求Response Header内Transfer-Encoding: chunked，代表服务端是一边产生数据，一边发给客户端，所以监听data事件，将获取到的data拼接成完整的html。

```
let html = '';
request
    .get(url)  //http://www.cjlu.edu.cn/do.jsp?dotype=newsmm&columnsid=13&currentPage=1
    .on('data',chunk =>{
        html += chunk;
    })
    .on('end',() =>{
        let $ = cheerio.load(html);
        let urlList = [];
        let length = $('.new-main-list li').length;
        $('.new-main-list li').each((index) =>{
            let link = 'http://www.cjlu.edu.cn'+$('.new-main-list li a').eq(index).attr('href');
            urlList.push(link);
        })
        resolve(urlList);
    })
```

3. 然后依次通过request请求每一条新闻，获取其中的新闻内容和图片，新闻内容使用fs.createWriteStream在本地创建txt文件，并使用fs.appendFileSync写入，图片下载保存到本地。

```
//将新闻文本内容一段一段添加到/data文件夹下，并用新闻的标题来命名文件
let fileName = `data/${i}_${news_title}.txt`;
fs.createWriteStream(fileName);  
$('.main_box').next().find('p').each(function (index, item) {
    let x = $(this).text(); 
    if (x != '') {
        x = x + '\n';   
        fs.appendFileSync(fileName, x, { encoding: 'utf8'});
    }
})
```

```
//将新闻内使用图片下载到本地
request
    .get(url)
    .on('error', err => {
        error_url.push(url);
    })
    .pipe(fs.createWriteStream('image/'+img_title));
```

4. 期间使用async.mapLimit限制并发请求的数量，防止服务器封IP，当然这样的方式是比较简陋的，可以使用动态IP的方式请求。

```
async.mapLimit(urlLists, 5, function (url, callback) {
    fetchPage(url, callback);
}, function (err,result) {
    console.log('final:');
    console.log(error_url);
    console.log(result)
});
```

### Nightmare介绍
nightmare号称爬虫的终极形态，它是一个基于electron的自动化库(自带浏览器)，常用于实现爬虫或自动化测试。<br>
#### 那么它能实现什么功能？
它可以通过获取dom的方式，模拟用户的操作，比如点击链接、输入账号密码自动登录等，可以通过wait底部元素得到页面dom加载完毕的事件。<br><br>

在爬豆瓣电影排行榜时，担心被反爬虫机制封IP，就使用了nightmare模拟用户操作，查看排行榜，点击翻页等操作。

```
nightmare
  .goto('https://movie.douban.com/')
  .click('#db-nav-movie > div.nav-secondary > div > ul > li:nth-child(4) > a')
  .click('.douban-top250-hd a')
  .wait('#footer')
  .evaluate(function () {
    let i = 1;
    let urlList = [];
    while(i<26){
      urlList.push(document.querySelector('#content > div > div.article > ol > li:nth-child('+i+') > div > div.info > div.hd > a').getAttribute('href'))
      i++;
    }
    return urlList;
  })
```

