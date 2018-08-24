---
title: spring mvc download file
date: 2018-04-25 16:52:17
tags: [Spring MVC]
---
# 文件下载踩坑记
# 1.需求
现在提了新需求，要求系统可以下载keytab文件，就是在keytab列表旁边加个链接，点击下载呗，有啥子。

# 1.5.自我认知的茶具
不是需求难，是你太菜了-_-

# 2.基本思路
## 1.后端
这个事情应该完全是后端的事情嘛，首先keytab文件原先就有导出的操作，是通过ssh调用kadmin.local导出到kdc本地机器，继续使用scp在集群的机器中传输来传输去。导出已经做了，然后将数据传输到web服务器所在的机器。使用File读出，序列化后返回就ok啊。返回ResponseEntity。
```
@RequestMapping(value = "downloadKeytab", method = RequestMethod.POST)
	public ResponseEntity<byte[]> downloadPricipal(String realm) throws IOException {
		String web_ip = IConfig.get("web_ip");
		//export the .keytab file to local /root/keytabs
		String cmd = "/usr/sbin/kadmin.local -q 'xst -k /root/keytabs/tmp.keytab -norandkey "+realm+"' && " +
				"scp /root/keytabs/tmp.keytab root@"+web_ip+":/root";
		cmd = cmd + "\n";
		String log = "export " + realm + " to " + "/root/tmp.keytab";
		List<String> rs = SSHUtil.sh(cmd);

		File file = new File("/root/tmp.keytab");
		byte[] body = null;
		InputStream is = null;
		try {
			is = new FileInputStream(file);
		} catch (FileNotFoundException e) {
			e.printStackTrace();
		}
		body = new byte[is.available()];
		is.read(body);
		
		HttpHeaders headers = new HttpHeaders();
		headers.add("Content-Dipostion","attchement;filename=" + file.getName());
		HttpStatus statusCode= HttpStatus.OK;
		headers.setContentType(MediaType.APPLICATION_OCTET_STREAM);
		ResponseEntity<byte[]> entity = new ResponseEntity<byte[]>(body, headers, statusCode);
		return entity;
```
## 2.前端
前端菜的要名，面向google编程，实验了好几个才弄到一个能下载的js
```
	function downloadKeytab(realm) {
		$.post('${basePath}/principal/downloadKeytab.shtml', {realm:realm}, function(result){
		    var blob=new Blob([array]);
            var link=document.createElement('a');
            link.style = "display: none";
            document.body.appendChild(link);
            link.href=window.URL.createObjectURL(blob);
            link.download="download.keytab";
            link.click();
            window.URL.revokeObjectURL(link);
		});
    }
```
当然还遇到几个小坑，比如freemarker里${foo}怎么以字符串的形式传入(加'')都得查半天，羞耻。
## 3.开始联调了，美滋滋
实验可以下载，激动，傻傻的严谨一把，用HxD比较下文件,竟然有个一个字节是错的，只有一个字节。
排除了自己犯傻、灵异、人品、程序重启后，我确定是代码实现有问题。

## 4.哪个环节的代码有问题
* 后端body byte数组，打印，没问题
* postman post,fiddler抓取（这样有个好处，可以绕过权限检测，直接用fiddler会返回权限不足的问题，postman不能看16进制，fiddler能，所以组合下）返回的数据还是没问题。那就是js数据处理的问题了

## 4.google你好
 以blob js file down搜，stackoverflow中有不少人说js那个实现会有corrupt的问题，一般说是解码的问题，有个js代码解码处理下
```
    function base64ToArrayBuffer(base64) {
        var binaryString = window.atob(base64);
        var binaryLen = binaryString.length;
        var bytes = new Uint8Array(binaryLen);
        for (var i = 0; i < binaryLen; i++) {
            var ascii = binaryString.charCodeAt(i);
            bytes[i] = ascii;
        }
        return bytes;
    }
```
套进去，提示
```
//Uncaught DOMException: Failed to execute 'atob' on 'Window': The string to be decoded contains characters outside of the Latin1 range.
```

## 5.折磨
无数次重启tomcat后（改js不能热更新吧），终于想到了，把后端的数据用base64 encode处理为String，
```
String bodyStr = Base64.encodeBase64String(body);
```
然后用base64ToArrayBuffer decode回来，ok

## 6.感想
小需求干掉大时间，cpu慢调试受阻？
