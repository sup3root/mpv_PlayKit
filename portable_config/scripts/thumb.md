# thumb_engine
基于 [thumbfast](https://github.com/po5/thumbfast) 重构的缩略图引擎。

**本脚本不直接显示缩略图**，需配合支持它的 OSD UI 脚本（例如 [uosc fork](https://github.com/hooke007/mpv_PlayKit/discussions/186)）使用，光标悬停在时间轴上时即可看到缩略图预览。

## 较原版的主要改进
- 双后端引擎
- 支持预缓存
- 性能优化
- 可运行时手动开关

## 安装与配置
将 [`thumb_engine/`](https://github.com/hooke007/mpv_PlayKit/blob/main/portable_config/scripts/thumb_engine/) 文件夹放入 mpv 的 `scripts` 目录。

用户选项参见仓库内 **script-opts/thumb_engine.conf** 中设置。
个别选项仅对单个后端有效，已在注释中标注。

## 开发接入

以下说明如何在你的 OSD UI 脚本中接入 thumb_engine 的缩略图能力。

### 接收缩略图信息

thumb_engine 在文件加载、视频参数变化、启用/禁用状态切换时，会广播 `thumb_engine-info` 消息。你需要注册该消息来维护本地的缩略图状态：

```lua
-- 初始状态声明（由 thumb_engine 自动更新，请勿手动修改这些值）
local thumbnail = {
    width = 0,
    height = 0,
    disabled = true,
    available = false,
}

mp.register_script_message("thumb_engine-info", function(json)
    local data = utils.parse_json(json)
    if type(data) ~= "table" or not data.width or not data.height then
        thumbnail.disabled = true
    else
        thumbnail = data
    end
end)
```

`thumb_engine-info` 回传的 JSON 对象包含以下字段：

| 字段 | 类型 | 说明 |
|---|---|---|
| `width` | number | 缩略图宽度（像素） |
| `height` | number | 缩略图高度（像素） |
| `disabled` | boolean | 当前是否禁用 |
| `available` | boolean | thumb_engine 是否已加载 |
| `socket` | string | mpv 后端的 IPC 管道路径（含 PID 后缀） |
| `tnpath` | string | 缩略图临时文件路径（含 PID 后缀） |
| `overlay_id` | number | 使用的 OSD overlay ID |

### 请求缩略图

当光标悬停在进度条上时，调用 `thumbnail_gen`（或 `thumb`）消息请求缩略图。此调用可以在每帧渲染中执行，引擎内部已有节流机制，会尽可能减少性能问题。

```lua
if not thumbnail.disabled then
    mp.commandv("script-message-to", "thumb_engine", "thumb",
        hovered_seconds, -- number: 光标悬停位置对应的视频时间（秒）
        thumb_x,         -- number: 缩略图左上角 x 坐标（像素，相对于 OSD）
        thumb_y          -- number: 缩略图左上角 y 坐标（像素，相对于 OSD）
    )
end
```

引擎收到请求后，会通过 mpv 的 `overlay-add` 命令在指定坐标处显示 BGRA 原始帧。坐标的计算（居中、边距约束等）由调用方负责。

### 清除缩略图

当光标离开进度条时，发送 `thumbnail_clr`（或 `clear`）消息来移除已显示的缩略图 overlay：

```lua
if thumbnail.available then
    mp.commandv("script-message-to", "thumb_engine", "clear")
end
```

> 注意：检查 `available` 而非 `disabled`。即使缩略图因各种原因被禁用，只要 thumb_engine 已加载（`available = true`），就应该在离开时发送 clear 以确保 overlay 被正确清除。

### 自定义渲染

默认情况下 thumb_engine 通过 `overlay-add` 直接渲染缩略图。如果你需要完全控制渲染方式（例如自行处理帧数据）：

1. 请求时将 `x`、`y` 设为空字符串，第 4 个参数传入你的脚本名：
   ```lua
   mp.commandv("script-message-to", "thumb_engine", "thumb",
       hovered_seconds, "", "", mp.get_script_name()
   )
   ```
2. 注册 `thumb_engine-render` 消息接收回调。当缩略图就绪时，你会收到一个 JSON 对象：
   ```lua
   mp.register_script_message("thumb_engine-render", function(json)
       local data = utils.parse_json(json)
       -- data.width, data.height: 帧尺寸
       -- data.x, data.y: nil（因为请求时传了空字符串）
       -- data.tnpath: 帧文件路径（BGRA 格式）
       -- data.socket, data.overlay_id: IPC 和 overlay 信息
   end)
   ```

### 消息接口

| 消息 | 参数 | 说明 |
|---|---|---|
| `thumb` | `time, x, y [, script]` | 请求指定时间点的缩略图（thumbfast 兼容，但不推荐使用） |
| `clear` | （无） | 清除当前缩略图（thumbfast 兼容，但不推荐使用） |
| `thumbnail_gen` | `time, x, y [, script]` | `thumb` 的别名，防歧义接口 |
| `thumbnail_clr` | （无） | `clear` 的别名，防歧义接口 |
| `thumbnail_hwdec` | `api \| toggle` | 运行时修改解码模式 |
