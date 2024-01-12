# Customized Blog Theme

Based on the [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, this is a personalized version by [Nihil](https://nihil.cc/). Special thanks to the original creator of Chirpy.

[![996.icu](https://img.shields.io/badge/link-996.icu-red.svg)](https://996.icu)

Before using, please refer to the original Chirpy version's [user guide](https://chirpy.cotes.page/posts/getting-started/).

## Customizations

Differences from the original Chirpy theme include:

* In addition to the supported comment systems [Disqus](https://disqus.com/), [Utterances](https://utteranc.es/) and [Giscus](https://giscus.app/zh-CN), support for the [Waline](https://waline.js.org/) comment system has been added, as seen in `_config.yml` under `comments.waline`.
* Use of a [Zhihu-style 404 page](https://404.life/564.html), which allows returning to the homepage or the previous page.
* New sharing options to Line, QQ, Qzone, and Weibo, as indicated in `_data/share.yml`.
* Replacement of [Font Awesome](https://fontawesome.com/) with [iconfont](https://www.iconfont.cn/) for a wider range of icon choices. Note: Since 2022-12-29, the iconfont path configuration has been moved from `_config.yml` to `_data/origin/cors.yml` and `_data/origin/basic.yml` in `iconfont.css`, to support downloading the CSS locally.
* Addition of an external links block to the right sidebar, as seen in `_data/external_links.yml`.
* Ability to freely control which blocks appear in the right sidebar of posts, as indicated in `_config.yml` under `panel`.
* Addition of subdomain pages, as seen in `_data/subdomain.yml`. Delete `_tabs/subdomain.md` if not needed.
* Addition of `<details>` tag style and adjustment of blockquote style.
* Use of table styles modified from [`just the docs`](https://github.com/pmarsceill/just-the-docs).
* Application of code highlighting to inline code segments, e.g., `` `let fuck_rust = 114514;`{:.language-rust} ``.
* Use of [Fira Code](https://github.com/tonsky/FiraCode) as the font for code segments. Ligatures are off by default for inline code but on for block code. Ligatures are disabled for the shell language for certain reasons.
* In dark mode, to make titles and bold content stand out against white text, a glowing effect has been added.
* Configuration of highlighting certain lines in highlighted code segments, refer to [this example](http://nihil.cc/posts/highlight_lines_for_jekyll/#%E4%BE%8B%E5%AD%90).
* Execution of code to display output results (under development, currently supports some languages). Add `{: run="lang" }` in the line following the code segment, e.g.:

    ````markdown
    ```rust
    fn main() {
        println!("hello world");
    }
    ```
    {: run="rust" }
    ````

    Supported languages include:
    | Supported Languages | `run="lang"` Parameter | Backend |
    | :-: | :-: | :-: |
    | C++ | `run="cpp"` | [Coliru](https://coliru.stacked-crooked.com/) |
    | JavaScript | `run="javascript"` | N/A (Local) |
    | Python | `run="python"` | [Online Python](https://www.online-python.com/) |
    | Rust | `run="rust"` | [Rust Playground](https://play.rust-lang.org/) |

* (2022-11-29) Support for multi-level categories with the same name. If updating from an older version, run `bundle update` locally. The original Chirpy uses the `jekyll-archives` plugin to generate categories, which treats all categories as equal, preventing the use of same-named secondary categories in Chirpy. This issue is fixed in this branch, allowing any multi-level categories with the same name. Note: Since `jekyll-archives` is downloaded to the local `.gems` directory and `Gemfile` is pointed to the local path, `bundle update` is required locally to take effect.
* (2022-12-12) Added animated background and mouse click effects. Considering not everyone enjoys special effects, they are disabled by default. To enable, configure `background_animation` and `mouse_click_effect` to `true` in `_config.yml`.

Typically, there will be at least one merge from [`upstream/master`](https://github.com/cotes2020/jekyll-theme-chirpy) every week to track new features.
