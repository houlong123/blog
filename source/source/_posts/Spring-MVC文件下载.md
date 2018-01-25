---
title: Spring MVC文件下载
date: 2017-07-06 11:41:48
tags: Spring
---

<i><font color=red>**文件下载的本质是从服务器读取要下载的文件，把文件转化为流，然后读取流。**</font></i>

#### 文件下载的方式

+ 普遍实现方式

```java
/**
 * 文件下载
 * @param response  响应体
 * @param file  文件目录
 * @return
 */
public static void downLoadFile(HttpServletResponse response, File file) { 
    if (file == null || !file.exists()) {
        return;
    } 
    OutputStream out = null; 
    try { 
    	//设置response信息头
        response.reset(); 
        response.setContentType("application/octet-stream; charset=utf-8"); 
        response.setHeader("Content-Disposition", "attachment; filename=" + file.getName()); 
        //读取文件流
        out = response.getOutputStream(); 
        out.write(FileUtils.readFileToByteArray(file)); 
        out.flush(); 
    } catch (IOException e) { 
        e.printStackTrace(); 
    } finally { 
        if (out != null) { 
            try { 
                out.close(); 
            } catch (IOException e) { 
                e.printStackTrace(); 
            } 
        } 
    } 
}
```
这种方式比较普通，是在之前的servlet中比较常用，但是在Spring MVC 项目中，还有一种比较优雅的实现方式，详情如下。

+ Spring MVC实现方式

```
/**
 * 文件下载
 * @param fileName  文件名
 * @param file  文件目录
 * @return
 */
public ResponseEntity<byte[]> download(String fileName, File file) throws IOException { 
    String dfileName = new String(fileName.getBytes("gb2312"), "iso8859-1"); 
    HttpHeaders headers = new HttpHeaders(); 
    headers.setContentType(MediaType.APPLICATION_OCTET_STREAM); 
    headers.setContentDispositionFormData("attachment", dfileName); 
    return new ResponseEntity<byte[]>(FileUtils.readFileToByteArray(file), headers, HttpStatus.CREATED); 
}
```