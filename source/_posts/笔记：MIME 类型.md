---
title: 笔记：MIME 类型
date: 2025-04-05
categories:
  - 网络服务
  - MIME 类型
tags: 
author: 霸天
layout: post
---
MIME 类型用于表示文档、文件或数据的类型，告诉客户端如何解析响应的内容,常用于 HTTP 协议的 `Content-Type` 和 `Accept` 头，指明发送或接收的数据格式，一般格式为：`type/subtype`。以下是一些常见的 MIME 类型：
1. ==文本类型（Text Types）==
    1. <font color="#00b0f0">text/html</font>：HTML 文档
    2. <font color="#00b0f0">text/plain</font>：纯文本
    3. <font color="#00b0f0">text/css</font>：CSS 样式表
    4. <font color="#00b0f0">text/javascript</font> 或 <font color="#00b0f0">application/javascript</font>：JavaScript 脚本
    5. <font color="#00b0f0">text/csv</font>：CSV 格式的文本文件
2. ==应用程序类型（Application Types）==
    6. <font color="#00b0f0">application/json</font>：JSON 数据格式
    7. <font color="#00b0f0">application/xml</font>：XML 数据格式
    8. <font color="#00b0f0">application/xhtml+xml</font>：XHTML 文档
    9. <font color="#00b0f0">application/pdf</font>：PDF 文件
    10. <font color="#00b0f0">application/zip</font>：ZIP 压缩文件
    11. <font color="#00b0f0">application/octet-stream</font>：任意的二进制数据（常用于文件下载）
    12. <font color="#00b0f0">application/msword</font>：Microsoft Word 文档
    13. <font color="#00b0f0">application/vnd.openxmlformats-officedocument.wordprocessingml.document</font>：Word 文档（.docx）
    14. <font color="#00b0f0">application/vnd.ms-excel</font>：Microsoft Excel 文档
    15. <font color="#00b0f0">application/vnd.openxmlformats-officedocument.spreadsheetml.sheet</font>：Excel 文档（.xlsx）
3. ==图像类型（Image Types）==
    1. <font color="#00b0f0">image/jpeg</font>：JPEG 图像
    2. <font color="#00b0f0">image/png</font>：PNG 图像
    3. <font color="#00b0f0">image/gif</font>：GIF 图像
    4. <font color="#00b0f0">image/svg+xml</font>：SVG 矢量图
    5. <font color="#00b0f0">image/webp</font>：WebP 图像格式
    6. <font color="#00b0f0">image/bmp</font>：BMP 图像
4. ==音频类型（Audio Types）==
    1. <font color="#00b0f0">audio/mpeg</font>：MP3 音频
    2. <font color="#00b0f0">audio/ogg</font>：OGG 音频
    3. <font color="#00b0f0">audio/wav</font>：WAV 音频
    4. <font color="#00b0f0">audio/webm</font>：WebM 音频
5. ==视频类型（Video Types）==
    1. <font color="#00b0f0">video/mp4</font>：MP4 视频
    2. <font color="#00b0f0">video/ogg</font>：OGG 视频
    3. <font color="#00b0f0">video/webm</font>：WebM 视频
    4. <font color="#00b0f0">video/x-msvideo</font>：AVI 视频
6. ==字体类型（Font Types）==
    1. <font color="#00b0f0">font/woff</font>：WOFF 字体
    2. <font color="#00b0f0">font/woff2</font>：WOFF2 字体
    3. <font color="#00b0f0">application/font-woff</font>：另一种 WOFF 的 MIME 类型
    4. <font color="#00b0f0">font/ttf</font>：TrueType 字体（TTF）
    5. <font color="#00b0f0">font/otf</font>：OpenType 字体（OTF）
7. ==其他常见类型==
    1. <font color="#00b0f0">multipart/form-data</font>：用于表单数据（文件上传）
    2. <font color="#00b0f0">application/x-www-form-urlencoded</font>：URL 编码表单数据
    3. <font color="#00b0f0">application/javascript</font>：JavaScript 代码