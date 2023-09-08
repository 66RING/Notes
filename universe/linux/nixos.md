---
title: NixOS
author: 66RING
date: 2023-09-01
tags: 
- linux
mathjax: true
---

# NixOS

> 被Linux依赖折磨的朋友应该考虑一下NixOS

[圣经](https://nixos-and-flakes.thiscute.world/zh/)

## 安装

直接看官方的[installation manual summary](https://nixos.org/manual/nixos/stable/#sec-installation-manual-summary)即可

如, uefi用户直接

```bash
parted /dev/vda -- mklabel gpt
parted /dev/vda -- mkpart primary 512MB 100%
parted /dev/vda -- mkpart ESP fat32 1MB 512MB
parted /dev/vda -- set 2 esp on

mkfs.ext4 -L nixos /dev/vda1
mkfs.fat -F 32 -n boot /dev/vda2
mount /dev/disk/by-label/nixos /mnt
mkdir -p /mnt/boot
mount /dev/disk/by-label/boot /mnt/boot
nixos-generate-config --root /mnt
vim /mnt/etc/nixos/configuration.nix

nixos-install --option substituters "https://mirrors.ustc.edu.cn/nix-channels/store https://cache.nixos.org/"
```

## 使用 & 方法论: 善用官网的NixOS Options

通过配置`/etc/nixos/configuration.nix`来管理系统, 包括用户, 程序等。

配置好后通过`nixos-rebuild switch`生成最新系统

具体查看官方的[NixOS Options](https://search.nixos.org/options)可以配置各种程序了

更新方面先update channel再rebuild。channel相当于镜像源, 添加channel并update后, 下次rebuild就会更新最新版本了

```bash
nix-channel --add https://mirrors.ustc.edu.cn/nix-channels/nixpkgs-unstable nixpkgs
nix-channel --add https://mirrors.ustc.edu.cn/nix-channels/nixos-22.11 nixos
nix-channel --update
```

## home manager

> 软件的用户配置也通过nix的方式管理

添加home manager的channel

```bash
# sudo nix-channel --add https://github.com/nix-community/home-manager/archive/release-22.11.tar.gz home-manager
# 用安装master分支版或者其他版详见官方github
sudo nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
sudo nix-channel --update
```

- `imports = [<home-manager/nixos>]`, import home-manager
- `users.users.<name>.isNormalUser = true`, 配置用户
- `home-manager.users.<name> = { pkgs, ... }`, 配置home manager管理app

如, 安装`atool`和`httpie`

```nix
home-manager.users.<name> = { pkgs, ... }: {
   home.packages = [ pkgs.atool pkgs.httpie ];
   home.stateVersion = "22.11";
};
```

配置解耦, 创建一个配置文件, 如`home.nix`:

```nix
{ config, pkgs, ... }:

{
  home.username = "<username>";
  home.homeDirectory = "/home/<username>";
  home.stateVersion = "22.11";
  home.packages = [ pkgs.atool pkgs.httpie ];
}
```

然后配置home-manager, import这个配置文件

```nix
  # home-manager.users.<name> = { pkgs, ... }: {
  #    home.packages = [ pkgs.atool pkgs.httpie ];
  #    home.stateVersion = "22.11";
  # };

  home-manager = {
    useGlobalPkgs = true;
    useUserPackages = true;
    users.<name> = import ./home.nix;
  };
```

再以分模块管理, 安装配置zsh为例:

创立独立文件`apps/zsh.nix`, 并配置文件

```nix
# app/zsh.nix
{
  programs.zsh = {
    enable = true;
    enableCompletion = true;
    enableAutosuggestions = true;
    enableSyntaxHighlighting = true;
  };

  programs.fzf = {
    enable = true;
    enableZshIntegration = true;
  };
}
```

然后import配置

```nix
# home.nix
{ config, pkgs, ... }:

{
  imports = [
    ./apps/zsh.nix
  ];

  home.username = "<username>";
  ...
}
```

修改默认shell

```nix
users.users.<name>.shell = pkgs.zsh;
environment.shells = with pkgs; [ zsh ];
```

另外也可以standalone的方式安装home-manager, 之后更新需要额外用一次`home-manager switch`

## Flake

> 管理channel和配置文件, 更加声明式的方式管理

将所有配置文件都保存到一个目录, 如flake

```
$ tree
.
├── home-manager
│   ├── apps
│   │   └── zsh.nix
│   └── home.nix
└── nixos
    ├── configuration.nix
    └── hardware-configuration.nix
```

创建一个flake配置文件用于管理HM和NixOS module的配置。之后在configuration.nix中关于HM的配置就不需要了

```nix
{
  description = "Your new nix config";

  nixConfig = {
    experimental-features = [ "nix-command" "flakes" ];
    substituters = [
      # replace official cache with a mirror located in China
      "https://mirrors.ustc.edu.cn/nix-channels/store"
      "https://cache.nixos.org/"
    ];

    # nix community's cache server
    extra-substituters = [
      "https://nix-community.cachix.org"
    ];
    extra-trusted-public-keys = [
      "nix-community.cachix.org-1:mB9FSh9qf2dCimDSUo8Zy7bkq5CX+/rkCWyvRCYg3Fs="
    ];
  };

  inputs = {
    # 声明channel和HM的来源
    nixpkgs.url = "github:nixos/nixpkgs/nixos-unstable";
    home-manager = {
      url = github:nix-community/home-manager;
      inputs.nixpkgs.follows = "nixpkgs";
    };
  };

  outputs = { nixpkgs, home-manager,  ... }:
  let
    system = "x86_64-linux";
  in
  {
    nixosConfigurations = {
      <your_host_name> = nixpkgs.lib.nixosSystem {
        inherit system;
        modules = [
          ./nixos/configuration.nix
          home-manager.nixosModules.home-manager
          {
            home-manager = {
              useUserPackages = true;
              useGlobalPkgs = true;
              users.<your_user_name> = ./home-manager/home.nix;
            };
          }
        ];
      };
    };
  };
}
```

应用修改

`sudo nixos-rebuild switch --flake '<path to flake dir>#<your_host_name>`

启动flake配套工具, 之后可以使用`nix repl`, `nix flake`等命令

```nix
# configuration.nix
  nix.settings.experimental-features = [ "nix-command" "flakes" ];
```

## cheat sheet

### 启用docker

配置启动docker并添加用户组

```
virtualisation.docker.enable = true

users.users.<name> = {
  isNormalUser = true;
  extraGroups = [ "wheel" "docker"];
  packages = with pkgs; [
    firefox
    tree
  ];
};
```

### 免密sudo

```
  security.sudo.extraRules= [
    {
      users = [ "<name>" ];
      commands = [
        { command = "ALL" ; # 全部
          options= [ "NOPASSWD" ];
        }
        { command = "<full path to binary>" ;
          options= [ "NOPASSWD" ];
        }
      ];
    }
  ];
```

### nix命令

- `nix-channel`, 软件包来源
- `nix-env`, 以脱离Nix声明管理的方式管理环境软件包
    * `nix-env -qa`列出系统中所有的包
- `nix-shell`, 创建一个临时shell环境
- `nix-build`, 构造nix包
- `nix-collect-garbage`, 垃圾回收, 清除`/nix/store`中未被引用的对象


