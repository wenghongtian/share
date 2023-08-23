# 实现一个vscode的gpt插件

### 简单介绍

1. ##### 搭建vscode开发环境

   ```bash
   $ pnpm install -g yo generator-code
   $ yo code
   #     _-----_     ╭──────────────────────────╮
   #    |       |    │   Welcome to the Visual  │
   #    |--(o)--|    │   Studio Code Extension  │
   #   `---------´   │        generator!        │
   #    ( _´U`_ )    ╰──────────────────────────╯
   #    /___A___\   /
   #     |  ~  |
   #   __'.___.'__
   # ´   `  |° ´ Y `
   
   #? What type of extension do you want to create? New Extension (TypeScript)
   #? What's the name of your extension? auto-input
   #? What's the identifier of your extension? auto-input
   #? What's the description of your extension? auto input
   #? Initialize a git repository? Yes
   #? Bundle the source code with webpack? Yes
   #? Which package manager to use? yarn
   ```

   此时按`f5`就能启动开发环境了

2. ##### 下面我们来到`src/extension.ts`文件，可以看到一下内容

   ```ts
   // Import the module and reference it with the alias vscode in your code below
   import * as vscode from 'vscode';
   
   // 扩展被激活时，会调用此方法
   export function activate(context: vscode.ExtensionContext) {
   	//...
   }
   
   export function deactivate() {}
   ```

3. ##### 如何激活插件？？？

   我们把关注点放到`package.json`

   ```json
   {
     "activationEvents": ["*"],
     //...
   }
   ```

   可以看到`json`文件中有一个`activationEvents`字段，我们可以往里面注册我们希望触发插件激活的多个事件

   * onLanguage:${language}  例如`onLanguate:javascript`，在文件时javascript文件时插件会被激活
   * onCommand:$(command) 调用命令时，插件被激活

   * onDebug  调试阶段被激活

   * workspaceContains:${toplevelfilename} 当打开文件夹并且该文件夹包含至少一个与 glob 模式匹配的文件时，插件被激活。
   * \* vscode启动，插件就被激活

   [更多事件](https://code.visualstudio.com/api/references/activation-events)

   > 在这里我们希望vscode启动，插件就被激活，因此使用*

4. ##### 开发第一个插件

   src/extension.ts

   ```ts
   //...
   export function activate(context: vscode.ExtensionContext) {
   	// 注册一个命令
   	let disposable = vscode.commands.registerCommand('auto-input.helloWorld', () => {
       // 当命令被调用时弹出一个hello world弹窗
   		vscode.window.showInformationMessage('Hello World from auto-input!');
   	});
   
   	context.subscriptions.push(disposable);
   }
   //...
   ```

   package.json

   ```ts
   {
     //...
     "contributes": {
       "commands": [
         {
           "command": "auto-input.helloWorld",
           "title": "Hello World"
         }
       ]
     },
     //...
   }
   ```

   按下`f5`，按下`command+shift+p`输入`hello world`可以看到
   ![image-20230417095928258.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1a0677a299b2414ca0216984423cb9f5~tplv-k3u1fbpfcp-watermark.image?)
		点一下就可以看到弹窗了。

5. 构建并发布vscode应用
   1. 构建应用
      ```bash
      $ yarn add @vscode/vsce
      ```
      
      在`package.json`中更改`sciprts`字段，并添加`icon`
      
      ```json
      {
        //...
        	"icon": "images/insnail.png",
          "scripts": {
          "vscode:prepublish": "pnpm run package",
          "compile": "webpack",
          "watch": "webpack --watch",
          "package": "webpack --mode production --devtool hidden-source-map",
          "compile-tests": "tsc -p . --outDir out",
          "watch-tests": "tsc -p . -w --outDir out",
          "pretest": "pnpm run compile-tests && pnpm run compile && pnpm run lint",
          "lint": "eslint src --ext ts",
          "test": "node ./out/test/runTest.js",
      +   "vscode:package": "vsce package",
      +   "vscode:publish": "vsce publish"
        },
        //...
      }
      ```
      
      ```bash
      $ yarn vscode:package
      
      #> auto-input@0.0.1 package /Users/wenghongtian/Desktop/auto-input
      #> webpack --mode production --devtool hidden-source-map
      #
      #    [webpack-cli] Compiler starting... 
      #    [webpack-cli] Compiler is using config: '/Users/wenghongtian/Desktop/auto-#input/webpack.config.js'
      #    [webpack-cli] Compiler finished
      #asset extension.js 608 bytes [emitted] [minimized] (name: main) 1 related asset
      #./src/extension.ts 1.4 KiB [built] [code generated]
      #external "vscode" 42 bytes [built] [code generated]
      #webpack 5.79.0 compiled successfully in 1360 ms
      ```
      
      执行完后我们会发现一个错误
      
      ```bash
      # ERROR  It seems the README.md still contains template text. Make sure to edit the README.md file before you package or publish your extension.
      ```
      
      这时需要我们去更改我们的`README.md`文件，就可以构建成功了，可以看到我们项目目录下有一个`auto-input-0.0.1.vsix`文件
      
      ```markdown
      # auto-input README
      
      一个gpt自动输入插件
      ```

   2. 发布vscode扩展
      1. 注册一个[azure](https://dev.azure.com)账号，生成一个token(还可以免费白嫖免费1年的服务器)
        <img src="https://code.visualstudio.com/assets/api/working-with-extensions/publishing-extension/token1.png">
        完成后把生成的token保存一下，回到我们的项目里
        package.json
        
        ```json
        {
        	//...,
          publisher: "$(token名字)"
        }
        ```
        
        命令行输入下面这个，然后一路'y'，直到遇到输入`token`，把刚才保存的token粘贴进去，过几分钟你就会从扩展市场搜到自己的插件了
        
        ```bash
        $ yarn vscode:publish
        ```
## 进入正题

1. 目标

   在代码中输入`//@insnail $(action)`时能自动输入对应的代码。

