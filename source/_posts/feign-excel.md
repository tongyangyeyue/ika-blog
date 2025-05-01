---
title: openfeign调用excel导出接口
date: 2019-02-21 18:32:21
tags: java
---

### 1.常见的excel导出核心方式
excel导出在企业应用中比较常见的.  
我们一般都是把接口的 **header** 的 **"ContentType"** 置为 **"application/octet-stream"**，
将 **"Content-Disposition"** 置为 **"attachment;filename=${fileName}"** 
之后我们将excel的流写入 **response** 的 **outputstream** 中，代码逻辑大体如下
```
    //这是被调用方(服务提供方)的excel导出接口。(这里excel导出被高度封装，请自行编写excel导出逻辑)
    @RequestMapping(value = "/export", method = RequestMethod.GET)
    public void export(HttpServletResponse response) throws IOException {
        sw.buildQuery()
                .exclude("staffFlag")
                .fileName("controller.xlsx")
                .doExport(response, Staff.class);
    }
    
    //其中 doExport函数
    public MpaasQuery doExport(HttpServletResponse response, Class pojo) {
        response.setContentType("application/octet-stream");

        try {
            String f = new String(this.excelFileName.getBytes("UTF-8"), "ISO8859_1");
            response.setHeader("Content-Disposition", "attachment;filename=" + f);
            this.doExport((OutputStream)response.getOutputStream(), pojo);
            return this;
        } catch (Exception var4) {
            throw new MpaasRuntimeException(var4);
        }
    }
```
### 2.openfeign如何调用excel导出的接口
而在 **openfeign** 中，调用excel导出接口不能像普通接口一样输入输出参数保持相同，我们需要得到 **excel** 的字节流，所以调用方的代码如下所示
```
//使用feign来调用已经写好的excel导出接口
@FeignClient(name = "staff",url = "http://localhost:8080/controller")
public interface TestFeign {

    @RequestMapping(value = "/export", method = RequestMethod.GET)
    public byte[] export();
}
```
### 3.最终暴露出来的接口
最后，我们在调用这个feign层的函数时候需要重新设置response的 **"ContentType"** 和 **"Content-Disposition"**
```
    //最终在我们系统中的接口
    @RequestMapping("/feignexport")
    public void health(HttpServletResponse response) throws IOException {
        response.setContentType("application/octet-stream");
        response.setHeader("Content-Disposition", "attachment;filename=controller.xlsx");
        response.getOutputStream().write(testFeign.export());
    }
```