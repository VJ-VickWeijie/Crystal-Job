账号密码如何避免明文写进代码。

一、先理解核心原理

自动化登录通常包含两类信息：

非敏感配置：登录网址、超时时间、下载目录。

敏感配置：用户名、密码、API Key、Token。


非敏感配置可以放在 appsettings.json；敏感信息不应该出现在：

.cs 源代码

Git 仓库

appsettings.json

Excel、TXT 等普通文件

测试日志和截图名称中


对于你目前的学习环境——Windows、VS Code、.NET 8、xUnit——我建议先使用 .NET User Secrets。

它的工作方式是：

测试代码
   ↓ 通过配置键读取
.NET User Secrets
   ↓
用户电脑项目目录之外的 secrets.json

代码里只写配置名称：

configuration["TestAccount:Username"]

而不写真实用户名和密码。

需要特别说明：User Secrets 主要解决的是“防止密码误提交到 Git”，它保存的内容本身并没有加密。微软也将其定位为开发环境的机密管理工具，而不是生产环境的安全保险箱。微软 User Secrets 文档


---

二、在 xUnit 项目中启用 User Secrets

假设你的测试项目目录为：

D:\PlaywrightTests

首先在 VS Code 终端进入项目目录：

cd D:\PlaywrightTests

确认这个目录中存在测试项目的 .csproj 文件，然后执行：

dotnet user-secrets init

执行后，.csproj 中会自动增加类似内容：

<PropertyGroup>
  <TargetFramework>net8.0</TargetFramework>
  <UserSecretsId>一串自动生成的唯一标识</UserSecretsId>
</PropertyGroup>

例如：

<UserSecretsId>36e87eb4-c1df-498d-bceb-7e74fa5a1721</UserSecretsId>

这个 ID 不是密码，可以提交到 Git。


---

三、安装配置读取组件

在测试项目目录执行：

dotnet add package Microsoft.Extensions.Configuration.UserSecrets

然后恢复依赖：

dotnet restore


---

四、写入用户名和密码

在终端执行：

dotnet user-secrets set "TestAccount:Username" "你的用户名"
dotnet user-secrets set "TestAccount:Password" "你的密码"

例如：

dotnet user-secrets set "TestAccount:Username" "student@example.com"
dotnet user-secrets set "TestAccount:Password" "Abc123456"

这里输入的值不会写进源代码，也不会写进项目目录。

Windows 系统通常会把它们保存在：

%APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json

但一般不需要自己打开这个文件。


---

五、检查和管理 User Secrets

查看当前项目保存了哪些配置：

dotnet user-secrets list

输出类似：

TestAccount:Username = student@example.com
TestAccount:Password = Abc123456

注意：这个命令会显示真实密码，不要在直播、录屏或共享终端时执行。

修改密码：

dotnet user-secrets set "TestAccount:Password" "新的密码"

删除一项：

dotnet user-secrets remove "TestAccount:Password"

清空当前项目的全部机密：

dotnet user-secrets clear


---

六、完整的 Playwright xUnit 示例

下面假设你的页面具有：

用户名输入框：#username

密码输入框：#password

登录按钮：button[type='submit']


你需要根据真实网站修改定位器。

using Microsoft.Extensions.Configuration;
using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class LoginTests : PageTest
{
    private readonly string _username;
    private readonly string _password;

    public LoginTests()
    {
        IConfiguration configuration = new ConfigurationBuilder()
            .AddUserSecrets<LoginTests>()
            .Build();

        _username = configuration["TestAccount:Username"]
            ?? throw new InvalidOperationException(
                "没有找到 TestAccount:Username。请使用 dotnet user-secrets set 设置。");

        _password = configuration["TestAccount:Password"]
            ?? throw new InvalidOperationException(
                "没有找到 TestAccount:Password。请使用 dotnet user-secrets set 设置。");
    }

    [Fact]
    public async Task Login_ShouldSucceed()
    {
        await Page.GotoAsync("https://example.com/login");

        await Page.Locator("#username").FillAsync(_username);
        await Page.Locator("#password").FillAsync(_password);

        await Page.Locator("button[type='submit']").ClickAsync();

        await Expect(
            Page.GetByText("登录成功")
        ).ToBeVisibleAsync();
    }
}

