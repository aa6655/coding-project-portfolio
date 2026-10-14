# 05 ExplorerPatcher 开源源码分析

这是第三方开源项目的源码阅读案例。原项目由 ExplorerPatcher Developers / 上游贡献者维护；本页不声明申请人是原作者，也不声明申请人参与过二次开发。

原项目：[valinet/ExplorerPatcher](https://github.com/valinet/ExplorerPatcher)。用途是调整 Windows 桌面工作环境。所引源码固定为标签 `22621.4317.67.1_b93337a`，提交 `b93337a98bdd5dbb4a6c33ea2c33143dbd835341`。

## 版本依据

所提供安装文件的版本元数据为 ExplorerPatcher `22621.4317.67.1`，大小 11,143,168 字节。上游[同版本发布页](https://github.com/valinet/ExplorerPatcher/releases/tag/22621.4317.67.1_b93337a)列出的 `ep_setup.exe` 大小相同。没有比较上游二进制哈希，因此该对应只证明版本与大小相符，不证明安装文件和发布资产逐字节一致。

安装器未运行、未反编译、未上传。本页代码直接取自上游公开源码，节选保持原样。

## 资源处理中的返回语义

未找到打包文件时，函数把该项视为无需处理；打开资源失败则返回 FALSE。后续资源处理使用辅助函数，并在对应分支关闭资源。阅读调用链时需要区分“跳过”与“实际完成写入”。

[上游文件第 590–614 行](https://github.com/valinet/ExplorerPatcher/blob/b93337a98bdd5dbb4a6c33ea2c33143dbd835341/ep_setup/ep_setup.c#L590-L614)：

```c
__declspec(noinline) BOOL InstallResource(BOOL bInstall, HMODULE hInstance, unzFile zipFile, const char* pszFileNameInZip, LPCWSTR pwszDirectory, LPCWSTR pwszFileName)
{
    if (bInstall && zipFile && pszFileNameInZip)
    {
        int resultLocateFile = unzLocateFile(zipFile, pszFileNameInZip, 0);
        if (resultLocateFile != UNZ_OK)
        {
            return resultLocateFile == UNZ_END_OF_LIST_OF_FILE; // Don't touch this file, we don't pack this file in the setup
        }

        if (unzOpenCurrentFile(zipFile) != UNZ_OK)
        {
            return FALSE;
        }
    }

    WCHAR wszPath[MAX_PATH];
    wcscpy_s(wszPath, MAX_PATH, pwszDirectory);
    wcscat_s(wszPath, MAX_PATH, L"\\");
    wcscat_s(wszPath, MAX_PATH, pwszFileName);
    BOOL bRet = InstallResourceHelper(bInstall, hInstance, zipFile, wszPath);
    if (bInstall && zipFile && pszFileNameInZip)
        unzCloseCurrentFile(zipFile);
    return bRet;
}
```
## 语言列表匹配

代码将 UTF-8 语言标识转换为宽字符，遍历以 NUL 分隔的语言列表并作不区分大小写比较。完整判断还依赖调用方提供的长度、缓冲区和列表契约。

[上游文件第 636–649 行](https://github.com/valinet/ExplorerPatcher/blob/b93337a98bdd5dbb4a6c33ea2c33143dbd835341/ep_setup/ep_setup.c#L636-L649)：

```c
BOOL SystemHasLanguageInstalled(const WCHAR* languages, const char* langCode, int cchLangCode)
{
    WCHAR szLangCode[100];
    MultiByteToWideChar(CP_UTF8, 0, langCode, cchLangCode, szLangCode, ARRAYSIZE(szLangCode));
    szLangCode[cchLangCode] = 0;
    for (const WCHAR* wszLang = languages; *wszLang; wszLang += wcslen(wszLang) + 1)
    {
        if (!_wcsicmp(wszLang, szLangCode))
        {
            return TRUE;
        }
    }
    return FALSE;
}
```

## 分析范围

这里能够展示资源生命周期、返回值语义、Windows 字符串处理和调用边界的阅读方法。没有执行安装/卸载、修改注册表、验证桌面兼容性或提交上游改动；源码分析不能替代个人开发贡献证明。

## 原作者与许可

节选归上游 ExplorerPatcher 项目及其相应贡献者所有。项目发布的许可证为 GNU GPL v2，完整文本见 [ExplorerPatcher GPL-2.0](../LICENSES/ExplorerPatcher-GPL-2.0.txt)。完整对应源码可从上述固定提交链接获取。本仓库仅复制两段未修改的代码，并另写中文分析；该许可证说明针对本页引用的第三方代码。
