实时视频录制与上传系统技术文档

目录

一、项目介绍
1. 项目概述
2. 功能特性
3. 技术栈
4. 系统架构
5. 前端详细说明
6. 后端详细说明

二、配置与部署
7. 配置文件说明
8. 部署指南
   - 8.1环境准备
   - 8.2安装与启动
   - 8.3使用ngrok将服务暴露到公网
   - 8.4目录权限
9.扩展与自定义
10. 常见问题与故障排除


一、项目介绍

 1. 项目概述

本项目是一个基于 Web 的实时视频录制与上传系统。用户在浏览器中开启摄像头录制视频，系统将录制的视频流按时间分片（chunk），并立即上传至后端服务器。录制完成后，所有分片可在服务器端合并为一个完整的视频文件。项目采用前后端分离架构，前端负责媒体采集、分片上传和状态展示，后端负责接收分片、异步写入磁盘和最终合并。通过 ngrok 等内网穿透工具，可以将本地服务发布到公网，实现远程访问。

 2. 功能特性

 2.1 核心功能（具体参数可自行修改）
- 实时视频录制：调用摄像头和麦克风，固定分辨率 640×480、24FPS，生成 WebM 格式视频。
- 动态分片：根据上传队列长度动态调整分片间隔（1.5~3秒），避免内存溢出。
- 并发上传：队列长度控制并发数（4~15个），失败自动重试（最多3次）。
- 后端异步写盘：使用任务队列处理分片写入，最大并发 20，超时保护。
- 视频合并：按分片索引顺序合并为完整 WebM 文件，自动清理临时分片。

 2.2 辅助功能
- 实时状态监控：前端实时显示总分片数、已上传数、待上传数、失败数、丢弃数、并发数、分片间隔。
- 健康检查接口：后端提供 /health，返回临时目录文件数、写盘队列长度等监控数据。
- 移动端适配：界面响应式，支持添加到主屏幕，横竖屏切换提示。
- 公网访问支持：通过 ngrok 轻松将本地服务暴露到公网，方便远程演示或使用。

 2.3 容错与优化
- 前端队列溢出时自动丢弃最早未上传分片（保护内存）。
- 上传失败自动重试（指数退避）。
- 后端写盘队列超时、合并超时均返回友好错误。
- 全局异常捕获，防止服务器崩溃。

 3. 技术栈

 前端
- HTML5/CSS3：页面结构与样式，Flex 布局，圆角卡片设计。
- JavaScript (ES6+)：核心逻辑，使用 MediaDevices、MediaRecorder、fetch。
- Web API：Canvas（预留处理帧）、FormData、Blob。

 后端
- Node.js + Express 5：Web 服务器框架。
- express-fileupload：处理文件上传中间件。
- async：异步队列控制（写盘队列）。
- fs / path：文件系统操作。
- cors：跨域支持。

 工具与环境
- ngrok：内网穿透，提供公网访问地址。
- npm：包管理。

 4. 系统架构

mermaid
flowchart TB
    subgraph 前端
        A[浏览器页面] -->|getUserMedia| B[摄像头/麦克风流]
        B --> C[MediaRecorder]
        C -->|分片数据| D[上传队列]
        D -->|动态并发控制| E[上传分片]
        E -->|POST /upload-video-chunk| F[后端服务器]
    end

    subgraph 后端
        F -->|express-fileupload| G[接收分片]
        G -->|写入任务| H[async写盘队列]
        H -->|保存到临时目录| I[temp-chunks]
        I -->|合并请求 POST /merge-chunks| J[合并流程]
        J -->|等待队列清空| K[读取所有分片]
        K -->|按序合并| L[完整视频]
        L -->|存入 recorded-videos| M[最终视频]
        I -->|合并后删除分片| N[清理]
    end

    前端 -->|GET /health| F


 5. 前端详细说明

 5.1 页面结构
- 容器：居中卡片，包含标题、服务器配置输入框、视频预览元素、统计信息、控制按钮、状态栏。
- CSS：圆角设计、阴影、响应式按钮、状态颜色（错误/成功/录制中）。

 5.2 核心变量
javascript
let mediaStream, mediaRecorder;
let isRecording = false;
let totalChunks = 0, uploadedChunks = 0, failedChunks = 0, droppedChunks = 0;
let videoFilename = '';
let uploadQueue = [];               // 待上传分片队列
let activeUploads = 0;              // 当前活跃上传数
let currentConcurrency = 4;         // 动态并发数
let currentChunkInterval = 1500;    // 动态分片间隔（ms）
const MAX_QUEUE_LENGTH = 30;        // 队列最大长度，超出丢弃队首


 5.3 主要函数

 startRecording()
- 请求摄像头/麦克风权限，应用约束（640×480，24FPS）。
- 初始化 MediaRecorder，选择合适的 MIME 类型。
- 绑定事件（ondataavailable、onstop、onerror）。
- 开始录制，设置初始分片间隔。
- 更新 UI，显示录制状态。

 bindMediaRecorderEvents()