这里真正重要的是：

.AddUserSecrets<LoginTests>()

它会根据当前测试项目的 UserSecretsId，找到项目对应的 User Secrets。

然后通过配置键读取：

configuration["TestAccount:Username"]
configuration["TestAccount:Password"]


---

七、建议把配置读取封装起来

如果以后有很多测试类，每个测试类都重复读取配置会很麻烦。可以建立一个专门的配置类。

TestSettings.cs

using Microsoft.Extensions.Configuration;

namespace PlaywrightTests;

public static class TestSettings
{
    private static readonly IConfiguration Configuration =
        new ConfigurationBuilder()
            .AddUserSecrets(typeof(TestSettings).Assembly)
            .AddEnvironmentVariables()
            .Build();

    public static string Username =>
        GetRequiredValue("TestAccount:Username");

    public static string Password =>
        GetRequiredValue("TestAccount:Password");

    private static string GetRequiredValue(string key)
    {
        string? value = Configuration[key];

        if (string.IsNullOrWhiteSpace(value))
        {
            throw new InvalidOperationException(
                $"缺少配置项：{key}。请通过 User Secrets 或环境变量设置。");
        }

        return value;
    }
}

注意这里采用的是：

.AddUserSecrets(typeof(TestSettings).Assembly)

因为 TestSettings 是静态类，不能作为 AddUserSecrets<T>() 的泛型类型使用实例化逻辑；直接指定其所在程序集更加明确。

同时加入：

.AddEnvironmentVariables()

这样以后放到 CI/CD 环境中运行时，可以用环境变量覆盖 User Secrets，不需要修改测试代码。

LoginTests.cs

using Microsoft.Playwright.Xunit;
using Xunit;

namespace PlaywrightTests;

public class LoginTests : PageTest
{
    [Fact]
    public async Task Login_ShouldSucceed()
    {
        await Page.GotoAsync("https://example.com/login");

        await Page
            .Locator("#username")
            .FillAsync(TestSettings.Username);

        await Page
            .Locator("#password")
            .FillAsync(TestSettings.Password);

        await Page
            .Locator("button[type='submit']")
            .ClickAsync();

        await Expect(
            Page.GetByText("登录成功")
        ).ToBeVisibleAsync();
    }
}

我更推荐这个封装版本，因为以后账号密码的读取方式发生变化，测试代码不需要跟着修改。


---

八、为什么不能只读一个 TXT 文件？

下面这种方式技术上当然可以：

string password = File.ReadAllText(@"D:\password.txt");

但它有明显问题：

文件仍然是明文。

其他能够访问 D 盘的程序可能读取它。

文件可能被误上传或复制。

多个账号不好管理。

密码和用户名缺少明确的配置结构。

换电脑、进入 CI/CD 后不方便。


所以：

普通 TXT 文件

只是让密码“不在代码里”，并没有真正做好机密管理。

User Secrets 至少能够：

把密码移出项目目录。

避免密码被 Git 提交。

按项目隔离配置。

使用统一的 .NET 配置接口。

将来平滑切换到环境变量或 Key Vault。



---

九、不同环境的推荐方案

使用环境	推荐方式

自己电脑学习	User Secrets
本地长期自动化	User Secrets 或 Windows Credential Manager
GitHub Actions、Azure DevOps	平台 Secrets + 环境变量
公司服务器	企业密钥管理系统
Azure 正式环境	Azure Key Vault
普通非敏感设置	appsettings.json


你目前先掌握 User Secrets 就够了。

最后记住三个重点：

1. User Secrets 防止密码进入源代码和 Git，但它并不是加密保险箱。


2. 不要在日志、报错信息或 Console.WriteLine() 中输出密码。


3. appsettings.json 适合普通配置，不适合真实账号密码。



