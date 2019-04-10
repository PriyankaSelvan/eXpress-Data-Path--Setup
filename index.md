## Setup

The following contains the XDP development environment setup information for an Ubuntu machine.

#### Native XDP support
Todo Punith

#### Prerequisites
First, certain packages need to be installed.
```markdown
sudo apt-get install -y make gcc libssl-dev bc libelf-dev libcap-dev clang gcc-multilib llvm libncurses5-dev git pkg-config libmnl-dev bison flex graphviz git exuberant-ctags
```

#### Compiling an XDP enabled kernel
Development of BPF and XDP features happens in the `net-next` tree. Obtain the kernel source.
```markdown
git clone git://git.kernel.org/pub/scm/linux/kernel/git/davem/net-next.git
```

To create a configuration file for compilation, copy the existing config file.
```markdown
cd net-next
cp /boot/config-`uname -r`* .config
```

Make sure that the following config options have the right values set.
```markdown
CONFIG_CGROUP_BPF=y
CONFIG_BPF=y
CONFIG_BPF_SYSCALL=y
CONFIG_NET_SCH_INGRESS=m
CONFIG_NET_CLS_BPF=m
CONFIG_NET_CLS_ACT=y
CONFIG_BPF_JIT=y
CONFIG_LWTUNNEL_BPF=y
CONFIG_HAVE_EBPF_JIT=y
CONFIG_BPF_EVENTS=y
CONFIG_TEST_BPF=m
```
## Welcome to GitHub Pages

You can use the [editor on GitHub](https://github.com/PriyankaSelvan/eXpress-Data-Path-XDP---Setup-and-Usage/edit/master/index.md) to maintain and preview the content for your website in Markdown files.

Whenever you commit to this repository, GitHub Pages will run [Jekyll](https://jekyllrb.com/) to rebuild the pages in your site, from the content in your Markdown files.

### Markdown

Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/PriyankaSelvan/eXpress-Data-Path-XDP---Setup-and-Usage/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
