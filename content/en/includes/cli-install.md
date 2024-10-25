---
draft: true
---

{{< tabpane text=true >}}
  {{% tab header="**Install From**:" disabled=true /%}}
  {{% tab header="Script" %}}
``` bash
curl --silent https://llmariner.ai/get-cli | bash
mv llma <your/PATH>
```
  {{% /tab %}}
  {{% tab header="Homebrew" %}}
```bash
brew install llmariner/tap/llma
```
  {{% /tab %}}
  {{% tab header="Go" %}}
```bash
go install github.com/llmariner/llmariner/cli/cmd@latest
```
  {{% /tab %}}
  {{% tab header="Binary" %}}
Download the binary from [GitHub Release Page](https://github.com/llmariner/llmariner/releases/latest).
  {{% /tab %}}
{{< /tabpane >}}
