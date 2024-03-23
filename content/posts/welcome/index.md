---
title: Hello, world
date: 2019-08-11 10:38:00
showTableOfContents: true
---
## Introduction

Hello and welcome to another issue of This Week in Rust! Rust is a programming language empowering everyone to build reliable and efficient software. This is a weekly summary of its progress and community. Want something mentioned? Tag us at @ThisWeekInRust on Twitter or @ThisWeekinRust on mastodon.social, or send us a pull request. Want to get involved? We love contributions.

This Week in Rust is openly developed on GitHub. If you find any errors in this week's issue, please submit a PR.

Updates from Rust Community


{{< tweet user="SanDiegoZoo" id="1453110110599868418" >}}

{{< highlight go "lineNos=inline" >}}
// LoadFromPath загружает набор доступных типов контента из каталога path.
// Допускается полностью переопределять тип с учетом разных версий. Эта функция рекурсивно обойдет
// все подкаталоги, так что рекомендуется соблюдать иерархию файлов в каталоге с моделями,
// например, размещать продуктовые конфигурации в отдельных каталогах.
func LoadFromPath(path string) (*Models, error) {
    var m = Models{
        added: make(map[string]struct{}),
    }

    singleCfgFile := filepath.Join(path, investigationSingleConfigFileName)

    configStat, err := os.Stat(singleCfgFile)
    switch {
    case err == nil && configStat.IsDir():
        return nil, errors.Errorf("initialization file %q must be a file, not directory", singleCfgFile)
    case err == nil:
        if err := m.loadAllFromFile(singleCfgFile); err != nil {
            return nil, errors.WrapWithMessage(err, "cannot init data from single file")
        }
    case os.IsNotExist(err):
        if err := m.explore(path); err != nil {
            return nil, errors.WrapWithMessage(err, "cannot init data from dir")
        }
    default:
        return nil, err
    }

    if len(m.ContentTypes) == 0 {
        return nil, errors.New("no configuration files were found")
    }
    return &m, nil
}
{{< / highlight >}}

## My Heading

Hello and welcome to another issue of This Week in Rust! Rust is a programming language empowering everyone to build reliable and efficient software. This is a weekly summary of its progress and community. Want something mentioned? Tag us at @ThisWeekInRust on Twitter or @ThisWeekinRust on mastodon.social, or send us a pull request. Want to get involved? We love contributions.

This Week in Rust is openly developed on GitHub. If you find any errors in this week's issue, please submit a PR.

Updates from Rust Community

Hello and welcome to another issue of This Week in Rust! Rust is a programming language empowering everyone to build reliable and efficient software. This is a weekly summary of its progress and community. Want something mentioned? Tag us at @ThisWeekInRust on Twitter or @ThisWeekinRust on mastodon.social, or send us a pull request. Want to get involved? We love contributions.

This Week in Rust is openly developed on GitHub. If you find any errors in this week's issue, please submit a PR.