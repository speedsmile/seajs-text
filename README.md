# seajs-text
seajs引用文本模块的插件。基于原始的seajs-text.js插件，修改url带有参数时模块加载错误的bug

## bug描述
seajs.config配置中如果使用了"map"配置项来重写文本模块的路径（例如给模块加上去掉缓存的参数），<br/>
该模块会被构建创建出2个模块对象，并且模块返回值是null。<br/>

例，加载的文本模块的url是
````
// 加载的文本模块的url是 a.html

// seajs.config配置map项

seajs.config({
    // 省略其它配置，只写上map配置
    map: [
        ["a.html", "a.html?v=aaa"]
    ]

});


seajs.use(["入口模块"], function(){

    require("a.html"); // 结果是null

});

// 入口模块

define(function(require){

    require("a.html"); // 结果是null。正确的结果应该是html文本

});


````


通过seajs.config("map")，观察到a.html模块生成了2个
````
{

    "a.html?v=aaa": {
        dependencies: [], // 没有依赖
        exports: null, // 文本类型，不设返回值
        uri: "a.html?v=aaa"
    },

    "a.html?v=aaa?v=aaa": { // 参数被多加了一遍
	id: "模块的路径",
	factory: "正确的文本内容", // 最关键的一处
        dependencies: [], // 没有依赖
        exports: null, // 文本类型，不设返回值
        uri: "a.html?v=aaa?v=aaa" // 参数被多加了一遍

    },
}
````
对比2个模块可以分析出：seajs-text插件不能正确处理带有参数或hash的url模块。没有id和factory的模块被当成了js模块，因此返回值是null<br/>

源代码中注册了3个文本解析模块，分别是[".tpl", ".html"]、[".json"]、[".handlebars"]。<br/>
在这3个模块的解析器中处理url的地方把带有参数的url转换成不带参数的url就正常了
````
// seajs-text源代码
// normal text
	register({
		name: "text",
		ext: [".tpl", ".html"],
		exec: function(uri, content) {
			uri = uri.replace(/([^?#]+).*/, "$1"); // 加上代码
			globalEval('define("' + uri + '#", [], "' + jsEscape(content) + '")')
		}
	})

// json
	register({
		name: "json",
		ext: [".json"],
		exec: function(uri, content) {
			uri = uri.replace(/([^?#]+).*/, "$1"); // 加上代码
			globalEval('define("' + uri + '#", [], ' + content + ')')
		}
	})

// for handlebars template
	register({
		name: "handlebars",
		ext: [".handlebars"],
		exec: function(uri, content) {
			uri = uri.replace(/([^?#]+).*/, "$1"); // 加上代码
			var code = [
				'define("' + uri + '#", ["handlebars"], function(require, exports, module) {',
				'  var source = "' + jsEscape(content) + '"',
				'  var Handlebars = require("handlebars")["default"]',
				'  module.exports = function(data, options) {',
				'    options || (options = {})',
				'    options.helpers || (options.helpers = {})',
				'    for (var key in Handlebars.helpers) {',
				'      options.helpers[key] = options.helpers[key] || Handlebars.helpers[key]',
				'    }',
				'    return Handlebars.compile(source)(data, options)',
				'  }',
				'})'
			].join('\n')
			
			globalEval(code)
		}
	})
````

