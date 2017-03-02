# Dactyl

Documentation tools for enterprise-quality documentation from Markdown source. Dactyl has advanced features to enable [single-sourcing](https://en.wikipedia.org/wiki/Single_source_publishing) and an extensible syntax for building well-organized, visually attractive docs. It generates output in HTML (natively), and can make PDFs if you have [Prince][] installed.

[Prince]: http://www.princexml.com/

## Installation

Dactyl requires [Python 3](https://python.org/). Install with [pip](https://pip.pypa.io/en/stable/):

```
sudo pip3 install dactyl
```

Or a local install in a virtualenv:

```sh
# Create an activate a virtualenv so the package and dependencies are localized
virtualenv -p `which python3` venv_dactyl
source venv_dactyl/bin/activate

# Check out this repo
git clone https://github.com/ripple/dactyl

# Install
pip3 install dactyl/

# Where 'dactyl/' is the top level directory of the repo, containing setup.py.
# And note the trailing '/' which tells pip to use a local directory to install it.
```

## Usage

Simple ("Ad-Hoc") usage:

```sh
$ dactyl_build --pages input1.md input2.md
```

By default, the resulting HTML pages are written to a folder called `out/` in the current working directory. You can specify a different output path in the config file or by using the `-o` parameter.

### Building PDF

Dactyl generates PDFs by making temporary HTML files and running [Prince][]. Use the `--pdf` command to generate a PDF. Dactyl tries to come up with a sensible output filename by default, or you can provide one (which must end in `.pdf`):

```sh
$ dactyl_build --pages input1.md input2.md --pdf MyGuide.pdf
```

### Advanced Usage

Dactyl is intended to be used with a config file containing a list of pages to parse. Pages are grouped into "targets" that represent a group of documents to be built together; a page can belong to multiple targets, and can even contain conditional syntax so that it builds slightly different depending on the target in question. Targets and pages can also use different templates from each other, and pages can inherit semi-arbitrary key/value pairs from the targets.

For more information on configuration, see the `default-config.yml` and the [examples](examples/) folder.

The input pages in the config file should be specified relative to the `content_path`, which is `content/` by default. You can also specify a URL to pull in a markdown file from a remote source, but if you do, Dactyl won't run any pre-processing on it.

For a full list of Dactyl options, use the `-h` parameter.

#### Specifying a Config File

By default, Dactyl looks for a config file named `dactyl-config.yml` in the current working directory. You can specify an alternate config file with the `-c` or `--config` parameter:

```sh
$ dactyl_build -c path/to/alt-config.yml
```

For more information on configuration, see the `default-config.yml` and the [examples](examples/) folder.

#### Specifying a Target

If your config file contains more than one **target**, Dactyl builds the first one by default. You can specify a different target by passing its `name` value with the `-t` parameter:

```sh
$ dactyl_build -t non-default-target
```

#### Copying Static Files

If your content or templates require certain static files (e.g. JavaScript, CSS, and images) to display properly, you can add the `-s` parameter to copy the files into the output directory. By default, Dactyl assumes that templates have static files in the `assets/` folder and documents have static files in the `content/static/` folder. Both of these paths are configurable. You can combine this flag with other flags, e.g.:

```sh
$ dactyl_build -st non-default-target
# Builds non-default target and copies static files to the output dir
```

#### Listing Available Targets

If you have a lot of targets, it can be hard to remember what the short names for each are. If you provide the `-l` flag, Dactyl will list available targets and then quit without doing anything:

```sh
$ dactyl_build -l
tests		Dactyl Test Suite
rc-install		Ripple Connect v2.6.3 Installation Guide
rc-release-notes		
kc-rt-faq		Ripple Trade Migration FAQ
```

#### Githubify Mode

This mode runs the preprocessor only, so you can generate Markdown files that are more likely to display properly in conventional Markdown parsers (like the one built into GitHub). Use the `-g` flag followed by a doc page (relative to the configured content dir):

```sh
$ dactyl_build -g page_with_dactyl_syntax.md
```

### Link Checking

The link checker is a separate script. It assumes that you've already built some documentation to an output path. Use it as follows:

```sh
$ dactyl_link_checker
```

This checks all the files in the output directory for links and confirms that any HTTP(S) links, including relative links to other files, are valid. For anchor links, it checks that an element with the correct ID exists in the target file. It also checks that the `src` of all image tags exists.

If there are links that are always reported as broken but you don't want to remove (for example, URLs that block Python's user-agent) you can add them to the `known_broken_links` array in the config.

In quiet mode (`-q`), the link checker still reports in every 30 seconds just so that it doesn't get treated as stalled and killed by continuous integration software (e.g. Jenkins).

To reduce the number of meaningless failure reports (because a particular website happened to be down momentarily while you ran the link checker), if there are any broken remote links, the link checker waits 2 minutes after finishing and then retries those links in case they came back up. (If they did, they're not considered broken for the link checker's final report.)

You can also run the link checker in offline mode (`-o`) to skip any remote links and just check that the files and anchors referenced exist in the output directory.

If you have a page that uses JavaScript or something to generate anchors dynamically, the link checker can't find those anchors (since it doesn't run any JS). You can add such pages to the `ignore_anchors_in` array in your config to skip checking for links that go to anchors in such pages.


