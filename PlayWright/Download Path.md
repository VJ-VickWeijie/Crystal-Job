## 点击网站下载按钮后，如何指定本地保存路径。

### 一、核心原理

网页触发下载时，Playwright 不会操作 Windows 的“另存为”窗口，而是直接捕获浏览器的下载事件。

标准流程是：

先监听下载事件
   ↓
点击下载按钮
   ↓
获得 IDownload 对象
   ↓
调用 SaveAsAsync()
   ↓
保存到 D 盘指定位置

有头和无头模式都可以完成，写法完全相同。

Playwright 官方提供两种下载路径控制方式：

1. DownloadsPath：设置浏览器的默认下载目录。


2. download.SaveAsAsync()：为具体文件指定最终保存路径和文件名。



对 xUnit 自动化测试来说，我更推荐 SaveAsAsync()，因为可以精确控制每一个文件。Playwright .NET 下载文档


---

### 二、最基本的下载代码

假设网页上有一个叫“下载报表”的按钮：
```
using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class DownloadTests : PageTest
{
    [Fact]
    public async Task DownloadReport_ShouldSaveToDDrive()
    {
        await Page.GotoAsync("https://example.com/report");

        // 必须在点击按钮之前开始监听下载
        Task<IDownload> downloadTask =
            Page.WaitForDownloadAsync();

        await Page.GetByRole(
            AriaRole.Button,
            new() { Name = "下载报表" }
        ).ClickAsync();

        IDownload download = await downloadTask;

        string savePath =
            @"D:\PlaywrightDownloads\报表.xlsx";

        Directory.CreateDirectory(
            Path.GetDirectoryName(savePath)!
        );

        await download.SaveAsAsync(savePath);

        Assert.True(
            File.Exists(savePath),
            $"下载文件不存在：{savePath}"
        );
    }
}
```
这里最重要的是顺序：
```
Task<IDownload> downloadTask = Page.WaitForDownloadAsync();

await Page.GetByRole(
    AriaRole.Button,
    new() { Name = "下载报表" }
).ClickAsync();

IDownload download = await downloadTask;
```
不能先点击，再开始监听。

错误写法：

`await Page.GetByText("下载报表").ClickAsync();`

// 这时下载事件可能已经发生
`IDownload download = await Page.WaitForDownloadAsync();`

如果下载速度很快，Playwright 就可能错过下载事件，测试最后一直等待。


---

### 三、保留网站提供的文件名

一般网站下载时会提供推荐文件名，例如：

2026年08月销售报表.xlsx

Playwright 可以通过以下属性取得：

`download.SuggestedFilename`

完整示例：
```
using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class DownloadTests : PageTest
{
    [Fact]
    public async Task DownloadReport_ShouldKeepOriginalFilename()
    {
        await Page.GotoAsync("https://example.com/report");

        Task<IDownload> downloadTask =
            Page.WaitForDownloadAsync();

        await Page.GetByRole(
            AriaRole.Button,
            new() { Name = "下载报表" }
        ).ClickAsync();

        IDownload download = await downloadTask;

        string downloadDirectory =
            @"D:\PlaywrightDownloads";

        Directory.CreateDirectory(downloadDirectory);

        string savePath = Path.Combine(
            downloadDirectory,
            download.SuggestedFilename
        );

        await download.SaveAsAsync(savePath);

        Assert.True(File.Exists(savePath));

        Console.WriteLine($"文件已保存：{savePath}");
    }
}
```
这里：

`download.SuggestedFilename`

只提供文件名，不包含完整路径。

例如：

报表.xlsx

通过 `Path.Combine()` 组合后变成：

D:\PlaywrightDownloads\报表.xlsx


---

### 四、自定义下载文件名

如果网站每次都下载相同名称，例如：

export.xlsx

可以在保存时自行改名：
```
string savePath =
    @"D:\PlaywrightDownloads\客户销售报表.xlsx";

await download.SaveAsAsync(savePath);
```
也可以在文件名中加入日期：
```
string fileName =
    $"销售报表_{DateTime.Now:yyyyMMdd}.xlsx";

string savePath = Path.Combine(
    @"D:\PlaywrightDownloads",
    fileName
);

await download.SaveAsAsync(savePath);
```
结果类似：

`D:\PlaywrightDownloads\销售报表_20260827.xlsx`

如果一天可能下载多次，可以加入时间：
```
string fileName =
    $"销售报表_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx";
```
结果：

销售报表_20260827_093015.xlsx


---

### 五、推荐的完整封装方法

以后每个测试都重复写监听、建目录和保存会比较麻烦，可以封装成通用方法。

