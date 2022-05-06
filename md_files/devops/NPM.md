### NPM

#### npm script

npm 允许在`package.json`文件里面，使用`scripts`字段定义脚本命令。

npm 脚本的原理非常简单。每当执行`npm run`，就会自动新建一个 Shell，在这个 Shell 里面执行指定的脚本命令。

通常，我们定义`build script`来打包编译js文件

Usually `npm run build` will create a production build.

The build process does a lot of things for you:

- transpiles JS code
- bundles code and assets
- uses cache busting techniques for assets
- removes dead code

Using the production build is the way to go for production.

`npm run build`打包成功后，会在dist目录下生成index.html和`static`文件夹，将dist下所有文件复制到你需要的目录下。

但是，如果`package.json`中没有定制`build`就好报错。

```bash
bash-4.2# npm  run build
npm ERR! missing script: build

npm ERR! A complete log of this run can be found in:
npm ERR!     /home/jenkins/.npm/_logs/2022-04-29T08_23_57_654Z-debug.log
bash-4.2# cat package.json
{
  "name": "npm-demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "mongoose": "^5.9.10",
    "underscore": "^1.10.2"
  }
}
bash-4.2# npm run
Lifecycle scripts included in npm-demo:
  test
    echo "Error: no test specified" && exit 1

```

一般来说，npm 脚本由用户提供。但是，npm 对两个脚本提供了默认值。也就是说，这两个脚本不用定义，就可以直接使用。

```bash
"start": "node server.js"，
"install": "node-gyp rebuild"
```

上面代码中，`npm run start`的默认值是`node server.js`，前提是项目根目录下有`server.js`这个脚本；`npm run install`的默认值是`node-gyp rebuild`，前提是项目根目录下有`binding.gyp`文件。

##### demo

使用`vue create hello-world`初始化一个demo可以查看package.json怎么写的。

```json
  "scripts": {
    "serve": "vue-cli-service serve",
    "build": "vue-cli-service build",
    "lint": "vue-cli-service lint"
  },

```

end

#### npm install

`npm install` (in a package directory, no arguments):

Install the dependencies to the local `node_modules` folder.

In global mode (ie, with `-g` or `--global` appended to the command), it installs the current package context (ie, the current working directory) as a global package.

