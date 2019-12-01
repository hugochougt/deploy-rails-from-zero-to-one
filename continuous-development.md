# 持续开发

现在我们打开网站首页是会出现 404 页面的，因为我们没有配置首页路由。

## 配置首页路由

编辑 `config/routes.rb` 文件，在第二行加上 `root to: "posts#index"`

```ruby
Rails.application.routes.draw do
  root to: "posts#index"
  resources :posts
  # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html
end
```

保存文件并提交修改，推送到远程仓库并部署代码。

```
$ git add config/routes.rb
$ git commit -m "Add root route"
$ git push origin master
$ cap production deploy
```

部署成功后访问 `https://deploy-demo.zq-dev.com/` 就是 posts 列表页。

后续有任何的代码更新，只要把代码提交到仓库的 master 分支，再执行 `cap production deploy` 就可以部署新的变动。
