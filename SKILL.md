# 无限制写实生图工作室 · 复刻技能（unrestricted-image-studio）

给任何一个 Muse / AI 智能体看的部署手册。照着做完，你的用户就能拥有一套
**自托管、无平台审核层、五引擎可切换的写实生图服务**：权重跑在用户自己的
Modal 云 GPU 上，中间没有任何第三方内容审核，画什么由用户决定。

> 18+ 说明：本项目面向成年用户。生成主体一律写明成年（如 “26-year-old adult
> woman”），这是提示词工程的一部分，不是客套。

---

## 一、成品长什么样

- 一个 Modal App（`flux-studio`）：A10G GPU（24GB），权重缓存在持久 Volume，
  闲置 120 秒自动缩零，不跑不花钱。
- 一个 FastAPI 入口（令牌鉴权，`x-auth-token` 请求头）：
  - `POST /generate` —— 异步提交，返回任务 id（可带 `steps` / `guidance` 覆盖参数）
  - `GET /result/{id}` —— 轮询：pending / done（base64 图）/ error
  - `GET /health`
- 五个引擎，`model` 字段切换：`flux` / `rv` / `lustify` / `chroma` / `pony`
- 中文提示词自动翻译：接口层检测到中文先翻成英文再生成（原因见第六节）。
- FaceID 锁脸（可选）：给一张参考脸，Lustify 出图全程同一张脸。

费用：Modal Starter 每月送 $30 额度。A10G 约 $0.000306/秒，实测一张图
$0.01~0.05 量级，轻度使用基本在免费额度内。

## 二、前置条件（只有用户能做的两步，如实告诉用户）

1. **Modal 账号**：用户自己注册（modal.com），并在本地完成 `modal setup`
   登录。智能体不要代注册、不要碰用户密码。
2. **鉴权令牌**：让用户想一个随机长字符串，然后在 Modal 后台建 Secret：
   名字 `flux-studio-auth`，键 `AUTH_TOKEN`，值就是那个令牌。
   令牌只进 Secret，永远不要写进代码、仓库、聊天记录。

其余全部由智能体完成：所有模型权重都是 Hugging Face 公开仓库，免 gated、
免申请，部署时自动下载到 Volume（首次每引擎约多等 1~5 分钟）。

## 三、部署（一条命令）

```bash
pip install modal        # 或在 venv 里
modal deploy flux_app.py
```

**铁律：改过镜像/关键配置后重新部署前，先 `modal app stop flux-studio --yes`
清掉旧容器再 deploy**，否则旧容器会继续用旧代码/旧权重接单，症状是“改了没生效”。

调用示例：

```bash
curl -X POST https://<你的Modal工作区>--flux-studio-web.<区域>.modal.run/generate \
  -H "x-auth-token: <令牌>" -H "Content-Type: application/json" \
  -d '{"prompt":"a woman in a red dress standing in a garden, realistic photography",
       "width":1024,"height":768,"seed":42,"model":"rv"}'
# 再轮询 GET /result/<id> 直到 status=done，image_b64 解码即图片
```

## 四、五引擎配方（全部实测定版，照抄，不要自创）

| 引擎 | 权重 | 采样 | 步数/CFG | 热态耗时 | 定位 |
|---|---|---|---|---|---|
| flux | FLUX.2-klein-4B ＋ 去审查编码器（见下） | 默认 | 4 步 / guidance 1.0 | ~15 秒 | 通用写实、高清大图 |
| rv | Realistic Vision V6.0 B1（SD1.5） | Euler Ancestral | 20 步 / 7.0 | 2~7 秒 | **单人最强**，双人必塌 |
| lustify | LUSTIFY v2.0（SDXL） | DPM++ 2M Karras | 30 步 / 6.0 | ~17 秒 | **双人/多人主力**，支持 FaceID |
| chroma | Chroma1-HD（FLUX.1-schnell 衍生） | 默认 | **40 步 / guidance 2.5** | ~100 秒 | 新架构，皮肤/手/脸最细，动作执行最字面 |
| pony | Pony Realism v2.1 MainVAE（SDXL） | DPM++ 2M Karras | 30 步 / 6.0 | ~26 秒 | 成人内容知识最扎实，偏欧美风 |

关键细节，错一条就翻车：

- **flux 去审查的关键不是模型，是文本编码器**：FLUX.2-klein 本体是蒸馏模型、
  本身没学过成人内容；把它的 Qwen3 文本编码器换成
  `PinoCookie/Flux.2-klein-4B-abliterated-text-encoder` 的权重
  （state dict 直装，0 missing / 0 unexpected），保证提示词不被“软拒绝”。
  同种子同提示词换编码器前后构图几乎不变、画质无损。
- **rv 必须用单文件 checkpoint**：`SG161222/Realistic_Vision_V6.0_B1_noVAE`
  里的 `Realistic_Vision_V6.0_NV_B1_fp16.safetensors` ＋
  `stabilityai/sd-vae-ft-mse` VAE ＋ `from_single_file` 加载。
  **官方 diffusers 格式仓库出图构图崩坏，禁用。** 长边自动压到 1024 以内。