By default, `npm install` will install all modules listed as dependencies in [`package.json`](https://docs.npmjs.com/cli/v8/configuring-npm/package-json).

With the `--production` flag (or when the `NODE_ENV` environment variable is set to `production`), npm will not install modules listed in `devDependencies`. To install all modules listed in both `dependencies` and `devDependencies` when `NODE_ENV` environment variable is set to `production`, you can use `--production=false`.

> NOTE: The `--production` flag has no particular meaning when adding a dependency to a project.

##### global

#### `global`

{prefix}就是node安装时指定的目录。

Operates in "global" mode, so that packages are installed into the `prefix` folder instead of the current working directory. See [folders](https://docs.npmjs.com/cli/v8/configuring-npm/folders) for more on the differences in behavior.

- packages are installed into the `{prefix}/lib/node_modules` folder, instead of the current working directory.
- bin files are linked to `{prefix}/bin`
- man pages are linked to `{prefix}/share/man`



#### npm config

npm gets its config settings from the command line, environment variables, `npmrc` files, and in some cases, the `package.json` file.

使用格式，必须是key-value模式才生效。

`npm config set <key> <value> [-g|--global]`

##### `.npmrc`

`.npmrc`可以理解成npm running cnfiguration, 即npm运行时配置文件.

在项目的根目录下新建 .npmrc 文件，在里面以 **key=value** 的格式进行配置。比如要把npm的源配置为淘宝源，可以参考一下代码：

```bash
# .npmrc
## npm 私服地址
registry=https://IP/repository/public/
## npm neuxs授权信息
_auth=<加密密钥>

## node-sass binary path （默认从github下载）
## Node-sass是一个库，它将Node.js绑定到LibSass（流行样式表预处理器Sass的C版本）
sass_binary_path=/root/node-sass/
```

如果你想删除一些配置，可以直接把对应的代码行给删除

npm按照如下顺序读取这些配置文件：

1. 项目配置文件：你可以在项目的根目录下创建一个.npmrc文件，只用于管理这个项目的npm安装。
2. 用户配置文件：在你使用一个账号登陆的电脑的时候，可以为当前用户创建一个.npmrc文件，之后用该用户登录电脑，就可以使用该配置文件。可以通过 **npm config get userconfig** 来获取该文件的位置。
3. 全局配置文件： 一台电脑可能有多个用户，在这些用户之上，你可以设置一个公共的.npmrc文件，供所有用户使用。该文件的路径为：**$PREFIX/etc/npmrc**，使用 **npm config get prefix** 获取$PREFIX。如果你不曾配置过全局文件，该文件不存在。
4. npm内嵌配置文件：最后还有npm内置配置文件，基本上用不到，不用过度关注。

```bash
bash-4.2# npm config get prefix
/var/jenkins_ext/tools/nodejs
bash-4.2# cat /var/jenkins_ext/tools/nodejs/
cat: /var/jenkins_ext/tools/nodejs/: Is a directory
bash-4.2# cat /var/jenkins_ext/tools/nodejs/
CHANGELOG.md  LICENSE       README.md     bin/          include/      lib/          share/
bash-4.2# cat /var/jenkins_ext/tools/nodejs/
```

end

官方说明：

npm gets its config settings from the command line, environment variables, and `npmrc` files.

The `npm config` command can be used to update and edit the contents of the user and global npmrc files.

For a list of available configuration options, see [config](https://docs.npmjs.com/cli/v8/using-npm/config).

Files

The four relevant files are:

- per-project config file (/path/to/my/project/.npmrc)
- per-user config file (~/.npmrc)
- global config file ($PREFIX/etc/npmrc)
- npm builtin config file (/path/to/npm/npmrc)

All npm config files are an ini-formatted list of `key = value` parameters. Environment variables can be replaced using `${VARIABLE_NAME}`.

##### `_auth`：有点意思

```bash
; Nexus proxy registry pointing to http://registry.npmjs.org/
registry = https://<host>/nexus/content/repositories/npmjs-registry/ 

; base64 encoded authentication token
_auth = <see question below>

; required by Nexus
email = <valid email>

; force auth to be used for GET requests
always-auth = true
```

相当于是对私有npm库的user-password进行加密保护。

`base64Encode(<username>:<password>)` 或者`$ echo -n 'username:password' | openssl base64`

#### node-gyp

https://github.com/tsy77/blog/issues/5

node-gyp是一个跨平台的命令行工具，目的是编译`node addon`模块。

NodeJS基于v8 JavaScript引擎，然后v8是基于C++语言写成的，所以Node可以通过C++来进行扩展，也就是增加所谓的Addon。

`node-gyp` - Node.js native addon build tool.

`node-gyp` is a cross-platform command-line tool written in Node.js for compiling native addon modules for Node.js. It contains a vendored copy of the [gyp-next](https://github.com/nodejs/gyp-next) project that was previously used by the Chromium team, extended to support the development of Node.js native addons.

Note that `node-gyp` is *not* used to build Node.js itself.、

C/C++对比javascript在位运算上具有极大优势，很多转码、编码的功能可以用C/C++扩展来提升性能。

C++模块通过预先编译为.node文件，然后调用process.dlopen() 加载执行。.node文件实际上在不同平台下是不一样的。如图。

```bash
*nix                                |           windows
                                 C/C++源码
g++/gcc编译成.node文件(.so文件)       |          VC++编译成.node文件(.dll文件)
                            dlopen加载.node文件导出给javascript
```

gyp的意思是generate your projects。node-gyp是一个node的扩展构建工具，通过`npm install -g node-gyp`安装。

node-gyp首先要知道什么是gyp([https://gyp.gsrc.io/index.md](https://link.zhihu.com/?target=https%3A//gyp.gsrc.io/index.md))。gyp其实是一个用来生成项目文件的工具，一开始是设计给chromium项目使用的，后来大家发现比较好用就用到了其他地方。生成项目文件后就可以调用GCC, vsbuild, xcode等编译平台来编译。至于为什么要有node-gyp，是由于node程序中需要调用一些其他语言编写的工具甚至是dll，需要先编译一下，否则就会有跨平台的问题，例如在windows上运行的软件copy到mac上就不能用了，但是如果源码支持，编译一下，在mac上还是可以用的。

node-gyp是一个跨平台的命令行工具，目的是编译`node addon`模块。

NodeJS基于v8 JavaScript引擎，然后v8是基于C++语言写成的，所以Node可以通过C++来进行扩展，也就是增加所谓的Addon。



##### avoid node-gyp downloading tgz

You can pass `--nodedir=/path/to/node/headers` to `npm install`, which should avoid downloading.

在安装和构建 native 模块(例如iconv，ref，ffi等)时，node-gyp从Internet下载以下文件:
node-v6.10.0-headers.tar.gz、node.lib等

如何使node-gyp使用本地文件夹(而不是Internet)中的这些文件？

利用`npm --nodedir`的解决方案:
1.下载node-v6.10.0-headers.tar.gz
2.将其解压缩到某些本地文件夹中或者代码目录。

> 因为这个可以能是个动态的构建pod，只能随着代码

3.在此本地文件夹中创建文件夹Release。
4.将文件node-v6.10.0-headers.tar.gz下载到文件夹Release中。
5.在`.npmrc`中设置属性nodedir，该属性将指向带有解压头的文件夹:
`nodedir = /node_src/node-v6.10.0-headers`

##### metrics-registry: 有点意思

```bash
### 这个就是错误的
### 这里将http://10.156.23.49:8081/repository/hbzynpm/ 作为key了....
npm config set http://10.156.23.49:8081/repository/hbzynpm/
npm config list
; cli configs
metrics-registry = "https://registry.npmjs.org/"
scope = ""
user-agent = "npm/6.9.0 node/v10.16.1 linux x64"
; project config /home/jenkins/workspace/job_1442316350088282112/.npmrc
nodedir = "/root/dep/node-v10.16.1-headers/node-v10.16.1/include/node"
sass_binary_path = "/root/node-sass/linux_musl-x64-57_binding.node"
; userconfig /home/jenkins/.npmrc
http://10.156.23.49:8081/repository/hbzynpm/ = ""
; node bin location = /var/jenkins_ext/tools/nodejs/bin/node
; cwd = /home/jenkins/workspace/job_1442316350088282112
; HOME = /home/jenkins
; "npm config ls -l" to show all defaults.

=== metrics-registry
npm config set http://10.156.23.49:8081/repository/hbzynpm/
npm config set registry http://10.156.23.49:8081/repository/hbzynpm/
npm config list
; cli configs
metrics-registry = "http://10.156.23.49:8081/repository/hbzynpm/"
scope = ""
user-agent = "npm/6.9.0 node/v10.16.1 linux x64"
; project config /home/jenkins/workspace/job_1442316350088282112/.npmrc
sass_binary_path = "/root/node-sass/linux_musl-x64-57_binding.node"
; userconfig /home/jenkins/.npmrc
http://10.156.23.49:8081/repository/hbzynpm/ = ""
registry = "http://10.156.23.49:8081/repository/hbzynpm/"
; node bin location = /var/jenkins_ext/tools/nodejs/bin/node
; cwd = /home/jenkins/workspace/job_1442316350088282112
; HOME = /home/jenkins
; "npm config ls -l" to show all defaults.

```

end

##### sass组件

自己编译sass时，还会提示Python报错。

原因：提示没有安装python、build失败，如果拉取binding.node失败，node-sass会尝试在本地编译binding.node，过程就需要用到python

ISV是在编译机器上预先下载了`sass`组件，然后通过`sass_binary_path`引用了`sass`。

```bash
# .npmrc
## npm 私服地址
registry=https://IP/repository/public/
## npm neuxs授权信息
_auth=<加密密钥>

## node-sass binary path （默认从github下载）
## Node-sass是一个库，它将Node.js绑定到LibSass（流行样式表预处理器Sass的C版本）
sass_binary_path=/root/node-sass/
```

自己构建时，可以设置`sass_binary_site `指向国内`sass`镜像站。

```bash
npm config set sass_binary_site https://npm.taobao.org/mirrors/node-sass
```

 end

#### 下载npm项目依赖

https://levelup.gitconnected.com/deploying-private-npm-packages-to-nexus-a16722cc8166

##### 思路出错了？

思路:

1. 在可以访问外面的机器上`npm install`，下载全部依赖到`node_modules`
2. 将`node_modules`拷贝到内网机器上，然后通过脚本将`dir/package.json`推送到内网
3. 默认`npm publish --registry=URL `是推送当前目录 

```bash
for i in $(ls ./node_modules/);
do
  echo $i
  cd  ./node_modules/$i
  npm publish --registry=http://172.31.212.222:18081/repository/test-npm/
  cd ../..
done
```

正确解法：

还是通过tgz包上传，无论是通过wget还是download-tgz从package-lock.josn下载依赖包，都要保障数量和版本都够...

##### package-lock.json

1、锁定安装时的包的版本号，需要上传到git，**保证大家的依赖包一致**。

2、package-lock.json 是在 `npm install`时候生成一份文件，用来记录当前状态下实际安装的各个npm package的具体来源和版本号。

3、它有什么用呢？因为npm是一个用于管理package之间依赖关系的管理器，它允许开发者在pacakge.json中间标出自己项目对npm各库包的依赖。你可以选择以如下方式来标明自己所需要库包的版本；例如：

```javascript
"dependencies": {
 "@types/node": "^8.0.33",
},
```

　　这里面的 向上标号**^**是定义了**向后（新）兼容依赖**，指如果 types/node的版本是超过8.0.33，并在大版本号（8）上相同，就允许下载最新版本的 types/node库包，例如实际上可能运行npm install时候下载的具体版本是8.0.35。

原来package.json文件只能锁定大版本，也就是版本号的第一位，并不能锁定后面的小版本，你每次npm install都是拉取的该大版本下的最新的版本，为了稳定性考虑我们几乎是不敢随意升级依赖包的，这将导致多出来很多工作量，测试/适配等，所以package-lock.json文件出来了，当你每次安装一个依赖的时候就锁定在你安装的这个版本。

安装依赖出问题的解决方式不同：

　　那如果我们安装时的包有bug，后面需要更新怎么办？

以前：在以前可能就是直接改package.json里面的版本，然后再npm install了。

现在：但是5版本后就不支持这样做了，因为版本已经锁定在package-lock.json里了，所以我们只能npm install xxx@x.x.x  这样去更新我们的依赖，然后package-lock.json也能随之更新。

###### resolved

```json
{
    "resolved": "https://registry.npmjs.org/camelcase/-/camelcase-4.1.0.tgz"
}
```

定义了npm tgz包从哪个仓库下载，纵然`npm config`指定了`registry`，也无法改变npm下载位置

```bash
bash-4.2# npm config set registry http://172.31.212.222:18081/repository/npm-public/
bash-4.2# npm install --unsafe-perm=true --allow-root
npm ERR! code E404
npm ERR! 404 Not Found - GET https://registry.npmjs.org/@types/q/-/q-1.5.5.tgz
npm ERR! 404 
npm ERR! 404  '@types/q@https://registry.npmjs.org/@types/q/-/q-1.5.5.tgz' is not in the npm registry.
npm ERR! 404 You should bug the author to publish it (or use the name yourself!)
npm ERR! 404 
npm ERR! 404 Note that you can also install from a
npm ERR! 404 tarball, folder, http url, or git url.

npm ERR! A complete log of this run can be found in:
npm ERR!     /home/jenkins/.npm/_logs/2022-05-05T15_41_16_712Z-debug.log
bash-4.2# 
```

end



###### integerity

integrity字段中回校验repo中npm tgz包和package-lock.json中是否记录的一样，如果不一致回报错。

解决办法：删除`integerity`字段即`:%g/integrity/d`

或者直接删除`package-lock.json`

```bash
"camelcase": {
          "version": "4.1.0",
          "resolved": "https://registry.npmjs.org/camelcase/-/camelcase-4.1.0.tgz",
          "integrity": "sha1-1UVjW+HjPFQmScaRc+Xeas+uNN0=",
          "dev": true
        }
```

end

##### 下载tgz

npm install 

先安装node-tgz-downloader

`npm install node-tgz-downloader -g`

然后执行

注意，必须执行`download-tgz package-lock package-lock.json`

因为有些package依赖不同的版本。

```bash
# 生产package-lock.json文件
$ npm install or npm install --package-lock-only
$ download-tgz package package.json
$ download-tgz package-lock package-lock.json
## download-tgz下载出差时，wget手动下载
$ download-tgz package-lock package-lock.json >> /tmp/log.txt
$ cat /tmp/log.txt|awk '{print $5}'|xargs wget
```

Downloads all of the tarballs based on one of the following:

- local `package-lock.json` file
- url to a `package-lock.json`
- name of package
- local `package.json` file
- url to a `package.json`
- search keyword

这个命令会根据package.json/package-lock.json文件，下载所需要的依赖包tgz，如果存在下载失败的情况，则多执行几次命令，直到所有依赖都下载完成。

下载的tgz文件会在项目根目录/tarballs下，这个时候基本上就完成了tgz的下载。

##### Nexus realm

https://blog.csdn.net/lqh4188/article/details/107384465

脚本批量推送tgz包时，报错报401  BASIC realm="Sonatype Nexus Repository Manager"。

```bash
$ npm login --registry=$REPOSITORY
npm ERR! code E401
npm ERR! Unable to authenticate, need: BASIC realm="Sonatype Nexus Repository Manager"

npm ERR! A complete log of this run can be found in:
npm ERR!     /home/jenkins/.npm/_logs/2022-04-29T09_12_53_149Z-debug.log

```

这个一般是 npm publish发包才会有此问题，npm publish时需要有本地仓库的权限，一般登录一下就可以解决。登录用  npm login    输入nexus上创建的用户、密码和邮箱就可以了

如果登录后还不能发，检查npm nexus的 Realms设置，把npm Bearer Token Reaim放入Active中，并保存。

![](https://image-1300760561.cos.ap-beijing.myqcloud.com/bgyq-blog/nexus-realm.png)



##### 上传tgz到nexus

1. 为了跟下面的脚本保持一直，将所有tgz都放到一个目录下，新建tgz文件夹，并在tarballs文件夹下执行下面的shell命令，这里用shell脚本找出tarballs文件夹下所有tgz包并复制到tgz文件夹下（这里都放到一个目录下是因为当有多个工程时，依赖包可能会重复，发布时tgz有重复则会报错，所以是一个去重的操作）。

   ```bash
   find . -name '*.tgz' -exec cp -f {} ../tgz \;
   ```

2. 创建发布脚本（单个工程时可利用find命令改动下面的脚本，省去第一步）

   ```bash
   #!/bin/bash
   
   PACKAGE_PATH=./tgz
   REPOSITORY=http://ip:port/repository/dataservice-web/
   
   npm login --registry=$REPOSITORY
   # 根据实际情况改动
   for file in $PACKAGE_PATH/*.tgz; do
    npm publish --registry=$REPOSITORY $file
   done
   ```

3. 执行发布脚本，会要求填写username，password以及email，填写完就会开始发布

##### devDependencies还是dependencies

？？？

#### @package_name

npm发布自己的包时，一直很疑惑，为啥@angular、@ionic他们的包， 都可以以@开头，为啥我的不可以，原来angular、ionic都属于一个组织（Organization）只有新创建一个Organization组织之后，才能创建@testorg/testpackname这样的包！！！

如果想发布的是个人的公共包即以@开头的包，需要使用以下命令：

By default all scoped packages are published privately. To publish a scoped package publicly, pass the `access` flag with the value `public`:

```bash
npm publish --access public
```


end



#### npm配置仓库优先级优: 从高到低

下面从优先级高到低的顺序来介绍一下各配置。

##### 命令行

```
> npm run commend --proxy http://server:port
```

命令行中将`proxy`的值设为`http://server:port`。

##### 环境变量

以`npm_config_`为前缀的环境变量会被识别为npm的配置属性。如设置proxy。

```
npm_config_proxy=http://server:port
```

##### 项目.npmrc文件

存在于项目根目录下的.npmrc配置文件`/path/to/project/.npmrc`。

##### 用户.npmrc文件

存在于用户根目录下的.npmrc文件。如windows下是`%USERPROFILE%/.npmrc`，MAC下是`$HOME/.npmrc`。

##### 全局.npmrc文件

存在于Node全局的.npmrc文件。如windows下`$PREFIX/etc/.npmrc`，MAC下是`%APPDATA%/etc/.npmrc`。

##### npm内置的.npmrc文件

存在于npm包的内置.npmrc文件`/path/to/npm/.npmrc`。

##### npm的默认配置

npm本身有默认配置。对于以上情况下都没有设置的配置，npm会使用默认配置。

### 引用

1. https://www.zhihu.com/question/36291768/answer/318429630
1. https://juejin.cn/post/7042124308050624525