- ondataavailable：收到分片数据。
  - 过滤无效分片（重启产生的空分片、过小分片、重复分片）。
  - 检查队列是否达到 MAX_QUEUE_LENGTH，若达到则丢弃队首元素（droppedChunks++）。
  - 将新分片加入队尾。
  - 调用 adjustConcurrency() 和 adjustChunkInterval() 动态调整上传策略。
  - 触发 processUploadQueue() 开始上传。

 processUploadQueue()
- 当活跃上传数小于当前并发数且队列不为空时，循环取出队首分片，调用 uploadChunk() 异步上传。
- 上传完成后减少活跃计数，并递归调用自身，实现持续处理。

 uploadChunk(chunkData, chunkIndex, retryCount)
- 构造 FormData，包含分片文件、索引、文件名。
- 使用 fetchWithRetry() 发送 POST 请求。
- 成功则更新 uploadedChunks；失败则根据重试次数决定重试（放回队列头部）或放弃。
- 更新统计显示。

 adjustConcurrency() 和 adjustChunkInterval()
- 根据 uploadQueue.length 调整并发数和分片间隔，并在变化时重启 MediaRecorder 应用新间隔。

 mergeVideoChunks()
- 检查文件名、队列状态、后端写盘队列（通过 /health）。
- 用户确认后调用 /merge-chunks 接口。
- 处理合并结果，更新状态。

 fetchWithRetry(url, options, retryCount)
- 带超时和指数退避重试的 fetch 封装。

 5.4 用户体验设计
- 状态栏根据不同情况显示不同颜色（错误红、成功绿、录制橙）。
- 按钮动态显示/隐藏（开始录制后隐藏“开始”，显示“停止”；停止后显示“合并”）。
- 文件名实时显示在界面中。
- 横竖屏切换时提示建议横屏录制。

 6. 后端详细说明

 6.1 服务器配置
- 端口：3000（可修改）。
- 核心目录：
  - ROOT_DIR：项目根目录。
  - TEMP_DIR：临时分片存储路径（temp-chunks）。
  - VIDEO_DIR：合并视频存储路径（recorded-videos）。
- 中间件：cors、fileUpload、express.json、express.urlencoded、express.static（前端页面）。

 6.2 异步写盘队列
使用 async.queue 实现写盘任务队列，最大并发 20。
- 任务：将上传的分片文件从临时位置移动到目标路径。
- 超时控制：每个任务 10 秒超时，超时则回调错误。
- 状态监控：队列满（saturated）、队列空（drain）、任务错误（error）均有日志。

 6.3 API 端点

 GET /health
返回服务器健康状态，包括：
- 临时目录文件数、视频目录文件数。
- 写盘队列当前长度和并发数。
- 时间戳。

 POST /upload-video-chunk
接收单分片上传。
- 请求体：multipart/form-data，包含字段 videoChunk（文件）、chunkIndex、filename。
- 流程：
  1. 参数校验。
  2. 生成存储路径 TEMP_DIR/filename.chunk{index}。
  3. 将写盘任务加入队列。
  4. 队列任务回调中返回成功/失败响应。

 POST /upload-video-chunk-batch
批量上传多个分片（预留接口，前端未使用）。
- 接收 filename 和 batchSize，以及对应的 chunkIndex_i 和 videoChunk_i。
- 并行加入写盘队列，所有任务完成后返回汇总结果。

 POST /merge-chunks
合并指定文件的所有分片。
- 请求体：{ filename }。
- 流程：
  1. 等待写盘队列清空（最多 20 秒）。
  2. 读取 TEMP_DIR，匹配 filename.chunk 文件。
  3. 按索引升序排序。
  4. 创建输出文件流，依次读取分片并写入。
  5. 合并完成后删除所有分片。
  6. 返回合并后的文件信息。

 6.4 错误处理
- 每个接口均使用 try-catch 捕获异常，返回 500 错误。
- 写盘队列任务失败会记录错误，但不会阻塞其他任务。
- 合并超时（40 秒）强制终止。
- 全局监听 uncaughtException 和 unhandledRejection，防止进程退出。


二、部署与配置
 7. 配置文件说明

ngrok.yml：Ngrok 隧道配置文件，用于将本地服务暴露到公网。

version: "2"
authtoken: <your-token>（登录 ngrok 官网 https://dashboard.ngrok.com 获取）
tunnels:
  frontend:
    addr: 51783    前端端口（通常不需要，前端通过后端地址访问）
    proto: http
  backend:
    addr: 3000     后端服务端口
    proto: http

实际使用时，前端页面中的 serverUrl（index.html文件第227行） 需替换为 Ngrok 分配的地址（如 https://xxx.ngrok-free.dev）。

package.json & package-lock.json
项目依赖：
- express：Web 框架
- cors：跨域
- express-fileupload：文件上传
- async：异步队列控制

8. 部署指南

 8.1 环境准备
- Node.js（建议 v18+）
- npm（通常随 Node 安装）

8.2 安装与启动
1. 将项目文件夹复制到目标电脑（例如 C:\Users\用户名\Desktop\video9）。
2. 打开终端（命令提示符或 PowerShell），进入项目根目录。
3. 终端安装依赖：
   npm install
   
