# fileDownloadAndUpload
文件下载和上传对应文档和代码
### 第十一章 文件上传与文件下载

#### 11.1 文件上传与文件下载底层IO逻辑

![image-20220521161932107](04SpringMVC.assets\image-20220521161932107.png)

- 文件上传：读本地文件，写到服务器
- 文件下载：读服务器文件，写到本地

#### 11.2 文件下载

- 实现【文件下载】步骤

  - 准备下载资源
    - 位置：【/WEB-INF/download/】
    - 名称：download
  - 使用ResponseEntity<T>对象作为方法返回值，实现文件下载
    - bytes[]：设置当前下载文件的所有字节
    - headers：设置响应头参数【插件&字符集问题】
    - statusCode：设置响应状态码【200：响应成功】

- 示例代码

  ```java
  package com.atguigu.controller;
  
  import org.springframework.http.HttpHeaders;
  import org.springframework.http.HttpStatus;
  import org.springframework.http.ResponseEntity;
  import org.springframework.stereotype.Controller;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.ResponseBody;
  import org.springframework.web.bind.annotation.ResponseStatus;
  
  import javax.servlet.http.HttpServletRequest;
  import java.io.FileInputStream;
  import java.io.InputStream;
  
  /**
   * @author Chunsheng Zhang 尚硅谷
   * @create 2022/5/21 16:30
   */
  @Controller
  public class FileDownController {
  
      @RequestMapping("/doFileDownController")
      public ResponseEntity<byte[]> doFileDownController(String fileName,
                                                         HttpServletRequest request) throws Exception{
          //获取【目标文件】真实路径
          String realPath = request.getServletContext().getRealPath("/WEB-INF/download/" + fileName);
          // - bytes[]：设置当前下载文件的所有字节
          InputStream is = new FileInputStream(realPath);
          byte[] bytes = new byte[is.available()];
          is.read(bytes);
          // - headers：设置响应头参数【插件&字符集问题】
          HttpHeaders headers = new HttpHeaders();
          //设置要下载的文件为附件格式【下载不打开】
          headers.add("Content-Disposition", "attachment;filename="+fileName);
          //处理中文文件名问题
          headers.setContentDispositionFormData("attachment", new String(fileName.getBytes("utf-8"), "ISO-8859-1"));
          // - statusCode：设置响应状态码【200：响应成功】
  
          ResponseEntity<byte[]> responseEntity =
                  new ResponseEntity<>(bytes,headers,HttpStatus.OK);
  
          return responseEntity;
      }
  }
  ```

### day13

#### 11.3 文件上传

- 实现【文件上传功能】思路

  - 导入jar包
  - 装配CommonsMultipartResolver
    - id必须是multipartResolver
  - 表单
    - 表单提交方式必须是POST
    - 设置表单enctype="multipart/form-data"
    - 表单中必须包含文件域【type="file"】
  - Controller中
    - 将文件域自动装配MultipartFile类型
    - 调用transferTo()方法实现文件上传

- 示例代码

  - 导包

  ```xml
  <!-- commons-fileupload -->
  <dependency>
      <groupId>commons-fileupload</groupId>
      <artifactId>commons-fileupload</artifactId>
      <version>1.4</version>
  </dependency>
  ```

  - 装配CommonsMultipartResolver

  ```xml
  <!--    装配CommonsMultipartResolve实现文件上传
              id必须是multipartResolver
  -->
  <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
      <property name="defaultEncoding" value="UTF-8"></property>
  </bean>
  ```

  - 表单

  ```html
  <div align="center">
      <h2>文件上传</h2>
      <form th:action="@{/doUploadFile}" method="post" enctype="multipart/form-data">
          用户名称：<input name="username" type="text"><br>
          上传文件：<input type="file" name="uploadFile"><br>
          <input type="submit" value="文件上传">
      </form>
  </div>
  ```

  - Controller

  ```java
  /**
   * 处理文件上传
   * @param username
   * @param uploadFile
   * @return
   */
  @PostMapping("/doUploadFile")
  public String doUploadFile(String username,
                             MultipartFile uploadFile,
                             HttpSession session){
      try {
          //获取上传文件路径
          String realPath = session.getServletContext().getRealPath("/WEB-INF/uploaddir/");
          //如上传文件目录不存在【新建目录】
  
          //获取上传文件名称
          String filename = uploadFile.getOriginalFilename();
          //File.separator：获取当前系统目录分隔符
          File uFile = new File(realPath+File.separator+filename);
          //实现文件上传
          uploadFile.transferTo(uFile);
      } catch (IOException e) {
          e.printStackTrace();
      }
  
      return "success";
  }
  ```

#### 11.4 优化文件上传

- 上传目录不存在

  - 解决方案：新建目录

    ```java
    File rpFile = new File(realPath);
    if(rpFile.exists()==false){
        rpFile.mkdirs();
    }
    ```

- 上传文件时【允许上传同名文件】

  - 解决方案：使用UUID处理文件名

    - UUID：是一个32位16进制随机数
    - 特点：全球唯一【当前项目中唯一】

    ```java
    String uuid = UUID.randomUUID().toString().replace("-", "");
    //获取上传文件名称
    String filename = uploadFile.getOriginalFilename();
    //File.separator：获取当前系统目录分隔符
    File uFile = new File(realPath+File.separator+uuid+filename);
    ```

- 上传文件时【设置文件上传大小上限】

  - 目的：降低服务器压力

  ```xml
  <!--    装配CommonsMultipartResolve实现文件上传
              id必须是multipartResolver
  -->
  <bean id="multipartResolver" class="org.springframework.web.multipart.commons.CommonsMultipartResolver">
      <!--设置字符集-->
      <property name="defaultEncoding" value="UTF-8"></property>
      <!--设置上传文件大小上限:100kb
              1b  [byte]
              1024b = 1kb
              1024kb = 1mb
              1024mb = 1gb
              1024gb = 1tb
              pb
              zb
      -->
      <property name="maxUploadSizePerFile" value="102400"></property>
      
  </bean>
  ```

  - 注意：如上传文件大小，超出设置大小上限，会报如下错

    org.apache.commons.fileupload.FileUploadBase$FileSizeLimitExceededException: The field uploadFile exceeds its maximum permitted size of 102400 bytes.
