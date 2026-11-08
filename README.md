# SoftGlass

Compose Multiplatform 的柔光玻璃渲染引擎。

**效果绘制**（倒角剖面、折射位移、方向光、背景处理）参考 HyperOS 渲染层的逆向结果复刻；
引擎其余部分 —— 采样录制层、7 级模糊金字塔、连续圆角几何、投影、三端后端与公开 API —— 都是本项目自研，
并在复刻之上做过调整，不完全等同于系统原版。

不依赖任何第三方模糊 / 形状 / 玻璃库，Android、JVM、iOS 三端输出同一套画面。

---

## 一、它由哪几层构成

玻璃的观感由五层叠加而成，各层来源不同：

| 层 | 做了什么 | 来源 |
|---|---|---|
| **几何** | 连续圆角（G2 Continuity）+ SDF 倒角带；剖面固定为 `smooth5_vertical`，再按 `edgePow` 收边 | 自研 |
| **折射** | 按折射率 `refractIOR` 与厚度 `thickness` 沿法线偏移采样，边缘被压缩、越靠边位移越大 | 复刻 |
| **方向光** | 倒角面上的高光带，随光源方向与 `lightAngleRange` 衰减 | 复刻 |
| **背景处理** | 采样源的饱和度、亮度、压暗（`darker`）、内阴影、底边提亮 | 复刻 |
| **模糊** | 7 级金字塔：基准 1024 边长、2×2 降采样、三趟盒式模糊，按半径选级 | 自研 |

标「复刻」的四项参数命名与逆向得到的 uniform 一一对应，所以每一档都能对回系统原值；剖面与采样偏移的具体数值在复刻时做过调整，不是逐位相同。

## 二、平台

| 平台 | 后端 | 最低要求 |
|---|---|---|
| Android | AGSL `RuntimeShader`（API 33+），`RenderEffect`（API 31+），更低版本自动退化 | minSdk 26 |
| JVM / Desktop | Skiko SkSL | — |
| iOS | Skiko SkSL（Kotlin/Native） | — |

能力探测内置，调用方不需要判断 API 等级。

## 三、安装

```kotlin
repositories {
    maven("https://jitpack.io")
}

dependencies {
    implementation("com.github.HaoZai000.softglass:softglass:0.1.0")
}
```

## 四、用法

### 1. 一块玻璃

玻璃读的是「它背后的像素」，所以先要有采样源：

```kotlin
import com.haooz.softglass.backdrop.backdrops.rememberLayerBackdrop
import com.haooz.softglass.backdrop.backdrops.layerBackdrop
import com.haooz.softglass.backdrop.drawBackdrop
import com.haooz.softglass.backdrop.effects.blur
import com.haooz.softglass.shape.ContinuousRoundedRectangle

@Composable
fun GlassCard() {
    val backdrop = rememberLayerBackdrop()   // 采样源

    Box(Modifier.fillMaxSize().layerBackdrop(backdrop)) {
        // 这一层会被录进 backdrop
        Image(wallpaper, contentDescription = null, modifier = Modifier.fillMaxSize())

        Box(
            Modifier
                .size(240.dp, 120.dp)
                .drawBackdrop(
                    backdrop = backdrop,
                    shape = { ContinuousRoundedRectangle(28.dp) },
                    effects = { blur(12.dp.toPx()) },
                )
                .background(Color.White.copy(alpha = 0.18f))
        )
    }
}
```

`drawBackdrop` 会按 `shape` 裁剪内容，并在内容之下画采样结果。

### 2. 投影

默认带一盏柔和投影（24dp / 黑 10% / y+4dp），传 `null` 关闭：

```kotlin
drawBackdrop(backdrop, shape, effects, shadow = null)
```

自定义：

```kotlin
import com.haooz.softglass.backdrop.shadow.Shadow

drawBackdrop(backdrop, shape, effects, shadow = { Shadow(alpha = 0.3f) })
```

投影由库自己画（离屏层 + 模糊轮廓挖空），形状与玻璃同源，三端一致。

### 3. 主题与材质档

一次性注入档位、深浅与降级口径：

```kotlin
import com.haooz.softglass.*

SoftGlassTheme(
    preset = GlassPreset.Default,
    level = GlassMaterialLevel.FROSTED,
    isDark = isSystemInDarkTheme(),
    hasWallpaper = true,
) {
    App()
}
```