4. 修改 server.js 中的 ROOT_DIR （第12行）常量，将其指向当前项目目录：
   const ROOT_DIR = 'C:\\Users\\用户名\\Desktop\\video9'; // 改为你的实际路径
   
5. 终端启动服务器：
   node server.js
   
6. 打开浏览器，访问 http://localhost:3000 即可看到录制页面。

 8.3 使用 ngrok 将服务暴露到公网

如果你希望从其他电脑或手机访问此服务（例如进行远程演示），可以使用 ngrok 创建安全的公网隧道。

 步骤 1：注册并下载 ngrok（项目文件夹中已包含ngrok，此步骤可省略）
- 访问 [ngrok 官网](https://ngrok.com/) 注册账号。
- 登录后，在 [dashboard](https://dashboard.ngrok.com) 获取你的 Authtoken。
- 下载对应操作系统的 ngrok 客户端，解压得到可执行文件。

 步骤 2：配置 Authtoken
打开终端，进入 ngrok 所在目录，执行：
ngrok config add-authtoken 你的Authtoken


 步骤 3：启动 ngrok 隧道
确保 Node.js 服务器已在运行（node server.js），然后在另一个终端中执行：
ngrok http 3000

你将看到类似以下输出：

Forwarding  https://a1b2c3d4.ngrok-free.dev -> http://localhost:3000

复制这个 https://a1b2c3d4.ngrok-free.dev 地址。

 步骤 4：在前端页面中设置公网地址
- 在浏览器中打开录制页面 http://localhost:3000（或直接访问 ngrok 地址）。
- 将页面顶部的 “后端服务地址” 输入框的内容替换为刚复制的 ngrok 地址（例如 https://a1b2c3d4.ngrok-free.dev）。
- 按回车或点击页面空白处，使新地址生效。

 步骤 5：在其他设备上访问
现在，任何设备（手机、平板、其他电脑）只要打开浏览器，输入 https://a1b2c3d4.ngrok-free.dev，即可使用你的录制服务。

 注意事项
- 免费版限制：每次启动 ngrok 分配的地址是随机的，重启后会改变；带宽约 1MB/s；隧道最长存活约 2 小时。
- 保持运行：ngrok 终端和 Node.js 终端都必须保持开启，关闭后服务将不可访问。
- 防火墙：确保防火墙允许 ngrok 出站连接（通常无需额外设置）。

 8.4 目录权限
程序启动时会自动创建 temp-chunks 和 recorded-videos 子目录，但请确保项目根目录有读写权限。如有必要，可以手动创建这两个空文件夹。

 9. 扩展与自定义

 9.1 修改录制参数
- 分辨率/帧率：修改 constraints 对象中的 width、height、frameRate。
- 分片间隔范围：调整 BASE_CHUNK_INTERVAL、MAX_CHUNK_INTERVAL 和阈值常量。
- 并发数范围：修改 MIN_CONCURRENT_UPLOADS 和 MAX_CONCURRENT_UPLOADS。

 9.2 添加实时处理
若需在录制过程中对视频帧进行处理（如人脸识别），可采用：
- 前端方案：使用 Canvas 绘制视频帧，分析像素，或上传图片。
- 后端方案：在分片保存后，调用 FFmpeg 提取帧或运行 AI 模型（需安装 FFmpeg 并增加处理队列）。

 9.3 修改存储路径
编辑 server.js 中的 ROOT_DIR、TEMP_DIR、VIDEO_DIR 常量即可。

 9.4 添加用户认证
可在 Express 中增加中间件验证请求头中的 token，与 ngrok 或其他认证服务集成。

 10. 常见问题与故障排除

 10.1 启动服务器时报错“端口已被占用”
- 修改 server.js 中的 PORT 常量为其他值（如 3001），然后重新启动。
- 或者关闭占用端口的程序。

 10.2 录制时提示“摄像头/麦克风权限被拒绝”
- 检查浏览器地址栏左侧的权限图标，确保已允许摄像头和麦克风。
- 如果之前拒绝过，可以在浏览器设置中重置权限。

 10.3 分片上传失败或一直等待
- 检查网络连接，确保前端能访问后端地址（本地为 http://localhost:3000，公网为 ngrok 地址）。
- 查看运行 server.js 的终端是否有错误日志。
- 如果使用 ngrok，确保 ngrok 终端正常运行，且地址正确。

 10.4 合并失败提示“后端还有 xx 个分片未写入磁盘”
- 这是正常的安全等待，说明分片正在写入磁盘。请稍等片刻再试。

 10.5 视频合并后无法播放
- 确保录制时浏览器支持 WebM 格式（Chrome/Edge 均支持）。
- 尝试使用 VLC 播放器打开生成的视频文件。

 10.6 使用 ngrok 时访问公网地址出现 404 或 502
- 确保 ngrok 终端和 Node.js 服务器都在运行。
- 检查 ngrok 分配的地址是否输入正确（注意 http/https）。
- 尝试重启 ngrok 隧道。

 10.7 ngrok 地址每次重启都会变，如何固定？
- 免费版无法固定域名。如果需要固定域名，可以升级到付费计划，或在 ngrok 官网配置保留域名。