2. 开发准备

   本插件开发中使用到的`vscode`api

   * `vscode.EventEmitter` 创建一个vscode事件

   * `vscode.window.activeTextEditor`获取当前激活的编辑窗口
   * `vscode.workspace.onDidChangeTextDocument`监听文本变化
   * `vscode.window.createStatusBarItem`在状态栏创建一个子项
   * `window.showErrorMessage`展示状态信息
   * `vscode.commands.registerCommand`注册vscode指令

   对接gpt3.5(AI Prompt工程师)

   ​	由于官方库`openai`在vscode打包时会导致类型错误，导致插件打包失败，因此可以使用`axios`自己手动发送请求

   [请求参数](https://platform.openai.com/docs/api-reference/completions/create)

   ```ts
   import { sleep } from './utils';
   import config from './config';
   import axios, { Canceler } from 'axios';
   import { httpsOverHttp } from 'tunnel';
   
   //data {"id":"cmpl-79WzDsXLjyQzOE91WRi70faSknLfD","object":"text_completion","created":1682506107,"choices":[{"text":"\n","index":0,"logprobs":null,"finish_reason":null}],"model":"text-davinci-003"}
   
   // 让axios发送的请求走我们的代理服务器，
   const tunnelAgent = httpsOverHttp({
   	proxy: {
   		host: config.PROXY_HOST,
   		port: config.PROXY_PORT,
   	}
   });
   
   const instance = axios.create({
   	headers: {
   		Authorization: `Bearer ${config.API_KEY}`,
   	},
   	httpsAgent: tunnelAgent,
   });
   
   for ()
   
   export function generateCodeAsStream(prompt: string) {
   	let done = false;
   	let err: Error | null = null;
   	let cancael: Canceler;
   
   	const completionPromise = instance.post('https://api.openai.com/v1/completions', {
   		model: 'text-davinci-003',
   		stream: true,
       // 采样温度：可选值0-2，值越小越精确
   		temperature: 0.2,
   		frequency_penalty: 0,
   		max_tokens: 1000,
       //一种与温度采样相对应的新方法——核采样。在核采样中，模型会考虑到概率质量函数中前top_p个概率最高的标记。例如，top_p为0.1表示只有包含前10%概率质量的标记会被考虑
   		top_p: 0.1,
   		prompt,
   	}, {
   		cancelToken: new axios.CancelToken(c => {
   			cancael = c;
   		}),
   		responseType: 'stream',
   	});
   
   	
   	const iterator = {
   		next: async function* () {
   			let gotFirstChar = false;
   			const codes: string[] = [];
   			const completion = await completionPromise;
   			(completion.data as any).on('data', (data: Buffer) => {
   				const lines = data.toString().split('\n').filter(line => line.trim() !== '');
   				for (const line of lines) {
             // 解析流数据
   					const message = line.replace(/^data: /, '');
   					if (message.startsWith('[DONE]')) {
   						done = true;
   						return;
   					}
   					try {
   						const parsed = JSON.parse(message);
   						const char = parsed.choices?.[0]?.text || '';
   						if (char === '\n' && !gotFirstChar) {
   							continue;
   						} else {
   							gotFirstChar = true;
   							codes.push(char);
   						}
   					} catch (e: unknown) {
   						err = e as Error;
   						done = true;
   					}
   				}
   			});
   			(completion.data as any).on('error', (e: Error) => {
   				err = e;
   			});
   			(completion.data as any).on('end', (data: Buffer) => {
   				done = true;
   			});
   
   			while (err == null && (!done || codes.length)) {
   				if (err !== null) {
   					throw err;
   				}
   				if (codes.length) {
   					await sleep();
   					yield codes.shift();
   					continue;
   				}
   				if (!codes.length && !done) {
   					await sleep();
   				}
   			}
   		},
   		abort() {
   			cancael?.();
   		},
   	};
   	return iterator;
   }
   ```

   

   

3. 代码思路设计-事件驱动
 ![image-20230417110745952.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2b024154aad4329b5ffa840bdab5b26~tplv-k3u1fbpfcp-watermark.image?)
  1. * 类型定义
  
       ```ts
       import { Event } from "vscode";
       
       // 事件中枢约束
       export interface IGPTAutoInputPlus {
       	readonly onLogin: Event<GPTLoginEvent>;
       	readonly onDidGPTOnline: Event<GPTOnlineEvent>;
       	readonly onDidGPTOffline: Event<GPTOfflineEvent>;
       	readonly onGPTStopOutput: Event<GPTStopOutputEvent>;
       	readonly onGPTGenerating: Event<GPTGeneratingEvent>;
       	readonly onGPTGenerated: Event<GPTGeneratedEvent>;
         
       	readonly shutdown: () => void;
       	readonly stopOutput: () => void;
       	readonly fireGenerating: () => void;
       	readonly fireGenerated: () => void;
       }
       
       export interface GPTAIPEvent {
       	readonly GAIP: IGPTAutoInputPlus;
       }
       
       export type GPTOnlineEvent = GPTAIPEvent;
       export type GPTOfflineEvent = GPTAIPEvent;
       export type GPTGeneratingEvent = GPTAIPEvent;
       export type GPTGeneratedEvent = GPTAIPEvent;
       export type GPTLoginEvent = GPTAIPEvent;
       export type GPTStopOutputEvent = GPTAIPEvent;
       
       // 服务注册约束
       export interface IGPTAutoInputPlusService {
       	register(): void;
       }
       
       // 服务构造函数约束
       export interface IGPTAutoInputPlusServiceCtor {
       	new(liveServerPlusPlus: IGPTAutoInputPlus): IGPTAutoInputPlusService;
       }
       ```
  
     * 实现事件中枢
  
       ```ts
       import { Disposable, EventEmitter } from "vscode";
       import { GPTGeneratingEvent, GPTLoginEvent, GPTOfflineEvent, GPTOnlineEvent, GPTStopOutputEvent, IGPTAutoInputPlus, IGPTAutoInputPlusServiceCtor } from "./types/IGPTAutoInputPlus";
       
       export default class GPTAutoInputPlus implements IGPTAutoInputPlus {
       	private loginEvent = new EventEmitter<GPTLoginEvent>();
       	private GPTOnlineEvent = new EventEmitter<GPTOnlineEvent>();
       	private GPTOfflineEvent = new EventEmitter<GPTOfflineEvent>();
       	private GPTGeneratingEvent = new EventEmitter<GPTGeneratingEvent>();
       	private GPTGeneratedEvent = new EventEmitter<GPTGeneratingEvent>();
       	private GPTStopOutputEvent = new EventEmitter<GPTStopOutputEvent>();
       
       	get onLogin() { return this.loginEvent.event; }
       	get onDidGPTOnline() { return this.GPTOnlineEvent.event; }
       	get onDidGPTOffline() { return this.GPTOfflineEvent.event; }
       	get onGPTGenerating() { return this.GPTGeneratingEvent.event; }
       	get onGPTGenerated() { return this.GPTGeneratedEvent.event; }
       	get onGPTStopOutput() { return this.GPTStopOutputEvent.event; }
       
         // 通过此方法进行service的注册，每个service可以订阅事件中枢的事件
       	useService(...fns: IGPTAutoInputPlusServiceCtor[]) {
       		fns.forEach(fn => {
       			const inst = new fn(this);
       			inst.register();
       		});
       	}
       
       	stopOutput() {
       		this.GPTStopOutputEvent.fire({ GAIP: this });
       	}
       
       	startup() {
       		this.GPTOnlineEvent.fire({ GAIP: this });
       	}
       
       	shutdown() {
       		this.GPTOfflineEvent.fire({ GAIP: this });
       	}
       
       	fireGenerating() {
       		this.GPTGeneratingEvent.fire({ GAIP: this });
       	}
       	fireGenerated() {
       		this.GPTGeneratedEvent.fire({ GAIP: this });
       	}
       }
       ```
  
     * 自动输入服务
  
       ```ts
       import { Disposable, window, workspace } from "vscode";
       import { IGPTAutoInputPlus, IGPTAutoInputPlusService } from "../core/types/IGPTAutoInputPlus";
       import { generateCodeAsStream } from "../openai";
       import { AxiosError } from "axios";
       import { showPopUpMsg } from "../utils";
       
       const regexp = /\/\/@insnail\s?(.*)/;
       
       export default class AutoInputService implements IGPTAutoInputPlusService {
       	private isOnline = false;
       	private textDispose?: Disposable;
       	private isStopOutput = false;
       	private stream: ReturnType<typeof generateCodeAsStream> | null = null;
       	constructor(private gptAutoInputPlus: IGPTAutoInputPlus) { }
       
       	register(): void {
       		this.gptAutoInputPlus.onDidGPTOnline(this.startup.bind(this));
       		this.gptAutoInputPlus.onDidGPTOffline(this.shutdown.bind(this));
       		this.gptAutoInputPlus.onGPTStopOutput(this.stopOutput.bind(this));
       	}
       
       	private stopOutput() {
       		this.isStopOutput = true;
       		this.stream?.abort();
       	}
       
       	private async writeCode(question: string) {
       		if (!question) { return; }
       		try {
       			const stream = generateCodeAsStream(question);
       			this.stream = stream;
       			const editor = window.activeTextEditor!;
       			for await (const code of stream.next()) {
       				if (!this.isOnline || this.isStopOutput) {
       					return;
       				}
       				await editor.edit(b => {
       					b.insert(editor.selection.active, code!);
       				});
       			}
       		} catch (err) {
       			if (err instanceof AxiosError) {
       				showPopUpMsg(err.message, { msgType: 'error' });
       				return;
       			}
       			throw err;
       		}
       	}
       
       	private startup() {
       		this.isOnline = true;
       		this.textDispose = workspace.onDidChangeTextDocument(async event => {
       			const change = event.contentChanges[event.contentChanges.length - 1];
       			const editor = window.activeTextEditor!;
       			const document = editor.document;
       			const languate = editor.document.languageId;
       			if (change?.text.includes('\n')) {
       				const cursorPosition = editor.selection.active;
       				const lineNumber = cursorPosition.line;
       				const lineText = document.lineAt(lineNumber).text;
       				const match = lineText.match(regexp);
       				const question = match?.[1];
       				if (!question) return;
       				this.gptAutoInputPlus.fireGenerating();
       				try {
       					await this.writeCode(`//${question}，以${languate}代码格式返回数据`);
       				} catch {
       					showPopUpMsg('Something error', { msgType: 'error' });
       				}
       				this.stream = null;
       				this.isStopOutput = false;
       				this.gptAutoInputPlus.fireGenerated();
       			}
       		});
       	}
       
       	private shutdown() {
       		this.isOnline = false;
       		this.dispose();
       	}
       
       	dispose() {
       		this.textDispose?.dispose();
       	}
       }
       ```
  
       
  
     * 注册vscode插件
  
       ```ts
       import * as vscode from 'vscode';
       import { COMMANDS } from './commands';
       import GPTAutoInputPlus from './core/GPTAutoInputPlus';
       import StatusbarService from './services/StatusbarService';
       import NotificationService from './services/NotificationService';
       import AutoInputService from './services/AutoInputService';
       
       export function activate(context: vscode.ExtensionContext) {
       	const gptAutoInputPlus = new GPTAutoInputPlus();
       	gptAutoInputPlus.useService(StatusbarService, NotificationService, AutoInputService);
         let stopOutputDisposor: vscode.Disposable;
       
       	const startGPT = vscode.commands.registerCommand(COMMANDS.START, () => {
       		gptAutoInputPlus.startup();
           stopOutputDisposor = vscode.commands.registerCommand(COMMANDS.STOP, () => {
       			gptAutoInputPlus.stopOutput();
       		});
       	});
       	const closeGpt = vscode.commands.registerCommand(COMMANDS.DOWN, () => {
       		gptAutoInputPlus.shutdown();
           stopOutputDisposor?.dispose();
       	});
       	context.subscriptions.push(startGPT, closeGpt, stopGptOutput);
       }
       ```



* [完整实现](https://git.woniubaoxian.com/wenghongtian/insnail-gpt-vscode)

      
