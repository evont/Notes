- Bundle

  Server Bundle中包含了所有要在服务端运行的代码列表，和一个入口文件名。

  Client Bundle包含了所有需要在客户端运行的脚本和静态资源，如：js、css图片、字体等。还有一份clientManifest文件清单，清单中initial数组中的js将会在ssr输出时插入到html字符串中作为preload和script脚本引用。async和modules将配合检索出异步组件和异步依赖库的js文件的引入，在输出阶段我们会详细解读。

- 优化

    - 使用`renderToStream` 渲染

    通过流式渲染将内容分批传送给浏览器而不是最后一次全部写入，这样就会使页面渲染速度加快，同时还可以引入 lru-cache 这个模块对数据进行缓存，并设置缓存时间

    ```javascript
    app.get('*', (req, res) => {
      // 等待编译
      if (!renderer) {
        return res.end('waiting for compilation... refresh in a moment.');
      }

      var s = Date.now();
      const context = { url: req.url };
      // 渲染我们的Vue实例作为流
      const renderStream = renderer.renderToStream(context);
        
      // 当块第一次被渲染时
      renderStream.once('data', () => {
          // 将预先的HTML写入响应
        res.write(indexHTML.head);
      });
        
      // 每当新的块被渲染
      renderStream.on('data', chunk => {
          // 将块写入响应
        res.write(chunk);
      });
        
      // 当所有的块被渲染完成
      renderStream.on('end', () => {
        // 当vuex初始状态存在
        if (context.initialState) {
            // 将vuex初始状态以script的方式写入到页面中
          res.write(
            `<script>window.__INITIAL_STATE__=${
              serialize(context.initialState, { isJSON: true })
            }</script>`
          );
        }
        
        // 将结尾的HTML写入响应
        res.end(indexHTML.tail);
      });
        
      // 当渲染时发生错误
      renderStream.on('error', err => {
        if (err && err.code === '404') {
          res.status(404).end('404 | Page Not Found');
          return;
        }
        res.status(500).end('Internal Error 500');
      });
    })
    ```

