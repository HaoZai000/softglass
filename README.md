# SoftGlass

Compose Multiplatform 的柔光玻璃渲染引擎。采样、折射、连续圆角几何、7 级模糊金字塔，三端同一套输出。

从 [Nexio 课程表](https://github.com/HaoZai000/NexioSchedule) 中抽出的独立库，不依赖任何第三方模糊 / 形状 / 玻璃制品。

## 平台

| 平台 | 后端 |
|---|---|
| Android | AGSL RuntimeShader（API 33+），RenderEffect（API 31+），低版本自动退化 |
| JVM / Desktop | Skiko SkSL |
| iOS | Skiko SkSL（经 Kotlin/Native） |

## 引入

```kotlin
repositories {
    maven("https://jitpack.io")
}

dependencies {
    implementation("com.github.HaoZai000.softglass:softglass:0.1.0")
}
```

只编 Android / JVM 时（跳过 Kotlin/Native 工具链）：

```bash
./gradlew build -Psoftglass.ios=false
```

## 快速开始

```kotlin
import com.haooz.softglass.backdrop.backdrops.rememberLayerBackdrop
import com.haooz.softglass.backdrop.backdrops.layerBackdrop
import com.haooz.softglass.backdrop.drawBackdrop
import com.haooz.softglass.backdrop.effects.blur
import com.haooz.softglass.shape.ContinuousRoundedRectangle

@Composable
fun GlassCard() {
    // 采样源：玻璃读的是这一层录制下来的像素
    val backdrop = rememberLayerBackdrop()

    Box(Modifier.fillMaxSize().layerBackdrop(backdrop)) {
        Image(bitmap = wallpaper, contentDescription = null, modifier = Modifier.fillMaxSize())

        Box(
            Modifier
                .size(240.dp, 120.dp)
                .drawBackdrop(
                    backdrop = backdrop,
                    shape = { ContinuousRoundedRectangle(28.dp) },
                    effects = { blur(12.dp.toPx()) },
                )
                .background(Color.White.copy(alpha = 0.2f))
        )
    }
}
```

`drawBackdrop` 默认带一盏 24dp / 黑 10% / y+4dp 的投影，传 `shadow = null` 关掉：

```kotlin
drawBackdrop(backdrop, shape, effects, shadow = null)
```

## 材质档位

三档质量，同一条渲染路径：

```kotlin
SoftGlassTheme(
    level = GlassMaterialLevel.FROSTED,   // SOFT / FROSTED / NONE
    isDark = isSystemInDarkTheme(),
) {
    App()
}
```

`SOFT` 完整柔光（折射 + 方向光），`FROSTED` 纯磨砂（关折射与方向光），`NONE` 全关（玻璃退化为实色，采样层不再录制）。

另有 13 档系统原值 `GlassToken`（HyperOS 玻璃档位）可直接取用：

```kotlin
val spec = GlassToken.BIONIC_MEDIUM_THIN.spec
```

## 设计约束

- **形状只有一个真相**：`G2Continuity()`（连续圆角）。玻璃 shader 只按标准圆弧算光学距离，可见轮廓由 G2 Path 裁剪——圆弧恒比 G2 更方，所以玻璃永远铺满、不露缝。
- **采样层不能采样自己**：玻璃节点必须画在它所采样的 `layerBackdrop` 之外，否则是自采样。
- **投影画在节点边界之外**：祖先若有 `graphicsLayer { alpha }` 的进出场动画，必须用 `CompositingStrategy.ModulateAlpha`，`Auto` 会离屏合成并按 bounds 把投影整圈裁掉。

## 构建与发布

```bash
./gradlew :jvmTest                 # 单测（含投影离屏渲染断言）
./gradlew publishToMavenLocal      # 本地验证产物
./gradlew publish                  # 发布到配置的仓库
```

## 许可证

Apache-2.0，见 [LICENSE](LICENSE)。