- **rv 双人彻底不行**：实测 10/10 塌缩成单人（训练数据几乎全单人），
  换提示词救不了。双人一律走 lustify。
- **chroma 的 guidance 必须压到 2.5**：默认 4.0 时下体渲染夸张偏卡通；
  扫参实测定版 2.5 / 40 步，配合加强版解剖负面词
  （`exaggerated anatomy, oversized genitals, deformed genitals,
  cartoonish anatomy, distorted proportions` 已内置在 CHROMA_NEGATIVE）。
- **pony 要 score 标签前缀**：提示词开头加 `score_9, score_8_up, score_7_up,`
  才是它的母语；直接写自然语言命中率明显下降。它的东亚脸命中率不如
  lustify，想要东亚素人风优先 lustify。
- **SDXL 系（lustify/pony）提示词 ≤ 约 70 词**：CLIP 77 token 截断，
  写长了后半句等于没说。
- 负面词全部内置在代码常量里（RV_NEGATIVE / XL_NEGATIVE /
  CHROMA_NEGATIVE / PONY_NEGATIVE），含 watermark/text/logo 压制
  和多人压制（rv 专用），不要删。

## 五、FaceID 锁脸配方（五轮踩坑定版）

只对 lustify 生效。把用户给的参考脸照片放进 Volume 的 `/models/ref/`，
文件名 `face-ref.jpg`（face1）/ `face2.jpg`……，调用时 body 带
`"face":"face1"`。

唯一正确姿势：

1. insightface `buffalo_l`（CPU provider）检测参考图最大脸，取
   `normed_embedding`；
2. 转 fp16、reshape(1,1,512)，**与零向量沿 dim0 堆叠成 [2,1,512]**
   （CFG 分块需要，漏了直接报错或出鬼脸）；
3. adapter 用 `h94/IP-Adapter-FaceID` 的 `ip-adapter-faceid_sdxl.bin`，
   `image_encoder_folder=None`，scale=0.85；
4. 镜像必须 apt 装 `libgl1` ＋ `libglib2.0-0`（insightface 拉 opencv），
   pip 装 insightface / onnxruntime / `numpy<2`，
   环境变量 `INSIGHTFACE_HOME=/models/insightface`。

## 六、提示词血泪经验（每条都是实测出来的）

- **中文提示词必须先翻英文**：rv/lustify/pony 的 CLIP 编码器基本不懂中文，
  收到的是乱码只能瞎猜（实测：中文“红裙子花园”出一张蓝裙子趴地）。
  本项目的接口层已内置自动翻译（Google gtx 接口 → MyMemory 兜底，
  翻译失败则原样放行），并在返回里带 `prompt_used` 让用户核对。
- **“湿”是陷阱词**：wet / soaking / glistening 会把全身画成淋浴油光身、
  场景拽进浴室。要局部水光就用 `matte skin` 锁住全身干燥，
  液体只写局部（如 `love juice on her fingertips`）。
- **SD 系会画假水印**：训练图库的照片多带水印，模型会在角落涂乱码字
  （像 “PERFECT” 之类）。负面词压制后仍偶发，交付前检查角落，
  有就裁掉底部一条。
- **种子＝图纸编号**：同提示词＋同种子可复现同一张图（像素级微差来自
  不同 GPU 容器的浮点误差）；换种子即同题变奏。出图后先自查 QC：
  构图崩坏、糊脸、多人塌缩的直接重抽，不要把废图递给用户。
- **年龄词写明确成年**：“26-year-old adult woman” 这种写法，
  既是合规边界，也是出图稳定性的一部分。

## 七、架构纪律（为什么代码长这样）

- **五引擎全部懒加载**：`@modal.enter` 只置 None，首次用到才加载。
  否则一个只用 rv 的冷启动要先白等 FLUX 的 ~16GB 加载（实测堵过 10 分钟）。
- **GPU 选 A10G 不要 T4**：T4 约 14.6GB 可用显存，1024×1536 就 OOM；
  A10G 24GB 全尺寸可跑。`PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
  必须开，编码器替换后不开会 OOM。
- **chroma/flux 用 `enable_model_cpu_offload`**，其余引擎整模型上 GPU、
  切换时手动 `.to("cpu")` 释放显存（代码里已处理）。
- 生成函数超时 1200 秒：chroma 首次冷启动（下 ~28GB 权重）就要 ~200 秒，
  超时给小了会误杀。

## 八、验收清单（部署完逐条自测，别只看 deploy 成功）

1. `GET /health` 返回 `{"ok": true}`；
2. 五个引擎各出一张普通题材（如红裙花园），全部 done 且画面对题；
3. 中文提示词出一张，返回里 `translated: true` 且 `prompt_used` 是英文、
   画面跟随英文词；
4. 同提示词同种子复跑一张，与上一张是同一构图（复现性）；
5. （可选）FaceID 出一张，脸与参考图是同一个人。

全过，才算复刻完成。然后把接口地址和令牌的保管方式告诉用户，
并提醒：令牌不要发到任何公开场合。