### Style Checking

The style checker is experimental. It reads lists of discouraged words and phrases from the `word_substitutions_file` and `phrase_substitutions_file` paths (respectively) in the config. For each such word or phrase that appears in the output HTML (excluding `code`, `pre`, and `tt` elements), it counts and prints a violation, suggesting a replacement based on the word/phrase file.

The style checker re-generates HTML in-memory (never writing it out). It uses the first target in the config file unless you specify another target with `-t`.

Example usage:

```sh
$ dactyl_style_checker -t rippledevportal
Style Checker - checking all pages in target rippledevportal
Found 6 issues:
Page: Gateway Guide
   Discouraged phrase: in order to (1 instances); suggest 'to' instead.
   Discouraged phrase: and/or (1 instances); suggest '__ or __ or both' instead.
   Discouraged word: feasible (1 instances); suggest 'can be done, workable' instead.
   Discouraged phrase: in an effort to (1 instances); suggest 'to' instead.
   Discouraged phrase: comply with (1 instances); suggest 'follow' instead.
Page: Amendments
   Discouraged phrase: limited number (1 instances); suggest 'limits' instead.
```

You can add an exemption to a specific style rule with an HTML comment. The exemption applies to the whole output (HTML) file in which it appears.

```html
Maybe the word "will" is a discouraged word, but you really want to use it here without flagging it as a violation? Adding a comment like this <!-- STYLE_OVERRIDE: will --> makes it so.
```

## Configuration

Many parts of Dactyl are configurable. An advanced setup would probably have the following folders in your directory structure:

```
./                      # Top-level dir; this is where you run dactyl_*
./dactyl-config.yml     # Default config file name
./content               # Dir containing your .md source files
---------/*/*.md        # You can sort .md files into subdirs if you like
---------/static/*      # Static images referencd in your .md files
./templates/template-*.html # Custom HTML Templates
./assets                # Directory for static files referenced by templates
./out                   # Directory where output gets generated. Can be deleted
```

(All of these paths can be configured.)

### Targets

A target represents a group of pages, which can be built together or concatenated into a single PDF. You should have at least one target defined in the `targets` array of your Dactyl config file. A target definition should consist of a short `name` (used to specify the target in the commandline and elsewhere in the config file) and a human-readable `display_name` (used mostly by templates but also when listing targets on the commandline).

A simple target definition:

```
targets:
    -   name: kc-rt-faq
        display_name: Ripple Trade Migration FAQ
```

In addition to `name` and `display_name`, a target definition can contain the following fields:

| Field        | Type       | Description                                      |
|:-------------|:-----------|:-------------------------------------------------|
| `filters`    | Array      | Names of filters to apply to all pages in this target. |
| `image_subs` | Dictionary | Mapping of image paths to replacement paths that should be used when building this target. (Use this, for example, to provide absolute paths to images uploaded to a CDN or CMS.) |
| ...          | (Various)  | Arbitrary key-values to be inherited by all pages in this target. (You can use this for pre-processing or in templates.) The following field names cannot be used: `name`, `display_name`, `image_subs`, `filters`, and `pages`. |

### Pages

