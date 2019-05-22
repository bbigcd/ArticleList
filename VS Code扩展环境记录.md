## 扩展

* [Chinese (Simplified) Language Pack for Visual Studio Code](<https://marketplace.visualstudio.com/items?itemName=MS-CEINTL.vscode-language-pack-zh-hans>) VS Code的中文语言包
* [vscode-icons](<https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons>) 图标
* [ESLint](<https://marketplace.visualstudio.com/items?itemName=dbaeumer.vscode-eslint>) JavaScript代码规范
* [Vetur](<https://marketplace.visualstudio.com/items?itemName=octref.vetur>) Vue tooling for VS Code
* [C#](<https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp>) C#
* [One Dark Pro](<https://marketplace.visualstudio.com/items?itemName=zhuangtongfa.Material-theme>) 黑色主题
* [Beautify](<https://marketplace.visualstudio.com/items?itemName=HookyQR.beautify>) 代码格式化工具

## 环境配置

* git.path

  git的安装路径，有默认路径，该扩展是系统的拓展，可以自定义git安装路径

  control+，进入配置修改页，输入git.path，将下方配置写入配置文件中:

  ```json
  {
      "git.path":"D:\\Program Files\\Git\\bin\\git.exe"
  }
  ```

  重启vscode，源代码插件即可正常操作

* eslint 保存自动格式化

  安装 ESLint 和 Vetur 插件，在配置文件中写入下方配置，保存即可

  ```json
  {
      "eslint.autoFixOnSave": true,
      "eslint.options": {
          "extensions": [
              ".js",
              ".vue"
          ]
      },
      "eslint.validate": [
          "javascript",
          {
              "language": "vue",
              "autoFix": true
          },
          "html",
          "vue"
      ]
  }
  ```

  

