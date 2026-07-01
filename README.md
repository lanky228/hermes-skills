# Hermes Skills

Personal skill collection for [Hermes Agent](https://github.com/NousResearch/hermes-agent) by Nous Research.

## Skills

| Skill | Description |
|-------|-------------|
| [hermes-shortcut](hermes-shortcut/) | Create a Windows desktop shortcut (.lnk) for Hermes Agent |
| [fix-github-access](fix-github-access/) | 修复中国大陆环境下 GitHub 访问问题（hosts + DoH 方案） |
| [install-hermes](install-hermes/) | 在中国大陆通过国内镜像安装 Hermes Agent |
| [apache-calcite-contribution](apache-calcite-contribution/) | 向 Apache Calcite 贡献代码的完整流程（JIRA/PR 规范、构建验证、Error Prone 清理、中国网络方案） |

## Usage

Copy the desired skill folder to `~/.hermes/skills/` to activate it:

```bash
cp -r hermes-shortcut ~/.hermes/skills/
cp -r fix-github-access ~/.hermes/skills/
cp -r install-hermes ~/.hermes/skills/
cp -r apache-calcite-contribution ~/.hermes/skills/
```

## License

MIT