Each page represents one HTML file in your output. A page can belong to one or more targets. When building a target, all the pages belonging to that target are built in the order they appear in the `pages` array of your Dactyl config file.

Example of a pages definition with two files:

```
pages:
    -   name: RippleAPI
        category: References
        html: reference-rippleapi.html
        md: https://raw.githubusercontent.com/ripple/ripple-lib/0.17.2/docs/index.md
        ripple.com: https://ripple.com/build/rippleapi/
        filters:
            - remove_doctoc
            - add_version
        targets:
            - local
            - ripple.com

    -   name: rippled
        category: References
        html: reference-rippled.html
        md: reference-rippled.md
        ripple.com: https://ripple.com/build/rippled-apis/
        targets:
            - local
            - ripple.com
```

Each individual page definition can have the following fields:

| Field                    | Type      | Description                           |
|:-------------------------|:----------|:--------------------------------------|
| `html`                   | String    | The filename where this file should be written in the output directory. |
| `targets`                | Array     | The short names of the targets that should include this page. |
| `name`                   | String    | _(Optional)_ Human-readable display name for this page. If omitted but `md` is provided, Dactyl tries to guess the right file name by looking at the first two lines of the `md` source file. |
| `md`                     | String    | _(Optional)_ The markdown filename to parse to generate this page, relative to the **content_path** in your config. If this is not provided, the source file is assumed to be empty. (You might do that if you use a nonstandard `template` for this page.) |
| `category` | String | _(Optional)_ The name of a category to group this page into. This is used by Dactyl's built-in templates to organize the table of contents. |
| `template`               | String    | _(Optional)_ The filename of a custom [Jinja][] HTML template to use when building this page for HTML, relative to the **template_path** in your config. |
| `pdf_template`           | String    | _(Optional)_ The filename of a custom [Jinja][] HTML template to use when building this page for PDF, relative to the **template_path** in your config. |
| (Short names of targets) | String    | _(Optional)_ If provided, use these values to replace links that would go to this file when building the specified targets. Use this if the page can't be accessed via its normal .html filename in some situations. |
| ...                      | (Various) | Additional arbitrary key-value pairs as desired. These values can be used by templates or pre-processing. |

[Jinja]: http://jinja.pocoo.org/

## Editing

Dactyl supports extended Markdown syntax with the [Python-Markdown Extra](https://pythonhosted.org/Markdown/extensions/extra.html) module. This correctly parses most GitHub-Flavored Markdown syntax (such as tables and fenced code blocks) as well as a few other features.

### Pre-processing

Dactyl pre-processes Markdown files by treating them as [Jinja][] Templates, so you can use [Jinja's templating syntax](http://jinja.pocoo.org/docs/dev/templates/) to do advanced stuff like include other files or pull in variables from the config or commandline. Dactyl passes the following fields to Markdown files when it pre-processes them:

| Field             | Value                                                    |
|:------------------|:---------------------------------------------------------|
| `target`          | The [target](#targets) definition of the current target. |
| `pages`           | The [array of page definitions](#pages) in the current target. |
| `currentpage`     | The definition of the page currently being rendered.     |

### Filters

Furthermore, Dactyl supports additional custom post-processing through the use of filters. Filters can operate on the markdown (after it's been pre-processed), on the raw HTML (after it's been parsed), or on a BeautifulSoup object representing the output HTML. Dactyl comes with several filters, which you can enable in your config file. Support for user-defined filters is planned but not yet implemented.

See the [examples](examples/) for examples of how to do many of these things.

## Templates

Dactyl provides the following information to templates, which you can access with Jinja's templating syntax (e.g. `{{ target.display_name }}`):

| Field             | Value                                                    |
|:------------------|:---------------------------------------------------------|
| `target`          | The [target](#targets) definition of the current target. |
| `pages`           | The [array of page definitions](#pages) in the current target. Use this to generate navigation across pages. (The default templates don't do this, but you should.) |
| `currentpage`     | The definition of the page currently being rendered.     |
| `categories`      | An array of categories that are used by at least one page in this target. |
| `content`         | The parsed HTML content of the page currently being rendered. |
| `current_time`    | The current date as of rendering, in the format "Monthname Day, Year" |
| `sidebar_content` | A table of contents generated from the current page's headers. Wrap this in a `<ul>` element. |
