### NPM

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