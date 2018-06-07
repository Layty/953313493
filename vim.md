先查看下用户配置：`$vi ~/.vimrc `

插件路径：` ./home/book/.vim_runtime/sources_non_forked/`

配置插件路径`/home/book/.vim_runtime/vimrcs`   `~/.vim_runtime/vimrcs/plugins_config.vim`

wiki上的虚拟机包mru路径创建权限没有,那么在配置文件中更改，添加一句

```
sudo 
```

```
""""""""""""""""""""""""""""""
" => MRU plugin
""""""""""""""""""""""""""""""
let MRU_File='/tmp/mru_files'  "------------------这句是加的
let MRU_Max_Entries = 400
map <leader>f :MRU<CR>
```

关于mru的一些其他配置：`http://it.taocms.org/06/712.htm`



