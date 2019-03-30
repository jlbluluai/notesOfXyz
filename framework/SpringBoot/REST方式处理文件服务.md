参考：



### 前言

​	处理文件服务在web开发中也是不可避免的存在，那么REST方式如何处理文件服务，该篇就以纯后端的方式模拟一下。



### 文件上传

​	文件服务自然包括文件上传和下载，但一般使用中，我个人认为还是文件上传的场景多一点（准确的说是文件上传由后端处理的场景），下载可以通过其它途径，比如ftp链接之类的（我理解上将下载的带宽分担到其它服务器会减少本服务器的压力吧），可以不通过后端代码的方式。

​	上传的话，我也不写前端的代码，用后端测试类模拟。

```java
    @Test
    public void whenUploadSuccess() throws Exception {
        String result = mockMvc.perform(MockMvcRequestBuilders.fileUpload("/file")
                .file(new MockMultipartFile("file", "test.txt", "multipart/form-data", "Hello".getBytes())))
                .andExpect(MockMvcResultMatchers.status().isOk())
                .andReturn().getResponse().getContentAsString();
        System.out.println(result);
    }
```

​	这算是一个标准的web测试方法，成功就返回200码，上传文件的模拟是由MockMultipartFile实现的，现在还没写服务，这个测试类跑起来很显然会报404错误码。

```java
@RestController
@RequestMapping("/file")
public class FileController {

    @PostMapping
    public FileInfo upload(MultipartFile file) throws IOException {
        System.out.println(file.getName());
        System.out.println(file.getOriginalFilename());
        System.out.println(file.getSize());
        String folder = "E:/";

        File localFile = new File(folder, System.currentTimeMillis() + ".txt");
        file.transferTo(localFile);
        return new FileInfo(localFile.getPath());
    }
}
```

​	一个标准REST风格的控制器，可以看到我们最后传进来的是一个MultipartFile类的file对象（测试类假定的名字必须和这边一致），然后我们定义本地文件路径，通过transferTo方法就可以将文件写在本地了，是不是很简单。后面我看看是不是补充下前端Ajax异步请求过来文件上传的操作。



### 文件下载

​	文件下载我们就直接用浏览器发送一个get请求了，先把服务写好。

```java
@RestController
@RequestMapping("/file")
public class FileController {

    @GetMapping("/{id}")
    public void download(@PathVariable String id, HttpServletRequest request, HttpServletResponse response) throws IOException {

        try (//JDK1.7开始的写法，这样处理流，不需要手动关闭，会自动关闭
                InputStream is = new FileInputStream(new File("E:/", id + ".txt"));
                OutputStream os = response.getOutputStream();
        ) {
            response.setContentType("application/x-download");//设置响应方式为下载
            //设置文件默认下载名
            response.addHeader("Content-Disposition", "attachment;filename=test.txt");

            IOUtils.copy(is, os);//commons-io提供的方法，将输入流直接拷贝到输出流
            os.flush();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

​	我们假定这是Spring Boot默认配置的工程并且假定文件名为1，那么我们启动工程在浏览器输入localhost:8080/file/1就会弹出下载框并且下载名已经默认为test.txt。