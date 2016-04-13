 一些思考
 以下是在实现这个解决方案过程中遇到的一些问题和思考：

【1】生成Js方法后，加载这段Js的时机是什么？
 刚开始时在当WebView正常加载URL后去加载Js，但发现会存在问题，如果当WebView跳转到下一个页面时，之前加载的Js就可能无效了，
 所以需要再次加载。这个问题经过尝试，需要在以下几个方法中加载Js，它们是WebChromeClient和WebViewClient的方法：
 · onLoadResource
 · doUpdateVisitedHistory
 · onPageStarted
 · onPageFinished
 · onReceivedTitle
 · onProgressChanged
 目前测试了这几个地方，没什么问题，这里我也不能完全确保没有问题。

【2】需要过滤掉Object类的方法
 由于通过反射的形式来得到指定对象的方法，他会把基类的方法也会得到，最顶层的基类就是Object，所以我们为了不把getClass方法注入到Js中，
 所以我们需要把Object的公有方法过滤掉。这里严格说来，应该有一个需要过滤方法的列表。目前我的实现中，需要过滤的方法有：
 "getClass",
 "hashCode",
 "notify",
 "notifyAll",
 "equals",
 "toString",
 "wait",

【3】通过手动loadUrl来加载一段js，这种方式难道js中的对象就不在window中吗？也就是说，通过遍历window的对象，不能找到我们通过loadUrl注入的js对象吗？
 关于这个问题，我们的方法是通过Js声明的，通过loadUrl的形式来注入到页面中，其实本质相当于把我们这动态生成的这一段Js直接写在Html页面中，
 所以，这些Js中的window中虽然包含了我们声明的对象，但是他们并不是Java对象，他们是通过Js语法声明的，所以不存在getClass之类的方法。本质上他们是Js对象。

【4】在Android 3.0以下，系统自己添加了一个叫searchBoxJavaBridge_的Js接口，要解决这个安全问题，我们也需要把这个接口删除，
 调用removeJavascriptInterface方法。这个searchBoxJavaBridge_好像是跟google的搜索框相关的。

【5】在实现过程中，我们需要判断系统版本是否在4.2以下，因为在4.2以上，Android修复了这个安全问题。我们只是需要针对4.2以下的系统作修复。


<!--动态生成类似这样的代码加载到html中-->
javascript:(function JsAddJavascriptInterface_(){
	if (typeof(window.jsInterface)!='undefined') {
		console.log('window.jsInterface_js_interface_name is exist!!');}
	else {
		window.jsInterface = {

			onButtonClick:function(arg0) {
				return prompt('MyApp:'+JSON.stringify({obj:'jsInterface',func:'onButtonClick',args:[arg0]}));
			},

			onImageClick:function(arg0,arg1,arg2) {
				prompt('MyApp:'+JSON.stringify({obj:'jsInterface',func:'onImageClick',args:[arg0,arg1,arg2]}));
			},
           <!--根据native中定义的方法动态的生成这一部分代码，并可以直接在js里面调用jsInterface.XXX();-->
		};
	}
}
)()

调用：

<script language="javascript">
      function onButtonClick()
      {
        // Call the method of injected object from Android source.
        var text = jsInterface.onButtonClick("从JS中传递过来的文本！！！");
        alert(text);
      }

      function onImageClick()
      {
        //Call the method of injected object from Android source.
        var src = document.getElementById("image").src;
        var width = document.getElementById("image").width;
        var height = document.getElementById("image").height;

        // Call the method of injected object from Android source.
        jsInterface.onImageClick(src, width, height);
      }
    </script>
  </head>

  <body>
      <p>点击图片把URL传到Java代码</p>
      <img class="curved_box" id="image"
         onclick="onImageClick()"
         width="328"
         height="185"
         src="http://t1.baidu.com/it/u=824022904,2596326488&fm=21&gp=0.jpg"
         onerror="this.src='background_chl.jpg'"/>
    </p>
    <button type="button" onclick="onButtonClick()">与Java代码交互</button>
  </body>
