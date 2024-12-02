title: PowerToys Run 插件开发入门
date: 2024-11-26 16:45:30
tags:
  - PowerToys
  - PowerToys Run
  - .Net
  - C#
  - 效率工具
categories: 效率工具
thumbnailImage: cover.jpg
---

{% asset_img "cover.jpg" %}

<!-- toc -->

前段时间突然发现 [PowerToys Run](https://github.com/microsoft/PowerToys/blob/main/doc/thirdPartyRunPlugins.md) 是[支持第三方开发插件](https://github.com/microsoft/PowerToys/blob/main/doc/thirdPartyRunPlugins.md) 的，但对于如何开发有关插件互联网上资料较少，官方也没有提供正式的文档，所以这边将基于 [hlaueriksson](https://github.com/hlaueriksson) 的 [《Creating custom PowerToys Run plugins》](https://conductofcode.io/post/creating-custom-powertoys-run-plugins/) 这篇教学文章进行翻译，加上最近自己开发 PowerToys Run 插件的一些理解，希望有更多同学参与第三方插件开发，丰富 PowerToys Run 的相关生态。

由于开发的资料相对缺乏，除了本篇文章，学习开发的最好方式还是直接阅读官方 PowerToys Run 插件的源码： [https://github.com/microsoft/PowerToys/tree/main/src/modules/launcher/Plugins](https://github.com/microsoft/PowerToys/tree/main/src/modules/launcher/Plugins)，当自己开发的时候可以去找实现了相似特性的插件源码，这样能快速掌握一些接口以及库的细节。

注意 PowerToys Run 的插件目前似乎只支持 .Net 开发（C#），不像其他 Launcher 工具比如 alfred 、 wox 等支持比较多的编程语言特别是脚本语言。不过由于插件实现其实都很简单，即便没学过 C#/.Net 我想照着其他项目源码然后结合 Claude/ChatGPT 来辅助编程应该也能够实现自己想要的插件。

# Demo 插件

为了便于入门，hlaueriksson 在文章中展示了一个 Demo 插件，这个插件用来记录输入的单词数（word）与字符数（characters）:

{% asset_img "demo_plugin_1.png" %}

这个插件触发的关键字(`ActionKeyword`)是 "demo"，同时这个插件还支持简单的配置：

{% asset_img "demo_plugin_2.png" %}

+ Count spaces: `true` | `false` ，配置是否把空格纳入计数的范围

插件虽然简单，但是涉及到的功能点是比较全面的： Query 查询的实现、Config 配置选项的实现、以及查询后出现条目对应的菜单项……所以了解这个 Demo 的实现基本可以知道怎么写一个基本的 PowerToys Run 插件了。这个 Demo 插件项目的源代码在：[
https://github.com/hlaueriksson/ConductOfCode/tree/master/PowerToysRun](
https://github.com/hlaueriksson/ConductOfCode/tree/master/PowerToysRun)


# Project 项目

在开始自己的项目之前，首先我们需要看下官方提供的新插件研发的 Checklist：

+ [New Plugin Checklist](https://github.com/microsoft/PowerToys/blob/main/doc/devdocs/modules/launcher/new-plugin-checklist.md)

这个清单的关键点主要是以下几条：

+ 项目名称遵循： `Community.PowerToys.Run.Plugin.<PluginName>`
+ .Net 配置目标框架（Target Framework）： `net8.0-windows`
+ 创建一个 `Main` 类，对应文件 `Main.cs`
+ 创建 `plugin.json` 插件配置文件

 下面我们围绕 Visual Studio 以及命令行等开发环境介绍下如何创建我们的项目脚手架

 ## Visual Studio

 在 Visual Studio 我们会介绍更多创建 PowerToys Run 插件项目的所有细节，首先我们在 Visual Studio 创建 [Class Library](https://learn.microsoft.com/en-us/dotnet/core/tutorials/library-with-visual-studio) 类库项目。

 然后我们按照下面的例子来编辑我们的 `.csproj` 项目文件：

 ```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Platforms>x64;ARM64</Platforms>
    <PlatformTarget>$(Platform)</PlatformTarget>
    <UseWPF>true</UseWPF>
  </PropertyGroup>

  <PropertyGroup>
    <LangVersion>preview</LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup Condition="'$(Platform)' == 'x64'">
    <Reference Include="..\libs\x64\PowerToys.Common.UI.dll" />
    <Reference Include="..\libs\x64\PowerToys.ManagedCommon.dll" />
    <Reference Include="..\libs\x64\PowerToys.Settings.UI.Lib.dll" />
    <Reference Include="..\libs\x64\Wox.Infrastructure.dll" />
    <Reference Include="..\libs\x64\Wox.Plugin.dll" />
  </ItemGroup>

  <ItemGroup Condition="'$(Platform)' == 'ARM64'">
    <Reference Include="..\libs\ARM64\PowerToys.Common.UI.dll" />
    <Reference Include="..\libs\ARM64\PowerToys.ManagedCommon.dll" />
    <Reference Include="..\libs\ARM64\PowerToys.Settings.UI.Lib.dll" />
    <Reference Include="..\libs\ARM64\Wox.Infrastructure.dll" />
    <Reference Include="..\libs\ARM64\Wox.Plugin.dll" />
  </ItemGroup>

  <ItemGroup>
    <None Include="plugin.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Include="Images\*.png">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>

</Project>
 ```

这里几个重点：

+ Platforms 平台：`x64` 与 `ARM64` 
+ 配置 `UseWPF` 以包含 WPF 的库引用
+ 依赖：PowerToys 与 Wox 的 `.dll` 库

在上面 `.csproj` 文件中引用的 `.dll` 文件是所需依赖项的示例，具体取决于你的插件需要支持的功能。

然而，由于上面这些 dll 文件没有对应 Nuget 包的官方封装，所以可能需要[自己编译](https://github.com/microsoft/PowerToys/tree/main/doc/devdocs#compiling-powertoys)，或者在 PowerToys 项目中直接引用 Dll 文件，像 [Lin Yu-Chieh](https://github.com/lin-ycv) 的 [EverythingPowerToys](https://github.com/lin-ycv/EverythingPowerToys/wiki) 插件一样。

而 [hlaueriksson](https://github.com/hlaueriksson) 直接创建了一个 Nuget 包将上述 PowerToys Run Plugin 依赖打包在了一起：

+ [https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Dependencies](https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Dependencies)

当使用 `Community.PowerToys.Run.Plugin.Dependencies` 时，`.csproj` 文件就变成了这样：

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Platforms>x64;ARM64</Platforms>
    <PlatformTarget>$(Platform)</PlatformTarget>
    <UseWPF>true</UseWPF>
  </PropertyGroup>

  <PropertyGroup>
    <LangVersion>preview</LangVersion>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Community.PowerToys.Run.Plugin.Dependencies" Version="0.84.1" />
  </ItemGroup>

  <ItemGroup>
    <None Include="plugin.json">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
    <None Include="Images\*.png">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </None>
  </ItemGroup>

</Project>
```

## CLI 命令行 `dotnet new`

更方便的方式是使用 [hlaueriksson](https://github.com/hlaueriksson) 搭建的 PowerToys Run 插件 .Net 项目脚手架模板： [https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Templates](https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Templates)

{% asset_img "dotnet_template.png" %}

我们可以使用 `dotnet new install Community.PowerToys.Run.Plugin.Templates` 安装这个模板，后面当我们想要创建新的 PowerToys Run 插件项目时，只需要利用 `dotnet new` 命令即可创建：

{% asset_img "dotnet_new.png" %}

当然各类 IDE（如 Visual Studio、Rider 等）对于 `dotnet new` 都有充分的支持，可以自行选择基于 IDE 功能还是自己手动命令行输入创建。`dotnet new` 运行完后就会生成一个解决方案或项目的脚手架，最主要需要关心的是项目已经为我们创建好的这几部分内容：

+ `Images/*.png` -Icon 图标图片，一般需要对应 Windows 暗色与亮色主题两版图片
+ `Main.cs` -整体插件代码逻辑的入口点
+ `plugin.json`- 插件元数据

# Metadata 元数据

项目的 `plugin.json` 元数据文件应该是像这样：

```json
{
  "ID": "AE953C974C2241878F282EA18A7769E4",
  "ActionKeyword": "demo",
  "IsGlobal": false,
  "Name": "Demo",
  "Author": "hlaueriksson",
  "Version": "1.0.0",
  "Language": "csharp",
  "Website": "https://github.com/hlaueriksson/ConductOfCode",
  "ExecuteFileName": "Community.PowerToys.Run.Plugin.Demo.dll",
  "IcoPathDark": "Images\\demo.dark.png",
  "IcoPathLight": "Images\\demo.light.png",
  "DynamicLoading": false
}
```

具体每个字段规定的类型以及含义在 [New plugin checklist](https://github.com/microsoft/PowerToys/blob/main/doc/devdocs/modules/launcher/new-plugin-checklist.md) 可以查到。

# Main 

`Main.cs` 包含是整体插件代码逻辑的入口，相当于 main 函数。以 Demo 插件的 `Main.cs` 为例：

```csharp
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using ManagedCommon;
using Microsoft.PowerToys.Settings.UI.Library;
using Wox.Plugin;
using Wox.Plugin.Logger;

namespace Community.PowerToys.Run.Plugin.Demo
{
    /// <summary>
    /// Main class of this plugin that implement all used interfaces.
    /// </summary>
    public class Main : IPlugin, IContextMenu, ISettingProvider, IDisposable
    {
        /// <summary>
        /// ID of the plugin.
        /// </summary>
        public static string PluginID => "AE953C974C2241878F282EA18A7769E4";

        /// <summary>
        /// Name of the plugin.
        /// </summary>
        public string Name => "Demo";

        /// <summary>
        /// Description of the plugin.
        /// </summary>
        public string Description => "Count words and characters in text";

        /// <summary>
        /// Additional options for the plugin.
        /// </summary>
        public IEnumerable<PluginAdditionalOption> AdditionalOptions => [
            new()
            {
                Key = nameof(CountSpaces),
                DisplayLabel = "Count spaces",
                DisplayDescription = "Count spaces as characters",
                PluginOptionType = PluginAdditionalOption.AdditionalOptionType.Checkbox,
                Value = CountSpaces,
            }
        ];

        private bool CountSpaces { get; set; }

        private PluginInitContext? Context { get; set; }

        private string? IconPath { get; set; }

        private bool Disposed { get; set; }

        /// <summary>
        /// Return a filtered list, based on the given query.
        /// </summary>
        /// <param name="query">The query to filter the list.</param>
        /// <returns>A filtered list, can be empty when nothing was found.</returns>
        public List<Result> Query(Query query)
        {
            Log.Info("Query: " + query.Search, GetType());

            var words = query.Terms.Count;
            // Average rate for transcription: 32.5 words per minute
            // https://en.wikipedia.org/wiki/Words_per_minute
            var transcription = TimeSpan.FromMinutes(words / 32.5);
            var minutes = $"{(int)transcription.TotalMinutes}:{transcription.Seconds:00}";

            var charactersWithSpaces = query.Search.Length;
            var charactersWithoutSpaces = query.Terms.Sum(x => x.Length);

            return [
                new()
                {
                    QueryTextDisplay = query.Search,
                    IcoPath = IconPath,
                    Title = $"Words: {words}",
                    SubTitle = $"Transcription: {minutes} minutes",
                    ToolTipData = new ToolTipData("Words", $"{words} words\n{minutes} minutes for transcription\nAverage rate for transcription: 32.5 words per minute"),
                    ContextData = (words, transcription),
                },
                new()
                {
                    QueryTextDisplay = query.Search,
                    IcoPath = IconPath,
                    Title = $"Characters: {(CountSpaces ? charactersWithSpaces : charactersWithoutSpaces)}",
                    SubTitle = CountSpaces ? "With spaces" : "Without spaces",
                    ToolTipData = new ToolTipData("Characters", $"{charactersWithSpaces} characters (with spaces)\n{charactersWithoutSpaces} characters (without spaces)"),
                    ContextData = CountSpaces ? charactersWithSpaces : charactersWithoutSpaces,
                },
            ];
        }

        /// <summary>
        /// Initialize the plugin with the given <see cref="PluginInitContext"/>.
        /// </summary>
        /// <param name="context">The <see cref="PluginInitContext"/> for this plugin.</param>
        public void Init(PluginInitContext context)
        {
            Log.Info("Init", GetType());

            Context = context ?? throw new ArgumentNullException(nameof(context));
            Context.API.ThemeChanged += OnThemeChanged;
            UpdateIconPath(Context.API.GetCurrentTheme());
        }

        /// <summary>
        /// Return a list context menu entries for a given <see cref="Result"/> (shown at the right side of the result).
        /// </summary>
        /// <param name="selectedResult">The <see cref="Result"/> for the list with context menu entries.</param>
        /// <returns>A list context menu entries.</returns>
        public List<ContextMenuResult> LoadContextMenus(Result selectedResult)
        {
            Log.Info("LoadContextMenus", GetType());

            if (selectedResult?.ContextData is (int words, TimeSpan transcription))
            {
                return
                [
                    new ContextMenuResult
                    {
                        PluginName = Name,
                        Title = "Copy (Enter)",
                        FontFamily = "Segoe Fluent Icons,Segoe MDL2 Assets",
                        Glyph = "\xE8C8", // Copy
                        AcceleratorKey = Key.Enter,
                        Action = _ => CopyToClipboard(words.ToString()),
                    },
                    new ContextMenuResult
                    {
                        PluginName = Name,
                        Title = "Copy time (Ctrl+Enter)",
                        FontFamily = "Segoe Fluent Icons,Segoe MDL2 Assets",
                        Glyph = "\xE916", // Stopwatch
                        AcceleratorKey = Key.Enter,
                        AcceleratorModifiers = ModifierKeys.Control,
                        Action = _ => CopyToClipboard(transcription.ToString()),
                    },
                ];
            }

            if (selectedResult?.ContextData is int characters)
            {
                return
                [
                    new ContextMenuResult
                    {
                        PluginName = Name,
                        Title = "Copy (Enter)",
                        FontFamily = "Segoe Fluent Icons,Segoe MDL2 Assets",
                        Glyph = "\xE8C8", // Copy
                        AcceleratorKey = Key.Enter,
                        Action = _ => CopyToClipboard(characters.ToString()),
                    },
                ];
            }

            return [];
        }

        /// <summary>
        /// Creates setting panel.
        /// </summary>
        /// <returns>The control.</returns>
        /// <exception cref="NotImplementedException">method is not implemented.</exception>
        public Control CreateSettingPanel() => throw new NotImplementedException();

        /// <summary>
        /// Updates settings.
        /// </summary>
        /// <param name="settings">The plugin settings.</param>
        public void UpdateSettings(PowerLauncherPluginSettings settings)
        {
            Log.Info("UpdateSettings", GetType());

            CountSpaces = settings.AdditionalOptions.SingleOrDefault(x => x.Key == nameof(CountSpaces))?.Value ?? false;
        }

        /// <inheritdoc/>
        public void Dispose()
        {
            Log.Info("Dispose", GetType());

            Dispose(true);
            GC.SuppressFinalize(this);
        }

        /// <summary>
        /// Wrapper method for <see cref="Dispose()"/> that dispose additional objects and events form the plugin itself.
        /// </summary>
        /// <param name="disposing">Indicate that the plugin is disposed.</param>
        protected virtual void Dispose(bool disposing)
        {
            if (Disposed || !disposing)
            {
                return;
            }

            if (Context?.API != null)
            {
                Context.API.ThemeChanged -= OnThemeChanged;
            }

            Disposed = true;
        }

        private void UpdateIconPath(Theme theme) => IconPath = theme == Theme.Light || theme == Theme.HighContrastWhite ? Context?.CurrentPluginMetadata.IcoPathLight : Context?.CurrentPluginMetadata.IcoPathDark;

        private void OnThemeChanged(Theme currentTheme, Theme newTheme) => UpdateIconPath(newTheme);

        private static bool CopyToClipboard(string? value)
        {
            if (value != null)
            {
                Clipboard.SetText(value);
            }

            return true;
        }
    }
}
```

这里面关键的类、接口，在后文会统一介绍。

# Interfaces 接口

插件需要实现的关键接口似乎都来自 `Wox.Plugin` ，虽然说是在开发 PowerToys Run 插件但好像还是遵循 Wox 扩展开发的类与接口的实践（x

需要关心的主要是这些接口：

+ `IPlugin`
+ `IPluginI18n`
+ `IDelayedExecutionPlugin`
+ `IContextMenu`
+ `ISettingProvider`

我们在 `Main.cs` 的 Main 类需要按需实现这些接口。

## IPlugin

最重要的接口就是 `IPlugin`:

```csharp
public interface IPlugin
{
    List<Result> Query(Query query);

    void Init(PluginInitContext context);

    string Name { get; }

    string Description { get; }
}
```

+ `Query` - 查询方法，执行主要代码逻辑的方法，当用户在 PowerToys Run 的搜索框中输入字符时插件的 `Query` 方法就会被调用，作为入参的 query 对象会包含用户输入的文本、唤醒关键字（`ActionKeyWord`）等信息，我们可以在 `Query` 方法里根据我们的需要基于用户的输入进行各种处理。
+ `Init` - 插件的初始化方法，入参有一个 `PluginInitContext`，我们可以获取到插件的对应上下文，并注册一些关心的事件。
+ `Name` - 应该与 `plugin.json` 中的 name 值一致，但这个 `Name` 是可以做 i18n 本地化的。

## IPluginI18n

如果需要做 i18n 国际化就需要实现这个 `IPluginI18n` 接口：

```csharp
public interface IPluginI18n
{
    string GetTranslatedPluginTitle();

    string GetTranslatedPluginDescription();
}
```

> 但目前第三方插件不知道为何似乎没法实现 i18n 国际化，即便按照官方插件的方案来做也无法生成对应语种的 Resource 资源。不知道是不是还没有支持。

## IDelayedExecutionPlugin

`IDelayedExecutionPlugin` 接口提供了另外一个可选的 `Query` 方法：

```csharp
public interface IDelayedExecutionPlugin
{
    List<Result> Query(Query query, bool delayedExecution);
}
```

这个 `Query` 方法可以在一些耗时场景下使用（如网络或者本地 IO）。PowerToys Run 会在每次 Query 方法被调用前添加一个小的延迟，这样可以在等用户完成自己额外的输入后再触发调用 `Query` 方法（有点像是 `debounce` 去抖动方法）

## IContextMenu

`IContextMenu` 接口用于给 Query 查询结果添加一个关联菜单（context menu）：

```csharp
public interface IContextMenu
{
    List<ContextMenuResult> LoadContextMenus(Result selectedResult);
}
```

## ISettingProvider

如果插件功能已经比较复杂了，可以实现 `ISettingProvider` 为插件添加设置选项：

```csharp
public interface ISettingProvider
{
    Control CreateSettingPanel();

    void UpdateSettings(PowerLauncherPluginSettings settings);

    IEnumerable<PluginAdditionalOption> AdditionalOptions { get; }
}
```

+ `UpdateSettings` - 这个方法在用户更新 PowerToys Run 的设置时会被触发调用，因此我们一般实现这个方法来保存我们的自定义设置以及根据用户传来的新设置来更新我们插件的状态
+ `AdditionalOptions` - PowerToys Run 的 GUI 界面显示我们的插件设置时会调用这个属性，因此我们通过这个属性来定义我们的设置如何在 PowerToys 的 GUI 图形界面渲染我们的设置项
+ `CreateSettingPanel` - 大部分场景不需要创建设置面板，所以直接抛出 `NotImplementedException` 即可。

# Classes 类

这里介绍一些比较重要的类对象，也主要是来自 `Wox.Plugin` 的：

+ `PluginInitContext`
+ `Query`
+ `Result`
+ `ContextMenuResult`

## PluginInitContext

之前也提到，`PluginInitContext` 会作为 `Init` 方法的参数传入：

```csharp
public class PluginInitContext
{
    public PluginMetadata CurrentPluginMetadata { get; internal set; }

    public IPublicAPI API { get; set; }
}
```

+ `PluginMetadata` 用于获取插件的元数据，比如 `PluginDirectory` 插件目录路径、`ActionKeyword` 触发关键字等
+ `IPublicAPI` - 主要就是获取 `GetCurrentTheme` 并基于此做些可视化的配置工作（比如图标），除此之外还有 `ShowMsg`、`ShowNotification` 以及 `ChangeQuery` 等方法。

## Query

这里的 Query 不是方法，而是作为参数传入 Query 方法的 Query 类对象。

需要关注的 Query 属性有：

+ `Search` 即用户输入的文本，除外了触发关键词的部分
+ `Terms` 同样是用户输入，但是基于空格分割的文本集合（collections）

## Result

`Query` 方法的返回对象。

创建 `Result` 对象的例子如下面代码所示：

```csharp
new Result
{
    QueryTextDisplay = query.Search, // displayed where the user types queries
    IcoPath = IconPath, // displayed on the left side
    Title = "A title displayed in the top of the result",
    SubTitle = "A subtitle displayed under the main title",
    ToolTipData = new ToolTipData("A tooltip title", "A tooltip text\nthat can have\nmultiple lines"),
    Action = _ =>
    {
        Log.Debug("The actual action of the result when pressing Enter.", GetType());
        /*
        For example:
        - Copy something to the clipboard
        - Open a URL in a browser
        */
    },
    Score = 1, // the higher, the better query match
    ContextData = someObject, // used together with the IContextMenu interface
}
```

## ContextMenuResult

`IContextMenu` 接口定义的 `LoadContextMenus` 方法返回的对象。这些对象会被渲染一系列的小按钮，然后显示在每条查询结果的右侧：

```csharp
new ContextMenuResult
{
    PluginName = Name,
    Title = "A title displayed as a tooltip",
    FontFamily = "Segoe Fluent Icons,Segoe MDL2 Assets",
    Glyph = "\xE8C8", // Copy
    AcceleratorKey = Key.C,
    AcceleratorModifiers = ModifierKeys.Control,
    Action = _ =>
    {
        Log.Debug("The actual action of the context menu result, when clicking the button or pressing the keyboard shortcut.", GetType());
        /*
        For example:
        - Copy something to the clipboard
        - Open a URL in a browser
        */
    },
}
```

这里可以找自己想要使用的 `Glyph` 图标：

+ [Segoe Fluent Icons font](https://learn.microsoft.com/en-us/windows/apps/design/style/segoe-fluent-icons-font)
+ [Segoe MDL2 Assets icons](https://learn.microsoft.com/en-us/windows/apps/design/style/segoe-ui-symbol-font)

# Actions 操作

在 Query 返回的 Result 中可以通过定义 Action 函数来定义查询结果被选择后的操作（比如弹出提示框、打开浏览器等…）

而在 ContextMenuResult 的 Action 函数可以定义关联菜单项被点击后的操作：

```csharp
Action = _ =>
{
    System.Windows.Clipboard.SetText("Some text to copy to the clipboard");
    return true;
}
```

```csharp
Action = _ =>
{
    var url = "https://conductofcode.io/";

    if (!Helper.OpenCommandInShell(DefaultBrowserInfo.Path, DefaultBrowserInfo.ArgumentsPattern, url))
    {
        Log.Error("Open default browser failed.", GetType());
        Context?.API.ShowMsg($"Plugin: {Name}", "Open default browser failed.");
        return false;
    }

    return true;
}
```

# Query 查询小插件代码示例

 [hlaueriksson](https://github.com/hlaueriksson) 提供的 Demo 插件涵盖插件功能比较完整，这里提供一个更简单的示例，就是我自己写自己用的一个简单的[卡路里热量值换算工具 KcalConverter](https://github.com/grzhan/KcalConverterPowerToys)，用来把热量值千焦单位换算成千卡，平常记录自己饮食热量值的时候经常会用到。

 {% asset_img "kcal_converter.gif" %}

 这个插件相比 Demo 插件还要简单，主要就是实现了 Query 方法：

 ```csharp
    public List<Result> Query(Query query)
    {
        ArgumentNullException.ThrowIfNull(query);
        var isGlobalQuery = string.IsNullOrEmpty(query.ActionKeyword);
        if (string.IsNullOrEmpty(query.Search) || isGlobalQuery)
        {
            return [];
        }

        var success = decimal.TryParse(query.Search, out var number);
        if (!success)
        {
            return ErrorHandler.OnError(IconPath, query.RawQuery, "Invalid number format");
        }

        try
        {
            var result = number * 0.239006m;
            var resultStr = result.ToString("F2");
            return [
                new Result
                {
                    Title = $"{number} kJ = {resultStr} kcal",
                    SubTitle = "将结果复制到剪贴板",
                    IcoPath = IconPath,
                    Action = _ =>
                    {
                        Clipboard.SetDataObject(resultStr);
                        return true;
                    },
                },
            ];
        }
        catch (Exception e)
        {
            return ErrorHandler.OnError(IconPath, query.RawQuery, errorMessage: e.Message, exception: e);
        }
    }
 ```

这段代码首先针对传入的 query 对象进行了 Search 与 ActionKeyword 属性的条件判断处理，只接收 Search 以及 ActionKeyword 有值的情况再推进后续逻辑。

然后是基于传入的 `query.Search` 文本尝试转换成 demical 数值类型（转换失败报错），进而将其值乘以 `0.239006` 换算成千卡单位的值，最后将结果封装成 Result 对象，通过 Action 匿名函数实现回车选中结果后将结果字符串复制到剪贴板，至此所有的逻辑结束。

（这里的 `ErrorHandler.OnError` 是自己实现的处理异常的工具方法，就是处理异常信息并输出日志，不是代码逻辑的主要内容）




# Logging 日志

通过来自 `Wox.Plugin.Logger` 命名空间的静态类 `Log`，我们可以输出插件日志。这个日志类的底层是 `NLog` 实现的。

```csharp
Log.Debug("A debug message", GetType());
Log.Info("An information message", GetType());
Log.Warn("A warning message", GetType());
Log.Error("An error message", GetType());
Log.Exception("An exceptional message", new Exception(), GetType());
```

日志会被写入到 `.txt` 后缀的文件中，并基于日期滚动，文件位置在：

+ `%LocalAppData%\Microsoft\PowerToys\PowerToys Run\Logs\<Version>\`

# Dependencies 第三方依赖

如果你需要添加第三方依赖，首先看一下已经被 PowerToys 在使用的这些库：

+ [Directory.Packages.props](https://github.com/microsoft/PowerToys/blob/main/Directory.Packages.props)

这些对应版本的依赖理论上你可以直接在 `.csproj` 里引入而不需要进一步的操作。

一些可能有用的 Packages：

+ `LazyCache`
+ `System.Text.Json`

如果要用到上面没包含的第三方库，那么就需要在 `plugin.json` 开启 `DynamicLoading` 配置：

```json
{
    // ...
    "DynamicLoading": true
}
```

设置为 `true` 将会使 PowerToys Run 动态加载放在插件目录中的所有 `.dll` 文件。

# Tests 测试

我们可以为自己的插件项目编写单元测试用例，官方插件使用 [MSTest](https://github.com/microsoft/testfx) 测试框架以及 `Moq` 用于 Mocking。

+ 项目名称： `Community.PowerToys.Run.Plugin.<PluginName>.UnitTests`
+ 目标框架（Target Framework）： `net8.0-windows`

对应单元测试项目 `.csproj` 文件差不多是这样：

```csharp

<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0-windows</TargetFramework>
    <Platforms>x64;ARM64</Platforms>
    <PlatformTarget>$(Platform)</PlatformTarget>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="MSTest.TestAdapter" Version="3.1.1" />
    <PackageReference Include="MSTest.TestFramework" Version="3.1.1" />
    <PackageReference Include="NLog" Version="5.0.4" />
    <PackageReference Include="System.IO.Abstractions" Version="17.2.3" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\Community.PowerToys.Run.Plugin.Demo\Community.PowerToys.Run.Plugin.Demo.csproj" />
  </ItemGroup>

  <ItemGroup Condition="'$(Platform)' == 'x64'">
    <Reference Include="..\libs\x64\Wox.Plugin.dll" />
    <Reference Include="..\libs\x64\PowerToys.Settings.UI.Lib.dll" />
  </ItemGroup>

  <ItemGroup Condition="'$(Platform)' == 'ARM64'">
    <Reference Include="..\libs\ARM64\Wox.Plugin.dll" />
    <Reference Include="..\libs\ARM64\PowerToys.Settings.UI.Lib.dll" />
  </ItemGroup>

</Project>
```

然后基于 Demo 插件的例子，对应的单元测试代码差不多是这样：

```csharp
using System;
using System.Linq;
using Microsoft.VisualStudio.TestTools.UnitTesting;

namespace Community.PowerToys.Run.Plugin.Demo.UnitTests
{
    [TestClass]
    public class MainTests
    {
        private Main _subject = null!;

        [TestInitialize]
        public void TestInitialize()
        {
            _subject = new Main();
        }

        [TestMethod]
        public void Query_should_calculate_the_number_of_words()
        {
            var results = _subject.Query(new(""));
            Assert.AreEqual("Words: 0", results[0].Title);

            results = _subject.Query(new("Hello World"));
            Assert.AreEqual("Words: 2", results[0].Title);
        }

        [TestMethod]
        public void Query_should_calculate_the_number_of_characters()
        {
            var results = _subject.Query(new(""));
            Assert.AreEqual("Characters: 0", results[1].Title);

            results = _subject.Query(new("Hello World"));
            Assert.AreEqual("Characters: 10", results[1].Title);
        }

        [TestMethod]
        public void LoadContextMenus_should_return_buttons_for_words_result()
        {
            var results = _subject.LoadContextMenus(new() { ContextData = (2, TimeSpan.FromSeconds(3)) });
            Assert.AreEqual(2, results.Count);
            Assert.AreEqual("Copy (Enter)", results[0].Title);
            Assert.AreEqual("Copy time (Ctrl+Enter)", results[1].Title);
        }

        [TestMethod]
        public void LoadContextMenus_should_return_button_for_characters_result()
        {
            var results = _subject.LoadContextMenus(new() { ContextData = 10 });
            Assert.AreEqual(1, results.Count);
            Assert.AreEqual("Copy (Enter)", results[0].Title);
        }

        [TestMethod]
        public void AdditionalOptions_should_return_option_for_CountSpaces()
        {
            var options = _subject.AdditionalOptions;
            Assert.AreEqual(1, options.Count());
            Assert.AreEqual("CountSpaces", options.ElementAt(0).Key);
            Assert.AreEqual(false, options.ElementAt(0).Value);
        }

        [TestMethod]
        public void UpdateSettings_should_set_CountSpaces()
        {
            _subject.UpdateSettings(new() { AdditionalOptions = [new() { Key = "CountSpaces", Value = true }] });

            var results = _subject.Query(new("Hello World"));
            Assert.AreEqual("Characters: 11", results[1].Title);
        }
    }
```

# Distribution 发布

目前 PowerToys Run 的插件管理器并不支持直接下载新插件。社区的插件目前的发布方式主要还是在 Github Releases 页面中下载 zip 压缩包。

不过现在社区目前有个 [8LWXpg](https://github.com/8LWXpg) 老师开发的 [ptr](https://github.com/8LWXpg/ptr) 命令行 PowerToys Run 插件管理器，可以方便安装社区插件，感兴趣的老板可以去了解下。

整体的发布过程可以参考这个非官方的 Checklist：

+ [Community plugin checklist](https://github.com/hlaueriksson/awesome-powertoys-run-plugins/blob/main/checklist.md)

而 `Everything` 插件作者 Lin Yu-Chieh (Victor) 更进一步， 其[发布](https://github.com/lin-ycv/EverythingPowerToys/releases) 内容物更加完整:

+ 可执行文件 (EXE)
+ 压缩文件 (ZIP)
+ WinGet
+ Chocolatey

# Linting 代码检查工具

[hlaueriksson](https://github.com/hlaueriksson) 还专门创建了一个针对 PowerToys Run 社区插件的 Linter（代码检查工具）：

+ [https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Lint](https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Lint)

{% asset_img "ptrun_lint.png" %}

运行 linter 之后这个 linter 会给你的插件报告任何可能的问题，每个问题会带上一个 Code 以及对应的描述。linter 的规则是正是基于之前提到的 [Community plugin checklist](https://github.com/hlaueriksson/awesome-powertoys-run-plugins/blob/main/checklist.md)。

# 总结

对于程序员而言，经常会有各种自动化的小需求，而 PowerToys 又是一个对于使用 Windows 作为操作环境的程序员来说必不可少的软件了，所以自带的 PowerToys Run 能够自己开发一些插件还是能够解决不少自己的痛点，我自己就会进一步写一些私用的扩展（比如 OKX 交易所加密货币的多空比查询）。

希望这篇文章能够帮助到你，进一步解放自己的生产力（x

# 参考资料

+ PowerToys Run Demo 插件 - https://github.com/hlaueriksson/ConductOfCode/tree/master/PowerToysRun
+ PowerToys Run Awesome - https://github.com/hlaueriksson/awesome-powertoys-run-plugins
+ 第三方插件列表 - https://github.com/microsoft/PowerToys/blob/main/doc/thirdPartyRunPlugins.md
+ 社区插件 `dotnet new` 模板 - https://github.com/hlaueriksson/Community.PowerToys.Run.Plugin.Templates
+ PowerToys Run 文档 - https://learn.microsoft.com/en-us/windows/powertoys/run
+ PowerToys 开发文档 - https://github.com/microsoft/PowerToys/tree/main/doc/devdocs/modules/launcher
+ PowerToys Github Repo - https://github.com/microsoft/PowerToys
+ PowerToys 编译文档 - [Compiling PowerToys](https://github.com/microsoft/PowerToys/tree/main/doc/devdocs#compiling-powertoys)