| 档 | 效果 |
|---|---|
| `SOFT` | 完整柔光：折射 + 方向光全开 |
| `FROSTED` | 纯磨砂：关折射与方向光，只留模糊 |
| `NONE` | 全关：玻璃退化为实色，采样层不再录制 |

### 4. 系统原值档位

逆向得到的 13 档 HyperOS 玻璃档位，可直接当 preset 用：

```kotlin
val spec = GlassToken.BIONIC_MEDIUM_THIN.spec
SoftGlassTheme(preset = spec, level = GlassMaterialLevel.SOFT, isDark = true) { App() }
```

13 档：`BIONIC_MEDIUM_THIN` / `BIONIC_MEDIUM_NORMAL` / `BIONIC_MEDIUM_REGULAR` / `BIONIC_SEMIBOLD_MOTION` / `COMMON_SMALL_REGULAR` / `COMMON_SMALL_BOLD` / `COMMON_MEDIUM_THIN_LOW` / `COMMON_MEDIUM_REGULAR_DARK` / `COMMON_LARGE_REGULAR` / `COMMON_LARGE_DEMIBOLD` / `TINT_PURPLE` / `TINT_BLACK_REGULAR` / `TINT_BLACK_HEAVY`。

### 5. 形状

```kotlin
ContinuousRoundedRectangle(28.dp)   // 连续圆角矩形
ContinuousCapsule()                 // 连续胶囊
```

两者的圆角都是 G2 连续性（`Continuity.Default`），不是普通圆弧。

### 6. 自定义效果链

```kotlin
import com.haooz.softglass.backdrop.effects.colorControls

drawBackdrop(backdrop, shape) {
    blur(12.dp.toPx())
    colorControls(saturation = 1.5f)
}
```

## 五、参数

`SoftGlassStyle` 是全部可调量的唯一来源，按用途分组：

| 组 | 字段（节选） |
|---|---|
| 几何 | `cornerRadius`、`edgeWidth`、`edgePow`、`thickness`、`reflectOffset` |
| 折射与反射 | `refractionScale`、`refractIOR`、`reflectionStrength`、`reflectionLighten`、`colorPow` |
| 方向光 | `lightDirection`、`lightAngleRange`、`lightIntensity`、`lightOppositeIntensity`、`edgeHighlight`、`edgeLightFalloff` |
| 颜色 | `glassColor`、`glassColorMix`、`saturation`、`brightness`、`darker`、`darkerRange`、`innerBottom`、`innerColorWhite` |
| 模糊 | `smallBlurRadius`、`bigBlurRadius` |

`GlassPreset.Default` 对应 HyperOS 大面板档位的完整参数；`GlassToken` 里每档只覆盖逆向表给出的列，其余字段从默认档继承。

## 六、使用约束

这四条踩过就会出画面问题，改代码前先看：

1. **采样层不能采样自己。** 玻璃节点必须画在它所采样的 `layerBackdrop` 之外。包在同一层里就是自采样，直接崩或画面全黑。
2. **投影画在节点边界之外。** 祖先若有 `graphicsLayer { alpha }` 的进出场动画，必须设 `compositingStrategy = CompositingStrategy.ModulateAlpha`；默认的 `Auto` 会走离屏合成并按 bounds 把整圈投影裁掉。
3. **形状只有一个真相。** 可见轮廓由 G2 Path 裁剪，shader 只按标准圆弧算光学距离。不要试图让 shader 复刻 G2 曲线，也不要把胶囊换回原生 `Outline.Rounded`。
4. **能力不足时自动退化。** 没有 `RuntimeShader` 走 `RenderEffect`，再没有就走实色底 —— 调用方不需要写分支，但要知道低版本设备看不到折射。

## 七、构建与测试

```bash
./gradlew jvmTest                  # 参数 golden / 形状 golden / shader 编译 / 投影离屏渲染
./gradlew publishToMavenLocal      # 本地验证产物
./gradlew build -Psoftglass.ios=false   # 跳过 Kotlin/Native，只编 Android + JVM
```

## 八、来源与许可

**效果绘制**部分（倒角剖面、折射、方向光、背景处理的参数与曲线）参考了对 HyperOS 系统渲染层的逆向分析并做了适配调整；
工程实现 —— 采样录制层、模糊金字塔、连续圆角几何、投影、平台后端与公开 API —— 均为本项目自研。

本项目与小米公司无关，不包含任何小米的私有代码或资源。

Apache License 2.0，见 [LICENSE](LICENSE)。
