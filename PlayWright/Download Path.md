思考了 12s

好的，现在进入问题 2：点击网站下载按钮后，如何指定本地保存路径。

一、核心原理

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

二、最基本的下载代码

假设网页上有一个叫“下载报表”的按钮：

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

这里最重要的是顺序：

Task<IDownload> downloadTask = Page.WaitForDownloadAsync();

await Page.GetByRole(
    AriaRole.Button,
    new() { Name = "下载报表" }
).ClickAsync();

IDownload download = await downloadTask;

不能先点击，再开始监听。

错误写法：

await Page.GetByText("下载报表").ClickAsync();

// 这时下载事件可能已经发生
IDownload download = await Page.WaitForDownloadAsync();

如果下载速度很快，Playwright 就可能错过下载事件，测试最后一直等待。


---

三、保留网站提供的文件名

一般网站下载时会提供推荐文件名，例如：

2026年08月销售报表.xlsx

Playwright 可以通过以下属性取得：

download.SuggestedFilename

完整示例：

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

这里：

download.SuggestedFilename

只提供文件名，不包含完整路径。

例如：

报表.xlsx

通过 Path.Combine() 组合后变成：

D:\PlaywrightDownloads\报表.xlsx


---

四、自定义下载文件名

如果网站每次都下载相同名称，例如：

export.xlsx

可以在保存时自行改名：

string savePath =
    @"D:\PlaywrightDownloads\客户销售报表.xlsx";

await download.SaveAsAsync(savePath);

也可以在文件名中加入日期：

string fileName =
    $"销售报表_{DateTime.Now:yyyyMMdd}.xlsx";

string savePath = Path.Combine(
    @"D:\PlaywrightDownloads",
    fileName
);

await download.SaveAsAsync(savePath);

结果类似：

D:\PlaywrightDownloads\销售报表_20260827.xlsx

如果一天可能下载多次，可以加入时间：

string fileName =
    $"销售报表_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx";

结果：

销售报表_20260827_093015.xlsx


---

五、推荐的完整封装方法

以后每个测试都重复写监听、建目录和保存会比较麻烦，可以封装成通用方法。

DownloadHelper.cs

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

测试中使用

using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class DownloadTests : PageTest
{
    [Fact]

