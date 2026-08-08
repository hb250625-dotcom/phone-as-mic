# Phone-as-Mic · 把手机变成电脑麦克风

一个轻量的本地工具：用手机的麦克风实时给电脑「喂」音频，让电脑上的任意软件
（Zoom / 腾讯会议 / 微信 / OBS / 录音软件……）把手机当成麦克风使用。

无需在手机上安装 App——手机端就是一个网页，浏览器即可完成麦克风采集与推送。

---

## 🚀 最简用法（推荐，免安装）

> 已经打包成了**独立 exe**，不用装 Python、不用敲命令，双击就能用。

1. **双击 `启动.bat`**（或直接双击 `phone_mic.exe`）→ 自动打开电脑面板，并弹出二维码。
2. **手机扫码**（或浏览器打开 `https://电脑IP:8080/phone`）→ 点「🎤」授权麦克风。
   - 首次打开会提示「证书风险」，点「高级 → 继续访问」即可（自签名证书，仅本机用）。
3. 想让会议/微信/OBS 用上它：电脑装一次免费的 **VB-Cable**，把麦克风选成 **CABLE Output**。

搞定。其余章节是原理和进阶说明。

---

## 原理

```
手机浏览器
  getUserMedia 采集麦克风
    → AudioWorklet/ScriptProcessor 取 PCM 样本
    → WebSocket 实时推送到电脑
电脑服务器 (Python)
  → 环形缓冲区 (解耦网络抖动)
  → sounddevice 写入输出设备
      ├─ 默认扬声器（仅测试）
      └─ VB-Cable 虚拟声卡 → 其它软件当作“麦克风”输入
```

---

## 一、安装

无需安装。直接用 **`phone_mic.exe`**（已打包所有依赖），双击 `启动.bat` 即可运行。

> 推荐安装 **VB-Cable**（免费虚拟声卡）：https://vb-audio.com/Cable/
> 安装后，电脑会多出一个 `CABLE Output` 录制设备，把音频输出到它，
> 其它会议/录音软件就能把它选为「麦克风」。不装也能用，只是只能外放/测试。

---

## 二、启动与参数

> 默认已启用 **HTTPS** 并自动打开电脑面板，手机端扫码即用，无需额外参数。

双击 `启动.bat`（或直接运行 `phone_mic.exe`）即可。若要自定义，可在命令行加参数：

```bash
# 关闭 HTTPS（仅建议配合 ADB 的 localhost 使用）
phone_mic.exe --no-https

# 指定输出到 VB-Cable 设备（按名字匹配，或传设备索引）
phone_mic.exe --device "CABLE"

# 自定义端口 / 采样率
phone_mic.exe --port 9000 --sample-rate 48000
```

启动后终端会打印：
- 本机控制面板地址（在电脑浏览器打开，可选设备、看状态）
- 手机页面地址（或二维码，用手机扫码）
- 检测到的输出设备

---

## 三、三种连接方式

### 1) ADB（最省事，USB 直连，手机端无证书提示）
1. 手机开启「开发者选项 → USB 调试」。
2. 电脑装好 [ADB](https://developer.android.com/tools/releases/platform-tools) 并连上手机。
3. 用 `--no-https` 启动（localhost 下 HTTP 即可用麦克风），再做端口转发：
   ```bash
   phone_mic.exe --no-https
   adb forward tcp:8080 tcp:8080
   ```
4. 手机浏览器打开 `http://localhost:8080/phone`。
5. 点「🎤」授权麦克风即可。

> `localhost` 属于安全上下文，即使服务器是 **HTTP** 也能用麦克风，手机端无证书提示。

### 2) WiFi / USB 网络共享（默认就是 HTTPS，最常用）
程序默认已启用 HTTPS，手机通过局域网 IP 访问即可（浏览器要求非 localhost 必须有 HTTPS）：

- 手机与电脑连同一 WiFi，或手机开启「USB 网络共享」后 USB 连电脑。
- 手机浏览器打开电脑面板里二维码对应的 `https://电脑IP:8080/phone`（也可直接扫码）。
- 首次访问会提示证书风险，点「高级 → 继续访问」即可（自签名证书，仅本机使用）。

> 不想用 HTTPS 的话，也可以给手机 Chrome 加白名单：
> 打开 `chrome://flags/#unsafely-treat-insecure-origin-as-secure`，
> 把 `http://电脑IP:8080` 加进去并重启浏览器。

### 3) 同一局域网直接 HTTP（仅当你已加 Chrome 白名单时）
见方式 2 末尾的 Chrome flag 方案，否则用 `--no-https` + ADB，或就用默认 HTTPS。

---

## 四、把手机设为电脑的“麦克风”

1. 电脑打开控制面板（启动地址）。
2. 在「音频输出设备」里选择 **VB-Cable / CABLE Output**（检测到会自动预选）。
3. 打开 Zoom / 腾讯会议 / 微信 / OBS → 麦克风选择 **CABLE Output (VB-Audio Virtual Cable)**。
4. 在手机端点「🎤」开始说话，对方/软件就能听到你的手机麦克风声音了。
5. 想自己监听，可在手机页面开启「监听（戴耳机）」开关——**务必戴耳机**否则会啸叫。

---

## 五、常见问题

**Q：手机点了按钮提示“无法访问麦克风”？**
- 多半是安全上下文限制。用 ADB 的 `localhost`（`phone_mic.exe --no-https` + `adb forward`）或默认 HTTPS 启动。
- 检查手机浏览器是否真的授予了麦克风权限。

**Q：电脑控制面板找不到 VB-Cable？**
- 确认已安装并重启电脑；在「声音设置 → 录制」里应能看到 `CABLE Output`。
- 没装就先用默认扬声器测试，能听到声音说明链路通了。

**Q：有延迟/卡顿？**
- 走 USB（ADB 或 USB 共享）延迟最低、最稳。
- WiFi 下尽量靠近路由器、避免 2.4G 拥堵。
- 本工具使用 48kHz PCM 直传，未做压缩，码率约 1.5 Mbps，普通 WiFi 无压力。

**Q：想把音频存成文件？**
- 当前版本主打“虚拟麦克风”实时通路。如需录音，可在电脑用 OBS / Audacity
  选择 CABLE Output 录制，或后续版本再加本地保存功能。

---

## 六、文件说明

| 文件 | 说明 |
|---|---|
| `phone_mic.exe`  | 已打包的独立程序（双击即用，免装 Python，已含全部依赖与手机网页） |
| `启动.bat`      | 一键启动器，直接运行 `phone_mic.exe` |
| `README.md`     | 本说明 |
| `~/.phone-as-mic/cert.pem` | 首次启动自动生成的 HTTPS 证书（用户目录，自动创建） |