DownloadHelper.cs
```
using Microsoft.Playwright;

namespace PlaywrightTests;

public static class DownloadHelper
{
    public static async Task<string> DownloadAsync(
        IPage page,
        ILocator downloadButton,
        string downloadDirectory,
        string? customFileName = null)
    {
        Directory.CreateDirectory(downloadDirectory);

        Task<IDownload> downloadTask =
            page.WaitForDownloadAsync();

        await downloadButton.ClickAsync();

        IDownload download = await downloadTask;

        string fileName = string.IsNullOrWhiteSpace(customFileName)
            ? download.SuggestedFilename
            : customFileName;

        fileName = SanitizeFileName(fileName);

        string savePath = Path.Combine(
            downloadDirectory,
            fileName
        );

        await download.SaveAsAsync(savePath);

        if (!File.Exists(savePath))
        {
            throw new FileNotFoundException(
                $"下载完成后没有找到文件：{savePath}"
            );
        }

        return savePath;
    }

    private static string SanitizeFileName(string fileName)
    {
        foreach (char invalidChar in
                 Path.GetInvalidFileNameChars())
        {
            fileName = fileName.Replace(
                invalidChar,
                '_'
            );
        }

        return fileName;
    }
}
```
测试中使用
```
using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class DownloadTests : PageTest
{
    [Fact]
    public async Task DownloadReport_ShouldSucceed()
    {
        await Page.GotoAsync("https://example.com/report");

        ILocator downloadButton = Page.GetByRole(
            AriaRole.Button,
            new() { Name = "下载报表" }
        );

        string savedFile = await DownloadHelper.DownloadAsync(
            page: Page,
            downloadButton: downloadButton,
            downloadDirectory: @"D:\PlaywrightDownloads",
            customFileName:
                $"销售报表_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx"
        );

        Assert.True(File.Exists(savedFile));

        FileInfo fileInfo = new(savedFile);

        Assert.True(
            fileInfo.Length > 0,
            "下载文件是空文件。"
        );
    }
}
```
如果不传自定义文件名：
```
string savedFile = await DownloadHelper.DownloadAsync(
    Page,
    downloadButton,
    @"D:\PlaywrightDownloads"
);
```
就会使用网站提供的：

download.SuggestedFilename


---

### 六、检查下载是否失败

文件下载失败时，可以通过：

`string? failure = await download.FailureAsync();`

进行检查。

可以把这段加入辅助方法：
```
string? failure = await download.FailureAsync();

if (failure is not null)
{
    throw new InvalidOperationException(
        $"浏览器下载失败：{failure}"
    );
}
```
放置位置：
```
IDownload download = await downloadTask;

string? failure = await download.FailureAsync();

if (failure is not null)
{
    throw new InvalidOperationException(
        $"浏览器下载失败：{failure}"
    );
}

await download.SaveAsAsync(savePath);
```
不过通常直接调用 `SaveAsAsync()` 时，如果下载失败，本身也会抛出异常。


---

### 七、下载按钮打开新页面怎么办？

有些网站点击“下载”后会先打开一个新标签页，然后新页面才触发下载。

如果下载是由新页面触发，但仍属于同一个 BrowserContext，可以在上下文级别监听：
```
Task<IDownload> downloadTask =
    Context.WaitForDownloadAsync();

await Page.GetByText("下载报表").ClickAsync();

IDownload download = await downloadTask;

await download.SaveAsAsync(
    @"D:\PlaywrightDownloads\报表.xlsx"
);
```
但多数普通下载使用：

`Page.WaitForDownloadAsync()`

就足够了。


---

### 八、如果同名文件已经存在

自动化程序最好不要无意识覆盖旧报表。最简单的处理方式是在文件名中加入时间：

`string fileName = $"报表_{DateTime.Now:yyyyMMdd_HHmmssfff}.xlsx";`

如果希望生成：

报表.xlsx
报表_1.xlsx
报表_2.xlsx

可以使用下面的方法：
```
private static string GetUniqueFilePath(
    string directory,
    string fileName)
{
    string name = Path.GetFileNameWithoutExtension(fileName);
    string extension = Path.GetExtension(fileName);

    string path = Path.Combine(directory, fileName);
    int number = 1;

    while (File.Exists(path))
    {
        path = Path.Combine(
            directory,
            $"{name}_{number}{extension}"
        );

        number++;
    }

    return path;
}
```
然后：
```
string savePath = GetUniqueFilePath(
    @"D:\PlaywrightDownloads",
    download.SuggestedFilename
);

await download.SaveAsAsync(savePath);
```

---

### 九、几个常见误区

1. 不需要按键盘操作“另存为”

不要使用：

`Alt + S`
`Enter`

也不要用 Windows 桌面自动化去操作保存窗口。

Playwright 能直接取得下载内容并调用：

`SaveAsAsync()`

2. 必须先监听，再点击

正确顺序：

`Task<IDownload> task = Page.WaitForDownloadAsync();`
`await button.ClickAsync();`
`IDownload download = await task;`

3. 保存目录必须存在

最好主动创建：

`Directory.CreateDirectory(downloadDirectory);`

即使目录已经存在，这行也不会报错。

4. 不要依赖临时下载路径

Playwright 会先把文件放进临时目录，浏览器上下文关闭后，临时下载文件会被删除。

因此需要永久保存时必须执行：

await download.SaveAsAsync(savePath);

5. 测试运行账户必须有 D 盘写入权限

本机运行通常没问题，但如果以后放到服务器、Docker 或 CI/CD 环境，可能根本没有 D: 盘。届时路径应该改成配置项，而不是永久写死在代码中。

最核心的标准模板就是：
```
Task<IDownload> downloadTask = Page.WaitForDownloadAsync();

await downloadButton.ClickAsync();

IDownload download = await downloadTask;

await download.SaveAsAsync(
    Path.Combine(
        @"D:\PlaywrightDownloads",
        download.SuggestedFilename
    )
);
```
