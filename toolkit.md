---
layout: page
title: Toolkit 🧰
---

Tools I use for work

## IDE & Text Editor: [VS Code](https://code.visualstudio.com) | [VSCodium](https://vscodium.com)

![vscode](https://code.visualstudio.com/assets/home/home-screenshot-win.png)

### Extensions

- [vscode-dbt](https://marketplace.visualstudio.com/items?itemName=analyst-snowflake.vscode-dbt)
- [dbt Power User](https://marketplace.visualstudio.com/items?itemName=analyst-collective.dbt-power-user)
- [Rainbow CSV](https://marketplace.visualstudio.com/items?itemName=mechatroner.rainbow-csv)
- [GitLens](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens)
- [Better Jinja](https://marketplace.visualstudio.com/items?itemName=samuelcolvin.jinjahtml)
- [GitHub Pull Requests and Issues](https://marketplace.visualstudio.com/items?itemName=GitHub.vscode-pull-request-github)
- [Todo Tree](https://marketplace.visualstudio.com/items?itemName=Gruntfuggly.todo-tree)
- [Copy file name](https://marketplace.visualstudio.com/items?itemName=nemesv.copy-file-name)

## DB Management: [DataGrip](https://www.jetbrains.com/datagrip/)

![datagrip](https://www.jetbrains.com/datagrip/img/screenshots/query-console.png)

## Shortcuts: [Raycast](https://www.raycast.com)

![raycast](https://files.raycast.com/acu4yqgt0s9lmuw9shxfpyhzn1im)

### Extensions
- [dbt Cloud](https://www.raycast.com/zsombor-flds/dbtcloud)
- [Docker](https://www.raycast.com/priithaamer/docker)
- [GitHub](https://www.raycast.com/raycast/github)

### Terminal: [Warp](https://www.warp.dev)

![warp](https://assets-global.website-files.com/60352b1db5736ada4741b380/60b943ef1026ad556075a5fd_RTC-p-1080.png)

```bash
function dbt_run_changed() {
    children=$1
    models=$(git diff --name-only | grep '\.sql$' | awk -F '/' '{ print $NF }' | sed "s/\.sql$/${children}/g" | tr '\n' ' ')
    echo "Running models: ${models}"
    dbt run --models $models
}
```

### Notes: [Obsidian](https://obsidian.md)

![obsidian](https://obsidian.md/images/screenshot-1.0-hero-combo.png)

### Drawing: [Excalidraw](https://excalidraw.com)

![excalidraw](https://raw.githubusercontent.com/bric3/excalidraw-jetbrains-plugin/main/excalidraw-for-jetbrains.png)

