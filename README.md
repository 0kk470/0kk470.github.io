# My Personal blog

* batchfile for jenkins
```bat
echo  update blog
if exist node_modules\ (
  echo skip install npm_module 
) else (
  call npm install
)

call hexo clean

call hexo g
````